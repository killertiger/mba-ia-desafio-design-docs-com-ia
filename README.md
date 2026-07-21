# Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre o desafio

O desafio consiste em gerar os documentos de design docs que irão auxiliar a guiar o projeto. Os documentos podem tanto ser consumidos por humanos quanto por outras IAs para geração de código e garantir a qualidade do sistema.

Em alguns pontos do documento, estão marcados como `PRECISA DE INFORMAÇÃO` de propósito.
O fato é que algumas decisões não foram tomadas e explicitadas na reunião, e como o próprio desafio pede para não inventar decisões, deixei esses pontos em aberto para que o time decida posteriormente. Porém, caso exista algum comportamento já definido no código, então este será também utilizado para auxiliar na decisão.

## Ferramentas utilizadas:
- Claude - todo o desenvolvimento dos documentos foi utilizado o Claude com o modelo Opus 4.8

## Workflow adotado:
1. Revisei as aulas e os prompts/skills disponíveis para fazer a geração dos documentos
2. Li toda a Transcrição
3. Assim como sugerido, segui a ordem de geração ADR -> RFC -> FDD -> PRD.
4. Entre cada geração:
  4.1 Revisei o documento por informações inventadas, ou pontos em abertos
  4.2 Refinei meu prompt e contextualização utilizando levando em conta o código atual
5. Após a geração de todos os documentos, pedi para a IA repassar documento por documento, identificando possíveis erros e possíveis melhorias considerando todos os documentos gerados + código

## Prompts customizados:


Prompt para revisão:
```
Preciso que você me ajude a revisar os documentos que gerei utilizando IA na pasta docs/...
Para cada documento você deve sempre considerar o código atual, e também o documento TRANSCRICAO.md. Esse doc é a base da geração dos outros documentos, pois está relacionado a uma conversa sobre discussão de features.

Quero começar pelos documentos docs/adrs/... e quero sua ajuda para revisar documento por documento.
Você deve ler o documento, verificar se todas as referências estão corretas em relação ao TRANSCRICAO.md e levantar se tem algum item pendente.
Se houver, quero que você verifique no código se existe já alguma definição para um caso semelhante.
Ao final da revisão do documento, quero que você me informe:
- Documento válido ou pendente de correção
- Se houver correção, qual deveria ser feita
-----
Quero fazer documento por documento, por causa disso, você deve começar pelo primeiro documento, e aguardar minha confirmação antes de prosseguir para o próximo. Além disso, você só deve ajustar caso eu confirme que você deva corrigir
```


```
Preciso gerar um documento do PRD referente a feature `OMS Sistema de Webhooks de Notificação de Pedidos`

Antes de iniciar a geração:
1. Leia o arquivo TRANSCRICAO.md - Neste arquivo contém uma transcrição de uma reunião com detalhes e decisições sobre a feature
2. Leia todo o código para entender a estrutura e auxiliar nas decisões relacionadas a transcrição. Em alguns casos, da transcrição simplesmente menciona seguir o padrão existente do código. Por causa disso, é importante ler o código para identificar possíveis decisões já todas.
3. Leia as documentações existentes na pasta docs/... pois várias decisões já foram tomadas
4. Não invente decisões ou pontos não discustidos.
 
 
Para realizar a geração do PRD, utilize o seguinte prompt:
---
[COLE AQUI O PROMPT DO PRD disponível em: https://devfullcycle.notion.site/Prompt-de-Entrevista-para-Gerar-PRD-para-desenvolvimento-de-Feature-2971423c038880f9bd94f3b46de9dd56]
```


## Iterações e ajustes

Eu não gerei todos os arquivos de uma só vez. Foram 4 interações para cada arquivo.
- Prompt para Geração do arquivo
- Revisão manual (Apenas leitura)
- Reajustar prompt por exemplo para considerar o código e verificar se é possível resolver algumas das opções
- Prompt refinado ajuste no arquivo
- Revisão manual (Apenas leitura)

Após a geração de todos os arquivos, revisão arquivo por arquivo:
- Prompt para revisar arquivo considerando documentos + código + TRANSCRICAO.md
- Atualização do TRACKER.md


## Como navegar a entrega
- Todos os arquivos estão disponíveis na pasta docs/...
Ordem de leitura:
- [PRD.md](docs/PRD.md)
- [FDD.md](docs/FDD.md)
- [RFC.md](docs/RFC.md)
- [ADRS.md](docs/ADRS.md)
- [ADR-001-padrao-outbox-no-mysql.md](docs/ADR-001-padrao-outbox-no-mysql.md)
- [ADR-002-worker-em-processo-separado-com-polling.md](docs/ADR-002-worker-em-processo-separado-com-polling.md)
- [ADR-003-retry-com-backoff-exponencial-e-dlq.md](docs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004-autenticacao-hmac-sha256-por-endpoint.md](docs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md)
- [ADR-005-garantia-at-least-once-com-x-event-id.md](docs/ADR-005-garantia-at-least-once-com-x-event-id.md)

Os documentos PRD, FDD, RFC, poderiam ser renomeados ou dentro de uma pasta específica da Feature, porém por decisão do desafio, decidi manter os arquivos na pasta docs/..
