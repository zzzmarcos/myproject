# Do ECS Fargate ao EKS: Como um time enxuto economizou milhares de reais enfrentando e contornando desafios de governaça e talz

Migrar cargas de trabalho para o Kubernetes com a promessa de reduzir custos e ganhar escala é o sonho de qualquer liderança de tecnologia. Mas quem está na operação disso sabe a real: sem uma boa estratégia de plataforma, o Kubernetes joga uma complexidade brutal no colo dos desenvolvedores, gerando fricção, problemas de segurança e desperdício de nuvem.

Nos últimos três anos, nós vivemos intensamente esse desafio. Quando começamos a desenhar essa transição, tínhamos 300+ desenvolvedores cuidando de inúmeros serviços rodando em Amazon ECS Fargate. O cenário era custos subindo devido ao superdimensionamento crônico, marretadas(?) para garantir a governança e lentidão no escalonamento de recursos.

Nossa vontade era criar algo mais controlável por um time de infra como o nosso, ai começamos a pensar nas alternativas, a resposta do do mercado seria: "ir para o EKS e adotar GitOps tradicional". O problema é que, em uma instituição gigante como a que estamos inseridos, nada é fácil assim, e as coisas não se mudam da noite para o dia. Tínhamos padrões organizacionais e imutáveis:

>1. Infraestrutura apenas via Terraform para provisionamento de recursos.
>2. Pipelines e workflows com padrões preestabelecidos bastante desafiadores (já vinham prontos e sem possibilidade de customização).

Adotamos o ArgoCD sem o modelo tradicional de GitOps para evitar fricção cultural e a sobrecarga de treinar centenas de desenvolvedores em novos processos. O deploy precisava ser feito enviando o pacote YAML diretamente para o Argo via API/CLI

Com um time extremamente enxuto, nossa missão parecia impossível. Criamos a primeira poc e esbarramos em um manifesto cheio de complexidade de complicaria ainda mais a vida dos times de desenvolvimento de produtos. Mas após uma analíse minuciosa dos dados exigidos pelos charts e de ferramentas disponíveis no mercado, nós resolvemos o problema criando um operador Kubernetes customizado dentro de casa e não usamos Go, usamos Python puro inicialmente e Metacontroller, depois migramos para FastAPI e seguimos usando o Metacontroller mesmo.

O Motivo de termos usado Python e não GO talvez era mais simples do que parece, era o que conhecíamos.

O resultado de três anos de evolução contínua? Uma economia de75% no custo de produção, e chegando em 90% em ambientes não produtivos. Essa economia chega a  mais de R$ 1 milhão por ano devolvidos em horas de engenharia que os times de produto gastavam brigando com a infraestrutura. No total, geramos uma eficiência combinada estimada é de mais de R$100 mil por mês, tudo isso sem alterar nenhum processo existente na empresa.

Neste artigo, vou te mostrar os bastidores dessa arquitetura, os principais desafios que enfrentamos na linha de frente ao longo desses três anos e como utilizamos o kuberntes para para forçar padrões sem matar a velocidade dos desenvolvedores.

---

## O Cenário no ECS e o Peso da Governança

O ECS Fargate é uma ótima tecnologia para iniciar algo, pois ele facilita muito a rampagem, mas conforme os times de produto crescem, a falta de padronização e os desafiadores padrões corporativos tem o seu preço. No nosso cenário original, os desenvolvedores tinham autonomia para criar suas Tasks definitions ao seu flavor. O resultado prático disso era um grande desafio de conformidade.

E para ser completamente honesto: nós tínhamos um processo interno que eu odiava com todas as minhas forças, chamávamos de  "Marathon".

O Marathon era uma reunião recorrente e extremamente desgastante. O objetivo era juntar o time de dev e ops para ficar resolvedo coisas que estavam fora dos padrões, por exemplo: caçando recursos órfãos na AWS e implorando para que os times colocassem tags básicas nos recursos, coisas simples, como a tag da squad dona do serviço. Ou em outros casos, apontar que os recursos estavam desperdiçando 95% da infraestrutura provisionada que ele deveria ter um trabalhao rightsizing. Era um trabalho de herói, totalmente manual, reativo e ineficiente. Mas talvez o nosso grande problema era mesmo a falta de tags.

>O grande impacto disso era a falta total de visibilidade de custos e isso impactava diretamente de onde o dinheiro estava sendo investido. Custos altíssimos na AWS chegavam e nós simplesmente não sabíamos e como muitas coisas não tinham tag, não conseguíamos ter uma visão mais granularizada, quanto cada time estava gastando na plataforma, o que eles faziam e, o pior de tudo, se aquele custo sequer se justificava para o negócio. Era impossível auditar e otimizar o que não conseguíamos enxergar.

Além dessa dor de cabeça com FinOps, o modelo descentralizado do ECS gerava um fardo imenso para os times de produto:

* **A Fadiga de Atualizações:** Os desenvolvedores precisavam estar constantemente de olho para ver se havia atualizações nos módulos de infraestrutura. Qualquer mudança institucional, por menor que fosse, se transformava em um pesadelo operacional: precisava ser aplicada app por app, de forma isolada.
* **Overprovisioning:** Na dúvida de quanta CPU ou Memória a aplicação precisava, o dev jogava o valor para beeeem alto. Como o Fargate cobra exatamente pelo que está alocado (e não pelo que está sendo usado), a conta da AWS explodiu.
* **O Gargalo do Escalonamento:** Devido à lentidão do scaling no ECS — que precisava bater na API da AWS a cada nova réplica para buscar secrets e parameters via API —, nossa velocidade de resposta a picos de tráfego era severamente afetada.


Sabíamos que o Amazon EKS resolveria o problema do custo e da velocidade. Mas como fazer isso sem obrigar 300 desenvolvedores a aprenderem o manifesto complexo do Kubernetes (Deployments, Services, Ingress, HPA)? E pior: como garantir que a gente matasse o Marathon e a dependência de intervenções manuais app por app de uma vez por todas?

Mas a complexidade que o YAML padrão para atender os charts da organizacao seria um desafio com uma curva de aprendizado enorme, e talvez comprometesse toda nossa estratégia, pois traria uma certa complexidade que os times de desenvolvimento nao estavam preparados, e nós sabíamos deste desafio:

A única forma de conseguirmos fazer o EKS ser aceito pela nossas lideranças seria se ele fosse mais fácil, pois as outras vantagens apesar de ter muito valor, não justificava a uma complexidade tão grande.
Percebemos que a única saída era interceptar o deploy dentro do próprio cluster. Precisávamos de um operador que recebesse um YAML ultra simples do desenvolvedor e injetasse toda a complexidade e governança corporativa por baixo dos panos, de forma 100% automatizada.


---

## Desafio de Arquitetura: Por que Python e Metacontroller para um time tão enxuto?

Quando se fala em estender o Kubernetes e criar CRDs (Custom Resource Definitions), o padrão de mercado quase absoluto é usar Go com Kubebuilder ou Operator SDK. No entanto, precisávamos ser pragmáticos. O Python estava no nosso dia a dia, sabíamos trabalhar com python, teríamos que aprender Go e não era o momento ainda.

Foi aí que tomamos uma decisão de arquitetura pouco ortodoxa para grandes corporações, mas extremamente eficiente: escolhemos o Metacontroller e escrevemos nossa lógica de negócios em Python.

>O Metacontroller funciona como um "facilitador". Ele roda como um operador dentro do cluster e cuida de toda a parte chata e complexa do Kubernetes: gerenciar os watchers, a fila de reconciliação e as chamadas de API. Tudo o que ele pede em troca é um Webhook HTTP.

Nossa arquitetura foi desenhada para ser simples:

1. O desenvolvedor submete um YAML minimalista (de no máximo 3 níveis de profundidade).
2. O pipeline fechado do GitHub Actions envia esse pacote diretamente para a API do ArgoCD.
3. O ArgoCD aplica esse recurso customizado no cluster.
4. O Metacontroller percebe a mudança e faz uma requisição HTTP POST para a nossa API em Python.
5. Nosso código recebe o estado atual, injeta os padrões do banco (tags de billing obrigatórias por squad, variáveis do Datadog, sidecars de segurança e health checks) e devolve a resposta.
6. O Metacontroller se encarrega de criar ou atualizar os Deployments, HPAs e Services gerados.

---

## O real ganho com Secrets e Parameters: Diga adeus aos Rate Limits da AWS

Uma das maiores dores que tínhamos no ECS Fargate estava diretamente ligada à forma como as aplicações lidavam com segredos e parâmetros de configuração (provenientes do AWS SSM Parameter Store e AWS Secrets Manager).

### O Pesadelo das Requisições Síncronas no Startup

No modelo do ECS, cada vez que uma nova *task* (réplica) iniciava, ela precisava se conectar e baixar de forma síncrona suas variáveis de ambiente diretamente da API da AWS. Esse processo trazia dois grandes problemas:

1. **Latência de Inicialização (Cold Start):** As tarefas demoravam preciosos segundos extras apenas esperando o download dos segredos antes do container efetivamente iniciar.
2. **O Temido *Rate Limit* da AWS:** Em momentos de pico de tráfego, quando a plataforma disparava um escalonamento agressivo (subindo dezenas ou centenas de tarefas de uma vez), todas as novas instâncias bombardeavam a API da AWS simultaneamente. O resultado? Estouro de limite de requisições (rate limit). Isso travava completamente o escalonamento das nossas aplicações, deixando o sistema instável, e causava um efeito cascata que de quebra impactava toda a nossa conta da AWS.

### A Solução: Cache Local com External Secrets Operator (ESO)

Com a migração para o EKS, resolvemos isso redesenhando o fluxo com o External Secrets Operator (ESO) e integrando-o ao nosso operador customizado.

Agora, o fluxo funciona de forma assíncrona e inteligente:

1. **Declaração Simples:** O desenvolvedor apenas lista quais parâmetros e segredos sua aplicação precisa diretamente no arquivo `kubernetes.yaml` simplificado.
2. **Abstração pelo Operador:** Nosso operador Python intercepta essa declaração e gera de forma automatizada o manifesto do `ExternalSecret`.
3. **Sincronização Assíncrona:** O ESO faz a ponte com a AWS, baixa os dados e cria um `Secret` nativo dentro do Kubernetes. 
4. **Consumo Local no Cluster:** As aplicações consomem esses segredos nativos, montados como volumes ou injetados diretamente como variáveis de ambiente no container.

**O impacto prático disso na nossa resiliência foi brutal.** Como os segredos ficam salvos localmente no cluster em um `Secret` do Kubernetes, o escalonamento das aplicações passou a ser instantâneo. 

Essa mudança se provou ainda mais crucial no Kubernetes do que no ECS: os Pods no EKS são infinitamente mais voláteis. Eles nascem, morrem, são realocados entre nós e escalados para cima e para baixo constantemente. Se tivéssemos mantido o modelo antigo de buscar os dados diretamente na AWS em cada inicialização, a volatilidade do Kubernetes teria gerado uma tempestade perfeita de requisições, estourando o rate limit do Parameter Store em minutos. 

Com o cache local provido pelo ESO, se precisarmos escalar de 10 para 100 réplicas em segundos, o Kubernetes apenas replica os Pods lendo as definições locais no `etcd`, sem fazer uma única chamada sequer à API da AWS no momento do boot. Eliminamos o gargalo de inicialização, zeramos os problemas de rate limit e garantimos estabilidade total para a nossa conta de nuvem mesmo nos maiores picos de estresse.



---

## Governança Centralizada na Prática: O Caso do Datadog e da Sanitização de Logs

O operador nos trouxe uma governança centralizada que nos permite fazer ate mesmo uma virada na plataforma de observabilidade, onde no cenario antigo os times de desenvolvimento precisavam alterar individualmente app por app para onde iriam mandar seus logs, com o operador fazemos isso em um piscar de olhos alterando apenas em um lugar e replicando isso para todas as apps. Isso tambem nos permitiu que criassemos até mesmo uma peça para sanitizaçao de logs (este é um papo para outro artigo) e injetamos em todos os serviços.

---

## A Evolução Técnico-Pragmática: Do Script Puro ao FastAPI


### O Desafio da Escala e a Virada para o FastAPI

---

## Conclusão: A Linha do Tempo de uma Jornada Viva

Olhando para trás, os desafios de sustentar e evoluir uma solução "feita em casa" ao longo desses três anos nos ensinaram que o sucesso de uma plataforma de engenharia corporativa depende de paciência, consistência e processos bem desenhados. Essa jornada de 36 meses dividiu-se em três grandes fases:

1. **Os Primeiros 6 Meses (Estudos e Provas de Conceito):** O período em que validamos o Metacontroller, rodamos os primeiros scripts em Python puro e provamos para a lideranas que conseguiríamos injetar os padrões institucionais de forma segura dentro do cluster.
2. **Os 2 Anos Seguintes (A Jornada de Migração):** O momento da linha de frente. A migração dos serviços foi sendo feita de forma gradual pelos próprios times de produto e assistidas por nós. Fomos refinando o operador (onde nasceu a virada para o FastAPI) e dando autonomia escalável para os desenvolvedores descerem do ECS.
3. **Os Últimos 6 Meses (A Consolidação do Valor):** Com boa parte dos serviços migrados e o ecossistema estabilizado, alcançamos a maturidade atual da plataforma. 

Hoje, colhemos uma **redução de 75% nos custos de infraestrutura em produção** e impressionantes **90% em desenvolvimento e homologação**. Matamos o fantasma do Marathon de FinOps, transformando a governança em um processo invisível e compulsório, devolvendo mais de R$ 1 milhão por ano em custos de infra e horas de trabalho para os times de produto. 

O projeto, porém, está longe de estar "concluído". Mesmo sem atingirmos 100% de migração total de toda a carga de trabalho histórica, a eficiência que já geramos nos abriu os olhos para novas frentes. Atualmente, estamos estudando e desenhando otimizações de arquitetura com a meta ousada de trazer uma redução ainda maior nesse custo mensal de infra. 

Em uma grande organizaçao como a nossa, provamos que times enxutos usando tecnologia de forma pragmática conseguem sim mudar o rumo da governança, da experiência do desenvolvedor e do faturamento de nuvem. E o trabalho continua.
