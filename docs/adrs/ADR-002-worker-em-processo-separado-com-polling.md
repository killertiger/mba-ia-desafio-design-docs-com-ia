# ADR-002: Worker em Processo Separado com Polling para Processamento da Outbox de Webhooks

**Status:** Aceito
**Data:** 2026-07-14
**ADRs Relacionados:** ADR-001 (Padrão Outbox no MySQL), ADR-003 (Retry com Backoff e DLQ)

---

## 1. Contexto e Problema

O sistema OMS precisa notificar clientes B2B sobre mudanças de status de pedidos por meio de webhooks outbound. A necessidade surgiu de um pedido formal de três clientes — Atlas Comercial, MaxDistribuição e Nova Cargo — que realizavam polling ativo sobre o endpoint de listagem de pedidos, gerando carga desnecessária e latência na integração ([09:00] Marcos).

O modelo de padrão Outbox no MySQL foi adotado como estratégia de confiabilidade: dentro da mesma transação que altera o status do pedido, um evento é inserido na tabela `webhook_outbox`. Isso elimina a possibilidade de inconsistência entre a mudança de estado e o registro do evento a notificar ([09:06] Diego). Uma vez adotado o Outbox, surge o problema central: **como o processo responsável por ler a tabela e disparar as chamadas HTTP deve ser estruturado e acionado?**

As alternativas consideradas se concentram no mecanismo de acionamento do worker e no modelo de isolamento de processo. O requisito de negócio é latência máxima de 10 segundos entre a mudança de status e a notificação ao cliente ([09:02] Marcos).

## 2. Fatores de Decisão

- Isolamento de falhas: reinicializações da API não devem interromper o processamento de eventos pendentes ([09:11] Diego).
- Simplicidade operacional para um time pequeno, sem necessidade de nova infraestrutura ([09:07] Diego).
- Atender à latência máxima de 10 segundos exigida pelos clientes B2B ([09:02] Marcos, [09:10] Larissa).
- Manutenção do padrão arquitetural existente no projeto (entry-point, configuração de banco, shutdown gracioso).
- Garantia de ordering de eventos por pedido enquanto a operação for single-worker ([09:12]–[09:13] Diego).

## 3. Opções Consideradas

- **Opção A — Worker em processo separado com polling periódico** (escolhida): processo Node.js independente (`src/worker.ts`) com loop de polling a cada 2 segundos sobre a tabela `webhook_outbox`.
- **Opção B — Worker embutido no processo da API**: thread ou `setInterval` interno ao mesmo processo da API, sem separação de ciclo de vida.
- **Opção C — Acionamento reativo via broker de mensagens (Redis Streams/Pub-Sub)**: worker acionado por eventos publicados em um broker externo ao banco.

## 4. Decisão

Escolhida a **Opção A — Worker em processo separado com polling periódico**, porque o MySQL não oferece mecanismo nativo de notificação de processo externo equivalente ao `NOTIFY/LISTEN` do PostgreSQL ([09:09] Diego), tornando o polling a abordagem mais simples e confiável disponível. O intervalo de 2 segundos atende com folga o requisito de latência máxima de 10 segundos ([09:10] Larissa). A separação em processo distinto garante que reinicializações da API não interrompam eventos em processamento ([09:11] Diego).

A entry-point `src/worker.ts` espelha o padrão do `src/server.ts` existente, incluindo instância própria do cliente de banco e shutdown gracioso. O script `npm run worker` é adicionado ao `package.json` de forma análoga aos scripts de servidor já existentes ([09:11] Larissa).

## 5. Prós e Contras das Opções

### Opção A — Worker em processo separado com polling periódico

- Positivo: isolamento de falhas — crash ou reinicialização da API não afeta o worker ([09:11] Diego).
- Positivo: sem nova infraestrutura — usa o MySQL existente e o mesmo modelo de configuração do projeto.
- Positivo: latência de até 2 segundos atende com margem o requisito de 10 segundos ([09:09] Diego, [09:10] Marcos).
- Negativo: latência mínima garantida de até 2 segundos no pior caso; não é reativo a eventos.
- Negativo: operacionalmente requer que dois processos estejam em execução para a feature funcionar completamente — `npm run dev` isolado não ativa webhooks.

### Opção B — Worker embutido no processo da API

- Positivo: deploy simplificado — apenas um processo para gerenciar.
- Negativo: ciclo de vida acoplado — reinicialização da API mata o worker e pode deixar eventos com status `PROCESSING` travados ([09:11] Diego).
- Negativo: falha no worker pode impactar a estabilidade da API e vice-versa.

### Opção C — Acionamento reativo via broker de mensagens

- Positivo: latência próxima de zero entre o insert na outbox e o acionamento do worker.
- Negativo: exige Redis ou infraestrutura equivalente — nova dependência operacional ([09:07] Diego).
- Negativo: complexidade desproporcional para um time pequeno com prazo de três sprints ([09:07] Diego).

## 6. Consequências

O sistema passa a ter dois processos de longa duração em todos os ambientes: o servidor da API e o worker de webhooks. Ambientes de desenvolvimento, staging e produção precisam executar `npm run worker` (ou equivalente via container/supervisor) além do processo da API. Engenheiros que executarem apenas `npm run dev` não terão o processamento de webhooks ativo, o que deve constar de forma explícita no guia de setup local e no onboarding.

A decisão de single-worker com ordering implícita por `created_at` é uma limitação documentada e consciente, não um defeito: garante ordenação dos eventos de um mesmo pedido enquanto houver apenas uma instância do worker. Escalar para múltiplos workers em paralelo exigiria particionamento por `order_id` ou locking pessimista com `SELECT ... FOR UPDATE SKIP LOCKED` — problema explicitamente adiado para o futuro ([09:13] Diego, [09:13] Larissa).

O worker deve implementar shutdown gracioso (SIGINT/SIGTERM) para garantir que eventos com status `PROCESSING` retornem a `PENDING` caso o processo seja encerrado durante uma tentativa, evitando que eventos fiquem presos indefinidamente. [PRECISA DE INFORMAÇÃO: Qual é o comportamento esperado para eventos com status `PROCESSING` no momento de um shutdown não-gracioso (ex: SIGKILL)? Devem retornar automaticamente a `PENDING` por lógica de health check periódico, ou é responsabilidade do operador intervir manualmente?]

## 7. Referências

- `src/server.ts` — entry-point da API; `src/worker.ts` deve espelhar este padrão com loop de polling no lugar do servidor HTTP.
- `src/config/database.ts` — factory do cliente de banco; o worker instancia o cliente de banco de forma independente usando a mesma `DATABASE_URL`.
- `package.json` — local onde o script `npm run worker` deve ser adicionado, análogo a `npm run dev` e `npm start`.
- `TRANSCRICAO.md:[09:09]–[09:13]` — decisões de polling, processo separado, ordering e limitações de escala horizontal.
