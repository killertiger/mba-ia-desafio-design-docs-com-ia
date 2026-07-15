# ADR-005: Garantia de Entrega At-Least-Once com Idempotência via X-Event-Id

**Status:** Aceito
**Data:** 2026-07-14
**ADRs Relacionados:** ADR-001 (Padrão Outbox no MySQL), ADR-002 (Worker com Polling), ADR-003 (Retry com Backoff e DLQ), ADR-004 (Autenticação HMAC-SHA256)

---

## 1. Contexto e Declaração do Problema

O sistema de webhooks do OMS precisa definir o contrato de confiabilidade de entrega entre a plataforma e os clientes B2B que consomem eventos de mudança de status de pedidos. O problema central é: o que acontece quando a chamada HTTP ao endpoint do cliente é bem-sucedida (HTTP 200), mas o worker falha antes de marcar o evento como entregue na tabela de outbox? Nesse cenário, o worker vai reprocessar o evento na próxima execução — resultando em entrega duplicada ao cliente.

A reunião técnica de quinta-feira decidiu explicitamente adotar a semântica **at-least-once** para entrega de eventos ([09:24] Diego). Para permitir que os clientes detectem e descartam duplicatas com segurança, cada evento carrega um header `X-Event-Id` contendo um UUID gerado no momento da inserção do evento na outbox — imutável em todas as tentativas de reentrega. Diego referenciou Stripe e GitHub como precedentes de mercado que adotam o mesmo padrão ([09:25]): "At-least-once com event_id resolve 99% dos casos." Sofia reconheceu a transferência de responsabilidade ao cliente ([09:25]), e Marcos comprometeu-se a documentar o comportamento no portal de desenvolvedores ([09:26]).

O UUID gerado na inserção da outbox é o mesmo identificador disponível no histórico de entregas via endpoint de consulta. A escolha de geração na inserção — e não no momento do envio — garante que múltiplas tentativas do mesmo evento compartilhem sempre o mesmo identificador, tornando a deduplicação do lado do cliente determinística. A decisão foi formalmente fechada por Larissa em [09:26]: "Beleza. At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."

## 2. Fatores de Decisão

- Complexidade de implementação: exactly-once requer coordenação bidirecional entre plataforma e cliente, inviável para MVP ([09:25] Diego)
- Alinhamento com padrões de mercado reconhecidos pelos clientes B2B (Stripe, GitHub), reduzindo curva de adoção ([09:25] Diego)
- Necessidade de deduplicação determinística: o UUID deve ser gerado uma única vez na inserção da outbox, nunca regenerado em retentativas
- Contrato de API público: o formato e semântica do `X-Event-Id` tornam-se parte do contrato externo da plataforma, impactando versionamento futuro
- Rastreabilidade operacional: o mesmo identificador deve aparecer em logs de processamento e no histórico de entregas para correlação entre tentativas

## 3. Opções Consideradas

- **Opção A — At-least-once com idempotência via X-Event-Id** (escolhida): UUID gerado na inserção da outbox, enviado como header em todas as tentativas; cliente é responsável pela deduplicação
- **Opção B — Exactly-once delivery**: requereria protocolo de confirmação bidirecional (Two-Phase Commit ou idempotency key com acknowledgment explícito do cliente)
- **Opção C — At-most-once (sem retry)**: evento entregue no máximo uma vez; falhas de entrega resultam em perda permanente do evento

## 4. Decisão

Adotada a **Opção A — at-least-once com idempotência via X-Event-Id**, porque elimina a complexidade de coordenação bidirecional do exactly-once enquanto oferece mecanismo suficiente para que clientes B2B implementem deduplicação — padrão amplamente documentado e adotado por plataformas de referência como Stripe e GitHub ([09:25] Diego). A semântica at-least-once é compatível com a estratégia de retry com backoff exponencial adotada pelo worker, onde tentativas subsequentes são esperadas e necessárias.

[NECESSITA INFORMAÇÃO: O UUID deve ser do tipo v4 (aleatório) ou v7 (baseado em timestamp, com melhor ordenação para debugging e consultas temporais)? A escolha afeta a estratégia de indexação na tabela de outbox e a interpretação do identificador pelos clientes.]

## 5. Prós e Contras das Opções

### Opção A — At-least-once com X-Event-Id

- Bom: implementação unilateral — apenas a plataforma precisa garantir o identificador; cliente pode implementar deduplicação a seu tempo
- Bom: alinhado com padrões de mercado consolidados, reduzindo resistência de adoção dos clientes ([09:25] Diego)
- Bom: compatível com retry automático sem modificações adicionais no worker
- Ruim: transfere responsabilidade de deduplicação ao cliente — clientes que não implementarem deduplicação podem processar eventos duplicados ([09:25] Sofia)

### Opção B — Exactly-once delivery

- Bom: elimina a necessidade de lógica de deduplicação nos sistemas dos clientes
- Ruim: requer protocolo de confirmação bidirecional, aumentando significativamente escopo e complexidade ([09:25] Diego)
- Ruim: dependente de comportamento correto do cliente para funcionar — qualquer falha no acknowledgment quebra a garantia

### Opção C — At-most-once (sem retry)

- Bom: implementação mais simples — sem lógica de retry ou rastreamento de tentativas
- Ruim: perda permanente de eventos em caso de indisponibilidade temporária do cliente, inaceitável para integrações B2B críticas
- Ruim: contradiz diretamente a estratégia de retry com backoff exponencial decidida para a feature ([09:15] Diego)

## 6. Consequências

A adoção de at-least-once como contrato público implica que qualquer cliente B2B que implemente integração com o sistema de webhooks deve ser informado explicitamente dessa semântica. Marcos comprometeu-se a documentar o comportamento no portal de desenvolvedores ([09:26]), cobrindo o mecanismo de deduplicação via `X-Event-Id` e exemplos de implementação para os cenários mais comuns. Uma vez que clientes tenham integrado com base nessa semântica, migrar para exactly-once constituiria um breaking change no contrato da API de webhooks.

O UUID gerado na inserção da outbox deve ser imutável em todas as retentativas. Uma falha nesse contrato interno — por exemplo, geração do UUID no momento do envio em vez da inserção — quebraria silenciosamente a garantia de idempotência sem erro explícito, tornando a deduplicação do lado do cliente não determinística. Esse é um ponto de atenção crítico para engenheiros que implementarem o worker.

[NECESSITA INFORMAÇÃO: O `event_id` deve ser exposto tanto no header `X-Event-Id` quanto no corpo do payload JSON? A especificação mencionada em [09:43] sugere que sim, mas não foi formalmente confirmada. A decisão afeta os clientes que processam apenas o corpo do payload sem inspecionar headers.]

## 7. Referências

- `prisma/schema.prisma` — schema atual sem a tabela `webhook_outbox`; o campo `event_id` deve ser indexado para suportar consultas do histórico de entregas
- `src/shared/logger/index.ts` — logger Pino; o `event_id` deve ser incluído em todos os logs de processamento do worker para correlação entre tentativas
- `src/modules/orders/order.service.ts:126` — método `changeStatus()`, ponto de integração onde o evento é publicado na outbox dentro da transação existente
