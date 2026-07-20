# ADR-004: Autenticação HMAC-SHA256 com Secret por Endpoint e Rotação com Grace Period

**Status:** Aceito
**Data:** 2026-07-14
**ADRs Relacionados:** ADR-001 (Padrão Outbox no MySQL), ADR-002 (Worker com Polling), ADR-003 (Retry com Backoff e DLQ)

---

## 1. Contexto e Definição do Problema

A plataforma OMS precisa notificar sistemas externos de clientes B2B sobre mudanças de status de pedidos via webhooks outbound. Como os eventos trafegam por redes externas e contêm dados sensíveis de pedidos, é necessário um mecanismo que permita ao cliente verificar que a requisição foi originada legítimamente pela plataforma e que o payload não foi adulterado em trânsito.

A motivação para adotar autenticação em nível de payload — e não apenas TLS — surgiu de experiência operacional concreta: um cliente anterior vazou uma secret de integração em logs de aplicação (`[09:22]` Diego), expondo a necessidade de isolar o raio de blast por endpoint. Sofia (`[09:19]`) contextualizou o problema: "a gente tá expondo eventos com dados de pedidos pra um endpoint fora da nossa infra. O cliente tem que conseguir validar que a requisição veio realmente da gente."

A rotação de secrets sem janela de inatividade é requisito operacional para clientes B2B com múltiplos sistemas que precisam ser atualizados de forma escalonada, sem coordenação em tempo real com a plataforma.

## 2. Fatores de Decisão

- Autenticidade e integridade do payload devem ser verificáveis pelo cliente independentemente do canal de transporte
- O comprometimento de uma secret não pode afetar outros endpoints ou clientes (isolamento de blast radius)
- A rotação de secrets deve ser possível sem janela de manutenção sincronizada entre plataforma e cliente
- O mecanismo deve ser compatível com bibliotecas disponíveis em qualquer linguagem de programação (`[09:20]` Sofia: "todo cliente sério tem biblioteca pra isso")
- Todas as URLs de destino devem usar TLS obrigatoriamente — URLs `http://` devem ser rejeitadas em validação de schema (`[09:23]` Sofia)

## 3. Opções Consideradas

- **Opção A:** HMAC-SHA256 com secret única por endpoint e grace period de 24 horas na rotação
- **Opção B:** Secret global da plataforma (um único segredo compartilhado com todos os endpoints)
- **Opção C:** TLS como único mecanismo de segurança, sem assinatura de payload

## 4. Decisão

Escolhida a **Opção A — HMAC-SHA256 com secret única por endpoint e grace period de 24 horas**, conforme declarado por Sofia em `[09:22]`: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h."

O algoritmo SHA-256 foi escolhido por ser padrão de mercado, garantindo que clientes B2B já possuam bibliotecas disponíveis para verificação (`[09:20]` Sofia: "HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso"). A granularidade por endpoint — em vez de secret global — foi motivada pelo princípio de blast radius: o vazamento de uma secret compromete apenas o endpoint afetado, não toda a integração do cliente. O grace period de 24 horas permite que o cliente atualize seus sistemas de forma escalonada sem coordenação em tempo real com a plataforma.

Cada request de webhook inclui os headers: `X-Event-Id` (UUID do evento para idempotência), `X-Signature` (HMAC-SHA256 sobre o corpo), `X-Timestamp` (timestamp de envio para detecção opcional de replay attack), `X-Webhook-Id` (UUID do endpoint, para clientes com múltiplos endpoints cadastrados — `[09:44]`–`[09:45]` Sofia), e `Content-Type: application/json`.

A secret é gerada pela plataforma e devolvida ao cliente no momento da criação do webhook (`[09:31]` Marcos).

## 5. Prós e Contras das Opções

### Opção A: HMAC-SHA256 com secret por endpoint e grace period

**Prós:**
- Vazamento de uma secret isola o impacto a um único endpoint (`[09:21]` Sofia — confirmado por caso real, `[09:22]` Diego)
- SHA-256 disponível em bibliotecas nativas de qualquer linguagem relevante no mercado B2B
- Grace period de 24 horas elimina janelas de manutenção na rotação de secrets

**Contras:**
- Responsabilidade de armazenar a secret com segurança recai sobre o cliente (modelo assimétrico)
- Grace period cria janela de vulnerabilidade caso ambas as secrets — atual e anterior — sejam comprometidas simultaneamente
- Schema de dados requer dois campos de secret simultâneos por endpoint, com timestamp de expiração da secret anterior

### Opção B: Secret global da plataforma

**Prós:**
- Implementação mais simples — um único segredo a gerenciar
- Rotação centralizada, sem necessidade de notificar clientes individualmente

**Contras:**
- Vazamento de uma única secret compromete todos os clientes e endpoints simultaneamente (`[09:21]` Sofia: "Senão se vaza uma, vaza tudo")
- Caso real documentado: cliente que vazou secret em log de aplicação (`[09:22]` Diego) causaria dano total neste modelo

### Opção C: TLS sem assinatura de payload

**Prós:**
- Zero implementação adicional no lado da plataforma

**Contras:**
- TLS protege o transporte, mas não garante autenticidade da origem — man-in-the-middle no nível de infraestrutura do cliente comprometeria os dados
- Clientes não conseguem verificar se o payload veio da plataforma ou de terceiro que tenha acesso ao endpoint deles

## 6. Consequências

A adoção desta arquitetura de segurança cria um gate de processo explícito: Sofia reservou 2 dias úteis para revisão do código de segurança antes do deploy (`[09:46]`), com foco específico na lógica HMAC e na geração de secrets. Este requisito de revisão deve ser registrado no processo de desenvolvimento da feature — não é opcional sob pressão de prazo.

O schema de dados do módulo de webhooks deve suportar dois campos de secret simultâneos por endpoint (`secret_current` e `secret_previous`), com timestamp de expiração da secret anterior. A lógica do worker deve usar a secret vigente para assinar; qualquer endpoint de validação de entrada deve aceitar ambas durante o grace period de 24 horas. As secrets também devem ser adicionadas às regras de redação de logs existentes em `src/shared/logger/index.ts` para evitar exposição acidental.

[PRECISA DE INFORMAÇÃO: Qual é a política de comprimento mínimo e fonte de entropia para as secrets geradas automaticamente? A transcrição (`[09:31]` Marcos) estabelece que a secret é gerada pela plataforma, mas não especifica requisitos de entropia, e não há precedente no código atual — o projeto usa bcrypt para senhas e valida `JWT_SECRET` com no mínimo 16 caracteres (`src/config/env.ts:8`), sem geração aleatória de segredos.]

[PRECISA DE INFORMAÇÃO: A secret é recuperável via GET após a criação, ou é entregue apenas uma vez? A transcrição define que ela é devolvida na criação (`[09:31]` Marcos), mas não a política de exposição posterior. A prática recomendada é entrega única — recuperação exigiria nova rotação.]

[PRECISA DE INFORMAÇÃO: Como o sistema deve se comportar quando um cliente ainda usa a secret anterior após o grace period de 24 horas expirar? O erro deve ser explícito (HTTP 401 com código `WEBHOOK_SECRET_EXPIRED`) ou silencioso? A transcrição não define este comportamento de borda.]

[PRECISA DE INFORMAÇÃO: A proteção contra replay attack via `X-Timestamp` ficará para implementação futura obrigatória ou é opcional para o cliente? `[09:44]` Diego menciona o header, mas a reunião não define se a plataforma fará validação da janela temporal ou apenas documentará a semântica para os clientes.]

## 7. Referências

- `src/middlewares/auth.middleware.ts:1` — Padrão de autenticação existente (JWT) que estabelece convenção para novos middlewares de segurança
- `src/shared/logger/index.ts:4` — Lista de redação de campos sensíveis onde secrets de webhook devem ser adicionadas
- `src/shared/errors/http-errors.ts` — Hierarquia de erros para novos códigos `WEBHOOK_SECRET_*`
- `prisma/schema.prisma` — Schema a ser estendido com campos `secret_current`, `secret_previous`, `secret_rotated_at` na tabela `webhook_endpoints`
