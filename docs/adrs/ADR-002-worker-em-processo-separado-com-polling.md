# ADR-002: Worker em Processo Separado com Polling para Processamento da Outbox de Webhooks

**Status:** Aceito
**Data:** 2026-07-14
**ADRs Relacionados:** ADR-001 (Padrão Outbox no MySQL), ADR-003 (Retry com Backoff e DLQ)

---

## 1. Contexto e Problema

O sistema OMS precisa notificar clientes B2B sobre mudanças de status de pedidos por meio de webhooks outbound. A necessidade surge de três clientes — Atlas Comercial, MaxDistribuição e Nova Cargo — que hoje fazem polling ativo sobre o endpoint de listagem de pedidos, gerando carga desnecessária e latência na integração.

O modelo de padrão Outbox no MySQL foi adotado como estratégia de confiabilidade (ver ADR-001): dentro da mesma transação que altera o status do pedido, um evento é inserido na tabela `webhook_outbox`. Isso elimina a possibilidade de inconsistência entre a mudança de estado e o registro do evento a notificar. Uma vez adotado o Outbox, surge o problema central: **como o processo responsável por ler a tabela e disparar as chamadas HTTP deve ser estruturado e acionado?**

As alternativas consideradas se concentram no mecanismo de acionamento do worker e no modelo de isolamento de processo. O requisito de negócio é latência máxima de 10 segundos entre a mudança de status e a notificação ao cliente.

## 2. Fatores de Decisão

- Isolamento de falhas: reinicializações da API não devem interromper o processamento de eventos pendentes.
- Simplicidade operacional para um time pequeno, sem necessidade de nova infraestrutura.
- Atender à latência máxima de 10 segundos exigida pelos clientes B2B.
- Manutenção do padrão arquitetural existente no projeto (entry-point, configuração de banco, shutdown gracioso).
- Garantia de ordering de eventos por pedido enquanto a operação for single-worker.

## 3. Opções Consideradas

- **Opção A — Worker em processo separado com polling periódico** (escolhida): processo Node.js independente (`src/worker.ts`) com loop de polling a cada 2 segundos sobre a tabela `webhook_outbox`.
- **Opção B — Worker embutido no processo da API**: thread ou `setInterval` interno ao mesmo processo da API, sem separação de ciclo de vida.
- **Opção C — Acionamento reativo via broker de mensagens (Redis Streams/Pub-Sub)**: worker acionado por eventos publicados em um broker externo ao banco.

## 4. Decisão

Escolhida a **Opção A — Worker em processo separado com polling periódico**, porque o MySQL não oferece mecanismo nativo de notificação de processo externo equivalente ao `NOTIFY/LISTEN` do PostgreSQL, tornando o polling a abordagem mais simples e confiável disponível. O intervalo de 2 segundos atende com folga o requisito de latência máxima de 10 segundos. A separação em processo distinto garante que reinicializações da API não interrompam eventos em processamento.

A entry-point `src/worker.ts` espelha o padrão do `src/server.ts` existente: bootstrap assíncrono, logger compartilhado e shutdown gracioso em SIGINT/SIGTERM. O cliente de banco é próprio do worker — mesmo banco e mesma `DATABASE_URL`, instância nova porque é outro processo Node. Como o `server.ts` importa o singleton `prisma` de `src/config/database.ts` e a factory `createPrismaClient()` está exportada no mesmo módulo, qualquer um dos dois caminhos satisfaz a decisão: em um processo distinto, ambos produzem um cliente novo. O script `npm run worker` é adicionado ao `package.json` de forma análoga aos scripts de servidor já existentes.

## 5. Prós e Contras das Opções

### Opção A — Worker em processo separado com polling periódico

- Positivo: isolamento de falhas — crash ou reinicialização da API não afeta o worker.
- Positivo: sem nova infraestrutura — usa o MySQL existente e o mesmo modelo de configuração do projeto.
- Positivo: latência de até 2 segundos atende com margem o requisito de 10 segundos.
- Negativo: latência mínima garantida de até 2 segundos no pior caso; não é reativo a eventos.
- Negativo: operacionalmente requer que dois processos estejam em execução para a feature funcionar completamente — `npm run dev` isolado não ativa webhooks.

### Opção B — Worker embutido no processo da API

- Positivo: deploy simplificado — apenas um processo para gerenciar.
- Negativo: ciclo de vida acoplado — reinicialização da API mata o worker, o que pode deixar eventos com status `PROCESSING` travados.
- Negativo: falha no worker pode impactar a estabilidade da API e vice-versa.

### Opção C — Acionamento reativo via broker de mensagens

- Positivo: latência próxima de zero entre o insert na outbox e o acionamento do worker.
- Negativo: exige Redis ou infraestrutura equivalente — nova dependência operacional.
- Negativo: complexidade desproporcional para um time pequeno com prazo de três sprints.

## 6. Consequências

O sistema passa a ter dois processos de longa duração em todos os ambientes: o servidor da API e o worker de webhooks. Ambientes de desenvolvimento, staging e produção precisam executar `npm run worker` (ou equivalente via container/supervisor) além do processo da API. Engenheiros que executarem apenas `npm run dev` não terão o processamento de webhooks ativo, o que deve constar de forma explícita no guia de setup local e no onboarding.

A decisão de single-worker com ordering implícita por `order_id` é uma limitação documentada e consciente, não um defeito. O worker processa os eventos na ordem de `created_at` da outbox, e é esse mecanismo que garante a ordenação dos eventos de um mesmo pedido enquanto houver apenas uma instância em execução. Não há garantia de ordenação global, apenas por `order_id` e enquanto for single-worker; os clientes não requisitaram ordenação global. Escalar para múltiplos workers em paralelo exigiria particionamento por `order_id` ou lock pessimista — problema explicitamente adiado para o futuro.

O worker deve implementar shutdown gracioso para SIGINT/SIGTERM, espelhando o tratamento já existente em `src/server.ts:13-21` — que encerra o processo e chama `prisma.$disconnect()`, sem qualquer reconciliação de estado.

O destino de eventos que fiquem presos em `PROCESSING` quando o worker é encerrado no meio de uma tentativa permanece em aberto: o tema não foi decidido e não há precedente no código atual, que não possui reaper, lease, lock ou timeout de job em nenhum módulo de `src/`. [PRECISA DE INFORMAÇÃO: Eventos que fiquem em `PROCESSING` após um encerramento abrupto (ex: SIGKILL) devem retornar automaticamente a `PENDING` — por lógica de recuperação periódica baseada na idade do registro — ou é responsabilidade do operador intervir manualmente? A definição deve estabelecer também se o shutdown gracioso reconcilia o estado dos eventos em andamento antes de encerrar, indo além do que o `server.ts` faz hoje.]

## 7. Referências

- `src/server.ts` — entry-point da API; `src/worker.ts` deve espelhar este padrão com loop de polling no lugar do servidor HTTP. O shutdown gracioso de referência está em `src/server.ts:13-21`.
- `src/config/database.ts` — factory `createPrismaClient()` (linha 4) e singleton `prisma` (linha 10); o worker usa o mesmo módulo e a mesma `DATABASE_URL`, com cliente próprio por ser outro processo.
- `package.json` — local onde o script `npm run worker` deve ser adicionado, análogo a `npm run dev` e `npm start`.
