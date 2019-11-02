---
layout: post
cover: 'assets/images/glue_etl.jpg'
title: Criando um Data Pipeline com AWS Glue Orquestrado por AWS Lambda + StepFunctions
date: 2018-09-07 16:35:00
tags: aws_glue data_engineer 
author: alexsandro
---

Nesse texto vou abordar um método de criação de Data Pipeline atrelado a Amazon Web Services, usando sobretudo do [AWS Glue](https://aws.amazon.com/pt/glue/) para administrar nossos Jobs Spark e uma função lambda para orquestrar as tarefas de nosso Pipeline usando do AWS Step Functions.

Esse texto é um resumo em palavras da demonstração _ETL Orchestration with AWS Glue and Step-Functions_ feita por mim na _PAPIS.io LATAM 2019_ e inspirado pelo texto do [Moataz Anany](https://amzn.to/2Yazsiz) no AWS Big Data Blog.

Aqui vou falar de diversas tecnologias da AWS, entre elas _Glue_, _StepFunctions_, _Lambda_,  _DynamoDB_ e _SES_. Mas para entender a motivação de usar tantas outras ferramentas é importante entender inicialmente o AWS Glue e as limitações que encontramos em nosso dia dia de trabalho no [Stoodi](www.stoodi.com.br).

### AWS Glue

<blockquote>
<h3>"O AWS Glue é um serviço de extração, transformação e carga (ETL) gerenciado que facilita a preparação e a carga de dados para análises pelos clientes. [...]"</h3>
<span>~ AWS</span>
</blockquote>

AWS Glue é um serviço da Amazon focado em disponibilizar uma plataforma para rodar jobs Spark de maneira serveless. Isso significa que você deixa de se preocupar com um cluster Spark porque a Amazon já vai cuidar disso para você. O Custo deixa de ser por manter instancias sparks rodando a todo momento e se torna algo mais _pay as you go_.

O Glue pode ser entendido através de algumas funções chaves:

* __Data Catalog:__
			Data Catalog é uma das funcionalidades mais uteis do AWS Glue, nele podemos, através de nossas _conexões_ e _crawlers_, mapear todas as nossas tabelas de diferentes fontes de dados. Ele é capaz de identificar os _schemas_ de sua tabela e já armazena esse metadado para podermos usar com facilidade em nossos scripts ETLs.
	*	__Conexões:__
			Aqui Somos capazes de conectar bancos de dados em nosso Data Catalog, aqui podemos escolher entre Bancos de dados que estiverem no RDS, um banco Redshift e ainda uma opção para conectar qualquer banco que possa se conectar através de JDBC.
	*	__Crawlers__
			Em Crawlers podemos criar processos que farão o trabalho de varrer buckets em seu S3 para analisar Arquivos csv, orc ou parquet e inferir seu schema. Dessa forma podemos usa-los como tabelas dentro do Spark.
* __Jobs:__
		O Job é nada mais do que o seu código ETL (_Extraction, Transform and Load_). Pode ser escrito em Python ou Scala e através da biblioteca do glue junto ao Spark consege carregar dados das tabelas mapeadas no Data Catalog e fazer as transformações necessarias.

<amp-img src="{{ site.baseurl }}assets/images/create_job.gif" layout="responsive" width="580px" height="407px" alt="" class="mb3"></amp-img>

* __Triggers:__
		Triggers são como _crons_ para os seus jobs, aqui a gente pode tentar orquestrar alguns poucos jobs definindo uma frequência que ele irá executar.

Apesar dos _triggers_ do glue serem uma mão na roda quando precisamos orquestrar poucos jobs, a coisa começa a ficar complexa demais para deixar toda ali. Então por isso usamos O Step Functions para nos auxiliar com Pipelines um pouco mais complexos.

### AWS Step Functions

Aws Step Functions é um coordenador de fluxos em estrutura de maquina de estados. Para quem esta acostumado com ferramentas de Data Engineer usaremos ele de forma parecida a o que o Apache AirFlow desempenha em outras arquiteturas.

<blockquote>
<h3>"AWS Step Functions makes it easy to coordinate the components of distributed applications as a series of steps in a visual workflow."</h3>
<span>~ AWS</span>
</blockquote>

Podemos sem muita dificuldade criar um arquivo que descreve uma step-function através da [Amazon State Language](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language.html).

<amp-img src="{{ site.baseurl }}assets/images/step-function-exemplo.png" layout="responsive" width="580px" height="407px" alt="" class="mb3"></amp-img>

No Step Functions podemos criar maquinas de estado onde cada estado representa a execução de um Job do Glue.

Quando você iniciar uma maquina de estados dentro do Step Functions ela irá passar de estado a estado mudando seu status de pendente, executando e por fim completo.

Porem, essa orquestração é bem simples, dado que não conseguimos ter total liberdade da execução do job (passar parâmetros para o job, armazenar status da execução em logs, informar a equipe de dados sobre erros na execução, etc..). Por isso, temos a opção de usar _Activities_. Com ela conseguimos fazer com que a execução de um estado na step-function seja orquestrado através de uma API. Com isso podemos fazer o que bem entendermos num único estado. Antes podiamos apenas chamar um um Job no glue, agora podemos ativar crawlers, triggers ou enviar emails e caso de falhas.


### AWS Lambda

De forma bem resumida, AWS Lambda é o serviço da Amazon que nos permite rodar scripts de maneira serverless sob demanda. Com esse serviço, podemos criar um script que irá consumir os status dos estados do Step-Functions, interpretar cada estado e tomar uma ação.

A essa altura, uma vez que você faça um script (que, dentro do Stoodi usamos como molde inicial o script do Moataz Anany disponibilizado [nesse repositorio](https://github.com/aws-samples/aws-etl-orchestrator/blob/master/lambda/gluerunner/gluerunner.py)).

O script já trata os casos do job ainda estar em execução e casos de falha. O ideal é roda-lo em uma função lambda com alguma frequência que faça sentido para o seu pipeline. no Caso do stoodi, como temos muitos jobs de curto tempo de execução rodamos um equivalente a essa função a cada 10 minutos.

A função lambda tem dois métodos que são executados em toda chamada: _start_glue_jobs_ que faz uma requisição para o step-function perguntando sobre estados pendentes, interpreta-os e executa o job necessario, por fim, guarda um objeto no _DynamoDB_ com o identificar da execução daquele job. E _check_glue_jobs_ faz uma busca nessa mesma tabela do _DynamoDB_ e, para os jobs que foram executados, verifica se já concluíram com sucesso ou falharam.

### Juntando as peças

Agora que já foram apresentadas as principais ferramentas que compõe nosso pipeline, vamos olhar como elas se encaixam!

No próprio texto do Moataz Anany existe um fluxograma do funcionamento do pipeline, mas eu particularmente penei para entende-lo e preferi criar um mais simplificado para me guiar durante a implementação.

<amp-img src="{{ site.baseurl }}assets/images/etl_glue_lambda.png" layout="responsive" width="283px" height="130px" alt="" class="mb3"></amp-img>

Nesse, _start_glue_jobs_:

* Faz um request para o step-functions, pedindo por todas as etapas pendentes que tenham como ```resource``` a activity.

* Dentro da definição das ```steps``` podemos adicionar um dicionario de variáveis chamado ```parameters```. É la que colocamos as configurações que acharmos útil. como o numero de ```DPUs```. Também colocamos o nome do Job para poder chama-lo pela função Lambda.

* Após executar o job, o Glue retornara o identificador dessa execução do Job, armazenaremos no DynamoDB para que o _check_glue_jobs_ possa saber quais jobs estão pendentes.	

Para o _check_glue_jobs_:

* Busca no DynamoDB por todos os Jobs que estão pendentes

* Requisita o status de cada job retornado pelo DynamoDB, requisita o status daquele Job para o AWS Glue e notifica o Step-Functions de acordo com o status:
	- Caso sucedido, envia que a tarefa foi concluída e apaga os dados no DynamoDB.
	- Caso ainda esteja em execução, envia uma mensagem de heartbeat
  - Caso falha ou de timeout, envia que a tarefa foi mal sucedida.