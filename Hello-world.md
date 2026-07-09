# A ilusão do Piloto Automático: Por que decidi voltar para a base na era da IA

Se você trabalha criando, desenvolvendo ou mantendo infraestrutura e sistemas distribuídos, já deve ter percebido a mágica. Eu abro o ChatGPT ou o Copilot, peço um script complexo, um manifesto do Kubernetes ou uma pipeline inteira, e em cinco segundos o código está na minha tela. Confesso que a primeira sensação é a de que me tornei um super-humano.

É tentador pensar que, a partir de agora, a Inteligência Artificial vai resolver tudo na nossa área. Que basta saber fazer a pergunta certa (prompt engineering) e a máquina faz o trabalho pesado. Mas, vivendo a realidade de quem atua com SRE, Cloud e operações, eu percebi uma armadilha perigosa nessa história: **a IA sabe como construir a sintaxe, mas não entende o porquê as coisas quebram na madrugada.**

Quando o sistema está em cruzeiro, o piloto automático é perfeito. Mas quando a turbulência começa, o painel pisca em vermelho e a aplicação trava sem motivo aparente, o piloto automático desliga e o controle volta sempre para as minhas mãos. E é nessa hora, entre war room e war room, que saber "escovar bits" separa os passageiros dos verdadeiros engenheiros.

Foi por causa dessa percepção que eu decidi iniciar a minha série que ainda não dei um nome "fancy". Um movimento pessoal de voltar aos fundamentos, e decidi compartilhar cada passo desse caminho com vocês por aqui.


## 1. Fugindo da Síndrome do "Copiador de Código Premium"

A IA democratizou o "como fazer". Qualquer pessoa hoje consegue gerar um deploy funcional na AWS. O problema é que a IA, por padrão, gera a resposta mais estatisticamente provável, não necessariamente a melhor para o contexto arquitetural da minha (ou da sua) empresa.

Se eu pedir para a IA configurar uma VPC, ela vai me dar um bloco CIDR. Se eu não souber os fundamentos de redes e subnet, eu vou simplesmente copiar e colar. Meses depois, o Auto-scaling precisa subir novos pods e não consegue porque os IPs acabaram. 

Se eu colar o erro de volta na IA, ela vai sugerir cinco soluções genéricas: reiniciar o cluster, mudar permissões, mexer em Security Groups... E eu vou passar 12 horas rodando em círculos. Ela não tem o contexto macro para enxergar o erro básico de design de rede que eu cometi meses atrás. Eu não quero ser refém desse ciclo.


## 2. Sintaxe vs Semântica (Por que eu estou voltando aos "Bits")

Na computação, sintaxe é a gramática; semântica é o significado real. A Inteligência Artificial é a rainha indiscutível da sintaxe. Ela não erra ponto e vírgula, não esquece de fechar chaves e indenta arquivos YAML com perfeição.

Mas o troubleshooting em ambientes críticos exige conhecimento semântico. Requer entender o conceito base. E é isso que eu decidi aprofundar nessa nova fase.

![A ponta visível fora da água é o treabalho feito pela IA. A parte massiva submersa representa os fundamentos: SO, Kernel, Rede TCP/IP, Memória, onde os problemas reais acontecem](imagens/iceberg.jpg)

* A IA sabe escrever uma query SQL. Mas se o banco der lock em produção, eu preciso entender o conceito de concorrência e transações ACID para destravar o sistema.
* A IA escreve o arquivo do Docker perfeitamente. Mas se o container estourar o limite e matar o nó inteiro por OOMKilled (Out of Memory), eu preciso entender como o Kernel do Linux gerencia recursos e memória virtual.
* A IA faz o fetch de uma API externa. Mas se ocorrer um estrangulamento de portas (port exhaustion), eu tenho que voltar para a base do protocolo TCP/IP e entender como conexões presas em TIME_WAIT estão derrubando meu servidor.

Saber a resposta que a máquina me dá é inútil se eu não consigo diagnosticar a pergunta certa.


## 3. O meu plano nessa série que ainda não dei um nome "fancy"

O avanço da IA não vai eliminar o profissional de tecnologia, mas vai aniquilar o "tarefeiro" – aquele que sobreviveu até hoje decorando comandos e copiando tutoriais sem entender como as peças se conectam por debaixo dos panos. 

Para não me tornar obsoleto e continuar evoluindo para níveis de especialização cada vez maiores, a minha relação com a tecnologia mudou, e o meu convite é para que você faça o mesmo:

1.  **Vou deixar a IA ser o meu pedreiro:** Vou usar a ferramenta para gerar as fundações rápidas, boilerplates e automações repetitivas. Quero ganhar velocidade.
2.  **Serei o Engenheiro Estrutural:** O tempo que eu economizar na digitação, vou investir em estudar a **base**. Vou dissecar protocolos de rede, gerenciamento de SO, certificações robustas de arquitetura e como sistemas distribuídos lidam com falhas.
3.  **Desenvolver o instinto investigativo:** Quando a IA me der uma resposta pronta, o meu novo padrão é me perguntar: "Por que essa é a melhor abordagem? O que acontece com esse código se a rede oscilar por 5 segundos?"

O futuro pertence a quem entende as abstrações tecnológicas profundamente o suficiente para saber quando a Inteligência Artificial está alucinando. A máquina pode pilotar o avião na calmaria, mas é o nosso conhecimento de base que vai impedir a queda durante a tempestade.

Bem-vindos a nessa série que ainda não dei um nome "fancy". Nos próximos artigos, vou postar os artigos que sairam das anotações que eu fiz nos caminhos que usei para evoluir meus conhecimentos e compartilhar com vocês os fundamentos que estou (re)estudando para não ser engolido pelo piloto automático. Vamos juntos?
