# Tracker de Rastreabilidade

Este documento estabelece a **rastreabilidade** entre os documentos de design produzidos e suas fontes de origem: a transcrição da reunião técnica (`TRANSCRICAO.md`) ou o código-fonte da aplicação existente.

Cada linha liga um item registrado em um documento a uma origem verificável. Se um item não tem origem identificável, ele não deveria existir no documento — o tracker é a defesa contra informação inventada.

## Escopo atual

> ⚠️ Neste momento o tracker cobre **apenas os documentos da pasta `docs/adrs/`**. Os demais documentos (PRD, RFC, FDD) serão adicionados posteriormente, cada um em sua própria seção.

## Como ler a tabela

| Coluna | Significado |
| --- | --- |
| **ID** | Identificador único do item no tracker (`A<n>-<seq>`, onde `<n>` é o número do ADR). |
| **Documento** | ADR onde o item aparece (ver legenda abaixo). |
| **Linha** | Linha(s) aproximada(s) no documento onde o item está registrado. |
| **Tipo** | Natureza do item: Decisão, Requisito Funcional (RF), Requisito Não Funcional (RNF), Restrição, Trade-off, Alternativa, Limitação, Ponto em Aberto, Integração, Contexto. |
| **Conteúdo (resumo)** | Descrição de uma linha do item. |
| **Fonte** | `TRANSCRICAO`, `CODIGO` ou `—` (lacuna sem origem, ponto em aberto). |
| **Localização** | Para `TRANSCRICAO`: `[hh:mm] Nome`. Para `CODIGO`: caminho do arquivo. Para lacunas: nota explicativa. |

### Legenda dos documentos

| Doc | Arquivo |
| --- | --- |
| ADR-001 | [`docs/adrs/ADR-001-padrao-outbox-no-mysql.md`](./adrs/ADR-001-padrao-outbox-no-mysql.md) |
| ADR-002 | [`docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md`](./adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| ADR-003 | [`docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md`](./adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| ADR-004 | [`docs/adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md`](./adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md) |
| ADR-005 | [`docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md`](./adrs/ADR-005-garantia-at-least-once-com-x-event-id.md) |

---

## Rastreabilidade — ADRs

### ADR-001 — Padrão Outbox no MySQL

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| A1-01 | ADR-001 | L11 | Contexto | Pedido formal de 3 clientes B2B (Atlas, MaxDistribuição, Nova Cargo) por notificação em tempo real | TRANSCRICAO | `[09:00] Marcos` |
| A1-02 | ADR-001 | L15 | RNF | Latência de notificação abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| A1-03 | ADR-001 | L15 | Restrição | Escopo limitado a webhooks de saída (outbound) | TRANSCRICAO | `[09:02] Sofia` |
| A1-04 | ADR-001 | L19 | Fator | HTTP síncrono dentro da transação cria acoplamento inaceitável | TRANSCRICAO | `[09:04] Bruno` |
| A1-05 | ADR-001 | L20 | Fator | Indisponibilidade do cliente não pode forçar rollback do status | TRANSCRICAO | `[09:04] Bruno` |
| A1-06 | ADR-001 | L33 | Decisão | Adoção do Padrão Outbox no MySQL, insert na mesma transação de status | TRANSCRICAO | `[09:06] Diego` |
| A1-07 | ADR-001 | L37 | Decisão | Payload gravado como snapshot no momento da inserção na outbox | TRANSCRICAO | `[09:52] Larissa` |
| A1-08 | ADR-001 | L37 | Decisão | Filtragem por status subscrito ocorre na inserção da outbox | TRANSCRICAO | `[09:34] Bruno` |
| A1-09 | ADR-001 | L48-53 | Alternativa | Disparo síncrono no service de pedidos descartado | TRANSCRICAO | `[09:04] Bruno` |
| A1-10 | ADR-001 | L55-59 | Alternativa | Redis Streams / fila externa descartada (overengineering p/ time pequeno) | TRANSCRICAO | `[09:07] Diego` |
| A1-11 | ADR-001 | L65 | Consequência | Arquivamento de eventos entregues após ~30 dias (fora do escopo) | TRANSCRICAO | `[09:08] Diego` |
| A1-12 | ADR-001 | L67 | Limitação | Ordenação implícita por `order_id` válida apenas com single-worker | TRANSCRICAO | `[09:12] Diego` |
| A1-13 | ADR-001 | L67 | Restrição | Clientes não requisitaram ordenação global | TRANSCRICAO | `[09:14] Marcos` |
| A1-14 | ADR-001 | L35, L71 | Integração | Inserção na outbox como última operação de `changeStatus()` | CODIGO | `src/modules/orders/order.service.ts` |
| A1-15 | ADR-001 | L72 | Integração | Novas tabelas `webhook_*` adicionadas via migração Prisma | CODIGO | `prisma/schema.prisma` |

### ADR-002 — Worker em Processo Separado com Polling

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| A2-01 | ADR-002 | L15 | RNF | Latência máxima de 10 segundos entre mudança de status e envio | TRANSCRICAO | `[09:02] Marcos` |
| A2-02 | ADR-002 | L19 | Fator | Reinício da API não pode interromper processamento de eventos pendentes | TRANSCRICAO | `[09:11] Diego` |
| A2-03 | ADR-002 | L33 | Decisão | Worker em processo separado, polling a cada 2 segundos | TRANSCRICAO | `[09:09] Diego` |
| A2-04 | ADR-002 | L33 | Fator | MySQL não possui `NOTIFY/LISTEN` nativo como o PostgreSQL | TRANSCRICAO | `[09:09] Diego` |
| A2-05 | ADR-002 | L33 | Decisão | Intervalo de 2s atende com folga o requisito de 10s | TRANSCRICAO | `[09:10] Larissa` |
| A2-06 | ADR-002 | L35 | Decisão | Entry-point `src/worker.ts` + script `npm run worker`, espelhando `server.ts` | TRANSCRICAO | `[09:11] Larissa` |
| A2-07 | ADR-002 | L47-51 | Alternativa | Worker embutido na API descartado (ciclo de vida acoplado) | TRANSCRICAO | `[09:11] Diego` |
| A2-08 | ADR-002 | L53-57 | Alternativa | Broker reativo (Redis Pub/Sub) descartado (nova dependência) | TRANSCRICAO | `[09:07] Diego` |
| A2-09 | ADR-002 | L63 | Limitação | Single-worker; escala horizontal exigiria particionamento por `order_id` | TRANSCRICAO | `[09:13] Diego` |
| A2-10 | ADR-002 | L65 | Ponto em Aberto | Recuperação de eventos `PROCESSING` em shutdown não-gracioso (SIGKILL) | — | Lacuna — não discutido na reunião |
| A2-11 | ADR-002 | L69 | Integração | `worker.ts` espelha o entry-point da API | CODIGO | `src/server.ts` |
| A2-12 | ADR-002 | L70 | Integração | Worker instancia `PrismaClient` próprio com a mesma `DATABASE_URL` | CODIGO | `src/config/database.ts` |
| A2-13 | ADR-002 | L71 | Integração | Novo script `npm run worker` | CODIGO | `package.json` |

### ADR-003 — Retry com Backoff Exponencial e DLQ

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| A3-01 | ADR-003 | L11 | Contexto | Clientes com indisponibilidade real de até 2h (manutenção planejada) | TRANSCRICAO | `[09:16] Diego` |
| A3-02 | ADR-003 | L20 | Fator | Retry sem limite deixa evento pendurado se o cliente sumir | TRANSCRICAO | `[09:15] Diego` |
| A3-03 | ADR-003 | L42 | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Diego` |
| A3-04 | ADR-003 | L44 | Decisão | Ciclo total de ~15h validado como aceitável pelo negócio | TRANSCRICAO | `[09:17] Marcos` |
| A3-05 | ADR-003 | L44 | RNF | Timeout de 10s por chamada HTTP; resposta lenta conta como falha | TRANSCRICAO | `[09:42] Diego` |
| A3-06 | ADR-003 | L46 | Decisão | DLQ em tabela separada (`webhook_dead_letter`) | TRANSCRICAO | `[09:18] Diego` |
| A3-07 | ADR-003 | L46 | Decisão | Replay manual via endpoint admin, exige role ADMIN | TRANSCRICAO | `[09:35] Larissa` |
| A3-08 | ADR-003 | L32-35, L64-72 | Alternativa | 3 tentativas com backoff agressivo descartada | TRANSCRICAO | `[09:16] Bruno` |
| A3-09 | ADR-003 | L36-38, L74-81 | Alternativa | Retry indefinido com `FAILED` na outbox descartada | TRANSCRICAO | `[09:18] Diego` |
| A3-10 | ADR-003 | L87 | Requisito | Auditoria: registrar quem executou o replay (`replayed_by`/`replayed_at`) | TRANSCRICAO | `[09:36] Sofia` |
| A3-11 | ADR-003 | L85 | Consequência | Comportamento de retry documentado no portal do desenvolvedor | TRANSCRICAO | `[09:26] Marcos` |
| A3-12 | ADR-003 | L48 | Ponto em Aberto | Política de retenção da tabela `webhook_dead_letter` | — | Lacuna — não discutido na reunião |
| A3-13 | ADR-003 | L89 | Ponto em Aberto | Semântica do `attempt_count` após replay manual | — | Lacuna — não discutido na reunião |
| A3-14 | ADR-003 | L91 | Ponto em Aberto | Recuperação de eventos travados em `PROCESSING` | — | Lacuna — não discutido na reunião |
| A3-15 | ADR-003 | L57, L95 | Integração | Reuso de `requireRole` no endpoint de replay | CODIGO | `src/middlewares/auth.middleware.ts` |
| A3-16 | ADR-003 | L96 | Integração | Erros `WEBHOOK_*` estendem a classe base `AppError` | CODIGO | `src/shared/errors/app-error.ts` |

### ADR-004 — Autenticação HMAC-SHA256 por Endpoint

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| A4-01 | ADR-004 | L13 | Motivação | Caso real: cliente vazou secret em log de aplicação | TRANSCRICAO | `[09:22] Diego` |
| A4-02 | ADR-004 | L13 | Fator | Cliente precisa validar origem e integridade do payload | TRANSCRICAO | `[09:19] Sofia` |
| A4-03 | ADR-004 | L34 | Decisão | HMAC-SHA256 sobre o corpo do request | TRANSCRICAO | `[09:22] Sofia` |
| A4-04 | ADR-004 | L36 | Fator | SHA-256 é padrão de mercado (Stripe, GitHub) | TRANSCRICAO | `[09:20] Sofia` |
| A4-05 | ADR-004 | L36 | Decisão | Secret única por endpoint (isolamento de blast radius) | TRANSCRICAO | `[09:21] Sofia` |
| A4-06 | ADR-004 | L36 | Decisão | Rotação de secret com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| A4-07 | ADR-004 | L38 | Decisão | Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` | TRANSCRICAO | `[09:44] Sofia` |
| A4-08 | ADR-004 | L23 | RNF | TLS obrigatório; URL `http` rejeitada na validação Zod | TRANSCRICAO | `[09:23] Sofia` |
| A4-09 | ADR-004 | L56-64 | Alternativa | Secret global descartada ("vaza uma, vaza tudo") | TRANSCRICAO | `[09:21] Sofia` |
| A4-10 | ADR-004 | L66-73 | Alternativa | TLS sem assinatura de payload descartada | TRANSCRICAO | `[09:20] Sofia` |
| A4-11 | ADR-004 | L77 | Consequência | Sofia reserva 2 dias úteis para revisão de segurança pré-deploy | TRANSCRICAO | `[09:46] Sofia` |
| A4-12 | ADR-004 | L24 | Ponto em Aberto | Política de comprimento/entropia das secrets geradas | TRANSCRICAO | `[09:31] Marcos` (não fechado) |
| A4-13 | ADR-004 | L40 | Ponto em Aberto | Exposição da secret: entrega única vs. recuperável | TRANSCRICAO | `[09:31] Marcos` (não fechado) |
| A4-14 | ADR-004 | L81 | Ponto em Aberto | Comportamento após expiração do grace period de 24h | — | Lacuna — não discutido na reunião |
| A4-15 | ADR-004 | L83 | Ponto em Aberto | Validação de replay attack via `X-Timestamp` (obrigatória?) | TRANSCRICAO | `[09:44] Diego` (não fechado) |
| A4-16 | ADR-004 | L79, L88 | Integração | Secrets adicionadas à redação de campos sensíveis do Pino | CODIGO | `src/shared/logger/index.ts` |
| A4-17 | ADR-004 | L87 | Integração | Reuso do padrão de middleware de autenticação (JWT) | CODIGO | `src/middlewares/auth.middleware.ts` |
| A4-18 | ADR-004 | L89 | Integração | Novos códigos `WEBHOOK_SECRET_*` na hierarquia de erros HTTP | CODIGO | `src/shared/errors/http-errors.ts` |

### ADR-005 — Garantia At-Least-Once com X-Event-Id

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| A5-01 | ADR-005 | L13 | Decisão | Semântica de entrega at-least-once | TRANSCRICAO | `[09:24] Diego` |
| A5-02 | ADR-005 | L13 | Fator | Padrão de mercado (Stripe, GitHub); resolve 99% dos casos | TRANSCRICAO | `[09:25] Diego` |
| A5-03 | ADR-005 | L15 | Decisão | `X-Event-Id` (UUID gerado na inserção) para dedup no cliente | TRANSCRICAO | `[09:26] Larissa` |
| A5-04 | ADR-005 | L19 | Fator | Exactly-once inviável para o MVP (coordenação bidirecional) | TRANSCRICAO | `[09:25] Diego` |
| A5-05 | ADR-005 | L28, L46-50 | Alternativa | Exactly-once delivery descartada | TRANSCRICAO | `[09:25] Diego` |
| A5-06 | ADR-005 | L29, L52-56 | Alternativa | At-most-once (sem retry) descartada | TRANSCRICAO | `[09:15] Diego` |
| A5-07 | ADR-005 | L44 | Trade-off | Responsabilidade de deduplicação transferida ao cliente | TRANSCRICAO | `[09:25] Sofia` |
| A5-08 | ADR-005 | L60 | Consequência | Semântica documentada no portal do desenvolvedor | TRANSCRICAO | `[09:26] Marcos` |
| A5-09 | ADR-005 | L35 | Ponto em Aberto | Tipo de UUID: v4 (aleatório) vs. v7 (baseado em timestamp) | — | Lacuna — não discutido na reunião |
| A5-10 | ADR-005 | L64 | Ponto em Aberto | `event_id` também no corpo do payload além do header? | TRANSCRICAO | `[09:43] Diego` (não fechado) |
| A5-11 | ADR-005 | L70 | Integração | Publicação do evento na outbox dentro de `changeStatus()` | CODIGO | `src/modules/orders/order.service.ts` |
| A5-12 | ADR-005 | L68 | Integração | `event_id` indexado no schema e incluído nos logs do Pino | CODIGO | `prisma/schema.prisma` |

---

## Resumo de cobertura (ADRs)

| Métrica | Valor |
| --- | --- |
| Total de itens rastreados | 74 |
| Itens com fonte `TRANSCRICAO` | 58 |
| Itens com fonte `CODIGO` | 12 |
| Pontos em aberto / lacunas | 6 lacunas puras + 4 levantados mas não fechados |
| ADRs cobertos | 5 de 5 |

> Os "Pontos em Aberto" registrados aqui são as lacunas que a IA identificou honestamente durante a geração dos ADRs (marcadores `[NECESSITA INPUT]`). Eles não têm decisão fechada na reunião e alimentarão a seção **"Questões em Aberto"** do futuro RFC.

## Próximas seções (a adicionar)

- [ ] Rastreabilidade — PRD (`docs/PRD.md`)
- [ ] Rastreabilidade — RFC (`docs/RFC.md`)
- [ ] Rastreabilidade — FDD (`docs/FDD.md`)
