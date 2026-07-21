### PRD: OMS Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-07-15
Responsável: Marcos (Product Manager)

> **Convenção do documento**
> Caminhos como `src/...` referenciam código existente e verificado.
> **[PENDÊNCIA]** marca ponto sem decisão fechada e sem base verificável no código. Nenhum valor foi inventado para preencher esses espaços. A lista consolidada está no fim do documento.
> Documentos relacionados: [RFC](./RFC.md), [FDD](./FDD.md), [ADRs](./adrs/), [Tracker](./TRACKER.md).

---

### Resumo

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para serem notificados em tempo real quando o status dos pedidos deles muda na plataforma. Hoje eles descobrem mudanças fazendo polling repetido contra `GET /orders`, o que torna a integração lenta e cara do lado deles. A Atlas sinalizou que pode migrar para o concorrente se a entrega não sair até o fim do trimestre.

Esta feature entrega webhooks de saída: quando o status de um pedido muda, a plataforma envia um POST assinado para uma URL cadastrada pelo cliente, em menos de 10 segundos. O cliente cadastra e gerencia seus endpoints via API, escolhe quais status quer receber, e consulta o histórico de entregas. Falhas são retentadas automaticamente ao longo de cerca de 15 horas antes de irem para uma fila de falhas com reprocessamento manual.

O escopo é estritamente de saída: a plataforma envia, os clientes recebem. Não há painel visual nem notificação por email nesta fase.

---

### Contexto e problema

Público-alvo

- Clientes B2B integrados via API que precisam reagir a mudanças de status de pedidos: Atlas Comercial, MaxDistribuição e Nova Cargo.
- Times de desenvolvimento desses clientes, que constroem e mantêm a integração do lado deles e passam a ser responsáveis por validar assinatura e deduplicar eventos.
- Operação interna da plataforma, que ganha uma nova superfície para diagnosticar e reprocessar falhas de entrega.

Cenários de uso chave

- Um pedido da Atlas muda de PROCESSING para SHIPPED e o sistema da Atlas é notificado em segundos, sem precisar consultar a API.
- Um cliente cadastra um endpoint e escolhe receber apenas SHIPPED e DELIVERED, ignorando os demais status.
- O endpoint de um cliente fica indisponível durante uma manutenção planejada de 2 horas e recebe os eventos quando volta, sem perda.
- Um cliente vazou a secret em log e precisa rotacionar sem parar a integração, migrando os sistemas dele dentro de 24 horas.
- Um cliente que tem vários endpoints cadastrados precisa saber qual deles corresponde a um envio recebido.
- O suporte investiga por que um cliente não recebeu um evento e consulta o histórico de entregas com payload, resposta e tempo de resposta.
- Um evento esgotou todas as tentativas e um administrador o reprocessa manualmente após o cliente confirmar que voltou.

Onde essa feature será implantada

- Sistema existente. A feature entra no OMS atual, um Order Management System em Node.js e TypeScript com Express, Prisma e MySQL. O código já tem módulos de autenticação, usuários, clientes, produtos e pedidos, com máquina de estados de pedido, controle transacional de estoque e auditoria de mudanças de status.
- A aplicação não possui hoje nenhum mecanismo de notificação externa, eventos ou filas. Essa é exatamente a lacuna que a feature preenche.
- A entrega acrescenta um segundo processo de longa duração em todos os ambientes: além da API, passa a existir um worker independente. Nenhuma infraestrutura nova é introduzida: a feature usa o MySQL já operacional, sem Redis nem broker externo.

Problemas priorizados

- **Polling ineficiente dos clientes B2B.** Os três clientes consultam `GET /orders` repetidamente para descobrir mudanças. Impacto: integração lenta e cara do lado deles, e carga desnecessária na plataforma. Prioridade: alta.
- **Risco comercial concreto de churn.** A Atlas Comercial declarou que pode migrar para o concorrente se a entrega não sair até o fim do trimestre. Impacto: perda de receita de um cliente B2B e sinal negativo para os outros dois. Prioridade: alta.
- **Atualização manual do lado do cliente.** Sem notificação, os clientes precisam ficar atualizando para saber o que mudou. Impacto: atraso operacional na cadeia deles, que depende do status para agir. Prioridade: alta.
- **Ausência de qualquer canal de eventos na plataforma.** Não existe base sobre a qual construir. Impacto: qualquer necessidade futura de notificação externa esbarra na mesma lacuna. Prioridade: média.

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Notificar mudanças de status em tempo real, conforme a definição dos clientes | Tempo entre o commit da mudança de status e o primeiro POST ao endpoint do cliente | Abaixo de 10 segundos em 100% dos casos, com worker saudável. Alvo prático de até 2 segundos pelo intervalo de polling do worker |
| Eliminar o polling dos clientes B2B contra `GET /orders` | Número de clientes B2B migrados para webhooks | 3 (Atlas Comercial, MaxDistribuição e Nova Cargo) |
| Reter a Atlas Comercial, evitando a migração para o concorrente | Entrega em produção dentro da janela comercial acordada | Fim de novembro, dentro do trimestre |
| Não degradar a operação de mudança de status, que é o caminho crítico do OMS | Latência de `PATCH /orders/:id/status` comparada ao baseline atual | Sem aumento mensurável acima do baseline |
| Absorver indisponibilidade real dos clientes sem perder eventos | Janela de indisponibilidade do cliente coberta pelo ciclo de retry | Cerca de 15 horas, com 5 tentativas em 1m, 5m, 30m, 2h e 12h |
| Entregar dentro da capacidade estimada do time | Sprints consumidas, incluindo a revisão de segurança | 3 sprints |

> **[PENDÊNCIA]** Não existe meta definida para taxa de entrega bem sucedida sem intervenção manual, nem para volume aceitável de eventos na fila de falhas. O escopo atual só tratou do comportamento em caso de falha, não de metas operacionais em regime. Definir antes da operação em produção.

---

### Escopo

Incluso

- Cadastro de endpoint de webhook por cliente, com URL, lista de status a receber e secret gerada pela plataforma.
- Listagem, edição e remoção de endpoints de webhook.
- Filtro de eventos por status: cada endpoint escolhe quais transições quer receber.
- Rotação de secret pela API, com a secret anterior válida por 24 horas em paralelo.
- Publicação do evento de mudança de status de forma atômica com a própria mudança.
- Entrega do evento por POST assinado com HMAC-SHA256, com timeout de 10 segundos.
- Retentativa automática com backoff exponencial, 5 tentativas em 1m, 5m, 30m, 2h e 12h.
- Fila de falhas permanentes (dead letter) com payload, motivo e timestamp preservados.
- Reprocessamento manual de itens da fila de falhas, restrito a administradores e auditado.
- Histórico de entregas por endpoint, com sucesso ou falha, payload, resposta e tempo de resposta.
- Validação de TLS obrigatório na URL cadastrada e limite de 64KB por payload.

Fora de escopo

- **Notificação por email ao cliente quando o webhook dele está falhando.** Fora de escopo desta fase; possível próxima fase, após medir o impacto.
- **Dashboard visual para o cliente acompanhar os webhooks dele.** Descartado nesta fase; painel é projeto separado do time de frontend.
- **Rate limiting de saída.** O cenário de 50 pedidos mudando de status em um minuto foi considerado, e a decisão foi adiar: observar e implementar se virar problema.
- **Webhooks de entrada (inbound).** O escopo é apenas de saída; os clientes querem apenas receber.
- **Escala horizontal do worker.** Múltiplos workers em paralelo removeriam a garantia de ordenação. Adiado como problema futuro.
- **Arquivamento ou expurgo de eventos já entregues.** Janela de referência de cerca de 30 dias, mas declarada fora do escopo desta feature.
- **Detalhes completos do pedido no payload.** Os itens do pedido não são enviados, para manter o payload enxuto. O cliente consulta `GET /orders/:id` se precisar.
- **Endurecimento do controle de acesso no cadastro de webhooks.** Por ora qualquer usuário autenticado configura endpoints. Endurecimento adiado.

> Nenhum item de "Fora de escopo" contradiz o que está incluso. O reprocessamento manual está incluso e é justamente o que substitui o email de aviso, que ficou fora.

---

### Requisitos funcionais

#### FR-001 Cadastrar endpoint de webhook

O cliente registra uma URL que passará a receber notificações de mudança de status dos pedidos dele.

**Fluxo principal**

- Usuário autenticado envia a requisição de cadastro com o identificador do cliente, a URL de destino e a lista de status que deseja receber.
- O sistema valida que a URL usa https.
- O sistema gera uma secret e a associa ao endpoint.
- O sistema persiste o endpoint como ativo e devolve o cadastro com a secret.

**Fluxos alternativos e exceções**

- O identificador do cliente vem no corpo ou no caminho da requisição, e não do token de autenticação, porque o token representa o usuário operador e não o cliente.
- Qualquer usuário autenticado pode cadastrar nesta fase, sem exigência de papel específico.
- Um mesmo cliente pode ter mais de um endpoint cadastrado, e cada envio identifica qual endpoint o originou.

**Erros previstos**

- URL sem https é recusada na validação, antes de qualquer persistência.
- Corpo inválido ou lista de status com valor inexistente é recusado na validação.
- Requisição sem autenticação é recusada.

**Prioridade:** alta

---

#### FR-002 Listar, editar e remover endpoints de webhook

O cliente gerencia os endpoints já cadastrados.

**Fluxo principal**

- Usuário autenticado consulta os endpoints de um cliente.
- Usuário edita um endpoint para alterar a URL, a lista de status ou o estado de ativação.
- Usuário remove um endpoint que não deseja mais.

**Fluxos alternativos e exceções**

- A secret nunca é devolvida em listagem, apenas no cadastro e na rotação.
- Desativar um endpoint interrompe a geração de novos eventos para ele, porque a filtragem acontece no momento da inserção.

**Erros previstos**

- Endpoint inexistente retorna erro de recurso não encontrado.
- Edição que introduza URL sem https é recusada.

**Prioridade:** alta

---

#### FR-003 Filtrar eventos por status

Cada endpoint recebe apenas as transições de status que assinou.

**Fluxo principal**

- No cadastro ou na edição, o cliente informa a lista de status que quer ouvir, por exemplo apenas SHIPPED e DELIVERED.
- Quando um pedido muda de status, o sistema consulta quais endpoints daquele cliente assinaram aquele status.
- O evento é criado apenas para os endpoints que assinaram.

**Fluxos alternativos e exceções**

- Se nenhum endpoint do cliente assinou aquele status, nenhum evento é criado. Isso não é erro, é o caminho esperado, e economiza registro na base.
- A filtragem ocorre no momento da inserção, e não no momento do envio, por decisão explícita.

**Erros previstos**

- Status inexistente na lista de assinatura é recusado na validação.

**Prioridade:** alta

---

#### FR-004 Gerar e rotacionar a secret de assinatura

Cada endpoint tem uma secret própria, usada para assinar os envios, e o cliente pode trocá-la sem parar a integração.

**Fluxo principal**

- A secret é gerada pela plataforma no cadastro do endpoint e devolvida ao cliente.
- O cliente solicita rotação pela API.
- O sistema gera uma nova secret vigente e mantém a anterior válida por 24 horas.
- Após 24 horas a secret anterior deixa de valer.

**Fluxos alternativos e exceções**

- Cada endpoint tem secret única. Não existe secret global da plataforma, para que o vazamento de uma não comprometa as demais.
- O período de 24 horas existe para o cliente migrar os sistemas dele de forma escalonada, sem coordenação em tempo real.

**Erros previstos**

- Rotação sobre endpoint inexistente retorna erro de recurso não encontrado.
- **[PENDÊNCIA]** O comportamento para o cliente que continua usando a secret anterior depois das 24 horas não está definido.

**Prioridade:** alta

---

#### FR-005 Publicar o evento de forma atômica com a mudança de status

Quando o status de um pedido muda, o evento correspondente é registrado na mesma transação.

**Fluxo principal**

- A mudança de status é solicitada e a transação existente é aberta.
- O sistema valida a transição, ajusta o estoque, grava o novo status e registra o histórico, exatamente como já faz hoje.
- Dentro da mesma transação, o sistema registra o evento a notificar.
- A transação confirma. O evento passa a existir e fica disponível para entrega.
- A resposta ao chamador não espera nenhuma entrega de webhook.

**Fluxos alternativos e exceções**

- Se qualquer passo falhar, incluindo o registro do evento, tudo é desfeito e nenhum evento existe. Nunca pode existir status mudado sem evento registrado.
- O conteúdo do evento é uma fotografia do pedido no instante da transição, e não é recalculado no envio. Se o pedido mudar depois, o evento continua refletindo o momento em que o status mudou.
- A indisponibilidade de um cliente nunca desfaz uma mudança de status: um sistema externo fora do ar não pode forçar rollback de uma operação de negócio legítima.

**Erros previstos**

- Falha ao registrar o evento desfaz a mudança de status, por decisão explícita.

**Prioridade:** alta

---

#### FR-006 Entregar o evento assinado ao endpoint do cliente

O sistema envia o evento e permite ao cliente verificar que ele veio mesmo da plataforma.

**Fluxo principal**

- Um processo independente busca periodicamente os eventos pendentes, a cada 2 segundos.
- Para cada evento, o sistema assina o conteúdo com HMAC-SHA256 usando a secret do endpoint.
- O sistema envia o POST com os cabeçalhos de identificação do evento, assinatura, horário de envio e identificação do endpoint.
- Qualquer resposta de sucesso encerra o evento como entregue, e a tentativa é registrada no histórico.

**Fluxos alternativos e exceções**

- A entrega é garantida ao menos uma vez. O cliente pode receber o mesmo evento duas vezes e deve descartar duplicatas usando o identificador do evento.
- O identificador do evento é gerado no registro e nunca muda entre tentativas, o que torna o descarte de duplicatas determinístico.
- A responsabilidade de descartar duplicatas é do cliente, justificada pelo padrão de mercado: Stripe e GitHub adotam o mesmo modelo.
- Enquanto houver um único processo de entrega, os eventos de um mesmo pedido chegam na ordem em que ocorreram. Não há garantia de ordenação global, e os clientes não pediram isso.
- O conteúdo enviado não inclui os itens do pedido, para não inflar o envio.

**Erros previstos**

- Evento acima de 64KB não é enviado e é tratado como erro, sem truncar.
- Cliente que não responde em 10 segundos é tratado como falha e entra no ciclo de retentativa.

**Prioridade:** alta

---

#### FR-007 Retentar entregas que falharam

Falhas temporárias do cliente não causam perda do evento.

**Fluxo principal**

- Uma entrega falha por resposta de erro, tempo esgotado ou problema de rede.
- O sistema registra a tentativa e agenda a próxima segundo o intervalo previsto.
- Os intervalos crescem progressivamente: 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, somando cerca de 15 horas.
- Se alguma tentativa der certo, o evento é encerrado como entregue.

**Fluxos alternativos e exceções**

- São 5 tentativas. A proposta de 3 foi recusada porque não cobre indisponibilidade real de 2 horas em manutenção planejada, que já aconteceu com clientes da base.
- A retentativa indefinida também foi recusada, para não deixar evento pendurado indefinidamente caso o cliente desapareça.
- A janela de 15 horas é aceitável do ponto de vista de negócio: uma indisponibilidade dessa ordem já representa um problema grave do lado do cliente.

**Erros previstos**

- Esgotadas as 5 tentativas, o evento é tratado como falha permanente e sai da fila ativa.

**Prioridade:** alta

---

#### FR-008 Preservar falhas permanentes em fila de falhas

Eventos que esgotaram as tentativas não são descartados.

**Fluxo principal**

- Após a quinta falha, o evento é movido para uma fila de falhas separada.
- O registro guarda o conteúdo do evento, o motivo da falha e o horário.

**Fluxos alternativos e exceções**

- A fila de falhas é uma estrutura separada da fila principal, e não uma marcação nela, para manter a leitura da fila ativa limpa e servir de evidência para diagnóstico.

**Erros previstos**

- Nenhum erro previsto para o cliente. É um estado terminal interno, visível para a operação.
- **[PENDÊNCIA]** Não há política definida de retenção para a fila de falhas. Sem isso, ela cresce indefinidamente.

**Prioridade:** alta

---

#### FR-009 Reprocessar manualmente um item da fila de falhas

A operação consegue reenviar um evento depois que o cliente se recuperou.

**Fluxo principal**

- Um administrador solicita o reprocessamento de um item específico da fila de falhas.
- O sistema recoloca o evento na fila ativa como pendente.
- O sistema registra quem executou a ação, para auditoria.

**Fluxos alternativos e exceções**

- O reprocessamento é manual e intencional, e não automático, para evitar reenvio contínuo a destinos permanentemente inválidos.
- Exige papel de administrador. Mexer em fila de entrega de notificação não é operação de operador.
- O controle de acesso por papel já existente no sistema é reaproveitado, sem construir nada novo.

**Erros previstos**

- Usuário sem papel de administrador é recusado.
- Item inexistente na fila de falhas retorna erro de recurso não encontrado.
- **[PENDÊNCIA]** Não está definido se o contador de tentativas reinicia ou mantém o histórico acumulado após o reprocessamento.

**Prioridade:** média

---

#### FR-010 Consultar o histórico de entregas

O cliente enxerga o que a plataforma tentou entregar para ele.

**Fluxo principal**

- Usuário autenticado consulta o histórico de um endpoint.
- O sistema retorna as entregas com indicação de sucesso ou falha, conteúdo enviado, resposta recebida e tempo de resposta.

**Fluxos alternativos e exceções**

- O histórico cobre as últimas entregas por endpoint (por exemplo, os últimos 100 envios), permitindo ao cliente auditar o que recebeu.
- Cada tentativa de um mesmo evento aparece como um registro próprio, permitindo ver o ciclo completo de retentativas.

**Erros previstos**

- Endpoint inexistente retorna erro de recurso não encontrado.

**Prioridade:** média

---

### Requisitos não funcionais

Performance

- Tempo entre a mudança de status e o primeiro envio ao cliente abaixo de 10 segundos. Essa é a definição de tempo real dada pelos próprios clientes: qualquer coisa abaixo de 10 segundos já é considerada tempo real.
- Intervalo de verificação de eventos pendentes de 2 segundos, o que estabelece a latência de pior caso em cerca de 2 segundos e atende o requisito com folga.
- Tempo limite de 10 segundos por chamada ao endpoint do cliente.
- A operação de mudança de status não pode ficar mais lenta de forma mensurável por causa desta feature. Foi justamente esse o argumento contra a alternativa síncrona.
- Tamanho máximo de 64KB por evento enviado.
- **[PENDÊNCIA]** Não há meta de latência p95 definida para os endpoints de configuração de webhook. Só a latência de notificação foi definida.

Disponibilidade

- O sistema tolera indisponibilidade do cliente de até cerca de 15 horas sem perder eventos, por meio do ciclo de 5 tentativas.
- Falha ou reinicialização da API não interrompe o processamento de eventos, porque o worker é um processo separado.
- A API continua operando normalmente se o worker estiver fora. Eventos acumulam como pendentes e são entregues quando ele voltar, sem perda.
- **[PENDÊNCIA]** Não existe meta de disponibilidade declarada para a plataforma nem para o worker. O escopo atual tratou a indisponibilidade do cliente, nunca a da própria plataforma. Uma meta precisa ser definida; nenhum número foi adotado por falta de base verificável.

Segurança e autorização

- Toda requisição enviada ao cliente é assinada com HMAC-SHA256 sobre o corpo, para que ele verifique origem e integridade.
- Cada endpoint tem secret única. Secret global foi recusada explicitamente.
- A secret é rotacionável, com 24 horas de validade paralela da anterior.
- TLS obrigatório. URL sem https é recusada na validação.
- Endpoints de configuração exigem autenticação. O reprocessamento da fila de falhas exige papel de administrador.
- Toda ação de reprocessamento registra quem a executou, para auditoria.
- Nenhuma secret pode aparecer em log. A motivação é um caso real de cliente que vazou secret em log de aplicação.
- Revisão de segurança obrigatória antes do deploy, com no mínimo 2 dias úteis reservados, focada na assinatura e na geração de secret.
- **[PENDÊNCIA]** Comprimento mínimo e fonte de entropia da secret não foram definidos. **[PENDÊNCIA]** Não está definido se a secret é recuperável após a criação ou entregue apenas uma vez.

Observabilidade

- Registro estruturado de cada tentativa de entrega, com o identificador do evento presente em todos os registros, permitindo reconstruir o ciclo completo de um evento, incluindo todas as retentativas.
- O registro de log já existente no projeto (Pino) é reaproveitado, sem introduzir biblioteca nova.
- A lista de campos sensíveis que já são ocultados em log precisa ser estendida para incluir as secrets de webhook.
- Histórico de entregas consultável por endpoint, com resposta e tempo de resposta, funcionando como observabilidade voltada ao cliente.
- **[PENDÊNCIA]** O projeto não possui nenhuma instrumentação de métricas nem de tracing, e não há decisão registrada sobre o assunto. Adotar métricas de erro por endpoint e tracing distribuído exigiria decisão de arquitetura e dependência nova, sem base no material de origem. Ver seção 7 do FDD.

Confiabilidade e integridade de dados

- O registro do evento acontece na mesma transação da mudança de status. Ou os dois acontecem, ou nenhum acontece: se a transação principal commita, o evento foi registrado; se sofre rollback, o evento some junto. Não há inconsistência possível.
- O controle transacional de estoque já existente permanece intacto. A feature não altera a máquina de estados nem as regras de estoque.
- O conteúdo do evento é uma fotografia do instante da transição e não é recalculado no envio.
- A entrega é garantida ao menos uma vez, com identificador estável por evento para descarte de duplicatas pelo cliente.
- A ordenação por pedido é garantida enquanto houver um único processo de entrega. Ordenação global não é garantida e não foi pedida.

Compatibilidade e portabilidade

- Nenhuma rota, contrato ou comportamento atual da API muda. A feature apenas acrescenta rotas novas sob o mesmo prefixo versionado já em uso.
- Nenhuma infraestrutura nova. A feature usa o MySQL, o Prisma e o Node já existentes, sem Redis nem broker: subir um cluster dedicado seria overengineering para um time pequeno.
- A feature segue os padrões já consolidados no projeto: mesma estrutura de módulos, mesmas classes de erro, mesmo tratamento centralizado de erro, mesmo padrão de validação e mesmo padrão de identificadores.
- O formato do conteúdo enviado e a garantia de entrega ao menos uma vez tornam-se contrato público a partir do primeiro cliente integrado. Mudar isso depois exigiria versionamento.

Compliance

- Trilha de auditoria de reprocessamento, com identificação de quem executou a ação, disponível para consulta.
- A auditoria de mudanças de status já existente no sistema permanece inalterada e continua sendo a fonte de verdade sobre o ciclo de vida do pedido.
- **[PENDÊNCIA]** Não houve definição sobre requisitos regulatórios relativos ao envio de dados de pedidos para fora da infraestrutura da plataforma. Nada foi assumido a respeito.

Acessibilidade no frontend consumidor

- Não aplicável nesta fase. A entrega é composta apenas por endpoints de API, sem interface visual. O painel para o cliente foi explicitamente colocado fora de escopo e tratado como projeto separado do time de frontend.

---

### Arquitetura e abordagem

Abordagem

- A feature é implementada no sistema existente, seguindo o padrão de módulos já adotado. Não é um novo serviço.
- O evento é registrado de forma atômica com a mudança de status, usando o banco que já existe como fila. Um processo separado consome esse registro e faz as chamadas HTTP.
- A comunicação é assíncrona da plataforma para o cliente. Não há chamada síncrona a sistema externo dentro de nenhuma transação.
- Não há fila dedicada, mensageria, streaming nem cache. A decisão foi deliberada, para não introduzir infraestrutura nova em um time pequeno.
- O detalhamento técnico está no [RFC](./RFC.md) e no [FDD](./FDD.md).

Componentes

- Módulo de webhooks dentro da aplicação existente, seguindo a mesma estrutura dos demais domínios: cada domínio é um módulo em `src/modules` com controller, service, repository, routes e schemas.
- Ponto de publicação do evento dentro do serviço de pedidos, acionado na mudança de status, recebendo a transação em andamento.
- Processo independente de entrega, com entrada própria e comando próprio de execução, espelhando o padrão do servidor já existente.
- Quatro estruturas novas no banco: cadastro de endpoints, fila de eventos a entregar, histórico de entregas e fila de falhas permanentes.
- Banco MySQL já existente, sem mudança de versão. O processo de entrega usa a mesma base e a mesma configuração de conexão, em instância própria por ser outro processo.

Integrações

- Endpoints HTTP dos clientes B2B, cadastrados por eles, obrigatoriamente sobre TLS, recebendo POST com conteúdo em JSON assinado.
- Integração interna com o serviço de pedidos, no ponto de mudança de status. Essa é a única alteração em código existente.
- Nenhuma integração com serviço externo de terceiros. Nenhum provedor de fila, de email ou de notificação.

### Decisões e trade-offs

#### Decisão: registrar o evento no próprio banco, na mesma transação da mudança de status

- **Justificativa:** garante que status alterado e evento registrado sejam inseparáveis, sem introduzir infraestrutura nova em um time pequeno. Ver [ADR-001](./adrs/ADR-001-padrao-outbox-no-mysql.md).
- **Trade-off:** as tabelas de eventos convivem no mesmo banco transacional de pedidos, o que cria uma responsabilidade operacional nova de arquivamento periódico, hoje sem política definida.

#### Decisão: entregar por um processo separado que verifica eventos pendentes a cada 2 segundos

- **Justificativa:** o MySQL não tem mecanismo nativo de notificação de processo externo, ao contrário do PostgreSQL, e verificar a cada 2 segundos atende o requisito de 10 segundos com folga. Manter o processo separado impede que uma reinicialização da API interrompa entregas. Ver [ADR-002](./adrs/ADR-002-worker-em-processo-separado-com-polling.md).
- **Trade-off:** a latência mínima passa a ser de até 2 segundos, e a operação passa a exigir dois processos rodando em todos os ambientes. Quem rodar apenas a API não terá webhooks ativos.

#### Decisão: 5 tentativas com intervalos crescentes de 1m, 5m, 30m, 2h e 12h, depois fila de falhas

- **Justificativa:** cobre indisponibilidade real de 2 horas já observada na base de clientes, o que 3 tentativas não fariam, e evita evento pendurado indefinidamente, o que a retentativa sem limite causaria. Ver [ADR-003](./adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md).
- **Trade-off:** um evento pode permanecer até cerca de 15 horas na fila antes de ser tratado como falha permanente, o que aumenta o volume da tabela em cenários de degradação prolongada.

#### Decisão: assinar com HMAC-SHA256 usando secret única por endpoint, com rotação de 24 horas

- **Justificativa:** permite ao cliente verificar origem e integridade, é padrão de mercado com biblioteca disponível em qualquer linguagem, e isola o impacto de um vazamento a um único endpoint, motivado por caso real. Ver [ADR-004](./adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md).
- **Trade-off:** a responsabilidade de guardar a secret com segurança passa a ser do cliente, e o período de 24 horas cria uma janela em que duas secrets são válidas ao mesmo tempo.

#### Decisão: garantir entrega ao menos uma vez, com descarte de duplicatas pelo cliente

- **Justificativa:** garantir entrega exatamente uma vez exigiria coordenação dos dois lados e complexidade desproporcional. O padrão adotado é o mesmo de Stripe e GitHub, o que reduz resistência de adoção e resolve a grande maioria dos casos. Ver [ADR-005](./adrs/ADR-005-garantia-at-least-once-com-x-event-id.md).
- **Trade-off:** transfere responsabilidade ao cliente. A ressalva é reconhecida, e a documentação do comportamento no portal do desenvolvedor mitiga o risco.

#### Decisão: filtrar eventos no momento do registro, e não no momento do envio

- **Justificativa:** se nenhum endpoint do cliente quer aquele status, não faz sentido criar o registro. Economiza linha na tabela.
- **Trade-off:** mudar a assinatura de status de um endpoint não afeta eventos já registrados, e desativar um endpoint interrompe imediatamente a criação de novos eventos para ele.

#### Decisão: reaproveitar ao máximo os padrões já existentes no projeto

- **Justificativa:** a codebase já tem estrutura de módulos, classes de erro, tratamento centralizado de erro, padrão de validação e de identificadores. Reaproveitar reduz risco e tempo: o middleware de erro centralizado já trata `AppError`, Zod e Prisma, capturando os novos erros sem precisar mudar nada.
- **Trade-off:** nenhum trade-off relevante. A decisão é aderência ao padrão já vigente, sem alternativa em disputa, e por isso não gerou ADR próprio.

#### Decisão: guardar o conteúdo do evento como fotografia do instante da transição

- **Justificativa:** se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. O contrário produziria casos estranhos.
- **Trade-off:** o conteúdo é duplicado no registro do evento em vez de derivado do pedido no envio, o que aumenta o volume armazenado.

---

### Dependências

#### Organizacional: revisão de segurança antes do deploy

Há no mínimo 2 dias úteis reservados para revisão do código de segurança antes da subida, com foco na assinatura HMAC e na geração de secret. A estimativa de três sprints já inclui essa revisão. Esta dependência é um gate de processo e não deve ser cortada sob pressão de prazo.

#### Organizacional: documentação no portal do desenvolvedor

A documentação do comportamento da integração para os clientes é responsabilidade de produto, com destaque para a possibilidade de receber o mesmo evento duas vezes e para a necessidade de descartar duplicatas. Sem isso, a decisão de entrega ao menos uma vez vira surpresa para o cliente em produção.

#### Técnica: operação de um segundo processo em todos os ambientes

A feature só funciona completamente com o processo de entrega em execução, além da API. Desenvolvimento, homologação e produção precisam executá-lo. Isso exige ajuste de deploy, de supervisão e de guia de setup local. Um desenvolvedor que rode apenas a API não terá webhooks ativos.

#### Técnica: decisões pendentes antes do início ou durante a implementação

Onze pontos seguem sem decisão e afetam trechos específicos da implementação. A lista completa está na seção 9 do [FDD](./FDD.md). Os mais críticos: recuperação de eventos travados após queda do processo de entrega, comprimento e entropia da secret, e escolha do tipo de identificador do evento.

#### Externa: implementação da validação de assinatura pelos clientes B2B

Os clientes precisam implementar a verificação da assinatura do lado deles. A escolha de HMAC-SHA256 foi motivada exatamente por isso: qualquer cliente sério já dispõe de biblioteca para o algoritmo.

#### Externa: implementação do descarte de duplicatas pelos clientes B2B

Os clientes precisam descartar eventos repetidos usando o identificador do evento. Um cliente que não fizer isso pode processar o mesmo evento mais de uma vez. É consequência direta e aceita da garantia de entrega ao menos uma vez.

#### Externa: exposição de endpoint sobre TLS pelos clientes B2B

Cada cliente precisa expor uma URL https válida. URLs sem https são recusadas no cadastro.

---

### Riscos e mitigação

#### A alteração no fluxo de mudança de status pode causar regressão no caminho mais crítico do OMS

- **Probabilidade:** media
- **Impacto:** alto. A mudança de status concentra transição de estado, ajuste de estoque e auditoria. Uma regressão afeta o núcleo do produto, não apenas a feature nova. É a única alteração em código existente.
- **Mitigação:**
  - Acrescentar um único passo à transação, sem tocar em validação, estoque ou histórico.
  - Usar uma função que recebe a transação em andamento em vez de injetar dependência maior no serviço de pedidos, limitando a superfície da mudança.
  - Reservar meia sprint especificamente para a integração e os testes ponta a ponta.
  - Usar a suíte de testes de pedidos já existente como linha de base de não regressão.
- **Plano de contingência:** desativar todos os endpoints cadastrados. Como a filtragem acontece no registro, isso reduz a publicação a uma consulta que não retorna nada, sem tocar no restante da transação.

#### O prazo de três sprints pode não caber na janela comercial da Atlas

- **Probabilidade:** media
- **Impacto:** alto. A Atlas declarou que pode migrar para o concorrente se a entrega não sair até o fim do trimestre. Perder o cliente também sinaliza mal para MaxDistribuição e Nova Cargo.
- **Mitigação:**
  - Manter o escopo enxuto, com email, painel e limitação de envio explicitamente fora.
  - Trabalhar com a decomposição já acordada, que distribui modelagem, processo de entrega, cadastro, integração e segurança ao longo das três sprints.
  - Manter alinhamento contínuo de expectativa com os clientes durante a execução.
- **Plano de contingência:** não cortar a revisão de segurança. Se houver necessidade de corte, o candidato natural é o histórico de entregas, que é o único requisito sem dependência de outro componente.

#### Uma falha na assinatura ou na geração de secret comprometeria a confiança no canal

- **Probabilidade:** baixa
- **Impacto:** alto. Um erro aqui compromete a confiança dos clientes B2B no canal inteiro. Já existe caso real de secret vazada em log de cliente.
- **Mitigação:**
  - Reservar os 2 dias úteis de revisão de segurança antes da subida, tratando-os como obrigatórios.
  - Usar secret única por endpoint, isolando o impacto de um vazamento.
  - Estender a lista de campos ocultados em log para incluir as secrets.
  - Resolver as pendências de comprimento e entropia da secret durante a revisão, e não depois dela.
- **Plano de contingência:** rotação forçada da secret do endpoint afetado, usando o mecanismo de rotação com 24 horas de validade paralela que já faz parte do escopo.

#### As tabelas de eventos podem crescer sem controle

- **Probabilidade:** alta
- **Impacto:** medio. As tabelas convivem no banco transacional de pedidos. Um evento pode ficar até cerca de 15 horas na fila antes de virar falha permanente, e não há política de retenção definida nem para os eventos entregues nem para a fila de falhas.
- **Mitigação:**
  - Criar os índices necessários desde a primeira migração.
  - Ler os eventos pendentes em lotes pequenos.
  - Definir política de retenção antes da operação em produção, resolvendo as pendências correspondentes.
- **Plano de contingência:** expurgo manual de eventos entregues mais antigos que 30 dias, janela de referência, executado como tarefa operacional pontual até haver política formal.

#### Um cliente pode não implementar o descarte de duplicatas e processar o mesmo evento mais de uma vez

- **Probabilidade:** media
- **Impacto:** medio. Consequência direta e aceita da garantia de entrega ao menos uma vez.
- **Mitigação:**
  - Manter o identificador do evento estável entre todas as tentativas, tornando o descarte determinístico.
  - Documentar o comportamento de forma destacada no portal do desenvolvedor.
  - Apoiar-se no precedente de mercado de Stripe e GitHub para reduzir resistência de adoção.
- **Plano de contingência:** nenhum do lado da plataforma. A entrega exatamente uma vez foi descartada por exigir coordenação dos dois lados.

#### Um cliente pode ser bombardeado com muitas chamadas em sequência

- **Probabilidade:** media
- **Impacto:** medio. Cenário concreto: um cliente com 50 pedidos mudando de status em um minuto recebe 50 chamadas.
- **Mitigação:**
  - Risco aceito conscientemente nesta fase. A decisão foi observar e agir se virar problema.
  - O intervalo crescente entre tentativas já espaça os reenvios, embora não as primeiras entregas.
- **Plano de contingência:** desativar temporariamente o endpoint do cliente afetado, o que interrompe a criação de novos eventos para ele pela filtragem no registro.

#### Eventos podem ficar presos se o processo de entrega cair no meio de uma tentativa

- **Probabilidade:** media
- **Impacto:** medio. Eventos ficariam presos indefinidamente, sem serem entregues nem virarem falha permanente, e sem erro visível.
- **Mitigação:**
  - Encerramento controlado do processo, devolvendo à fila os eventos em andamento.
  - Essa medida é insuficiente sozinha, porque não cobre queda abrupta. A definição do comportamento nesse caso é uma pendência aberta.
- **Plano de contingência:** consulta manual por eventos presos e devolução deles à fila, até que a pendência seja resolvida.

---

### Critérios de aceitação

Checklist objetivo que define se a feature está pronta.

- Uma mudança de status para um status assinado por um endpoint ativo gera exatamente um evento por endpoint assinante.
- Uma mudança de status sem nenhum endpoint assinante daquele status não gera nenhum evento.
- Uma transação de mudança de status desfeita por qualquer motivo não deixa nenhum evento registrado.
- Uma falha ao registrar o evento desfaz a mudança de status e o histórico do pedido.
- O conteúdo do evento reflete o estado do pedido no instante da transição, mesmo que o pedido seja alterado depois.
- O tempo entre a confirmação da mudança de status e o primeiro envio ao cliente é inferior a 10 segundos em 100 por cento dos casos, com o processo de entrega saudável.
- Um endpoint de cliente que demora 30 segundos para responder não afeta a latência da mudança de status nem a de outros pedidos.
- Um endpoint que responde com erro recebe exatamente 5 tentativas, nos intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas.
- Um endpoint que não responde em 10 segundos tem a tentativa tratada como falha e uma nova tentativa agendada.
- Após a quinta falha, o evento aparece na fila de falhas com conteúdo, motivo e horário preservados, e sai da fila ativa.
- Um reprocessamento solicitado por usuário sem papel de administrador é recusado, e o mesmo pedido feito por um administrador recoloca o evento na fila.
- Todo reprocessamento fica registrado com a identificação de quem o executou.
- Todas as tentativas de um mesmo evento carregam o mesmo identificador de evento.
- A assinatura enviada é verificável com a secret do endpoint e falha se um único byte do conteúdo for alterado.
- Após uma rotação de secret, a secret anterior continua válida por 24 horas e deixa de valer depois disso.
- Um cadastro com URL sem https é recusado na validação, antes de qualquer persistência.
- Um evento acima de 64KB não é enviado e é tratado como erro, sem truncamento.
- O histórico de entregas retorna, para cada tentativa, indicação de sucesso ou falha, conteúdo enviado, resposta recebida e tempo de resposta.
- Com um único processo de entrega em execução, os eventos de um mesmo pedido chegam ao cliente na ordem em que ocorreram.
- A queda da API não interrompe entregas em andamento no processo de entrega.
- Nenhuma secret e nenhuma assinatura aparece em log, em nenhum nível de detalhamento.
- Todo registro de log do processo de entrega contém o identificador do evento, permitindo reconstruir o ciclo completo de tentativas.
- Nenhuma rota ou contrato existente da API teve comportamento alterado.
- Nenhuma dependência nova foi adicionada ao projeto.
- A revisão de segurança foi concluída antes do deploy, com no mínimo 2 dias úteis dedicados.
- O comportamento de entrega ao menos uma vez está documentado no portal do desenvolvedor.

---

### Testes e validação

Tipos de teste obrigatórios

- Testes de integração para o fluxo principal de publicação do evento dentro da transação de mudança de status, cobrindo o caso em que a transação é desfeita e o caso em que nenhum endpoint assinou o status. Este é o ponto de maior risco da entrega.
- Testes de integração para o cadastro, edição, remoção e listagem de endpoints, incluindo a recusa de URL sem https.
- Testes unitários para a regra de cálculo dos intervalos entre tentativas e para a transição do evento até a fila de falhas.
- Testes unitários para a geração e a verificação da assinatura, incluindo a rejeição de conteúdo adulterado.
- Testes de integração para o ciclo completo de tentativas, incluindo a chegada à fila de falhas e o reprocessamento.
- Teste de segurança de controle de acesso, verificando que o reprocessamento é recusado para usuário sem papel de administrador e aceito para administrador.
- Teste de comportamento sob endpoint lento, verificando que o tempo limite de 10 segundos é respeitado e que a mudança de status não é afetada.
- Revisão manual de segurança, com no mínimo 2 dias úteis, focada na assinatura e na geração de secret. Este item não é automatizável e é obrigatório antes do deploy.

Estratégia de validação

- Reaproveitar a suíte de testes já existente no projeto, que roda contra um banco MySQL real, sem simulação, em processo único e com limpeza de tabelas entre os testes. Os testes da feature seguem esse mesmo padrão.
- Usar os testes de pedidos já existentes como linha de base de não regressão para o fluxo de mudança de status, que é o principal risco da entrega.
- Validar o ciclo de tentativas com um endpoint controlado que simule os cenários reais previstos: erro persistente, tempo esgotado e indisponibilidade temporária seguida de recuperação.
- Submeter a feature à revisão técnica das engenharias de Pedidos e Plataforma antes do início da implementação.
- Concluir com a revisão de segurança como último passo antes do deploy.
- **[PENDÊNCIA]** Não foi definida necessidade de teste de carga. Não há definição de volume esperado de eventos por minuto, e o cenário de 50 chamadas em um minuto foi levantado sem meta associada.

---

### Pendências consolidadas

Pontos sem decisão fechada e sem base verificável no código. Nenhum valor foi inventado para preenchê-los.

| # | Pendência | Onde aparece | Referência |
| --- | --- | --- | --- |
| 1 | Meta de entrega bem sucedida sem intervenção manual e volume aceitável na fila de falhas | Objetivos e métricas | Não decidido |
| 2 | Meta de latência p95 para os endpoints de configuração | Requisitos não funcionais, Performance | Não decidido |
| 3 | Meta de disponibilidade da plataforma e do processo de entrega | Requisitos não funcionais, Disponibilidade | Não decidido |
| 4 | Comprimento mínimo e fonte de entropia da secret | RF-004, Segurança | Não fechado, ver ADR-004 |
| 5 | Política de exposição da secret após a criação | Segurança | Não fechado, ver ADR-004 |
| 6 | Comportamento do cliente que usa a secret anterior após as 24 horas | RF-004 | Não decidido, ver ADR-004 |
| 7 | Contador de tentativas após reprocessamento manual | RF-009 | Não decidido, ver ADR-003 |
| 8 | Política de retenção da fila de falhas | RF-008, Riscos | Não decidido, ver ADR-003 |
| 9 | Política de retenção e arquivamento dos eventos entregues | Riscos | Declarado fora do escopo |
| 10 | Recuperação de eventos presos após queda abrupta do processo de entrega | Riscos, Dependências | Não decidido, ver ADR-002 |
| 11 | Instrumentação de métricas e tracing | Requisitos não funcionais, Observabilidade | Não decidido, ausente do código |
| 12 | Requisitos regulatórios sobre envio de dados de pedidos para fora da plataforma | Requisitos não funcionais, Compliance | Não decidido |
| 13 | Necessidade de teste de carga e volume esperado de eventos | Testes e validação | Levantado sem meta |

Os itens 4 a 10 têm rastreabilidade direta aos ADRs e ao RFC, onde já constam como questões em aberto. Os itens 1, 2, 3, 11, 12 e 13 são lacunas de produto identificadas na elaboração deste PRD e não aparecem nos demais documentos.
