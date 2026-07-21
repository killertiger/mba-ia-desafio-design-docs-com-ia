# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Autor** | Larissa (Tech Lead)  |
| **Status** | Em revisão |
| **Data** | 2026-07-15 |
| **Revisores** | Marcos (Product Manager) · Bruno (Engenheiro Pleno, time de Pedidos) · Diego (Engenheiro Sênior, time de Plataforma) · Sofia (Engenheira de Segurança) |
| **Revisão obrigatória** | Segurança — 2 dias úteis reservados para revisão (HMAC e geração de secret) antes do deploy |
| **Prazo alvo** | 3 sprints, entrega até o fim de novembro |

---

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B sobre mudanças de status de pedidos via **webhooks outbound**, usando o **padrão Outbox no MySQL já existente**: a inserção do evento acontece dentro da mesma transação do `changeStatus`, garantindo que status alterado e evento registrado sejam atômicos. Um **worker em processo separado** faz polling a cada 2 segundos e dispara as chamadas HTTP, com **retry em backoff exponencial (5 tentativas, ~15h)** e **Dead Letter Queue** em tabela dedicada. Cada request é assinado com **HMAC-SHA256 usando secret única por endpoint**, rotacionável com grace period de 24h. A entrega é **at-least-once**, com `X-Event-Id` para deduplicação do lado do cliente.

Nenhuma infraestrutura nova é introduzida. A feature reutiliza MySQL, Prisma, Pino, `AppError` e o padrão de módulos já consolidados no projeto.

## 2. Contexto e problema

Três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — solicitaram formalmente notificação em tempo real de mudanças de status de pedidos. Hoje eles fazem polling contra `GET /orders`, o que torna a integração lenta e cara do lado deles. A Atlas sinalizou risco de churn caso a entrega não ocorra até o fim do trimestre, o que dá à feature caráter de retenção, não apenas de conveniência.

O requisito de latência é qualificado pelos clientes: **qualquer coisa abaixo de 10 segundos é considerada "tempo real"**. O escopo é estritamente **outbound** — a plataforma envia, os clientes recebem; não há webhooks de entrada.

A aplicação não possui hoje nenhum mecanismo de notificação externa, eventos ou filas. O problema técnico central é: **como disparar notificações confiáveis a sistemas externos sem acoplar a disponibilidade deles à transação de mudança de status do pedido?** Essa transação já é composta e crítica — atualiza `orders`, insere em `order_status_history` e decrementa `stockQuantity` dos produtos. Introduzir um HTTP call nela significaria que um cliente lento trava mudanças de status de outros pedidos, e que a indisponibilidade de um terceiro forçaria rollback de uma operação de negócio legítima. Um sistema externo fora do ar nunca pode desfazer uma mudança de status já válida.

## 3. Proposta técnica

### 3.1 Visão geral do fluxo

```
changeStatus()                        webhook_outbox            worker (processo separado)
─────────────────                     ──────────────            ──────────────────────────
BEGIN TRANSACTION
  update orders
  insert order_status_history    ──▶  insert evento              polling a cada 2s
  decrement stockQuantity             (snapshot do payload)  ──▶ lê pendentes em batch
COMMIT ──────────────────────────────────────────────────       assina HMAC-SHA256
  (rollback ⇒ evento some junto)                                POST ao endpoint (timeout 10s)
                                                                 │
                                                    sucesso ◀────┴────▶ falha
                                                       │                  │
                                                  marca entregue     retry backoff
                                                                    1m/5m/30m/2h/12h
                                                                          │
                                                                    5ª falha ⇒ webhook_dead_letter
```

### 3.2 Pilares da proposta

**Outbox no MySQL existente.** Dentro da mesma transação Prisma do `changeStatus`, uma linha é inserida em `webhook_outbox`. Se a transação commitou, o evento está registrado; se deu rollback, o evento some junto, sem inconsistência possível. A integração se dá por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o client transacional, evitando injetar um repository inteiro no `OrderService`. O payload é gravado como **snapshot no momento da inserção**, não renderizado no envio, para que o evento reflita o estado exato da transição. A **filtragem por status subscrito ocorre na inserção**: se nenhum endpoint do customer escuta aquele status, a linha nem é criada.

**Worker separado com polling de 2s.** O worker roda como processo Node independente (`src/worker.ts`, script `npm run worker`), espelhando o padrão de `src/server.ts`, com PrismaClient próprio sobre a mesma `DATABASE_URL`. Não pode viver dentro do processo da API, pois uma reinicialização da API interromperia o processamento de eventos. O polling de 2s atende o requisito de 10s com folga, e o MySQL não oferece alternativa reativa nativa comparável ao `NOTIFY/LISTEN` do Postgres.

**Retry e DLQ.** 5 tentativas em backoff exponencial — 1m / 5m / 30m / 2h / 12h, cobrindo ~15 horas. Timeout de 10s por chamada; resposta mais lenta conta como falha. Esgotadas as tentativas, o evento vai para `webhook_dead_letter` — tabela separada, com payload, motivo da falha e timestamp, mantendo a outbox principal com semântica limpa de eventos ativos. Reprocessamento é manual e intencional, via `POST /admin/webhooks/dead-letter/:id/replay`, restrito a role **ADMIN** e auditado.

**Segurança.** HMAC-SHA256 sobre o corpo do request, com **secret única por endpoint** — não global, para que o vazamento de uma secret não comprometa toda a base. Rotação via API com grace period de 24h, durante o qual a secret anterior continua válida. TLS obrigatório: URLs `http://` são recusadas na validação Zod. Payload limitado a 64KB, com erro acima disso. Headers: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.

**Contrato de entrega.** **At-least-once**: o cliente pode receber o mesmo evento duas vezes e deve deduplicar pelo `X-Event-Id` — UUID gerado na inserção da outbox e imutável em todas as retentativas. Exactly-once exigiria coordenação bidirecional e foi descartado; o padrão adotado é o mesmo de Stripe e GitHub. A semântica será documentada no portal de desenvolvedores.

**Reuso máximo do que já existe.** Módulo `src/modules/webhooks` seguindo controller/service/repository/routes/schemas; `AppError` e códigos com prefixo `WEBHOOK_` (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`); logger Pino e error middleware centralizado sem alterações, pois o middleware já captura qualquer `AppError` sem modificação; IDs em UUID, seguindo o padrão do projeto.

### 3.3 Superfície de API proposta

CRUD de configuração autenticado com JWT comum (`POST`, `PATCH`, `DELETE`, `GET` de webhooks por customer), histórico de entregas (`GET /webhooks/:id/deliveries`) e replay de DLQ restrito a ADMIN. O `customer_id` vem do body ou path, **não do JWT** — os usuários do sistema representam o cliente, mas o JWT atual é do operador. Contratos, payloads e status codes são detalhados no FDD.

## 4. Alternativas consideradas

| # | Alternativa | Trade-off que levou ao descarte |
| --- | --- | --- |
| 1 | **Disparo síncrono no `changeStatus`** | Acopla a transação de negócio à disponibilidade de terceiros: cliente lento trava mudança de status de outros pedidos, e cliente offline forçaria rollback de uma operação legítima. |
| 2 | **Redis Streams / fila externa** | Exige nova infraestrutura (Redis Cluster) para um time pequeno — overengineering. Além disso, a consistência entre transação e publicação exigiria Outbox de qualquer forma. |
| 3 | **Trigger de banco em vez de polling** | MySQL não tem listener nativo (`NOTIFY/LISTEN`); a trigger só executa SQL e não notifica processo externo. Improvisar a notificação seria frágil, e polling de 2s já atende os 10s. |
| 4 | **Worker embutido no processo da API** | Ciclo de vida acoplado: reinicialização da API mata o worker e pode deixar eventos travados em `PROCESSING`. |
| 5 | **3 tentativas de retry (mais agressivo)** | Não cobre indisponibilidade real de 2h em manutenção planejada já observada na base: retentaria 3× em 30 minutos e mataria o evento. |
| 6 | **Retry indefinido com backoff** | Evento fica pendurado indefinidamente quando o destino desaparece, acumulando para destinos permanentemente inativos. |
| 7 | **DLQ como status `FAILED` na própria outbox** | Mistura estados terminais e ativos, poluindo a leitura da outbox principal e prejudicando debug. |
| 8 | **Secret global da plataforma** | Vazamento único compromete todos os clientes. Há caso real de cliente que vazou secret em log de aplicação. |
| 9 | **TLS sem assinatura de payload** | TLS protege o transporte, mas não permite ao cliente verificar a autenticidade da origem do evento. |
| 10 | **Exactly-once delivery** | Exigiria coordenação bidirecional entre plataforma e cliente; complexidade desproporcional. At-least-once com `event_id` resolve a grande maioria dos casos. |

## 5. Questões em aberto

Pontos **não decididos** ou explicitamente adiados. Cada um precisa de resolução antes ou durante a implementação, conforme indicado.

| # | Questão | Situação | Referência |
| --- | --- | --- | --- |
| 1 | **Rate limiting de saída.** Se um customer tem 50 pedidos mudando de status em um minuto, a plataforma dispara 50 chamadas contra ele? | Adiado: observar e implementar se virar problema. | Decisão adiada |
| 2 | **Política de retenção da outbox.** Linhas entregues seriam arquivadas após cerca de 30 dias. | Fora do escopo desta feature, mas precisa existir antes da operação em produção. | ADR-001 |
| 3 | **Política de retenção da DLQ.** Eventos em `webhook_dead_letter` são purgados após um período ou permanecem até replay manual? | Não definido — sem política, a tabela cresce indefinidamente. | ADR-003 |
| 4 | **Ordering com múltiplos workers.** A garantia de ordem por `order_id` só vale enquanto for single-worker. | Adiado como problema futuro. Escalar exigiria particionamento por `order_id` ou lock pessimista. | ADR-002 |
| 5 | **Recuperação de eventos travados em `PROCESSING`.** Como o worker detecta e recupera eventos órfãos após crash ou SIGKILL durante uma tentativa? | Não definido — precisa de estratégia (ex.: timeout por estado) para evitar eventos presos. | ADR-002, ADR-003 |
| 6 | **`attempt_count` após replay.** O evento recolocado na outbox reinicia a contagem ou mantém o histórico acumulado? | Não definido — afeta a lógica do endpoint e a semântica dos campos da DLQ. | ADR-003 |
| 7 | **Entropia e exposição da secret.** Qual o comprimento mínimo e a fonte de entropia? A secret é recuperável via GET ou entregue uma única vez na criação? | Não definido. Item da revisão de segurança. | ADR-004 |
| 8 | **Comportamento pós-grace period.** Cliente que ainda usa a secret anterior após 24h recebe erro explícito (`WEBHOOK_SECRET_EXPIRED`) ou falha silenciosa? | Comportamento de borda não definido. | ADR-004 |
| 9 | **Validação de replay attack.** A plataforma validará a janela temporal do `X-Timestamp` ou apenas documentará a semântica para o cliente? | O header está definido, mas não de quem é a validação. | ADR-004 |
| 10 | **Endurecimento de RBAC no CRUD.** Hoje qualquer role autenticada configura webhooks. | Adiado para endurecimento futuro. | Decisão adiada |

**Fora de escopo desta fase** (decidido, não em aberto): notificação por email ao cliente com webhook falhando, adiada para uma próxima fase após medir o impacto; e dashboard visual para o cliente, tratado como projeto separado do time de frontend.

## 6. Impacto e riscos

### 6.1 Impacto

**No código existente.** A alteração crítica é cirúrgica mas sensível: o `changeStatus()` em `src/modules/orders/order.service.ts:126` passa a inserir na outbox dentro da transação já existente. A consequência é uma invariante forte: se a inserção na outbox falhar, a transação inteira sofre rollback, para que nunca exista o caso de status mudar e evento não sair. O resto da integração é aditivo — novo módulo, novas tabelas, nova entry-point.

**Operacional.** O sistema passa a ter **dois processos de longa duração** em todos os ambientes. Quem rodar apenas `npm run dev` não terá webhooks ativos, o que precisa constar no guia de setup e no onboarding. Deploy, supervisão e health check do worker são responsabilidades novas.

**No banco.** Quatro tabelas novas (`webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`) convivendo no banco OLTP de pedidos, com índices em status e `created_at` para a leitura em batch do worker.

**Contratual.** O formato do payload, os headers e a semântica at-least-once tornam-se contrato público. Migrar para exactly-once depois seria breaking change.

### 6.2 Riscos

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| **Regressão no `changeStatus`**, o caminho mais crítico do OMS (estoque + histórico + status) | Média | Alto | Testes ponta a ponta na transação; função pura recebendo `tx` em vez de injetar repository; meia sprint reservada para integração e testes |
| **Crescimento descontrolado das tabelas** de outbox/DLQ sem política de retenção definida | Alta | Médio | Definir retenção antes de produção (questões em aberto 2 e 3); índices já previstos |
| **Prazo**: 3 sprints vs. expectativa da Atlas para fim de novembro, com risco de churn declarado | Média | Alto | Estimativa já inclui a revisão de segurança; escopo enxuto com email, dashboard e rate limiting explicitamente fora |
| **Falha de segurança em HMAC ou geração de secret** | Baixa | Alto | Gate obrigatório de revisão de segurança antes do deploy, não negociável sob pressão de prazo; secrets adicionadas à redação de logs em `src/shared/logger/index.ts` |
| **Cliente não implementa deduplicação** e processa eventos duplicados | Média | Médio | Documentação destacada no portal do desenvolvedor; `X-Event-Id` imutável entre retentativas |
| **Bombardeio de cliente** sem rate limiting de saída (50 eventos/min) | Média | Médio | Aceito conscientemente nesta fase; observar e implementar se virar problema |
| **Eventos órfãos em `PROCESSING`** após crash do worker | Média | Médio | Shutdown gracioso (SIGINT/SIGTERM); estratégia para SIGKILL ainda em aberto (questão 5) |

## 7. Decisões relacionadas

| ADR | Decisão | Escopo |
| --- | --- | --- |
| [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md) | Padrão Outbox no MySQL para garantia de consistência dos eventos | Fundacional — âncora arquitetural da feature |
| [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) | Worker em processo separado com polling de 2s | Processamento |
| [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada | Resiliência |
| [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h | Segurança |
| [ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md) | Garantia at-least-once com idempotência via `X-Event-Id` | Contrato de entrega |

A decisão de **reuso máximo dos padrões existentes** não foi registrada como ADR isolada por não constituir escolha arquitetural com alternativas reais em disputa — trata-se de aderência ao padrão já vigente no projeto. Ela aparece transversalmente nos ADRs acima (`AppError`, Pino, `requireRole`, estrutura de módulos).

## 8. Referências

- `docs/TRACKER.md` — rastreabilidade item a item entre a documentação, o código e a origem de cada decisão.
- `src/modules/orders/order.service.ts:126` — método `changeStatus()`, ponto de integração da outbox.
- `src/middlewares/auth.middleware.ts:49` — `requireRole`, reutilizado no endpoint de replay.
- `src/shared/errors/app-error.ts` — classe base para os erros `WEBHOOK_*`.
- `src/shared/logger/index.ts` — logger Pino e lista de redação de campos sensíveis.
- `src/server.ts` / `package.json` — padrão de entry-point e scripts que `src/worker.ts` e `npm run worker` devem espelhar.
- `prisma/schema.prisma` — schema a ser estendido com as tabelas de webhook.
