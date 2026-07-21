# ADR-003: Estratégia de Retry com Backoff Exponencial e Dead Letter Queue para Webhooks

**Status:** Aceito
**Data:** 2026-07-14
**ADRs Relacionados:** ADR-001 (Padrão Outbox no MySQL), ADR-002 (Worker em Processo Separado)

---

## 1. Contexto e Problema

O sistema de webhooks precisa entregar notificações de mudança de status de pedidos a clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) de forma confiável. Clientes podem estar temporariamente indisponíveis por falhas de rede, reinicializações de servidor ou janelas de manutenção planejada — cenário concreto na operação da plataforma, com casos de indisponibilidade de até duas horas em manutenção planejada.

A questão central é: quando uma tentativa de entrega de webhook falha, por quanto tempo e com qual frequência o sistema deve tentar novamente antes de declarar falha permanente? E o que acontece com eventos que esgotam todas as tentativas — eles são descartados, marcados na tabela principal ou movidos para uma estrutura separada que permita reprocessamento manual?

A decisão sobre esses parâmetros define o contrato implícito de confiabilidade da plataforma com os integradores B2B e determina o comportamento observável do sistema em cenários de degradação.

## 2. Direcionadores da Decisão

- **Cobertura de janelas de manutenção reais**: o histórico da plataforma inclui clientes com indisponibilidade de até 2 horas; a estratégia de retry deve cobrir esse cenário sem descartar eventos prematuramente.
- **Prevenção de eventos pendentes indefinidamente**: retry sem limite de tentativas cria risco de eventos ficarem pendurados indefinidamente quando o destino desaparece, consumindo recursos sem resolução.
- **Observabilidade e auditabilidade de falhas permanentes**: eventos que esgotam retries precisam ser preservados com contexto suficiente para diagnóstico e reprocessamento controlado.
- **Separação de responsabilidades operacionais**: o reprocessamento de falhas permanentes deve exigir ação intencional de um administrador, não ocorrer automaticamente, para evitar loops de retry em destinos permanentemente inválidos.
- **Integração com o controle de acesso existente**: a infraestrutura de autorização por papel (`requireRole`) já disponível no sistema deve ser reutilizada sem modificação.

## 3. Opções Consideradas

**Opção A — 5 tentativas com backoff exponencial e DLQ em tabela separada** (escolhida)
- Intervalos progressivos: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas (~15 horas de cobertura total).
- Após a 5ª falha, o evento é movido para uma tabela `webhook_dead_letter` separada da outbox principal.
- Endpoint administrativo para reprocessamento manual com controle de acesso ADMIN.

**Opção B — 3 tentativas com backoff agressivo**
- Ciclo total de aproximadamente 30 minutos.
- Descartada porque não cobre janelas de manutenção planejada de 2 horas que já ocorreram com clientes reais.

**Opção C — Retry indefinido com backoff e marcação como FAILED na outbox**
- Eventos nunca seriam descartados; a tabela outbox absorveria estados terminais.
- Descartada pelo risco de acúmulo ilimitado de eventos para destinos permanentemente inativos e por poluir a leitura da outbox com registros em estado terminal misturados aos pendentes.

## 4. Decisão

Escolhida a **Opção A**: 5 tentativas com backoff exponencial nos intervalos 1 minuto / 5 minutos / 30 minutos / 2 horas / 12 horas, seguidas de movimentação para uma Dead Letter Queue implementada como tabela MySQL separada (`webhook_dead_letter`).

O número 5 de tentativas foi calibrado para cobrir a janela de até ~15 horas entre primeira falha e última tentativa — suficiente para manutenções planejadas reais observadas na base de clientes, sem comprometer recursos com destinos permanentemente inativos. O limite temporal é aceitável do ponto de vista de negócio: uma indisponibilidade de 15 horas já representa um problema grave do lado do cliente. O timeout de 10 segundos por chamada HTTP integra-se ao ciclo: respostas lentas acima do limite são tratadas como falha e contam para o ciclo de retry.

A DLQ como tabela separada, e não como status FAILED na outbox, foi escolhida para manter a outbox principal com semântica clara de eventos ativos, enquanto a DLQ serve de evidência auditável para debug e reprocessamento. O reprocessamento manual via endpoint `POST /admin/webhooks/dead-letter/:id/replay`, protegido por papel ADMIN, garante que ações sobre a DLQ sejam intencionais e rastreáveis.

[PRECISA DE INFORMAÇÃO: Qual é a política de retenção da tabela `webhook_dead_letter`? Os eventos devem ser purgados automaticamente após um período definido (ex: 90 dias) ou permanecem indefinidamente até reprocessamento manual?]

## 5. Prós e Contras das Opções

### Opção A — 5 tentativas com backoff exponencial e DLQ em tabela separada

**Prós:**
- Cobre janelas de manutenção planejada de até 2 horas: com os intervalos acumulados (1m, 6m, 36m, ~2h36m), a tentativa em torno de 2h36 recupera o evento, margem que a Opção B de 3 tentativas (~30 min) não alcança.
- DLQ separada preserva a legibilidade da outbox principal e serve como evidência estruturada para suporte.
- Endpoint de replay com restrição de papel ADMIN reutiliza infraestrutura existente (`src/middlewares/auth.middleware.ts:49`).
- Ciclo de ~15 horas validado como aceitável pelos stakeholders de negócio.

**Contras:**
- Eventos podem permanecer na outbox por até ~15 horas antes de ir para a DLQ, impactando o tamanho da tabela em cenários de degradação prolongada.
- Sem política de retenção definida, a tabela `webhook_dead_letter` cresce indefinidamente.

### Opção B — 3 tentativas com backoff agressivo

**Prós:**
- Ciclo mais curto reduz o tempo máximo de eventos pendentes na outbox.
- Operacionalmente mais simples de monitorar.

**Contras:**
- Insuficiente para cobrir indisponibilidades reais de 2 horas documentadas na base de clientes.
- Descarta eventos prematuramente em cenários de manutenção legítima.

### Opção C — Retry indefinido com marcação como FAILED na outbox

**Prós:**
- Nenhum evento é descartado sem reprocessamento explícito.

**Contras:**
- Eventos para destinos permanentemente inativos acumulam indefinidamente, consumindo recursos e poluindo monitoramento.
- Mistura de eventos pendentes e terminais na outbox prejudica a legibilidade e eficiência do worker.

## 6. Consequências

A implementação desta estratégia estabelece um contrato operacional implícito com os clientes B2B: o sistema fará até 5 tentativas ao longo de ~15 horas antes de mover um evento para a DLQ. Este comportamento deve ser documentado no portal de desenvolvedores, pois afeta as expectativas de integração da Atlas Comercial, MaxDistribuição e Nova Cargo.

A tabela `webhook_dead_letter` acumula evidência de falhas permanentes. Os campos de rastreabilidade de replay (`replayed_by`, `replayed_at`) são requisito explícito de auditoria: o endpoint de admin precisa registrar quem executou o replay. O logging estruturado via Pino deve registrar cada ação de replay com identidade do executor e identificador do evento.

[PRECISA DE INFORMAÇÃO: Após um replay manual, o evento recolocado na outbox deve reiniciar o `attempt_count` para zero ou manter o histórico acumulado de tentativas anteriores? A resposta afeta tanto a lógica do endpoint de replay quanto a semântica dos campos da DLQ.]

A detecção e recuperação de eventos travados no estado `PROCESSING` após crash do worker é um item em aberto do ciclo de vida do worker, tratado no ADR-002 (Worker em Processo Separado com Polling) — ver o marcador de pendência na seção 6 daquele documento. A definição adotada lá governa também o comportamento de retry descrito aqui.

## 7. Referências

- `src/middlewares/auth.middleware.ts:49` — Implementação de `requireRole` reutilizada no endpoint de replay da DLQ.
- `src/shared/errors/app-error.ts` — Classe base para os erros específicos do módulo de webhooks, seguindo o prefixo `WEBHOOK_`.
- `src/modules/orders/order.service.ts` — Ponto de integração onde a inserção na outbox ocorre dentro da transação de mudança de status.
- `prisma/schema.prisma` — Schema a ser estendido com os modelos `webhook_outbox` e `webhook_dead_letter`, incluindo campos de controle de retry (`attempt_count`, `next_attempt_at`, `last_error`).
