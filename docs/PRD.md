### PRD: OMS Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-07-15
Responsável: Marcos (Product Manager), que abriu o contexto da feature `[09:00]`, levantou os requisitos funcionais `[09:31]`, assumiu a documentação no portal do desenvolvedor `[09:26]` e é o ponto de contato com os clientes `[09:47]`

> **Convenção de rastreabilidade**
> `[hh:mm]` referencia a transcrição da reunião técnica (`TRANSCRICAO.md`). Caminhos como `src/...` referenciam código existente e verificado.
> **[PENDÊNCIA]**x marca ponto sem origem na transcrição nem no código. Seguindo a regra do desafio, nenhum valor foi inventado para preencher esses espaços. A lista consolidada está no fim do documento.
> Documentos relacionados: [RFC](./RFC.md), [FDD](./FDD.md), [ADRs](./adrs/), [Tracker](./TRACKER.md).

---

### Resumo

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para serem notificados em tempo real quando o status dos pedidos deles muda na plataforma `[09:00]`. Hoje eles descobrem mudanças fazendo polling repetido contra `GET /orders`, o que torna a integração lenta e cara do lado deles `[09:00]`. A Atlas sinalizou que pode migrar para o concorrente se a entrega não sair até o fim do trimestre `[09:00]`.

Esta feature entrega webhooks de saída: quando o status de um pedido muda, a plataforma envia um POST assinado para uma URL cadastrada pelo cliente, em menos de 10 segundos. O cliente cadastra e gerencia seus endpoints via API, escolhe quais status quer receber, e consulta o histórico de entregas. Falhas são retentadas automaticamente ao longo de cerca de 15 horas antes de irem para uma fila de falhas com reprocessamento manual.

O escopo é estritamente de saída: a plataforma envia, os clientes recebem `[09:02]`. Não há painel visual nem notificação por email nesta fase.

---

### Contexto e problema

Público-alvo

- Clientes B2B integrados via API que precisam reagir a mudanças de status de pedidos: Atlas Comercial, MaxDistribuição e Nova Cargo `[09:00]`.
- Times de desenvolvimento desses clientes, que constroem e mantêm a integração do lado deles e passam a ser responsáveis por validar assinatura e deduplicar eventos `[09:25]`.
- Operação interna da plataforma, que ganha uma nova superfície para diagnosticar e reprocessar falhas de entrega `[09:18]`.

Cenários de uso chave

- Um pedido da Atlas muda de PROCESSING para SHIPPED e o sistema da Atlas é notificado em segundos, sem precisar consultar a API `[09:00]`.
- Um cliente cadastra um endpoint e escolhe receber apenas SHIPPED e DELIVERED, ignorando os demais status `[09:34]`.
- O endpoint de um cliente fica indisponível durante uma manutenção planejada de 2 horas e recebe os eventos quando volta, sem perda `[09:16]`.
- Um cliente vazou a secret em log e precisa rotacionar sem parar a integração, migrando os sistemas dele dentro de 24 horas `[09:21]`, `[09:22]`.
- Um cliente que tem vários endpoints cadastrados precisa saber qual deles corresponde a um envio recebido `[09:44]`.
- O suporte investiga por que um cliente não recebeu um evento e consulta o histórico de entregas com payload, resposta e tempo de resposta `[09:34]`.
- Um evento esgotou todas as tentativas e um administrador o reprocessa manualmente após o cliente confirmar que voltou `[09:18]`.

Onde essa feature será implantada

- Sistema existente. A feature entra no OMS atual, um Order Management System em Node.js e TypeScript com Express, Prisma e MySQL. O código já tem módulos de autenticação, usuários, clientes, produtos e pedidos, com máquina de estados de pedido, controle transacional de estoque e auditoria de mudanças de status.
- A aplicação não possui hoje nenhum mecanismo de notificação externa, eventos ou filas. Essa é exatamente a lacuna que a feature preenche.
- A entrega acrescenta um segundo processo de longa duração em todos os ambientes: além da API, passa a existir um worker independente `[09:11]`. Nenhuma infraestrutura nova é introduzida: a feature usa o MySQL já operacional, sem Redis nem broker externo `[09:07]`.

Problemas priorizados

- **Polling ineficiente dos clientes B2B.** Os três clientes consultam `GET /orders` repetidamente para descobrir mudanças. Impacto: integração lenta e cara do lado deles, e carga desnecessária na plataforma. Prioridade: alta `[09:00]`.
- **Risco comercial concreto de churn.** A Atlas Comercial declarou que pode migrar para o concorrente se a entrega não sair até o fim do trimestre. Impacto: perda de receita de um cliente B2B e sinal negativo para os outros dois. Prioridade: alta `[09:00]`.
- **Atualização manual do lado do cliente.** Sem notificação, os clientes precisam ficar atualizando para saber o que mudou. Impacto: atraso operacional na cadeia deles, que depende do status para agir. Prioridade: alta `[09:02]`.
- **Ausência de qualquer canal de eventos na plataforma.** Não existe base sobre a qual construir. Impacto: qualquer necessidade futura de notificação externa esbarra na mesma lacuna. Prioridade: média.

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Notificar mudanças de status em tempo real, conforme a definição dos clientes | Tempo entre o commit da mudança de status e o primeiro POST ao endpoint do cliente | Abaixo de 10 segundos em 100% dos casos, com worker saudável. Alvo prático de até 2 segundos pelo intervalo de polling do worker `[09:02]`, `[09:09]` |
| Eliminar o polling dos clientes B2B contra `GET /orders` | Número de clientes B2B migrados para webhooks | 3 (Atlas Comercial, MaxDistribuição e Nova Cargo) `[09:00]` |
| Reter a Atlas Comercial, evitando a migração para o concorrente | Entrega em produção dentro da janela comercial acordada | Fim de novembro, dentro do trimestre `[09:00]`, `[09:45]` |
| Não degradar a operação de mudança de status, que é o caminho crítico do OMS | Latência de `PATCH /orders/:id/status` comparada ao baseline atual | Sem aumento mensurável acima do baseline `[09:04]` |
| Absorver indisponibilidade real dos clientes sem perder eventos | Janela de indisponibilidade do cliente coberta pelo ciclo de retry | Cerca de 15 horas, com 5 tentativas em 1m, 5m, 30m, 2h e 12h `[09:17]` |
| Entregar dentro da capacidade estimada do time | Sprints consumidas, incluindo a revisão de segurança | 3 sprints `[09:46]`, `[09:47]` |

> **[PENDÊNCIA]** Não existe meta definida para taxa de entrega bem sucedida sem intervenção manual, nem para volume aceitável de eventos na fila de falhas. A reunião não tratou de metas operacionais em regime, apenas do comportamento em caso de falha. Definir antes da operação em produção.

---

### Escopo

Incluso

- Cadastro de endpoint de webhook por cliente, com URL, lista de status a receber e secret gerada pela plataforma `[09:31]`.
- Listagem, edição e remoção de endpoints de webhook `[09:33]`.
- Filtro de eventos por status: cada endpoint escolhe quais transições quer receber `[09:33]`, `[09:34]`.
- Rotação de secret pela API, com a secret anterior válida por 24 horas em paralelo `[09:21]`.
- Publicação do evento de mudança de status de forma atômica com a própria mudança `[09:40]`.
- Entrega do evento por POST assinado com HMAC-SHA256, com timeout de 10 segundos `[09:20]`, `[09:42]`.
- Retentativa automática com backoff exponencial, 5 tentativas em 1m, 5m, 30m, 2h e 12h `[09:17]`.
- Fila de falhas permanentes (dead letter) com payload, motivo e timestamp preservados `[09:18]`.
- Reprocessamento manual de itens da fila de falhas, restrito a administradores e auditado `[09:35]`, `[09:36]`.
- Histórico de entregas por endpoint, com sucesso ou falha, payload, resposta e tempo de resposta `[09:34]`.
- Validação de TLS obrigatório na URL cadastrada e limite de 64KB por payload `[09:23]`, `[09:24]`.

Fora de escopo

- **Notificação por email ao cliente quando o webhook dele está falhando.** Marcos perguntou explicitamente e a resposta foi negativa: "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto" `[09:37]`.
- **Dashboard visual para o cliente acompanhar os webhooks dele.** Descartado nesta fase: "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend" `[09:40]`.
- **Rate limiting de saída.** Diego levantou o cenário de 50 pedidos mudando de status em um minuto e a decisão foi adiar: "A gente observa e implementa se virar problema" `[09:38]`, `[09:39]`.
- **Webhooks de entrada (inbound).** Sofia delimitou o escopo logo no início e Marcos confirmou que os clientes querem apenas receber `[09:02]`.
- **Escala horizontal do worker.** Múltiplos workers em paralelo removeriam a garantia de ordenação. Adiado como "problema do futuro" `[09:13]`.
- **Arquivamento ou expurgo de eventos já entregues.** Citado como algo a fazer "depois de 30 dias ou assim", mas declarado fora do escopo desta feature `[09:08]`.
- **Detalhes completos do pedido no payload.** Os itens do pedido não são enviados, para manter o payload enxuto. O cliente consulta `GET /orders/:id` se precisar `[09:43]`.
- **Endurecimento do controle de acesso no cadastro de webhooks.** Por ora qualquer usuário autenticado configura endpoints. Sofia aceitou e adiou: "Por enquanto sim. Mais pra frente a gente pode endurecer" `[09:37]`.

> Nenhum item de "Fora de escopo" contradiz o que está incluso. O reprocessamento manual está incluso e é justamente o que substitui o email de aviso, que ficou fora.

---

### Requisitos funcionais

#### FR-001 Cadastrar endpoint de webhook

O cliente registra uma URL que passará a receber notificações de mudança de status dos pedidos dele.

**Fluxo principal**

- Usuário autenticado envia a requisição de cadastro com o identificador do cliente, a URL de destino e a lista de status que deseja receber `[09:31]`.
- O sistema valida que a URL usa https `[09:23]`.
- O sistema gera uma secret e a associa ao endpoint `[09:31]`.
- O sistema persiste o endpoint como ativo e devolve o cadastro com a secret.

**Fluxos alternativos e exceções**

- O identificador do cliente vem no corpo ou no caminho da requisição, e não do token de autenticação, porque o token representa o usuário operador e não o cliente `[09:32]`.
- Qualquer usuário autenticado pode cadastrar nesta fase, sem exigência de papel específico `[09:37]`.
- Um mesmo cliente pode ter mais de um endpoint cadastrado, e cada envio identifica qual endpoint o originou `[09:44]`.

**Erros previstos**

- URL sem https é recusada na validação, antes de qualquer persistência `[09:23]`.
- Corpo inválido ou lista de status com valor inexistente é recusado na validação.
- Requisição sem autenticação é recusada.

**Prioridade:** alta

---

#### FR-002 Listar, editar e remover endpoints de webhook

O cliente gerencia os endpoints já cadastrados.

**Fluxo principal**

- Usuário autenticado consulta os endpoints de um cliente `[09:33]`.
- Usuário edita um endpoint para alterar a URL, a lista de status ou o estado de ativação `[09:33]`.
- Usuário remove um endpoint que não deseja mais `[09:33]`.

**Fluxos alternativos e exceções**

- A secret nunca é devolvida em listagem, apenas no cadastro e na rotação.
- Desativar um endpoint interrompe a geração de novos eventos para ele, porque a filtragem acontece no momento da inserção `[09:34]`.

**Erros previstos**

- Endpoint inexistente retorna erro de recurso não encontrado.
- Edição que introduza URL sem https é recusada `[09:23]`.

**Prioridade:** alta

---

#### FR-003 Filtrar eventos por status

Cada endpoint recebe apenas as transições de status que assinou.

**Fluxo principal**

- No cadastro ou na edição, o cliente informa a lista de status que quer ouvir, por exemplo apenas SHIPPED e DELIVERED `[09:33]`.
- Quando um pedido muda de status, o sistema consulta quais endpoints daquele cliente assinaram aquele status `[09:34]`.
- O evento é criado apenas para os endpoints que assinaram.

**Fluxos alternativos e exceções**

- Se nenhum endpoint do cliente assinou aquele status, nenhum evento é criado. Isso não é erro, é o caminho esperado, e economiza registro na base `[09:34]`.
- A filtragem ocorre no momento da inserção, e não no momento do envio, por decisão explícita `[09:34]`.

**Erros previstos**

- Status inexistente na lista de assinatura é recusado na validação.

**Prioridade:** alta

---

#### FR-004 Gerar e rotacionar a secret de assinatura

Cada endpoint tem uma secret própria, usada para assinar os envios, e o cliente pode trocá-la sem parar a integração.

**Fluxo principal**

- A secret é gerada pela plataforma no cadastro do endpoint e devolvida ao cliente `[09:31]`.
- O cliente solicita rotação pela API `[09:21]`.
- O sistema gera uma nova secret vigente e mantém a anterior válida por 24 horas `[09:21]`.
- Após 24 horas a secret anterior deixa de valer `[09:21]`.

**Fluxos alternativos e exceções**

- Cada endpoint tem secret única. Não existe secret global da plataforma, para que o vazamento de uma não comprometa as demais: "Senão se vaza uma, vaza tudo" `[09:21]`.
- O período de 24 horas existe para o cliente migrar os sistemas dele de forma escalonada, sem coordenação em tempo real `[09:21]`.

**Erros previstos**

- Rotação sobre endpoint inexistente retorna erro de recurso não encontrado.
- **[PENDÊNCIA]** O comportamento para o cliente que continua usando a secret anterior depois das 24 horas não foi definido na reunião.

**Prioridade:** alta

---

#### FR-005 Publicar o evento de forma atômica com a mudança de status

Quando o status de um pedido muda, o evento correspondente é registrado na mesma transação.

**Fluxo principal**

- A mudança de status é solicitada e a transação existente é aberta.
- O sistema valida a transição, ajusta o estoque, grava o novo status e registra o histórico, exatamente como já faz hoje.
- Dentro da mesma transação, o sistema registra o evento a notificar `[09:06]`, `[09:40]`.
- A transação confirma. O evento passa a existir e fica disponível para entrega.
- A resposta ao chamador não espera nenhuma entrega de webhook.

**Fluxos alternativos e exceções**

- Se qualquer passo falhar, incluindo o registro do evento, tudo é desfeito e nenhum evento existe: "Se a outbox falhar de inserir, rollback. Não pode ter caso de status mudar e evento não sair" `[09:40]`.
- O conteúdo do evento é uma fotografia do pedido no instante da transição, e não é recalculado no envio. Se o pedido mudar depois, o evento continua refletindo o momento em que o status mudou `[09:52]`.
- A indisponibilidade de um cliente nunca desfaz uma mudança de status: "se o cliente tiver fora do ar, o que a gente faz, dá rollback na mudança de status? Não dá" `[09:04]`.

**Erros previstos**

- Falha ao registrar o evento desfaz a mudança de status, por decisão explícita `[09:40]`.

**Prioridade:** alta

---

#### FR-006 Entregar o evento assinado ao endpoint do cliente

O sistema envia o evento e permite ao cliente verificar que ele veio mesmo da plataforma.

**Fluxo principal**

- Um processo independente busca periodicamente os eventos pendentes, a cada 2 segundos `[09:09]`.
- Para cada evento, o sistema assina o conteúdo com HMAC-SHA256 usando a secret do endpoint `[09:20]`.
- O sistema envia o POST com os cabeçalhos de identificação do evento, assinatura, horário de envio e identificação do endpoint `[09:44]`, `[09:45]`.
- Qualquer resposta de sucesso encerra o evento como entregue, e a tentativa é registrada no histórico `[09:34]`.

**Fluxos alternativos e exceções**

- A entrega é garantida ao menos uma vez. O cliente pode receber o mesmo evento duas vezes e deve descartar duplicatas usando o identificador do evento `[09:24]`, `[09:25]`.
- O identificador do evento é gerado no registro e nunca muda entre tentativas, o que torna o descarte de duplicatas determinístico `[09:25]`.
- A responsabilidade de descartar duplicatas é do cliente. Sofia registrou a ressalva e Diego justificou pelo padrão de mercado: "Stripe faz assim, GitHub faz assim" `[09:25]`.
- Enquanto houver um único processo de entrega, os eventos de um mesmo pedido chegam na ordem em que ocorreram. Não há garantia de ordenação global, e os clientes não pediram isso `[09:12]`, `[09:14]`.
- O conteúdo enviado não inclui os itens do pedido, para não inflar o envio `[09:43]`.

**Erros previstos**

- Evento acima de 64KB não é enviado e é tratado como erro, sem truncar: "Se chegou nesse tamanho, tem algo errado" `[09:23]`, `[09:24]`.
- Cliente que não responde em 10 segundos é tratado como falha e entra no ciclo de retentativa `[09:42]`.

**Prioridade:** alta

---

#### FR-007 Retentar entregas que falharam

Falhas temporárias do cliente não causam perda do evento.

**Fluxo principal**

- Uma entrega falha por resposta de erro, tempo esgotado ou problema de rede.
- O sistema registra a tentativa e agenda a próxima segundo o intervalo previsto.
- Os intervalos crescem progressivamente: 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, somando cerca de 15 horas `[09:17]`.
- Se alguma tentativa der certo, o evento é encerrado como entregue.

**Fluxos alternativos e exceções**

- São 5 tentativas. A proposta de 3 foi recusada porque não cobre indisponibilidade real de 2 horas em manutenção planejada, que já aconteceu com clientes da base `[09:16]`.
- A retentativa indefinida também foi recusada, para não deixar evento pendurado para sempre caso o cliente suma `[09:15]`.
- A janela de 15 horas foi validada pelo negócio: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele. Acho aceitável" `[09:17]`.

**Erros previstos**

- Esgotadas as 5 tentativas, o evento é tratado como falha permanente e sai da fila ativa `[09:17]`.

**Prioridade:** alta

---

#### FR-008 Preservar falhas permanentes em fila de falhas

Eventos que esgotaram as tentativas não são descartados.

**Fluxo principal**

- Após a quinta falha, o evento é movido para uma fila de falhas separada `[09:18]`.
- O registro guarda o conteúdo do evento, o motivo da falha e o horário `[09:18]`.

**Fluxos alternativos e exceções**

- A fila de falhas é uma estrutura separada da fila principal, e não uma marcação nela, para manter a leitura da fila ativa limpa e servir de evidência para diagnóstico `[09:18]`.

**Erros previstos**

- Nenhum erro previsto para o cliente. É um estado terminal interno, visível para a operação.
- **[PENDÊNCIA]** Não há política definida de retenção para a fila de falhas. Sem isso, ela cresce indefinidamente.

**Prioridade:** alta

---

#### FR-009 Reprocessar manualmente um item da fila de falhas

A operação consegue reenviar um evento depois que o cliente se recuperou.

**Fluxo principal**

- Um administrador solicita o reprocessamento de um item específico da fila de falhas `[09:35]`.
- O sistema recoloca o evento na fila ativa como pendente `[09:18]`.
- O sistema registra quem executou a ação, para auditoria `[09:36]`.

**Fluxos alternativos e exceções**

- O reprocessamento é manual e intencional, e não automático, para evitar reenvio contínuo a destinos permanentemente inválidos `[09:18]`.
- Exige papel de administrador. Sofia foi explícita: "Mexer em fila de entrega de notificação não é coisa de operador" `[09:36]`.
- O controle de acesso por papel já existente no sistema é reaproveitado, sem construir nada novo `[09:36]`.

**Erros previstos**

- Usuário sem papel de administrador é recusado `[09:36]`.
- Item inexistente na fila de falhas retorna erro de recurso não encontrado.
- **[PENDÊNCIA]** Não foi definido se o contador de tentativas reinicia ou mantém o histórico acumulado após o reprocessamento.

**Prioridade:** média

---

#### FR-010 Consultar o histórico de entregas

O cliente enxerga o que a plataforma tentou entregar para ele.

**Fluxo principal**

- Usuário autenticado consulta o histórico de um endpoint `[09:34]`.
- O sistema retorna as entregas com indicação de sucesso ou falha, conteúdo enviado, resposta recebida e tempo de resposta `[09:34]`.

**Fluxos alternativos e exceções**

- O pedido original de Marcos foi: "esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta" `[09:34]`.
- Cada tentativa de um mesmo evento aparece como um registro próprio, permitindo ver o ciclo completo de retentativas.

**Erros previstos**

- Endpoint inexistente retorna erro de recurso não encontrado.

**Prioridade:** média

---

### Requisitos não funcionais

Performance

- Tempo entre a mudança de status e o primeiro envio ao cliente abaixo de 10 segundos. Essa é a definição de tempo real dada pelos próprios clientes: "Pra eles, qualquer coisa abaixo de 10 segundos já é tempo real" `[09:02]`.
- Intervalo de verificação de eventos pendentes de 2 segundos, o que estabelece a latência de pior caso em cerca de 2 segundos e atende o requisito com folga `[09:09]`, `[09:10]`.
- Tempo limite de 10 segundos por chamada ao endpoint do cliente `[09:42]`.
- A operação de mudança de status não pode ficar mais lenta de forma mensurável por causa desta feature. Foi justamente esse o argumento contra a alternativa síncrona `[09:04]`.
- Tamanho máximo de 64KB por evento enviado `[09:24]`.
- **[PENDÊNCIA]** Não há meta de latência p95 definida para os endpoints de configuração de webhook. A reunião só tratou da latência de notificação.

Disponibilidade

- O sistema tolera indisponibilidade do cliente de até cerca de 15 horas sem perder eventos, por meio do ciclo de 5 tentativas `[09:17]`.
- Falha ou reinicialização da API não interrompe o processamento de eventos, porque o worker é um processo separado: "se a API reinicia, perde o worker" `[09:11]`.
- A API continua operando normalmente se o worker estiver fora. Eventos acumulam como pendentes e são entregues quando ele voltar, sem perda.
- **[PENDÊNCIA]** Não existe meta de disponibilidade declarada para a plataforma nem para o worker. A reunião discutiu a indisponibilidade do cliente, nunca a nossa. O prompt de PRD sugere 99.9 por cento como default, mas esse número não tem origem na transcrição nem no código e por isso não foi adotado.

Segurança e autorização

- Toda requisição enviada ao cliente é assinada com HMAC-SHA256 sobre o corpo, para que ele verifique origem e integridade `[09:20]`, `[09:22]`.
- Cada endpoint tem secret única. Secret global foi recusada explicitamente `[09:21]`.
- A secret é rotacionável, com 24 horas de validade paralela da anterior `[09:21]`.
- TLS obrigatório. URL sem https é recusada na validação `[09:23]`.
- Endpoints de configuração exigem autenticação. O reprocessamento da fila de falhas exige papel de administrador `[09:36]`, `[09:37]`.
- Toda ação de reprocessamento registra quem a executou, para auditoria `[09:36]`.
- Nenhuma secret pode aparecer em log. A motivação é um caso real: "A gente já teve cliente que vazou secret em log de aplicação dele uma vez" `[09:22]`.
- Revisão de segurança obrigatória antes do deploy, com no mínimo 2 dias úteis reservados, focada na assinatura e na geração de secret `[09:46]`.
- **[PENDÊNCIA]** Comprimento mínimo e fonte de entropia da secret não foram definidos. **[PENDÊNCIA]** Não foi definido se a secret é recuperável após a criação ou entregue apenas uma vez.

Observabilidade

- Registro estruturado de cada tentativa de entrega, com o identificador do evento presente em todos os registros, permitindo reconstruir o ciclo completo de um evento, incluindo todas as retentativas.
- O registro de log já existente no projeto é reaproveitado, sem introduzir biblioteca nova: "o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo" `[09:29]`.
- A lista de campos sensíveis que já são ocultados em log precisa ser estendida para incluir as secrets de webhook.
- Histórico de entregas consultável por endpoint, com resposta e tempo de resposta, funcionando como observabilidade voltada ao cliente `[09:34]`.
- **[PENDÊNCIA]** O projeto não possui nenhuma instrumentação de métricas nem de tracing, e a reunião não tratou do assunto. O prompt de PRD sugere métricas de erro por endpoint e tracing distribuído como default, mas adotar isso exigiria decisão de arquitetura e dependência nova, sem origem na transcrição. Ver seção 7 do FDD.

Confiabilidade e integridade de dados

- O registro do evento acontece na mesma transação da mudança de status. Ou os dois acontecem, ou nenhum acontece: "Garante que se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto. Não tem inconsistência possível" `[09:06]`.
- O controle transacional de estoque já existente permanece intacto. A feature não altera a máquina de estados nem as regras de estoque.
- O conteúdo do evento é uma fotografia do instante da transição e não é recalculado no envio `[09:52]`.
- A entrega é garantida ao menos uma vez, com identificador estável por evento para descarte de duplicatas pelo cliente `[09:24]`, `[09:25]`.
- A ordenação por pedido é garantida enquanto houver um único processo de entrega. Ordenação global não é garantida e não foi pedida `[09:12]`, `[09:14]`.

Compatibilidade e portabilidade

- Nenhuma rota, contrato ou comportamento atual da API muda. A feature apenas acrescenta rotas novas sob o mesmo prefixo versionado já em uso.
- Nenhuma infraestrutura nova. A feature usa o MySQL, o Prisma e o Node já existentes, sem Redis nem broker: "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering" `[09:07]`.
- A feature segue os padrões já consolidados no projeto: mesma estrutura de módulos, mesmas classes de erro, mesmo tratamento centralizado de erro, mesmo padrão de validação e mesmo padrão de identificadores `[09:30]`, `[09:51]`.
- O formato do conteúdo enviado e a garantia de entrega ao menos uma vez tornam-se contrato público a partir do primeiro cliente integrado. Mudar isso depois exigiria versionamento `[09:26]`.

Compliance

- Trilha de auditoria de reprocessamento, com identificação de quem executou a ação, disponível para consulta `[09:36]`.
- A auditoria de mudanças de status já existente no sistema permanece inalterada e continua sendo a fonte de verdade sobre o ciclo de vida do pedido.
- **[PENDÊNCIA]** Não houve discussão sobre requisitos regulatórios relativos ao envio de dados de pedidos para fora da infraestrutura da plataforma. Nada foi assumido a respeito.

Acessibilidade no frontend consumidor

- Não aplicável nesta fase. A entrega é composta apenas por endpoints de API, sem interface visual. O painel para o cliente foi explicitamente colocado fora de escopo e tratado como projeto separado do time de frontend `[09:40]`.

---

### Arquitetura e abordagem

Abordagem

- A feature é implementada no sistema existente, seguindo o padrão de módulos já adotado. Não é um novo serviço.
- O evento é registrado de forma atômica com a mudança de status, usando o banco que já existe como fila. Um processo separado consome esse registro e faz as chamadas HTTP `[09:06]`.
- A comunicação é assíncrona da plataforma para o cliente. Não há chamada síncrona a sistema externo dentro de nenhuma transação `[09:04]`.
- Não há fila dedicada, mensageria, streaming nem cache. A decisão foi deliberada, para não introduzir infraestrutura nova em um time pequeno `[09:07]`.
- O detalhamento técnico está no [RFC](./RFC.md) e no [FDD](./FDD.md).

Componentes

- Módulo de webhooks dentro da aplicação existente, seguindo a mesma estrutura dos demais domínios: "A gente tem um padrão claro na codebase. Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual" `[09:27]`.
- Ponto de publicação do evento dentro do serviço de pedidos, acionado na mudança de status, recebendo a transação em andamento `[09:41]`.
- Processo independente de entrega, com entrada própria e comando próprio de execução, espelhando o padrão do servidor já existente `[09:11]`.
- Quatro estruturas novas no banco: cadastro de endpoints, fila de eventos a entregar, histórico de entregas e fila de falhas permanentes.
- Banco MySQL já existente, sem mudança de versão. O processo de entrega usa a mesma base e a mesma configuração de conexão, em instância própria por ser outro processo `[09:30]`.

Integrações

- Endpoints HTTP dos clientes B2B, cadastrados por eles, obrigatoriamente sobre TLS, recebendo POST com conteúdo em JSON assinado `[09:23]`, `[09:44]`.
- Integração interna com o serviço de pedidos, no ponto de mudança de status. Essa é a única alteração em código existente `[09:40]`.
- Nenhuma integração com serviço externo de terceiros. Nenhum provedor de fila, de email ou de notificação.

### Decisões e trade-offs

#### Decisão: registrar o evento no próprio banco, na mesma transação da mudança de status

- **Justificativa:** garante que status alterado e evento registrado sejam inseparáveis, sem introduzir infraestrutura nova em um time pequeno `[09:06]`, `[09:07]`. Ver [ADR-001](./adrs/ADR-001-padrao-outbox-no-mysql.md).
- **Trade-off:** as tabelas de eventos convivem no mesmo banco transacional de pedidos, o que cria uma responsabilidade operacional nova de arquivamento periódico, hoje sem política definida `[09:08]`.

#### Decisão: entregar por um processo separado que verifica eventos pendentes a cada 2 segundos

- **Justificativa:** o MySQL não tem mecanismo nativo de notificação de processo externo, ao contrário do PostgreSQL, e verificar a cada 2 segundos atende o requisito de 10 segundos com folga `[09:09]`. Manter o processo separado impede que uma reinicialização da API interrompa entregas `[09:11]`. Ver [ADR-002](./adrs/ADR-002-worker-em-processo-separado-com-polling.md).
- **Trade-off:** a latência mínima passa a ser de até 2 segundos, e a operação passa a exigir dois processos rodando em todos os ambientes. Quem rodar apenas a API não terá webhooks ativos.

#### Decisão: 5 tentativas com intervalos crescentes de 1m, 5m, 30m, 2h e 12h, depois fila de falhas

- **Justificativa:** cobre indisponibilidade real de 2 horas já observada na base de clientes, o que 3 tentativas não fariam, e evita evento pendurado para sempre, o que a retentativa indefinida causaria `[09:15]`, `[09:16]`, `[09:17]`. Ver [ADR-003](./adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md).
- **Trade-off:** um evento pode permanecer até cerca de 15 horas na fila antes de ser tratado como falha permanente, o que aumenta o volume da tabela em cenários de degradação prolongada.

#### Decisão: assinar com HMAC-SHA256 usando secret única por endpoint, com rotação de 24 horas

- **Justificativa:** permite ao cliente verificar origem e integridade, é padrão de mercado com biblioteca disponível em qualquer linguagem, e isola o impacto de um vazamento a um único endpoint, motivado por caso real `[09:20]`, `[09:21]`, `[09:22]`. Ver [ADR-004](./adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md).
- **Trade-off:** a responsabilidade de guardar a secret com segurança passa a ser do cliente, e o período de 24 horas cria uma janela em que duas secrets são válidas ao mesmo tempo.

#### Decisão: garantir entrega ao menos uma vez, com descarte de duplicatas pelo cliente

- **Justificativa:** garantir entrega exatamente uma vez exigiria coordenação dos dois lados e complexidade desproporcional. O padrão adotado é o mesmo de Stripe e GitHub, o que reduz resistência de adoção: "At-least-once com event_id resolve 99% dos casos" `[09:25]`. Ver [ADR-005](./adrs/ADR-005-garantia-at-least-once-com-x-event-id.md).
- **Trade-off:** transfere responsabilidade ao cliente. Sofia levantou a ressalva explicitamente, e Marcos assumiu documentar o comportamento de forma destacada no portal `[09:25]`, `[09:26]`.

#### Decisão: filtrar eventos no momento do registro, e não no momento do envio

- **Justificativa:** se nenhum endpoint do cliente quer aquele status, não faz sentido criar o registro. Economiza linha na tabela `[09:34]`.
- **Trade-off:** mudar a assinatura de status de um endpoint não afeta eventos já registrados, e desativar um endpoint interrompe imediatamente a criação de novos eventos para ele.

#### Decisão: reaproveitar ao máximo os padrões já existentes no projeto

- **Justificativa:** a codebase já tem estrutura de módulos, classes de erro, tratamento centralizado de erro, padrão de validação e de identificadores. Reaproveitar reduz risco e tempo: "o middleware de erro centralizado já trata AppError, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada" `[09:29]`, `[09:30]`.
- **Trade-off:** nenhum trade-off relevante foi levantado na reunião. A decisão foi tomada por consenso imediato, sem alternativa em disputa, e por isso não gerou ADR próprio.

#### Decisão: guardar o conteúdo do evento como fotografia do instante da transição

- **Justificativa:** se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. O contrário produziria casos estranhos `[09:52]`.
- **Trade-off:** o conteúdo é duplicado no registro do evento em vez de derivado do pedido no envio, o que aumenta o volume armazenado.

---

### Dependências

#### Organizacional: revisão de segurança antes do deploy

Sofia reservou no mínimo 2 dias úteis para revisar o código de segurança antes da subida, com foco na assinatura HMAC e na geração de secret: "Reservem pelo menos dois dias úteis pra eu revisar o código de segurança antes do deploy. HMAC e geração de secret eu quero olhar com calma" `[09:46]`. A estimativa de três sprints já inclui essa revisão `[09:47]`. Esta dependência é um gate de processo e não deve ser cortada sob pressão de prazo.

#### Organizacional: documentação no portal do desenvolvedor

Marcos assumiu documentar o comportamento da integração para os clientes, com destaque para a possibilidade de receber o mesmo evento duas vezes e para a necessidade de descartar duplicatas: "Eu posso documentar isso bem destacado no portal de desenvolvedor pros clientes, sem problema" `[09:26]`, e "Eu documento no portal pros clientes saberem como integrar via API" `[09:40]`. Sem isso, a decisão de entrega ao menos uma vez vira surpresa para o cliente em produção.

#### Técnica: operação de um segundo processo em todos os ambientes

A feature só funciona completamente com o processo de entrega em execução, além da API `[09:11]`. Desenvolvimento, homologação e produção precisam executá-lo. Isso exige ajuste de deploy, de supervisão e de guia de setup local. Um desenvolvedor que rode apenas a API não terá webhooks ativos.

#### Técnica: decisões pendentes antes do início ou durante a implementação

Onze pontos levantados ou omitidos na reunião seguem sem decisão e afetam trechos específicos da implementação. A lista completa está na seção 9 do [FDD](./FDD.md). Os mais críticos: recuperação de eventos travados após queda do processo de entrega, comprimento e entropia da secret, e escolha do tipo de identificador do evento.

#### Externa: implementação da validação de assinatura pelos clientes B2B

Os clientes precisam implementar a verificação da assinatura do lado deles. A escolha de HMAC-SHA256 foi motivada exatamente por isso: "todo cliente sério tem biblioteca pra isso" `[09:20]`.

#### Externa: implementação do descarte de duplicatas pelos clientes B2B

Os clientes precisam descartar eventos repetidos usando o identificador do evento `[09:25]`. Um cliente que não fizer isso pode processar o mesmo evento mais de uma vez. É consequência direta e aceita da garantia de entrega ao menos uma vez.

#### Externa: exposição de endpoint sobre TLS pelos clientes B2B

Cada cliente precisa expor uma URL https válida. URLs sem https são recusadas no cadastro `[09:23]`.

---

### Riscos e mitigação

#### A alteração no fluxo de mudança de status pode causar regressão no caminho mais crítico do OMS

- **Probabilidade:** media
- **Impacto:** alto. A mudança de status concentra transição de estado, ajuste de estoque e auditoria. Uma regressão afeta o núcleo do produto, não apenas a feature nova. É a única alteração em código existente.
- **Mitigação:**
  - Acrescentar um único passo à transação, sem tocar em validação, estoque ou histórico `[09:40]`.
  - Usar uma função que recebe a transação em andamento em vez de injetar dependência maior no serviço de pedidos: "função pura recebendo o tx. Não precisa injetar repository inteiro" `[09:41]`.
  - Reservar meia sprint especificamente para a integração e os testes ponta a ponta `[09:46]`.
  - Usar a suíte de testes de pedidos já existente como linha de base de não regressão.
- **Plano de contingência:** desativar todos os endpoints cadastrados. Como a filtragem acontece no registro, isso reduz a publicação a uma consulta que não retorna nada, sem tocar no restante da transação `[09:34]`.

#### O prazo de três sprints pode não caber na janela comercial da Atlas

- **Probabilidade:** media
- **Impacto:** alto. A Atlas declarou que pode migrar para o concorrente se a entrega não sair até o fim do trimestre `[09:00]`. Perder o cliente também sinaliza mal para MaxDistribuição e Nova Cargo.
- **Mitigação:**
  - Manter o escopo enxuto, com email, painel e limitação de envio explicitamente fora `[09:37]`, `[09:39]`, `[09:40]`.
  - Trabalhar com a decomposição já acordada na reunião, que distribui modelagem, processo de entrega, cadastro, integração e segurança ao longo das três sprints `[09:46]`.
  - Manter Marcos como ponto de contato para alinhar expectativa com os clientes durante a execução `[09:47]`.
- **Plano de contingência:** não cortar a revisão de segurança. Se houver necessidade de corte, o candidato natural é o histórico de entregas, que é o único requisito sem dependência de outro componente.

#### Uma falha na assinatura ou na geração de secret comprometeria a confiança no canal

- **Probabilidade:** baixa
- **Impacto:** alto. Um erro aqui compromete a confiança dos clientes B2B no canal inteiro. Já existe caso real de secret vazada em log de cliente `[09:22]`.
- **Mitigação:**
  - Reservar os 2 dias úteis de revisão de segurança antes da subida, tratando-os como obrigatórios `[09:46]`.
  - Usar secret única por endpoint, isolando o impacto de um vazamento `[09:21]`.
  - Estender a lista de campos ocultados em log para incluir as secrets.
  - Resolver as pendências de comprimento e entropia da secret durante a revisão, e não depois dela.
- **Plano de contingência:** rotação forçada da secret do endpoint afetado, usando o mecanismo de rotação com 24 horas de validade paralela que já faz parte do escopo `[09:21]`.

#### As tabelas de eventos podem crescer sem controle

- **Probabilidade:** alta
- **Impacto:** medio. As tabelas convivem no banco transacional de pedidos. Um evento pode ficar até cerca de 15 horas na fila antes de virar falha permanente, e não há política de retenção definida nem para os eventos entregues nem para a fila de falhas.
- **Mitigação:**
  - Criar os índices necessários desde a primeira migração `[09:08]`.
  - Ler os eventos pendentes em lotes pequenos `[09:08]`.
  - Definir política de retenção antes da operação em produção, resolvendo as pendências correspondentes.
- **Plano de contingência:** expurgo manual de eventos entregues mais antigos que 30 dias, janela já citada na reunião, executado como tarefa operacional pontual até haver política formal `[09:08]`.

#### Um cliente pode não implementar o descarte de duplicatas e processar o mesmo evento mais de uma vez

- **Probabilidade:** media
- **Impacto:** medio. Consequência direta e aceita da garantia de entrega ao menos uma vez. Sofia registrou a ressalva na reunião `[09:25]`.
- **Mitigação:**
  - Manter o identificador do evento estável entre todas as tentativas, tornando o descarte determinístico `[09:25]`.
  - Documentar o comportamento de forma destacada no portal do desenvolvedor `[09:26]`.
  - Apoiar-se no precedente de mercado de Stripe e GitHub para reduzir resistência de adoção `[09:25]`.
- **Plano de contingência:** nenhum do lado da plataforma. A entrega exatamente uma vez foi descartada por exigir coordenação dos dois lados `[09:25]`.

#### Um cliente pode ser bombardeado com muitas chamadas em sequência

- **Probabilidade:** media
- **Impacto:** medio. Diego levantou o cenário concreto: um cliente com 50 pedidos mudando de status em um minuto recebe 50 chamadas `[09:38]`.
- **Mitigação:**
  - Risco aceito conscientemente nesta fase. A decisão foi observar e agir se virar problema `[09:39]`.
  - O intervalo crescente entre tentativas já espaça os reenvios, embora não as primeiras entregas.
- **Plano de contingência:** desativar temporariamente o endpoint do cliente afetado, o que interrompe a criação de novos eventos para ele pela filtragem no registro `[09:34]`.

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
- Uma mudança de status sem nenhum endpoint assinante daquele status não gera nenhum evento `[09:34]`.
- Uma transação de mudança de status desfeita por qualquer motivo não deixa nenhum evento registrado `[09:06]`.
- Uma falha ao registrar o evento desfaz a mudança de status e o histórico do pedido `[09:40]`.
- O conteúdo do evento reflete o estado do pedido no instante da transição, mesmo que o pedido seja alterado depois `[09:52]`.
- O tempo entre a confirmação da mudança de status e o primeiro envio ao cliente é inferior a 10 segundos em 100 por cento dos casos, com o processo de entrega saudável `[09:02]`.
- Um endpoint de cliente que demora 30 segundos para responder não afeta a latência da mudança de status nem a de outros pedidos `[09:04]`.
- Um endpoint que responde com erro recebe exatamente 5 tentativas, nos intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas `[09:17]`.
- Um endpoint que não responde em 10 segundos tem a tentativa tratada como falha e uma nova tentativa agendada `[09:42]`.
- Após a quinta falha, o evento aparece na fila de falhas com conteúdo, motivo e horário preservados, e sai da fila ativa `[09:18]`.
- Um reprocessamento solicitado por usuário sem papel de administrador é recusado, e o mesmo pedido feito por um administrador recoloca o evento na fila `[09:36]`.
- Todo reprocessamento fica registrado com a identificação de quem o executou `[09:36]`.
- Todas as tentativas de um mesmo evento carregam o mesmo identificador de evento `[09:25]`.
- A assinatura enviada é verificável com a secret do endpoint e falha se um único byte do conteúdo for alterado `[09:20]`.
- Após uma rotação de secret, a secret anterior continua válida por 24 horas e deixa de valer depois disso `[09:21]`.
- Um cadastro com URL sem https é recusado na validação, antes de qualquer persistência `[09:23]`.
- Um evento acima de 64KB não é enviado e é tratado como erro, sem truncamento `[09:24]`.
- O histórico de entregas retorna, para cada tentativa, indicação de sucesso ou falha, conteúdo enviado, resposta recebida e tempo de resposta `[09:34]`.
- Com um único processo de entrega em execução, os eventos de um mesmo pedido chegam ao cliente na ordem em que ocorreram `[09:12]`.
- A queda da API não interrompe entregas em andamento no processo de entrega `[09:11]`.
- Nenhuma secret e nenhuma assinatura aparece em log, em nenhum nível de detalhamento.
- Todo registro de log do processo de entrega contém o identificador do evento, permitindo reconstruir o ciclo completo de tentativas.
- Nenhuma rota ou contrato existente da API teve comportamento alterado.
- Nenhuma dependência nova foi adicionada ao projeto `[09:07]`, `[09:29]`.
- A revisão de segurança foi concluída antes do deploy, com no mínimo 2 dias úteis dedicados `[09:46]`.
- O comportamento de entrega ao menos uma vez está documentado no portal do desenvolvedor `[09:26]`.

---

### Testes e validação

Tipos de teste obrigatórios

- Testes de integração para o fluxo principal de publicação do evento dentro da transação de mudança de status, cobrindo o caso em que a transação é desfeita e o caso em que nenhum endpoint assinou o status. Este é o ponto de maior risco da entrega.
- Testes de integração para o cadastro, edição, remoção e listagem de endpoints, incluindo a recusa de URL sem https.
- Testes unitários para a regra de cálculo dos intervalos entre tentativas e para a transição do evento até a fila de falhas.
- Testes unitários para a geração e a verificação da assinatura, incluindo a rejeição de conteúdo adulterado.
- Testes de integração para o ciclo completo de tentativas, incluindo a chegada à fila de falhas e o reprocessamento.
- Teste de segurança de controle de acesso, verificando que o reprocessamento é recusado para usuário sem papel de administrador e aceito para administrador `[09:36]`.
- Teste de comportamento sob endpoint lento, verificando que o tempo limite de 10 segundos é respeitado e que a mudança de status não é afetada `[09:04]`, `[09:42]`.
- Revisão manual de segurança conduzida por Sofia, com no mínimo 2 dias úteis, focada na assinatura e na geração de secret `[09:46]`. Este item não é automatizável e é obrigatório antes do deploy.

Estratégia de validação

- Reaproveitar a suíte de testes já existente no projeto, que roda contra um banco MySQL real, sem simulação, em processo único e com limpeza de tabelas entre os testes. Os testes da feature seguem esse mesmo padrão.
- Usar os testes de pedidos já existentes como linha de base de não regressão para o fluxo de mudança de status, que é o principal risco da entrega.
- Validar o ciclo de tentativas com um endpoint controlado que simule os cenários reais discutidos: erro persistente, tempo esgotado e indisponibilidade temporária seguida de recuperação `[09:16]`.
- Submeter a feature à revisão técnica de Bruno e Diego antes do início da implementação, conforme combinado na reunião: "Eu vou abrir o doc de design da feature e marcar uma sessão pro Bruno e o Diego revisarem comigo antes da gente começar a codar" `[09:50]`.
- Concluir com a revisão de segurança de Sofia como último passo antes do deploy `[09:47]`.
- **[PENDÊNCIA]** Não foi definida necessidade de teste de carga. A reunião não tratou de volume esperado de eventos por minuto, e o cenário de 50 chamadas em um minuto foi levantado sem meta associada `[09:38]`.

---

### Pendências consolidadas

Pontos sem origem na transcrição nem no código. Nenhum valor foi inventado para preenchê-los.

| # | Pendência | Onde aparece | Origem |
| --- | --- | --- | --- |
| 1 | Meta de entrega bem sucedida sem intervenção manual e volume aceitável na fila de falhas | Objetivos e métricas | Não discutido |
| 2 | Meta de latência p95 para os endpoints de configuração | Requisitos não funcionais, Performance | Não discutido |
| 3 | Meta de disponibilidade da plataforma e do processo de entrega | Requisitos não funcionais, Disponibilidade | Não discutido |
| 4 | Comprimento mínimo e fonte de entropia da secret | RF-004, Segurança | `[09:31]` não fechado, ver ADR-004 |
| 5 | Política de exposição da secret após a criação | Segurança | `[09:31]` não fechado, ver ADR-004 |
| 6 | Comportamento do cliente que usa a secret anterior após as 24 horas | RF-004 | Não discutido, ver ADR-004 |
| 7 | Contador de tentativas após reprocessamento manual | RF-009 | Não discutido, ver ADR-003 |
| 8 | Política de retenção da fila de falhas | RF-008, Riscos | Não discutido, ver ADR-003 |
| 9 | Política de retenção e arquivamento dos eventos entregues | Riscos | `[09:08]`, declarado fora do escopo |
| 10 | Recuperação de eventos presos após queda abrupta do processo de entrega | Riscos, Dependências | Não discutido, ver ADR-002 |
| 11 | Instrumentação de métricas e tracing | Requisitos não funcionais, Observabilidade | Não discutido, ausente do código |
| 12 | Requisitos regulatórios sobre envio de dados de pedidos para fora da plataforma | Requisitos não funcionais, Compliance | Não discutido |
| 13 | Necessidade de teste de carga e volume esperado de eventos | Testes e validação | `[09:38]` levantado sem meta |

Os itens 4 a 10 têm rastreabilidade direta aos ADRs e ao RFC, onde já constam como questões em aberto. Os itens 1, 2, 3, 11, 12 e 13 são lacunas de produto identificadas na elaboração deste PRD e não aparecem nos demais documentos.
