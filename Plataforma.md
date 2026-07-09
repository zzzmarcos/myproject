# Do ECS Fargate ao EKS: Como um time enxuto economizou milhares de reais enfrentando contornando desafios de governana e talz

Migrar cargas de trabalho para o Kubernetes com a promessa de reduzir custos e ganhar escala é o sonho de qualquer liderança de tecnologia. Mas quem está na operação disso sabe a real: sem uma boa estratégia de plataforma, o Kubernetes joga uma complexidade brutal no colo dos desenvolvedores, gerando fricção, problemas de segurança e desperdício de nuvem.

Nos últimos três anos, nós vivemos intensamente esse desafio. Quando começamos a desenhar essa transição, tínhamos 300+ desenvolvedores cuidando de inúmeros serviços rodando em Amazon ECS Fargate. O cenário era custos subindo devido ao superdimensionamento crônico, marretadas(?) para garantir a governança e lentidão no escalonamento de recursos.

Nossa vontade era criar algo mais controlável por um time de infra como o nosso, ai começamos a pensar nas alternativas, a resposta do do mercado seria: "ir para o EKS e adotar GitOps tradicional". O problema é que, em uma instituição gigante como a que estamos inseridos, nada é fácil assim, e as coisas não se mudam da noite para o dia. Tínhamos amarras de governança muito fortes e imutáveis:

1. Infraestrutura aapenas via Terraform para provisionamento de recursos.
2. Pipelines engessados e trancados no GitHub Actions, com workflows prontos e sem possibilidade de customização.
3.	Poderiamos optar pelo uso do ArgoCD, mas sem o modelo tradicional de GitOps (sem repositórios dedicados para os manifestos de apps). O deploy precisava ser feito enviando o pacote do YAML direto para o Argo via API/CLI.

Com um time enxuto de 4 pessoas, mas com mas com apenas 2 analistas dedicados a trabalhar nesse problema durante todo o início do ciclo, nossa missão parecia impossível. Criamos a primeira poc e esbarramos em um manifesto cheio de complexidade de complicaria ainda mais a vida de quem fosse subir as apps. Mas após uma analíse minuciosa dos dados exigidos pelos charts e de algumas coisas que tinha no mercado, nós resolvemos o problema criando um operador Kubernetes customizado dentro de casa — e não usamos Go, usamos Python puro inicialmente e Metacontroller, depois migramos para FastAPI e seguimos usando o Metacontroller mesmo.

O Motivo de termos usado Python e não GO talvez era mais simples do que parece, era o que conhecíamos.

O resultado de três anos de evolução contínua? Uma economia de75% no custo de produção, e chegando em 90% em ambientes não produtivos. Essa economia chega a  mais de R$ 1 milhão por ano devolvidos em horas de engenharia que os times de produto gastavam brigando com a infraestrutura. No total, geramos uma eficiência combinada estimada é de mais de R$100 mil por mês.

Neste artigo, vou te mostra os bastidores dessa arquitetura, os principais desafios que enfrentamos na linha de frente ao longo desses três anos e como utilizamos o kuberntes para para forçar padrões sem matar a velocidade dos desenvolvedores.

---

## O Cenário no ECS e o Peso da Governança

O ECS Fargate é uma ótima tecnologia para iniciar algo, pois ele facilita muito a rampagem, mas conforme os times de produto crescem, a falta de padronização e as travas corporativas tem o seu preço. No nosso cenário original, os desenvolvedores tinham autonomia para criar suas Tasks definitos ao seu flavor. O resultado prático disso era um grande desafio de conformidade.

E para ser completamente honesto: nós tínhamos um processo interno que eu odiava com todas as minhas forças, chamávamos de  "Marathon".

O Marathon era uma reunião recorrente e extremamente desgastante. O objetivo era juntar o time de infraestrutura e os responsáveis pelo desenvolvimento de produtos para ficar resolvedo coisas que estavam fora dos padrões, por exemplo:  caçando recursos órfãos na AWS e implorando para que os times colocassem tags básicas nos recursos — coisas simples, como a tag da `squad`, o dono do serviço ou o centro de custo. Ou em outros casos, apontar que os recursos estavam desperdiçando 95% da infraestrutura provisionada que ele deveria ter um trabalhao rightsizing. Era um trabalho de herói, totalmente manual, reativo e ineficiente. Mas talvez o nosso grande problema era mesmo a falta de tags.

>O grande impacto disso era a cegueira total de custos. A falta dessas tags fazia com que não tivéssemos a menor visibilidade de para onde o dinheiro estava indo. Custos altíssimos de AWS chegavam e nós simplesmente não sabíamos e como muitas coisas não tinham tag, não conseguíamos ter uma visão mais granularizada dos custos, quanto cada time estava gastando na AWS, o que eles faziam e, o pior de tudo, se aquele custo sequer se justificava para o negócio. Era impossível auditar e otimizar o que não conseguíamos enxergar.

Além dessa dor de cabeça com FinOps, o modelo descentralizado do ECS gerava um fardo imenso para os times de produto:

* **A Fadiga de Atualizações:** Os desenvolvedores precisavam estar constantemente de olho para ver se havia atualizações nos módulos de infraestrutura. Qualquer mudança institucional, por menor que fosse, se transformava em um pesadelo operacional: precisava ser aplicada app por app, de forma isolada.
* **Overprovisioning:** Na dúvida de quanta CPU ou Memória a aplicação precisava, o dev jogava o valor para beeeem alto. Como o Fargate cobra exatamente pelo que está alocado (e não pelo que está sendo usado), a conta da AWS explodiu.
* **O Gargalo do Escalonamento:** Devido à lentidão do scaling no ECS — que precisava bater na API da AWS a cada nova réplica para buscar secrets e parameters via API —, nossa velocidade de resposta a picos de tráfego era severamente afetada.


Sabíamos que o Amazon EKS resolveria o problema do custo e da velocidade. Mas como fazer isso sem obrigar 300 desenvolvedores a aprenderem o manifesto complexo do Kubernetes (Deployments, Services, Ingress, HPA)? E pior: como garantir que a gente matasse o Marathon e a dependência de intervenções manuais app por app de uma vez por todas?

Mas a complexidade que o YAML padrão para atender os charts do banco seria um desafio com uma curva de apresenzado enorme, e talvez comprometesse toda nossa estratégia, pois traria uma complexidade enorma, e nós sabíamos:

A única forma de conseguirmos fazer o EKS ser aceito pela nossas lideranças seria se ele fosse mais fácil, pois as outras vantagens apesar de ter muito valor, não justificava a uma complexidade tão grande.
Percebemos que a única saída era interceptar o deploy dentro do próprio cluster. Precisávamos de um operador que recebesse um YAML ultra simples do desenvolvedor e injetasse toda a complexidade e governança corporativa por baixo dos panos, de forma 100% automatizada.


---

## Desafio de Arquitetura: Por que Python e Metacontroller para um time tão enxuto?

Quando se fala em estender o Kubernetes e criar CRDs (*Custom Resource Definitions*), o padrão de mercado quase absoluto é usar Go com Kubebuilder ou Operator SDK. No entanto, precisávamos ser pragmáticos. O Python estava no nosso dia a dia, sabíamos trabalhar com python, teríamos que aprender Go e não era o momento ainda.

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

## O real ganho com Secrets e parameters

---

## Governança Centralizada na Prática: O Caso do Datadog e da Sanitização de Logs

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

Em uma grande instituição financeira como a nossa, provamos que times enxutos usando tecnologia de forma pragmática conseguem sim mudar o rumo da governança, da experiência do desenvolvedor e do faturamento de nuvem. E o trabalho continua.
