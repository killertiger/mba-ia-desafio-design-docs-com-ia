# Tracker de Rastreabilidade

Este documento estabelece a **rastreabilidade** entre os documentos de design produzidos e suas fontes de origem: a transcrição da reunião técnica (`TRANSCRICAO.md`) ou o código-fonte da aplicação existente.

Cada linha liga um item registrado em um documento a uma origem verificável. Se um item não tem origem identificável, ele não deveria existir no documento — o tracker é a defesa contra informação inventada.

## Escopo atual

> ✅ O tracker cobre o **pacote completo de documentação**: os 5 ADRs (`docs/adrs/`), o RFC (`docs/RFC.md`), o FDD (`docs/FDD.md`) e o PRD (`docs/PRD.md`).

## Como ler a tabela

| Coluna | Significado |
| --- | --- |
| **ID** | Identificador único do item no tracker: `A<n>-<seq>` para ADRs (onde `<n>` é o número do ADR), `R-<seq>` para o RFC, `F-<seq>` para o FDD e `P-<seq>` para o PRD. |
| **Documento** | Documento onde o item aparece (ver legenda abaixo). |
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
| RFC | [`docs/RFC.md`](./RFC.md) |
| FDD | [`docs/FDD.md`](./FDD.md) |
| PRD | [`docs/PRD.md`](./PRD.md) |

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

## Rastreabilidade — RFC

### Metadados e contexto (§ Metadados, § 1–2)

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| R-01 | RFC | L7 | Contexto | Larissa como autora — abriria o doc de design da feature | TRANSCRICAO | `[09:50] Larissa` |
| R-02 | RFC | L10 | Contexto | Revisores = os 4 demais participantes da reunião | TRANSCRICAO | Cabeçalho de `TRANSCRICAO.md` (L7-12) |
| R-03 | RFC | L11 | Restrição | Sofia reserva 2 dias úteis de revisão de segurança pré-deploy | TRANSCRICAO | `[09:46] Sofia` |
| R-04 | RFC | L13 | Restrição | Prazo de 3 sprints; Atlas espera entrega até fim de novembro | TRANSCRICAO | `[09:45]`–`[09:47] Marcos, Larissa` |
| R-05 | RFC | L19 | Decisão | TL;DR consolidando as 6 decisões fechadas na reunião | TRANSCRICAO | `[09:48] Larissa` (resumo de fechamento) |
| R-06 | RFC | L21 | Restrição | Nenhuma infraestrutura nova; reuso de MySQL, Prisma, Pino, `AppError` | TRANSCRICAO | `[09:30] Larissa` |
| R-07 | RFC | L25 | Contexto | 3 clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pediram notificação | TRANSCRICAO | `[09:00] Marcos` |
| R-08 | RFC | L25 | Contexto | Atlas sinalizou risco de churn se não entregar até fim do trimestre | TRANSCRICAO | `[09:00] Marcos` |
| R-09 | RFC | L27 | RNF | Latência abaixo de 10s é considerada "tempo real" pelos clientes | TRANSCRICAO | `[09:02] Marcos` |
| R-10 | RFC | L27 | Restrição | Escopo estritamente outbound — plataforma envia, cliente recebe | TRANSCRICAO | `[09:02] Sofia, Marcos` |
| R-11 | RFC | L29 | Contexto | Transação de status já compõe orders + history + decremento de estoque | CODIGO | `src/modules/orders/order.service.ts:126` |
| R-12 | RFC | L29 | Fator | Síncrono travaria outros pedidos e forçaria rollback indevido | TRANSCRICAO | `[09:04] Bruno` |

### Proposta técnica (§ 3)

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| R-13 | RFC | L36-51, L55 | Decisão | Insert na outbox dentro da mesma transação Prisma do `changeStatus` | TRANSCRICAO | `[09:06] Diego` |
| R-14 | RFC | L55 | Decisão | Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx` | TRANSCRICAO | `[09:41] Bruno, Diego` |
| R-15 | RFC | L55 | Decisão | Payload gravado como snapshot na inserção, não renderizado no envio | TRANSCRICAO | `[09:52] Larissa, Diego` |
| R-16 | RFC | L55 | Decisão | Filtragem por status subscrito ocorre na inserção da outbox | TRANSCRICAO | `[09:34] Bruno, Diego` |
| R-17 | RFC | L57 | Decisão | Worker em processo separado: `src/worker.ts` + `npm run worker` | TRANSCRICAO | `[09:11] Larissa, Diego` |
| R-18 | RFC | L57 | Decisão | `PrismaClient` próprio no worker, mesma `DATABASE_URL` | TRANSCRICAO | `[09:30] Bruno` |
| R-19 | RFC | L57 | Decisão | Polling de 2s; MySQL não tem `NOTIFY/LISTEN` como o Postgres | TRANSCRICAO | `[09:09] Diego` |
| R-20 | RFC | L59 | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h (~15h de cobertura) | TRANSCRICAO | `[09:17] Diego, Larissa` |
| R-21 | RFC | L59 | RNF | Timeout de 10s por chamada HTTP; resposta mais lenta conta como falha | TRANSCRICAO | `[09:42] Diego` |
| R-22 | RFC | L59 | Decisão | DLQ em tabela separada `webhook_dead_letter` | TRANSCRICAO | `[09:18] Diego` |
| R-23 | RFC | L59 | RF | `POST /admin/webhooks/dead-letter/:id/replay` restrito a role ADMIN | TRANSCRICAO | `[09:35]`–`[09:36] Diego, Sofia` |
| R-24 | RFC | L61 | Decisão | HMAC-SHA256 sobre o corpo, com secret única por endpoint | TRANSCRICAO | `[09:22] Sofia` |
| R-25 | RFC | L61 | Decisão | Rotação de secret com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| R-26 | RFC | L61 | RNF | TLS obrigatório; URL `http://` rejeitada na validação Zod | TRANSCRICAO | `[09:23] Sofia` |
| R-27 | RFC | L61 | RNF | Limite de 64KB por payload, com erro acima do teto | TRANSCRICAO | `[09:24] Diego, Larissa` |
| R-28 | RFC | L61 | RF | Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` | TRANSCRICAO | `[09:44]`–`[09:45] Diego, Sofia` |
| R-29 | RFC | L63 | Decisão | Entrega at-least-once com dedup pelo `X-Event-Id` no cliente | TRANSCRICAO | `[09:26] Larissa` |
| R-30 | RFC | L63 | Contexto | Precedente de mercado: Stripe e GitHub adotam o mesmo padrão | TRANSCRICAO | `[09:25] Diego` |
| R-31 | RFC | L63 | Consequência | Marcos documenta a semântica no portal do desenvolvedor | TRANSCRICAO | `[09:26] Marcos` |
| R-32 | RFC | L65 | Decisão | Módulo `src/modules/webhooks` seguindo o padrão dos demais módulos | TRANSCRICAO | `[09:27] Bruno` |
| R-33 | RFC | L65 | Decisão | Prefixo `WEBHOOK_` nos códigos de erro; error middleware sem alteração | TRANSCRICAO | `[09:29] Larissa, Bruno` |
| R-34 | RFC | L65 | Decisão | IDs em UUID, seguindo o padrão do resto do projeto | TRANSCRICAO | `[09:51] Larissa` |
| R-35 | RFC | L69 | RF | CRUD de configuração (POST/PATCH/DELETE/GET) + `GET /webhooks/:id/deliveries` | TRANSCRICAO | `[09:33] Bruno`, `[09:34] Marcos` |
| R-36 | RFC | L69 | Decisão | `customer_id` vem do body ou path, **não** do JWT (JWT é do operador) | TRANSCRICAO | `[09:32] Larissa` |

### Alternativas consideradas (§ 4)

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| R-37 | RFC | L75 | Alternativa | Disparo síncrono no `changeStatus` descartado | TRANSCRICAO | `[09:04] Bruno` |
| R-38 | RFC | L76 | Alternativa | Redis Streams / fila externa descartada (overengineering) | TRANSCRICAO | `[09:07] Diego` |
| R-39 | RFC | L77 | Alternativa | Trigger de banco descartada (não notifica processo externo) | TRANSCRICAO | `[09:09] Diego` |
| R-40 | RFC | L78 | Alternativa | Worker embutido na API descartado (ciclo de vida acoplado) | TRANSCRICAO | `[09:11] Diego` |
| R-41 | RFC | L79 | Alternativa | 3 tentativas de retry descartadas (não cobrem 2h de manutenção) | TRANSCRICAO | `[09:16] Bruno` propôs, `Diego` descartou |
| R-42 | RFC | L80 | Alternativa | Retry indefinido descartado (evento pendurado para sempre) | TRANSCRICAO | `[09:15] Diego` |
| R-43 | RFC | L81 | Alternativa | DLQ como status `FAILED` na outbox descartada (polui a leitura) | TRANSCRICAO | `[09:18] Diego` |
| R-44 | RFC | L82 | Alternativa | Secret global descartada ("se vaza uma, vaza tudo") | TRANSCRICAO | `[09:21] Sofia`, `[09:22] Diego` |
| R-45 | RFC | L83 | Alternativa | TLS sem assinatura de payload descartado | TRANSCRICAO | `[09:19]`–`[09:20] Sofia` |
| R-46 | RFC | L84 | Alternativa | Exactly-once descartado (coordenação bidirecional) | TRANSCRICAO | `[09:25] Diego` |

### Questões em aberto e exclusões (§ 5)

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| R-47 | RFC | L92 | Ponto em Aberto | Rate limiting de saída — "observar e decidir depois" | TRANSCRICAO | `[09:38]`–`[09:39] Diego, Larissa` (adiado) |
| R-48 | RFC | L93 | Ponto em Aberto | Política de retenção da outbox (~30 dias, fora do escopo) | TRANSCRICAO | `[09:08] Diego` (não fechado) |
| R-49 | RFC | L94 | Ponto em Aberto | Política de retenção da DLQ | — | Lacuna — ver A3-12 |
| R-50 | RFC | L95 | Ponto em Aberto | Ordering com múltiplos workers — "problema do futuro" | TRANSCRICAO | `[09:12]`–`[09:13] Diego` (adiado) |
| R-51 | RFC | L96 | Ponto em Aberto | Recuperação de eventos travados em `PROCESSING` | — | Lacuna — ver A2-10, A3-14 |
| R-52 | RFC | L97 | Ponto em Aberto | Semântica do `attempt_count` após replay | — | Lacuna — ver A3-13 |
| R-53 | RFC | L98 | Ponto em Aberto | Entropia e política de exposição da secret | TRANSCRICAO | `[09:31] Marcos` (não fechado) — ver A4-12, A4-13 |
| R-54 | RFC | L99 | Ponto em Aberto | Comportamento após expirar o grace period de 24h | — | Lacuna — ver A4-14 |
| R-55 | RFC | L100 | Ponto em Aberto | Quem valida a janela temporal do `X-Timestamp` | TRANSCRICAO | `[09:44] Diego` (não fechado) — ver A4-15 |
| R-56 | RFC | L101 | Ponto em Aberto | Tipo do UUID: v4 vs. v7 | — | Lacuna — ver A5-09 |
| R-57 | RFC | L102 | Ponto em Aberto | `event_id` também no corpo do payload | TRANSCRICAO | `[09:43] Diego` (não fechado) — ver A5-10 |
| R-58 | RFC | L103 | Ponto em Aberto | Endurecimento de RBAC no CRUD de configuração | TRANSCRICAO | `[09:37] Sofia` (adiado) |
| R-59 | RFC | L105 | Exclusão | Email de aviso ao cliente fora de escopo desta fase | TRANSCRICAO | `[09:37] Larissa` |
| R-60 | RFC | L105 | Exclusão | Dashboard visual fora de escopo (projeto do time de frontend) | TRANSCRICAO | `[09:40] Larissa` |

### Impacto e riscos (§ 6)

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| R-61 | RFC | L111 | Integração | `changeStatus()` passa a inserir na outbox; falha no insert ⇒ rollback | TRANSCRICAO | `[09:40] Bruno`, `[09:41] Diego` |
| R-62 | RFC | L111 | Integração | Ponto exato da alteração no código existente | CODIGO | `src/modules/orders/order.service.ts:126` |
| R-63 | RFC | L113 | Consequência | Dois processos de longa duração em todos os ambientes | TRANSCRICAO | `[09:11] Diego` |
| R-64 | RFC | L115 | Consequência | 4 tabelas novas no banco OLTP, com índices em status e `created_at` | TRANSCRICAO | `[09:08] Diego` |
| R-65 | RFC | L115 | Integração | Tabelas `webhook_*` adicionadas ao schema | CODIGO | `prisma/schema.prisma` |
| R-66 | RFC | L117 | Consequência | Payload, headers e at-least-once viram contrato público | TRANSCRICAO | `[09:26] Marcos, Larissa` |
| R-67 | RFC | L123 | Risco | Regressão no `changeStatus` — caminho mais crítico do OMS | CODIGO | `src/modules/orders/order.service.ts` |
| R-68 | RFC | L123 | Mitigação | Meia sprint reservada para integração e testes ponta a ponta | TRANSCRICAO | `[09:46] Larissa` |
| R-69 | RFC | L124 | Risco | Crescimento das tabelas sem política de retenção definida | — | Derivado das lacunas R-48 e R-49 |
| R-70 | RFC | L125 | Risco | Prazo de 3 sprints vs. expectativa da Atlas e risco de churn | TRANSCRICAO | `[09:45] Marcos`, `[09:00] Marcos` |
| R-71 | RFC | L126 | Mitigação | Gate obrigatório de revisão de segurança antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| R-72 | RFC | L126 | Integração | Secrets adicionadas à lista de redação de campos sensíveis do Pino | CODIGO | `src/shared/logger/index.ts` |
| R-73 | RFC | L127 | Risco | Cliente não implementa deduplicação e processa duplicatas | TRANSCRICAO | `[09:25] Sofia` |
| R-74 | RFC | L128 | Risco | Bombardeio do cliente sem rate limiting de saída | TRANSCRICAO | `[09:38] Diego` |
| R-75 | RFC | L129 | Risco | Eventos órfãos em `PROCESSING` após crash do worker | — | Lacuna — ver A2-10 |

### Decisões relacionadas (§ 7)

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| R-76 | RFC | L135-139 | Rastreabilidade | Links para os 5 ADRs correspondentes | — | Documentos internos — `docs/adrs/` |
| R-77 | RFC | L141 | Justificativa | "Reuso de padrões" não virou ADR isolada — sem alternativas em disputa | TRANSCRICAO | `[09:30] Larissa` |

---

## Rastreabilidade — FDD

### Contexto, objetivos e escopo (§ 1–3)

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| F-01 | FDD | L5 | Contexto | Larissa como responsável; Bruno, Diego e Sofia como revisores | TRANSCRICAO | `[09:50] Larissa` |
| F-02 | FDD | L6 | Restrição | Gate de 2 dias úteis de revisão de segurança antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| F-03 | FDD | L17 | Contexto | 3 clientes B2B fazem polling contra `GET /orders`; Atlas com risco de churn | TRANSCRICAO | `[09:00] Marcos` |
| F-04 | FDD | L19 | Contexto | `changeStatus` já compõe transição, estoque, update e histórico numa transação | CODIGO | `src/modules/orders/order.service.ts:126` |
| F-05 | FDD | L19 | Fator | HTTP síncrono na transação travaria outros pedidos e forçaria rollback | TRANSCRICAO | `[09:04] Bruno` |
| F-06 | FDD | L21 | Decisão | Outbox no MySQL sem nova infraestrutura | TRANSCRICAO | `[09:07] Diego` |
| F-07 | FDD | L21 | Decisão | Módulo `src/modules/webhooks` seguindo o padrão dos demais | TRANSCRICAO | `[09:27] Bruno` |
| F-08 | FDD | L21 | Decisão | `src/worker.ts` espelhando `src/server.ts` | TRANSCRICAO | `[09:11] Larissa` |
| F-09 | FDD | L25-31 | Contexto | Atores: produtor, worker, endpoint do cliente, usuário JWT, ADMIN | TRANSCRICAO | `[09:36] Sofia`, `[09:37] Marcos` |
| F-10 | FDD | L33 | Restrição | Escopo estritamente outbound | TRANSCRICAO | `[09:02] Sofia, Marcos` |
| F-11 | FDD | L37 | Restrição | Falha no insert da outbox faz rollback de tudo | TRANSCRICAO | `[09:40] Bruno` |
| F-12 | FDD | L38 | RNF | Latência abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| F-13 | FDD | L39 | Restrição | Worker obrigatoriamente em processo separado | TRANSCRICAO | `[09:11] Diego` |
| F-14 | FDD | L41 | Suposição | `PrismaClient` próprio no worker, mesma `DATABASE_URL` | TRANSCRICAO | `[09:30] Bruno` |
| F-15 | FDD | L49 | Objetivo | Atomicidade verificável por teste de rollback forçado | TRANSCRICAO | `[09:06] Diego` |
| F-16 | FDD | L52 | Objetivo | `event_id` gerado uma única vez e imutável entre tentativas | TRANSCRICAO | `[09:25] Diego` |
| F-17 | FDD | L56 | Objetivo | Snapshot do payload na inserção | TRANSCRICAO | `[09:52] Larissa` |
| F-18 | FDD | L57 | Objetivo | Zero dependência nova; HMAC via `node:crypto` nativo | CODIGO | `package.json` (dependencies) |
| F-19 | FDD | L64 | Escopo | Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` | TRANSCRICAO | `[09:41] Bruno, Diego` |
| F-20 | FDD | L65 | Escopo | Filtragem por status subscrito na inserção | TRANSCRICAO | `[09:34] Bruno` |
| F-21 | FDD | L68-73 | Escopo | Worker, HMAC, retry/DLQ, CRUD, TLS e 64KB inclusos | TRANSCRICAO | `[09:48] Larissa` (resumo de fechamento) |
| F-22 | FDD | L77 | Exclusão | Email de aviso fora desta fase | TRANSCRICAO | `[09:37] Larissa` |
| F-23 | FDD | L78 | Exclusão | Dashboard visual fora de escopo | TRANSCRICAO | `[09:40] Larissa` |
| F-24 | FDD | L79 | Exclusão | Rate limiting de saída adiado | TRANSCRICAO | `[09:39] Diego, Larissa` |
| F-25 | FDD | L80 | Exclusão | Webhooks inbound fora do escopo | TRANSCRICAO | `[09:02] Sofia` |
| F-26 | FDD | L81 | Exclusão | Escala horizontal do worker adiada | TRANSCRICAO | `[09:13] Diego` |
| F-27 | FDD | L82 | Exclusão | Arquivamento de eventos entregues fora do escopo | TRANSCRICAO | `[09:08] Diego` |
| F-28 | FDD | L83 | Exclusão | Endurecimento de RBAC adiado | TRANSCRICAO | `[09:37] Sofia` |
| F-29 | FDD | L84 | Exclusão | `items` fora do payload para não inflar | TRANSCRICAO | `[09:43] Diego` |

### Fluxos e diagramas (§ 4)

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| F-30 | FDD | L92-99 | Fluxo | Passos existentes de `changeStatus` preservados sem alteração | CODIGO | `src/modules/orders/order.service.ts:126-179` |
| F-31 | FDD | L100 | Fluxo | Novo passo: `publishWebhookEvent(tx, ...)` com o mesmo `tx` | TRANSCRICAO | `[09:41] Bruno` |
| F-32 | FDD | L102 | Fluxo | Resultado vazio na filtragem não insere nada | TRANSCRICAO | `[09:34] Bruno` |
| F-33 | FDD | L103 | Fluxo | Insert com `PENDING`, `event_id`, `attempt_count = 0` | TRANSCRICAO | `[09:06] Diego`, `[09:25] Diego` |
| F-34 | FDD | L104 | Fluxo | Rollback total em qualquer falha | TRANSCRICAO | `[09:40] Bruno` |
| F-35 | FDD | L110 | Fluxo | Polling a cada 2s de `PENDING` ordenado por `created_at` | TRANSCRICAO | `[09:09] Diego`, `[09:08] Diego` |
| F-36 | FDD | L112 | Fluxo | Assinatura HMAC-SHA256 e POST com timeout de 10s | TRANSCRICAO | `[09:42] Diego` |
| F-37 | FDD | L113 | Fluxo | Sucesso 2xx grava em `webhook_deliveries` com tempo de resposta | TRANSCRICAO | `[09:34] Marcos` |
| F-38 | FDD | L118 | Fluxo | Falha agenda `next_attempt_at` pelo backoff e volta a `PENDING` | TRANSCRICAO | `[09:17] Diego` |
| F-39 | FDD | L119 | Fluxo | Esgotadas as 5 tentativas, move para `webhook_dead_letter` | TRANSCRICAO | `[09:18] Diego` |
| F-40 | FDD | L120 | Fluxo | Replay por ADMIN devolve a `PENDING` e audita o executor | TRANSCRICAO | `[09:18] Diego`, `[09:36] Sofia` |
| F-41 | FDD | L121 | Fluxo | Payload acima de 64KB vira erro, sem truncar | TRANSCRICAO | `[09:23]`–`[09:24] Sofia, Diego` |
| F-42 | FDD | L123 | Fluxo | Rotação: worker assina com a secret vigente | TRANSCRICAO | `[09:21] Sofia` |
| F-43 | FDD | L124 | Fluxo | Shutdown gracioso devolve `PROCESSING` a `PENDING` | TRANSCRICAO | `[09:11] Diego` (derivado) |
| F-44 | FDD | L125 | Bloqueador | Recuperação de `PROCESSING` em SIGKILL ou crash não definida | — | Lacuna — ver A2-10, A3-14 |
| F-45 | FDD | L129-158 | Diagrama | Sequência do fluxo principal (Mermaid) | TRANSCRICAO | Derivado de `[09:06]`–`[09:18]` |
| F-46 | FDD | L162-172 | Diagrama | Estados do evento na outbox (Mermaid) | TRANSCRICAO | Derivado de `[09:08]`, `[09:17]`, `[09:18]` |

### Contratos públicos (§ 5)

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| F-47 | FDD | L178 | Integração | Base path `/api/v1` e `authenticate` nos endpoints de configuração | CODIGO | `src/app.ts`, `src/middlewares/auth.middleware.ts:27` |
| F-48 | FDD | L178 | Decisão | `customerId` vem do body ou path, não do JWT | TRANSCRICAO | `[09:32] Larissa` |
| F-49 | FDD | L180-184 | Contrato | Assinatura de `publishWebhookEvent` recebendo `tx` | TRANSCRICAO | `[09:41] Bruno, Diego` |
| F-50 | FDD | L186-219 | Contrato | `POST /api/v1/webhooks`; secret gerada e devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| F-51 | FDD | L190 | RNF | URL não https rejeitada com `WEBHOOK_INVALID_URL` | TRANSCRICAO | `[09:23] Sofia` |
| F-52 | FDD | L218 | Bloqueador | Comprimento e entropia da secret não definidos | — | Lacuna — ver A4-12 |
| F-53 | FDD | L218 | Bloqueador | Exposição da secret pós-criação não definida | — | Lacuna — ver A4-13 |
| F-54 | FDD | L221-235 | Contrato | `GET`, `PATCH` e `DELETE` de webhooks | TRANSCRICAO | `[09:33] Bruno` |
| F-55 | FDD | L237-251 | Contrato | `POST /webhooks/:id/rotate-secret` com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| F-56 | FDD | L253-277 | Contrato | `GET /webhooks/:id/deliveries` com payload, resposta e duração | TRANSCRICAO | `[09:34] Marcos` |
| F-57 | FDD | L279-299 | Contrato | `POST /admin/webhooks/dead-letter/:id/replay` restrito a ADMIN | TRANSCRICAO | `[09:35] Diego`, `[09:36] Sofia` |
| F-58 | FDD | L283 | Integração | Reuso de `requireRole('ADMIN')` sem modificação | CODIGO | `src/middlewares/auth.middleware.ts:49` |
| F-59 | FDD | L299 | Bloqueador | `attempt_count` após replay não definido | — | Lacuna — ver A3-13 |
| F-60 | FDD | L301-354 | Contrato | Evento outbound: POST em URL https do cliente | TRANSCRICAO | `[09:23] Sofia` |
| F-61 | FDD | L309-315 | Contrato | Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` | TRANSCRICAO | `[09:44] Diego`, `[09:45] Sofia` |
| F-62 | FDD | L319-331 | Contrato | Campos do payload do evento | TRANSCRICAO | `[09:43] Diego` |
| F-63 | FDD | L334 | Bloqueador | `event_id` no corpo além do header não confirmado | — | Lacuna — ver A5-10 |
| F-64 | FDD | L346-352 | RNF | Limites: timeout 10s, 64KB, sem rate limit, latência alvo | TRANSCRICAO | `[09:42]`, `[09:24]`, `[09:39]`, `[09:02]` |
| F-65 | FDD | L352 | Consequência | Payload e at-least-once viram contrato público versionado | TRANSCRICAO | `[09:26] Larissa` |

### Erros, resiliência e observabilidade (§ 6–7)

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| F-66 | FDD | L360 | Integração | Erros estendem `AppError` com prefixo `WEBHOOK_` | CODIGO | `src/shared/errors/app-error.ts` |
| F-67 | FDD | L360 | Integração | `errorMiddleware` formata `AppError` sem alteração | CODIGO | `src/middlewares/error.middleware.ts:14` |
| F-68 | FDD | L364-370 | Matriz | 7 códigos de erro previstos, 3 citados literalmente na reunião | TRANSCRICAO | `[09:28] Bruno`, `[09:29] Larissa` |
| F-69 | FDD | L373 | Bloqueador | Comportamento pós-grace period não definido | — | Lacuna — ver A4-14 |
| F-70 | FDD | L377-384 | Matriz | Falhas de entrega no worker alimentam o retry | TRANSCRICAO | `[09:42] Diego`, `[09:18] Diego` |
| F-71 | FDD | L388-392 | Resiliência | Timeout 10s, 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:42]`, `[09:16]`, `[09:17]` |
| F-72 | FDD | L391 | Resiliência | Circuit breaker não discutido e fora desta fase | — | Lacuna — não discutido na reunião |
| F-73 | FDD | L396 | Fallback | Sem canal alternativo; DLQ é o fallback efetivo | TRANSCRICAO | `[09:37] Larissa`, `[09:18] Diego` |
| F-74 | FDD | L400-404 | Invariante | 5 invariantes críticos da feature | TRANSCRICAO | `[09:06]`, `[09:25]`, `[09:52]`, `[09:12]` |
| F-75 | FDD | L408 | Contexto | Projeto não tem métricas, tracing, alertas nem APM | CODIGO | Busca em `src/` e `package.json` sem resultado |
| F-76 | FDD | L414 | Integração | Worker importa o mesmo logger Pino, sem lib nova | CODIGO | `src/shared/logger/index.ts:13` |
| F-77 | FDD | L415 | Integração | `base: { service, env }` no logger | CODIGO | `src/shared/logger/index.ts:20` |
| F-78 | FDD | L416 | Integração | Estender `redactPaths` com secrets de webhook | CODIGO | `src/shared/logger/index.ts:4-11` |
| F-79 | FDD | L417 | Integração | Worker respeita `LOG_LEVEL` | CODIGO | `src/config/env.ts:6` |
| F-80 | FDD | L418 | Integração | Correlação por `X-Request-Id` nos endpoints CRUD | CODIGO | `src/middlewares/request-logger.middleware.ts:6-8` |
| F-81 | FDD | L419 | Integração | Log `http_request` com `durationMs` aplicado sem alteração | CODIGO | `src/middlewares/request-logger.middleware.ts:14-24` |
| F-82 | FDD | L420 | Integração | Worker espelha o padrão do endpoint `/health` | CODIGO | `src/app.ts:62` |
| F-83 | FDD | L426-436 | Observabilidade | Campos de log do worker, com `eventId` correlacionando tentativas | TRANSCRICAO | `[09:29] Bruno` (reuso do Pino) |
| F-84 | FDD | L438 | Observabilidade | Eventos de log nomeados; replay loga o executor | TRANSCRICAO | `[09:36] Sofia` |
| F-85 | FDD | L440 | Segurança | Secret e `X-Signature` nunca em log | TRANSCRICAO | `[09:22] Diego` |
| F-86 | FDD | L446 | Bloqueador | Stack de métricas não definida | — | Lacuna — ausente do código e da reunião |
| F-87 | FDD | L452 | Bloqueador | Stack de tracing não definida | — | Lacuna — ausente do código e da reunião |
| F-88 | FDD | L458 | Bloqueador | Painéis e alertas não especificáveis | — | Lacuna — dependente de F-86 |

### Dependências, aceite e riscos (§ 8–10)

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| F-89 | FDD | L467-477 | Dependência | Versões reais: Node >=20, TS 5.6.3, Prisma 5.22.0, Pino 9.5.0, Zod 3.23.8, uuid 11.0.3, Express 4.21.1 | CODIGO | `package.json` |
| F-90 | FDD | L469 | Dependência | MySQL 8.0 sem `NOTIFY/LISTEN` motiva o polling | TRANSCRICAO | `[09:09] Diego` |
| F-91 | FDD | L477 | Dependência | `node:crypto` nativo para HMAC, sem dependência nova | CODIGO | `package.json` (ausência de lib de cripto) |
| F-92 | FDD | L479 | Decisão | Nenhuma dependência nova é necessária | TRANSCRICAO | `[09:07] Diego`, `[09:30] Larissa` |
| F-93 | FDD | L481 | Bloqueador | UUID v4 ou v7 não definido | — | Lacuna — ver A5-09 |
| F-94 | FDD | L485-489 | Compatibilidade | Nenhuma rota atual muda; schema só acrescenta; API roda sem o worker | CODIGO | `prisma/schema.prisma`, `src/app.ts` |
| F-95 | FDD | L495-502 | Aceite | 8 critérios funcionais verificáveis | TRANSCRICAO | `[09:34]`, `[09:40]`, `[09:52]`, `[09:36]`, `[09:23]` |
| F-96 | FDD | L506-508 | Aceite | 3 critérios de performance e latência | TRANSCRICAO | `[09:02]`, `[09:04]` |
| F-97 | FDD | L512-521 | Aceite | 9 critérios de resiliência | TRANSCRICAO | `[09:17]`, `[09:42]`, `[09:18]`, `[09:21]`, `[09:11]`, `[09:12]` |
| F-98 | FDD | L525-528 | Aceite | 4 critérios de observabilidade; o 24 depende de F-86 | TRANSCRICAO | `[09:36] Sofia` |
| F-99 | FDD | L532 | Aceite | Gate de processo: revisão de segurança concluída | TRANSCRICAO | `[09:46] Sofia` |
| F-100 | FDD | L539-551 | Bloqueador | Tabela consolidada de 13 pendências bloqueantes | — | 10 lacunas + 3 adiados; ver ADRs |
| F-101 | FDD | L559-569 | Risco | Regressão no `changeStatus`; mitigação com `tx` e meia sprint de testes | TRANSCRICAO | `[09:41] Diego`, `[09:46] Larissa` |
| F-102 | FDD | L569 | Contingência | Desativar endpoints reduz `publishWebhookEvent` a consulta vazia | TRANSCRICAO | `[09:34] Bruno` (derivado) |
| F-103 | FDD | L571-579 | Risco | Crescimento das tabelas sem política de retenção | TRANSCRICAO | `[09:08] Diego` |
| F-104 | FDD | L581-589 | Risco | Prazo de 3 sprints contra a expectativa da Atlas | TRANSCRICAO | `[09:00] Marcos`, `[09:46] Larissa` |
| F-105 | FDD | L591-601 | Risco | Falha de segurança em HMAC ou geração de secret | TRANSCRICAO | `[09:22] Diego`, `[09:46] Sofia` |
| F-106 | FDD | L603-611 | Risco | Cliente não implementa deduplicação | TRANSCRICAO | `[09:25] Sofia` |
| F-107 | FDD | L613-620 | Risco | Eventos órfãos em `PROCESSING`; mitigação insuficiente | — | Lacuna — ver F-44 |
| F-108 | FDD | L622-630 | Risco | Bombardeio do cliente sem rate limiting | TRANSCRICAO | `[09:38] Diego` |

### Integração com o sistema existente (§ 11, seção obrigatória do desafio)

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| F-109 | FDD | L637-643 | Integração | Um único passo acrescentado após `tx.orderStatusHistory.create` | CODIGO | `src/modules/orders/order.service.ts:126` |
| F-110 | FDD | L645-655 | Integração | Erros `WEBHOOK_*` seguindo o padrão de `InvalidStatusTransitionError` | CODIGO | `src/shared/errors/http-errors.ts` |
| F-111 | FDD | L657 | Integração | `errorMiddleware` captura os novos erros sem alteração | CODIGO | `src/middlewares/error.middleware.ts:14` |
| F-112 | FDD | L661 | Integração | `authenticate` e `requireRole` reusados; router segue o padrão de orders | CODIGO | `src/middlewares/auth.middleware.ts:27`, `src/modules/orders/order.routes.ts:14` |
| F-113 | FDD | L665 | Integração | Única alteração em `src/shared/`: estender `redactPaths` | CODIGO | `src/shared/logger/index.ts:4-11` |
| F-114 | FDD | L669 | Integração | `worker.ts` espelha `server.ts`; script `worker` análogo a `dev`/`start` | CODIGO | `src/server.ts`, `package.json` |
| F-115 | FDD | L673-680 | Integração | 4 modelos novos seguindo `@id @default(uuid()) @db.Char(36)` e `@@map` | CODIGO | `prisma/schema.prisma` |
| F-116 | FDD | L682 | Integração | Rotas sob `/api/v1`; `requestLogger` cobre os novos endpoints | CODIGO | `src/app.ts:66`, `src/app.ts:59` |
| F-117 | FDD | L686 | Integração | Testes seguem o padrão da suíte existente contra MySQL real | CODIGO | `tests/setup.ts`, `tests/helpers/factories.ts`, `tests/orders.test.ts` |

---

## Rastreabilidade — PRD

### Resumo, contexto e problema

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| P-01 | PRD | L5 | Contexto | Marcos como responsável pelo PRD | TRANSCRICAO | `[09:00]`, `[09:31]`, `[09:26]`, `[09:47] Marcos` |
| P-02 | PRD | L16 | Contexto | 3 clientes B2B pediram notificação em tempo real | TRANSCRICAO | `[09:00] Marcos` |
| P-03 | PRD | L16 | Contexto | Atlas pode migrar para o concorrente até o fim do trimestre | TRANSCRICAO | `[09:00] Marcos` |
| P-04 | PRD | L18 | Resumo | Webhooks de saída com POST assinado em menos de 10s | TRANSCRICAO | `[09:02] Marcos`, `[09:48] Larissa` |
| P-05 | PRD | L20 | Restrição | Escopo estritamente de saída; sem painel nem email | TRANSCRICAO | `[09:02] Sofia`, `[09:37]`, `[09:40] Larissa` |
| P-06 | PRD | L28 | Público | Clientes B2B: Atlas, MaxDistribuição e Nova Cargo | TRANSCRICAO | `[09:00] Marcos` |
| P-07 | PRD | L29 | Público | Times de dev dos clientes, responsáveis por assinatura e dedup | TRANSCRICAO | `[09:25] Diego, Sofia` |
| P-08 | PRD | L30 | Público | Operação interna, que diagnostica e reprocessa falhas | TRANSCRICAO | `[09:18] Diego` |
| P-09 | PRD | L34 | Cenário | Pedido muda para SHIPPED e o cliente é notificado em segundos | TRANSCRICAO | `[09:00] Marcos` |
| P-10 | PRD | L35 | Cenário | Cliente assina apenas SHIPPED e DELIVERED | TRANSCRICAO | `[09:34] Marcos` |
| P-11 | PRD | L36 | Cenário | Endpoint fora por 2h de manutenção recebe tudo ao voltar | TRANSCRICAO | `[09:16] Diego` |
| P-12 | PRD | L37 | Cenário | Cliente vazou secret e rotaciona com 24h de migração | TRANSCRICAO | `[09:21] Sofia`, `[09:22] Diego` |
| P-13 | PRD | L38 | Cenário | Cliente com vários endpoints identifica qual recebeu o envio | TRANSCRICAO | `[09:45] Sofia` |
| P-14 | PRD | L39 | Cenário | Suporte consulta histórico de entregas para diagnóstico | TRANSCRICAO | `[09:34] Marcos` |
| P-15 | PRD | L40 | Cenário | ADMIN reprocessa evento após o cliente voltar | TRANSCRICAO | `[09:18] Diego` |
| P-16 | PRD | L44 | Implantação | Sistema existente: OMS em Node, TS, Express, Prisma e MySQL | CODIGO | `package.json`, `prisma/schema.prisma` |
| P-17 | PRD | L45 | Implantação | Aplicação não tem hoje notificação externa, eventos nem filas | CODIGO | Ausência verificada em `src/` |
| P-18 | PRD | L46 | Implantação | Segundo processo em todos os ambientes; nenhuma infra nova | TRANSCRICAO | `[09:11] Diego`, `[09:07] Diego` |
| P-19 | PRD | L50 | Problema | Polling ineficiente dos clientes contra `GET /orders`; prioridade alta | TRANSCRICAO | `[09:00] Marcos` |
| P-20 | PRD | L51 | Problema | Risco comercial de churn da Atlas; prioridade alta | TRANSCRICAO | `[09:00] Marcos` |
| P-21 | PRD | L52 | Problema | Atualização manual do lado do cliente; prioridade alta | TRANSCRICAO | `[09:02] Marcos` |
| P-22 | PRD | L53 | Problema | Ausência de canal de eventos na plataforma; prioridade média | CODIGO | Ausência verificada em `src/` |

### Objetivos, métricas e escopo

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| P-23 | PRD | L61 | Métrica | Latência commit até primeiro POST abaixo de 10s; alvo de 2s | TRANSCRICAO | `[09:02] Marcos`, `[09:09] Diego` |
| P-24 | PRD | L62 | Métrica | 3 clientes B2B migrados para webhooks | TRANSCRICAO | `[09:00] Marcos` |
| P-25 | PRD | L63 | Métrica | Entrega até o fim de novembro, dentro do trimestre | TRANSCRICAO | `[09:00]`, `[09:45] Marcos` |
| P-26 | PRD | L64 | Métrica | Latência de `PATCH /orders/:id/status` sem aumento sobre o baseline | TRANSCRICAO | `[09:04] Bruno` |
| P-27 | PRD | L65 | Métrica | Janela de ~15h de indisponibilidade do cliente coberta | TRANSCRICAO | `[09:17] Diego` |
| P-28 | PRD | L66 | Métrica | 3 sprints incluindo a revisão de segurança | TRANSCRICAO | `[09:46]`, `[09:47] Larissa` |
| P-29 | PRD | L68 | Pendência | Sem meta de entrega sem intervenção manual nem de volume na DLQ | — | Lacuna — não discutido |
| P-30 | PRD | L76-86 | Escopo | 11 itens inclusos, todos rastreados à reunião | TRANSCRICAO | `[09:48] Larissa` (resumo de fechamento) |
| P-31 | PRD | L90 | Exclusão | Email de aviso fora desta fase | TRANSCRICAO | `[09:37] Larissa` |
| P-32 | PRD | L91 | Exclusão | Dashboard visual fora de escopo | TRANSCRICAO | `[09:40] Larissa` |
| P-33 | PRD | L92 | Exclusão | Rate limiting de saída adiado | TRANSCRICAO | `[09:38]`–`[09:39] Diego, Larissa` |
| P-34 | PRD | L93 | Exclusão | Webhooks inbound fora do escopo | TRANSCRICAO | `[09:02] Sofia, Marcos` |
| P-35 | PRD | L94 | Exclusão | Escala horizontal do worker adiada | TRANSCRICAO | `[09:13] Diego` |
| P-36 | PRD | L95 | Exclusão | Arquivamento de eventos entregues fora do escopo | TRANSCRICAO | `[09:08] Diego` |
| P-37 | PRD | L96 | Exclusão | `items` fora do payload | TRANSCRICAO | `[09:43] Diego` |
| P-38 | PRD | L97 | Exclusão | Endurecimento de RBAC adiado | TRANSCRICAO | `[09:37] Sofia` |

### Requisitos funcionais

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| P-39 | PRD | L105-130 | RF-001 | Cadastrar endpoint: URL, lista de status, secret gerada; prioridade alta | TRANSCRICAO | `[09:31] Marcos` |
| P-40 | PRD | L115 | RF-001 | `customerId` no body ou path, não no JWT | TRANSCRICAO | `[09:32] Larissa` |
| P-41 | PRD | L116 | RF-001 | Qualquer role autenticada cadastra nesta fase | TRANSCRICAO | `[09:37] Sofia` |
| P-42 | PRD | L122 | RF-001 | URL sem https recusada antes de persistir | TRANSCRICAO | `[09:23] Sofia` |
| P-43 | PRD | L132-154 | RF-002 | Listar, editar e remover endpoints; prioridade alta | TRANSCRICAO | `[09:33] Bruno` |
| P-44 | PRD | L156-177 | RF-003 | Filtro de eventos por status, aplicado na inserção; prioridade alta | TRANSCRICAO | `[09:33] Marcos`, `[09:34] Bruno` |
| P-45 | PRD | L179-202 | RF-004 | Gerar e rotacionar secret com grace period de 24h; prioridade alta | TRANSCRICAO | `[09:21] Sofia`, `[09:31] Marcos` |
| P-46 | PRD | L191 | RF-004 | Secret única por endpoint, não global | TRANSCRICAO | `[09:21] Sofia` |
| P-47 | PRD | L199 | Pendência | Comportamento do cliente com secret anterior após 24h | — | Lacuna — ver A4-14 |
| P-48 | PRD | L204-228 | RF-005 | Publicar evento atômico com a mudança de status; prioridade alta | TRANSCRICAO | `[09:06] Diego`, `[09:40] Bruno` |
| P-49 | PRD | L216 | RF-005 | Falha no registro desfaz a mudança de status | TRANSCRICAO | `[09:40] Bruno` |
| P-50 | PRD | L217 | RF-005 | Snapshot do payload no instante da transição | TRANSCRICAO | `[09:52] Larissa` |
| P-51 | PRD | L230-256 | RF-006 | Entregar evento assinado; polling de 2s; prioridade alta | TRANSCRICAO | `[09:09] Diego`, `[09:20] Sofia` |
| P-52 | PRD | L241 | RF-006 | Entrega at-least-once com dedup pelo cliente | TRANSCRICAO | `[09:24]`–`[09:25] Diego` |
| P-53 | PRD | L244 | RF-006 | Precedente de mercado: Stripe e GitHub | TRANSCRICAO | `[09:25] Diego` |
| P-54 | PRD | L245 | RF-006 | Ordenação por pedido garantida com worker único | TRANSCRICAO | `[09:12]`, `[09:14] Diego, Marcos` |
| P-55 | PRD | L250 | RF-006 | Evento acima de 64KB vira erro, sem truncar | TRANSCRICAO | `[09:23]`–`[09:24] Sofia, Diego` |
| P-56 | PRD | L251 | RF-006 | Timeout de 10s tratado como falha | TRANSCRICAO | `[09:42] Diego` |
| P-57 | PRD | L258-281 | RF-007 | 5 tentativas em backoff 1m/5m/30m/2h/12h; prioridade alta | TRANSCRICAO | `[09:17] Diego` |
| P-58 | PRD | L269 | RF-007 | 3 tentativas recusadas por não cobrir 2h de manutenção | TRANSCRICAO | `[09:16] Diego` |
| P-59 | PRD | L270 | RF-007 | Retry indefinido recusado | TRANSCRICAO | `[09:15] Diego` |
| P-60 | PRD | L271 | RF-007 | Janela de 15h validada pelo negócio | TRANSCRICAO | `[09:17] Marcos` |
| P-61 | PRD | L283-303 | RF-008 | DLQ em estrutura separada com payload, motivo e horário; prioridade alta | TRANSCRICAO | `[09:18] Diego` |
| P-62 | PRD | L301 | Pendência | Política de retenção da DLQ | — | Lacuna — ver A3-12 |
| P-63 | PRD | L305-329 | RF-009 | Replay manual restrito a ADMIN e auditado; prioridade média | TRANSCRICAO | `[09:35] Diego`, `[09:36] Sofia` |
| P-64 | PRD | L317 | RF-009 | Reuso do controle de acesso por papel existente | CODIGO | `src/middlewares/auth.middleware.ts:49` |
| P-65 | PRD | L327 | Pendência | Contador de tentativas após replay | — | Lacuna — ver A3-13 |
| P-66 | PRD | L331-351 | RF-010 | Histórico de entregas com resposta e duração; prioridade média | TRANSCRICAO | `[09:34] Marcos` |

### Requisitos não funcionais

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| P-67 | PRD | L357 | RNF | Latência abaixo de 10s: definição de tempo real dada pelos clientes | TRANSCRICAO | `[09:02] Marcos` |
| P-68 | PRD | L358 | RNF | Polling de 2s estabelece a latência de pior caso | TRANSCRICAO | `[09:09]`, `[09:10] Diego, Larissa` |
| P-69 | PRD | L359 | RNF | Timeout de 10s por chamada | TRANSCRICAO | `[09:42] Diego` |
| P-70 | PRD | L360 | RNF | Mudança de status não pode ficar mais lenta de forma mensurável | TRANSCRICAO | `[09:04] Bruno` |
| P-71 | PRD | L361 | RNF | Máximo de 64KB por evento | TRANSCRICAO | `[09:24] Diego` |
| P-72 | PRD | L362 | Pendência | Sem meta de p95 para os endpoints de configuração | — | Lacuna — não discutido |
| P-73 | PRD | L366 | RNF | Tolerância a ~15h de indisponibilidade do cliente | TRANSCRICAO | `[09:17] Diego` |
| P-74 | PRD | L367 | RNF | Reinício da API não interrompe o processamento | TRANSCRICAO | `[09:11] Diego` |
| P-75 | PRD | L368 | RNF | API opera normalmente sem o worker; eventos acumulam sem perda | TRANSCRICAO | `[09:06] Diego` (derivado) |
| P-76 | PRD | L369 | Pendência | Sem meta de disponibilidade da plataforma; default de 99.9% não adotado | — | Lacuna — não discutido |
| P-77 | PRD | L373-381 | RNF | Segurança: HMAC-SHA256, secret por endpoint, rotação, TLS, RBAC, auditoria | TRANSCRICAO | `[09:20]`–`[09:23]`, `[09:36] Sofia` |
| P-78 | PRD | L379 | RNF | Nenhuma secret em log; motivado por caso real | TRANSCRICAO | `[09:22] Diego` |
| P-79 | PRD | L380 | RNF | Revisão de segurança obrigatória de 2 dias úteis | TRANSCRICAO | `[09:46] Sofia` |
| P-80 | PRD | L381 | Pendência | Entropia e exposição da secret | — | Lacuna — ver A4-12, A4-13 |
| P-81 | PRD | L385-388 | RNF | Observabilidade: log estruturado com `eventId`, reuso do Pino, redação | CODIGO | `src/shared/logger/index.ts` |
| P-82 | PRD | L386 | RNF | Nenhuma biblioteca de log nova | TRANSCRICAO | `[09:29] Bruno` |
| P-83 | PRD | L389 | Pendência | Sem métricas nem tracing; defaults do prompt não adotados | — | Lacuna — ausente do código e da reunião |
| P-84 | PRD | L393 | RNF | Registro do evento na mesma transação; sem inconsistência possível | TRANSCRICAO | `[09:06] Diego` |
| P-85 | PRD | L394 | RNF | Controle transacional de estoque permanece intacto | CODIGO | `src/modules/orders/order.service.ts` |
| P-86 | PRD | L397 | RNF | At-least-once com identificador estável | TRANSCRICAO | `[09:24]`–`[09:25] Diego` |
| P-87 | PRD | L401-404 | RNF | Compatibilidade: nenhuma rota muda; nenhuma infra nova; padrões reusados | TRANSCRICAO | `[09:07]`, `[09:30] Diego, Larissa` |
| P-88 | PRD | L408 | RNF | Trilha de auditoria de replay | TRANSCRICAO | `[09:36] Sofia` |
| P-89 | PRD | L410 | Pendência | Sem discussão sobre requisitos regulatórios de envio de dados | — | Lacuna — não discutido |
| P-90 | PRD | L414 | RNF | Acessibilidade não aplicável: sem interface visual nesta fase | TRANSCRICAO | `[09:40] Larissa` |

### Arquitetura, decisões e dependências

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| P-91 | PRD | L422-426 | Arquitetura | Feature no sistema existente; comunicação assíncrona; sem fila nem cache | TRANSCRICAO | `[09:06]`, `[09:07] Diego` |
| P-92 | PRD | L430-434 | Componentes | Módulo de webhooks, ponto de publicação, worker, 4 tabelas, MySQL | TRANSCRICAO | `[09:27] Bruno`, `[09:11] Larissa` |
| P-93 | PRD | L438-440 | Integrações | Endpoints dos clientes sobre TLS; serviço de pedidos; sem terceiros | TRANSCRICAO | `[09:23] Sofia`, `[09:40] Bruno` |
| P-94 | PRD | L444-447 | Decisão | Outbox no banco, na mesma transação; trade-off de arquivamento | TRANSCRICAO | `[09:06]`, `[09:08] Diego` |
| P-95 | PRD | L449-452 | Decisão | Worker separado com polling de 2s; trade-off de 2 processos | TRANSCRICAO | `[09:09]`, `[09:11] Diego` |
| P-96 | PRD | L454-457 | Decisão | 5 tentativas e DLQ; trade-off de 15h na fila | TRANSCRICAO | `[09:15]`–`[09:17] Diego` |
| P-97 | PRD | L459-462 | Decisão | HMAC-SHA256 por endpoint; trade-off de secret sob guarda do cliente | TRANSCRICAO | `[09:20]`–`[09:22] Sofia, Diego` |
| P-98 | PRD | L464-467 | Decisão | At-least-once; trade-off de responsabilidade transferida | TRANSCRICAO | `[09:25]`–`[09:26] Diego, Sofia` |
| P-99 | PRD | L469-472 | Decisão | Filtragem no registro; trade-off de não afetar eventos já criados | TRANSCRICAO | `[09:34] Bruno, Diego` |
| P-100 | PRD | L474-477 | Decisão | Reuso máximo dos padrões; sem trade-off levantado, sem ADR próprio | TRANSCRICAO | `[09:29]`, `[09:30] Bruno, Larissa` |
| P-101 | PRD | L479-482 | Decisão | Snapshot na inserção; trade-off de conteúdo duplicado | TRANSCRICAO | `[09:52] Larissa, Diego` |
| P-102 | PRD | L488-490 | Dependência | Organizacional: revisão de segurança de 2 dias úteis antes do deploy | TRANSCRICAO | `[09:46] Sofia`, `[09:47] Larissa` |
| P-103 | PRD | L492-494 | Dependência | Organizacional: documentação no portal do desenvolvedor por Marcos | TRANSCRICAO | `[09:26]`, `[09:40] Marcos` |
| P-104 | PRD | L496-498 | Dependência | Técnica: segundo processo em todos os ambientes | TRANSCRICAO | `[09:11] Diego` |
| P-105 | PRD | L500-502 | Dependência | Técnica: 13 pendências sem decisão, listadas no FDD | — | Ver F-100 |
| P-106 | PRD | L504-506 | Dependência | Externa: clientes implementam validação de assinatura | TRANSCRICAO | `[09:20] Sofia` |
| P-107 | PRD | L508-510 | Dependência | Externa: clientes implementam descarte de duplicatas | TRANSCRICAO | `[09:25] Diego` |
| P-108 | PRD | L512-514 | Dependência | Externa: clientes expõem endpoint https | TRANSCRICAO | `[09:23] Sofia` |

### Riscos, aceite e testes

| ID | Documento | Linha | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- | --- |
| P-109 | PRD | L520-529 | Risco | Regressão no fluxo de mudança de status; média, alto | TRANSCRICAO | `[09:40]`, `[09:41] Bruno, Diego` |
| P-110 | PRD | L529 | Contingência | Desativar endpoints reduz a publicação a consulta vazia | TRANSCRICAO | `[09:34] Bruno` (derivado) |
| P-111 | PRD | L531-539 | Risco | Prazo de 3 sprints contra a janela comercial; média, alto | TRANSCRICAO | `[09:00]`, `[09:46] Marcos, Larissa` |
| P-112 | PRD | L541-550 | Risco | Falha em assinatura ou geração de secret; baixa, alto | TRANSCRICAO | `[09:22] Diego`, `[09:46] Sofia` |
| P-113 | PRD | L552-560 | Risco | Crescimento das tabelas sem controle; alta, médio | TRANSCRICAO | `[09:08] Diego` |
| P-114 | PRD | L562-570 | Risco | Cliente não implementa dedup; média, médio | TRANSCRICAO | `[09:25] Sofia` |
| P-115 | PRD | L572-579 | Risco | Bombardeio do cliente; média, médio | TRANSCRICAO | `[09:38]`–`[09:39] Diego` |
| P-116 | PRD | L581-589 | Risco | Eventos presos após queda do worker; média, médio | — | Lacuna — ver A2-10 |
| P-117 | PRD | L596-621 | Aceite | 26 critérios objetivos e verificáveis | TRANSCRICAO | `[09:02]`–`[09:52]` (múltiplos) |
| P-118 | PRD | L629-636 | Testes | 8 tipos de teste obrigatórios, incluindo revisão manual de segurança | TRANSCRICAO | `[09:36]`, `[09:42]`, `[09:46]` |
| P-119 | PRD | L640 | Testes | Reuso da suíte existente contra MySQL real, sem simulação | CODIGO | `tests/setup.ts`, `vitest.config.ts` |
| P-120 | PRD | L641 | Testes | Testes de pedidos como baseline de não regressão | CODIGO | `tests/orders.test.ts` |
| P-121 | PRD | L643 | Testes | Revisão técnica de Bruno e Diego antes de codar | TRANSCRICAO | `[09:50] Larissa` |
| P-122 | PRD | L644 | Testes | Revisão de segurança como último passo antes do deploy | TRANSCRICAO | `[09:47] Larissa` |
| P-123 | PRD | L645 | Pendência | Necessidade de teste de carga não definida | — | Lacuna — `[09:38]` sem meta |
| P-124 | PRD | L653-667 | Pendência | Tabela consolidada de 13 pendências do PRD | — | 6 novas de produto + 7 herdadas |

---

## Resumo de cobertura

| Métrica | ADRs | RFC | FDD | PRD | Total |
| --- | --- | --- | --- | --- | --- |
| Itens rastreados | 74 | 77 | 117 | 124 | **392** |
| Itens com fonte `TRANSCRICAO` | 56 | 64 | 77 | 103 | 300 |
| Itens com fonte `CODIGO` | 12 | 5 | 27 | 8 | 52 |
| Itens sem origem (`—`) | 6 | 8 | 13 | 13 | 40 |
| Documentos cobertos | 5 de 5 | 1 de 1 | 1 de 1 | 1 de 1 | **8 de 8** |

Por ADR: ADR-001 = 15 · ADR-002 = 13 · ADR-003 = 16 · ADR-004 = 18 · ADR-005 = 12.

Contagens conferidas por script sobre as próprias tabelas deste arquivo, não à mão. Não há IDs duplicados.

**Leitura da distribuição por documento**

A proporção entre `TRANSCRICAO` e `CODIGO` muda conforme a altura do documento, e isso é esperado:

- **PRD** é o mais ancorado na reunião (103 de 124). Faz sentido: é o documento de produto, e produto veio da conversa, não do código.
- **FDD** é o mais ancorado no código (27 de 117, mais que o dobro de qualquer outro). Também esperado: é o documento de implementação, e implementação precisa citar arquivo e linha reais.
- **ADRs** e **RFC** ficam no meio, com predominância da transcrição, porque registram decisão e não construção.

**Detalhamento dos itens sem origem (`—`)**

| Documento | Composição |
| --- | --- |
| ADRs | 6 lacunas puras (`[NECESSITA INPUT]`). Os outros 4 pontos em aberto foram levantados na reunião mas não fechados, e por isso contam como `TRANSCRICAO`. |
| RFC | 6 lacunas herdadas dos ADRs (R-49, R-51, R-52, R-54, R-56, R-75), 1 risco derivado de lacunas (R-69) e 1 remissão a documentos internos (R-76). |
| FDD | 13 marcadores `[BLOQUEADOR]`, dos quais 3 são novos e nasceram da análise do código (F-86 métricas, F-87 tracing, F-88 painéis, todos ausentes de `src/` e de `package.json`). Os demais são herdados dos ADRs. |
| PRD | 13 pendências, das quais 6 são lacunas de produto identificadas na elaboração do próprio PRD (P-29 metas em regime, P-72 p95, P-76 disponibilidade, P-83 métricas, P-89 compliance, P-123 carga) e 7 são herdadas dos ADRs e do FDD. |

**Onde as lacunas foram descobertas**

As 40 lacunas não são 40 problemas distintos. A maioria é a mesma pendência aparecendo em documentos de alturas diferentes, o que é o comportamento correto de um tracker. O que importa é onde cada uma **apareceu pela primeira vez**:

| Origem da descoberta | Quantidade | Exemplos |
| --- | --- | --- |
| Geração dos ADRs (`[NECESSITA INPUT]`) | 6 | retenção da DLQ, `attempt_count` após replay, UUID v4/v7 |
| Análise do código durante o FDD | 3 | ausência de métricas, tracing e alertas no projeto |
| Elaboração do PRD (lacunas de produto) | 6 | metas em regime, p95, disponibilidade, compliance, carga |

O padrão que isso revela: a reunião foi forte em decisão técnica e fraca em meta de produto. Definiu com precisão o que fazer quando o cliente cai, mas nunca definiu como é o sucesso em operação normal.

> **Sobre os pontos em aberto:** são as lacunas identificadas honestamente durante a produção dos documentos, e não decisões inventadas para preencher espaço. Elas aparecem como `[NECESSITA INPUT]` nos ADRs, `[BLOQUEADOR]` no FDD e `[PENDÊNCIA]` no PRD, e foram promovidas à seção **"Questões em Aberto"** do RFC (`docs/RFC.md` § 5). A rastreabilidade cruzada entre elas está na coluna *Localização* (ex.: `ver A3-12`), permitindo seguir a mesma pendência através das quatro alturas de documento.

## Cobertura dos documentos

- [x] Rastreabilidade — ADRs (`docs/adrs/`), 5 documentos
- [x] Rastreabilidade — RFC (`docs/RFC.md`)
- [x] Rastreabilidade — FDD (`docs/FDD.md`)
- [x] Rastreabilidade — PRD (`docs/PRD.md`)

O pacote de documentação está integralmente rastreado. Nenhum item de nenhum documento afirma algo sem origem verificável na transcrição, no código, ou sem estar explicitamente marcado como lacuna.
