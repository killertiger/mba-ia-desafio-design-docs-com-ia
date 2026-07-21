# ADR-001: Padrão Outbox no MySQL para Garantia de Consistência dos Eventos de Webhook

**Status:** Aceito
**Data:** 2026-07-14
**ADRs Relacionados:** (nenhum — este é o ADR fundacional da feature de webhooks)

---

## 1. Contexto e Problema

O sistema de gerenciamento de pedidos (OMS) precisa notificar três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — em tempo real sobre mudanças de status de pedidos. Hoje esses clientes fazem polling manual via `GET /orders`, o que torna a integração lenta e custosa, e há risco de churn caso a entrega não ocorra dentro do trimestre.

O núcleo do problema é arquitetural: como garantir que, ao mudar o status de um pedido, o evento de notificação seja disparado de forma confiável e consistente, sem comprometer a atomicidade da transação existente nem introduzir dependência em sistemas externos? A transação de mudança de status já é composta por múltiplas operações (atualização do pedido, registro de histórico e controle de estoque), e qualquer falha de entrega não pode forçar rollback do status do pedido.

A solução precisa atender ao requisito de latência definido pelos clientes: abaixo de 10 segundos entre a mudança de status e a notificação, com escopo restrito a webhooks de saída — o sistema envia, os clientes recebem.

## 2. Fatores de Decisão

- A transação de mudança de status é composta por múltiplas operações críticas; um HTTP call síncrono criaria acoplamento inaceitável com sistemas externos.
- Indisponibilidade do cliente receptor não pode forçar rollback do status do pedido — as duas operações devem ser desacopladas.
- O time é pequeno; soluções que exijam nova infraestrutura (Redis Cluster, Kafka) representam overengineering para o contexto atual.
- A garantia de consistência entre "status atualizado" e "evento registrado" deve ser atômica — ou ambos ocorrem, ou nenhum ocorre.
- O acúmulo de eventos na tabela é uma preocupação legítima de performance, endereçável por indexação e leitura em batch sem exigir infraestrutura adicional.
- O requisito de latência de menos de 10 segundos pode ser atendido sem reatividade em tempo real.

## 3. Opções Consideradas

1. **Padrão Outbox no MySQL existente** (opção escolhida)
2. **Disparo síncrono dentro do service de pedidos**
3. **Redis Streams ou fila externa**

## 4. Decisão

Escolhida a opção **Padrão Outbox no MySQL existente**, porque garante atomicidade entre a mudança de status e o registro do evento sem introduzir nova infraestrutura, atendendo ao requisito de latência e ao contexto de time pequeno.

A decisão estabelece que, dentro da mesma transação Prisma que atualiza o pedido, registra o histórico e ajusta o estoque, uma inserção na tabela `webhook_outbox` é realizada como operação adicional. O ponto de integração no código existente é `src/modules/orders/order.service.ts:126-179`. Um worker separado (processo distinto da API) realiza polling periódico dessa tabela e efetua as chamadas HTTP aos endpoints dos clientes.

O desenho operacional da tabela acompanha a decisão: cada evento transita entre quatro estados — pendente, processando, falhou e entregue — com índice no campo de status e em `created_at`. O worker lê apenas os eventos pendentes mais antigos, em batch pequeno, processa e marca como entregue. Isso mantém o custo de leitura estável independentemente do volume de linhas já entregues acumuladas na tabela, respondendo à preocupação de performance sobre acúmulo de eventos.

Três sub-decisões acompanham o desenho: (1) o payload é serializado como snapshot do estado do pedido no momento da inserção, não no momento do envio, garantindo que o evento reflita o estado exato da transição; (2) a filtragem por status subscrito ocorre na inserção, evitando linhas desnecessárias para eventos que o endpoint não consome; e (3) a chave primária da outbox é UUID, seguindo o padrão já adotado em todos os models do schema atual.

## 5. Prós e Contras das Opções

### Padrão Outbox no MySQL existente

- Positivo: Atomicidade entre transação de negócio e registro do evento é garantida pelo banco de dados.
- Positivo: Zero nova infraestrutura; aproveita MySQL já operacional.
- Positivo: Snapshot na inserção elimina risco de evento com dados desatualizados.
- Positivo: Índice em status e `created_at`, com leitura em batch apenas dos pendentes, mantém o custo de leitura do worker estável conforme a tabela cresce.
- Negativo: A tabela `webhook_outbox` co-existe no banco OLTP de pedidos, exigindo política de arquivamento periódico de eventos entregues.

### Disparo síncrono dentro do service de pedidos

- Positivo: Implementação mais simples, sem worker adicional.
- Negativo: Clientes lentos travam a transação de mudança de status para outros pedidos.
- Negativo: Indisponibilidade do cliente forçaria rollback da mudança de status.
- Negativo: Acoplamento forte entre disponibilidade de sistemas externos e operações internas.

### Redis Streams ou fila externa

- Positivo: Maior capacidade de escala horizontal e separação de responsabilidades.
- Negativo: Requer nova infraestrutura (Redis Cluster ou equivalente) — overengineering para o contexto atual.
- Negativo: A consistência entre transação de banco e publicação na fila exigiria padrões adicionais (two-phase commit ou Outbox de qualquer forma).

## 6. Consequências

A adoção do Padrão Outbox no MySQL estabelece a âncora arquitetural de toda a feature de webhooks. As decisões subsequentes — worker com polling, estratégia de retry, gerenciamento de Dead Letter Queue e autenticação HMAC — derivam diretamente desta. Alterar este padrão implicaria redesenhar a feature integralmente.

A tabela `webhook_outbox` co-existindo no banco OLTP de pedidos cria uma responsabilidade operacional nova: o arquivamento ou purge periódico de eventos com status entregue. A janela de referência é de cerca de 30 dias, e o arquivamento é explicitamente delimitado como fora do escopo desta feature. A política formal de retenção permanece, portanto, em aberto para trabalho futuro.

A garantia de ordenação de eventos é implícita por `order_id` enquanto houver um único worker em execução. Escalar para múltiplos workers simultâneos remove essa garantia; para esse cenário futuro, particionamento por `order_id` ou lock pessimista seriam necessários. Esta limitação é documentada e aceita, pois os clientes não requisitaram ordenação global.

## 7. Referências

- `src/modules/orders/order.service.ts:126-179` — método `changeStatus()`, ponto de integração onde a inserção na outbox deve ocorrer dentro da transação existente.
- `prisma/schema.prisma` — schema atual sem as tabelas de webhook; as tabelas `webhook_outbox`, `webhook_dead_letter`, `webhook_deliveries` e `webhook_endpoints` deverão ser adicionadas via migração Prisma.
