---
layout: post
cover: 'assets/images/dados_tse_cover.jpg'
title: Pandas + Matplolib - Analisando os bens dos candidatos as eleições
date: 2018-09-07 16:35:00
tags: pandas analises
author: alexsandro
---


Já a um tempo eu queria preparar um apanhado de coisas interessantes e básicas de Pandas e quando eu descobri que o TSE tem uma planilha com os [candidatos e suas declarações de bens](http://www.tse.jus.br/eleicoes/estatisticas/repositorio-de-dados-eleitorais-1/repositorio-de-dados-eleitorais) achei o momento certo para escrever esse "tutorial".

Vamos usar conceitos de Dataframes do Pandas, que são objetos que nos dão uma mãozinha na hora de trabalhar com planilhas. Na parte de visualização vamos brincar um pouco com os gráficos básicos da biblioteca matplotlib.

Mas primeiro vamos baixar as planilhas que iremos trabalhar. [Nesse link](http://www.tse.jus.br/eleicoes/estatisticas/repositorio-de-dados-eleitorais-1/repositorio-de-dados-eleitorais) acesse o ano de 2018. Agora faça o download dos arquivos [Candidatos (formato ZIP)](http://agencia.tse.jus.br/estatistica/sead/odsele/consulta_cand/consulta_cand_2018.zip) e [Bens de candidatos (formato ZIP)](http://agencia.tse.jus.br/estatistica/sead/odsele/bem_candidato/bem_candidato_2018.zip) (nesses links você também pode baixar diretamente).

São dois arquivos .zip. Vamos entender cada um, explicando brevemente as colunas que iremos usar. Cabe dizer que se você quiser usar outras colunas existe um arquivo PDF dentro de cada arquivo zipado explicando cada coluna.

* **Candidatos:** Planilha que lista todos os candidatos exceto presidente, ou seja: Deputado Estadual, Deputado Federal, Governador, Senador e seus respectivos suplentes e vices. Aqui vamos usar basicamente as colunas:

	* **SQ_CANDIDATO:** Essa sera nossa "chave primaria" para o candidato. É basicamente um número único relacionado a candidatura e que também pode ser vista na tabela de bens.

<br>
* **Bens de candidatos:** Planilha com a declaração de cada bem de todos os candidatos, ou seja, cada linha nessa tabela representa um bem de alguém. Se um candidato, por exemplo, declarou uma casa e um carro, terá duas linhas com o mesmo SQ_CANDIDATO. Nessa tabela vamos usar basicamente duas colunas:

	* **SQ_CANDIDATO:** Vamos usar para agregar, nessa tabela elá não é uma chave unica, mas sim uma coluna que usaremos pra juntar todos os bens de um único candidato
	
	* **VR_BEM_CANDIDATO:** O valor do bem declarado em reais. Importante dizer que alguns dados aqui estão populados de maneira errada e no decorrer do texto vamos ver exatamente onde da pra notar isso. 

Agora que já estamos familiarizado com as tabelas que iremos trabalhar vamos tentar agregar a soma de todos os bens de um candidato e adicionar esse resultado em uma nova coluna na tabela de candidatos. Isso irá nos ajudar a listar os top candidatos mais "ricos".

Vamos ao código! Primeira coisa: carregar as planilhas:

```python
import glob
import matplotlib.pyplot as plt
import pandas as pd
	
candidatos_csv_list = glob.glob("consulta_cand_2018_*.csv")

bens_csv_list = glob.glob("bem_candidato_2018_*.csv")
```

Para quem nunca usou: uso o método glob.glob para pegar todos os arquivos que o nome seja valido para algum regex, isso é necessário porque vamos unir todas as tabelas estaduais.

Vamos carregar todas as tabelas de candidatos, usar como índice a coluna "SQ_CANDIDATO"  e por fim unir todas as tabelas:

```python	
dataframes_cands = []

for candidato_csv in candidatos_csv_list:
    cand_df = pd.read_csv(candidato_csv, error_bad_lines=False, sep=';', encoding='latin-1')
    cand_df = cand_df.set_index('SQ_CANDIDATO')
    dataframes_cands.append(cand_df)

candidato_dfs = pd.concat(dataframes_cands)
```

Fazendo a mesma coisa para a planilha de bens:

```python
dataframes_bens = []

for bens_csv in bens_csv_list:
    cand_df = pd.read_csv(bens_csv, error_bad_lines=False, sep=';', encoding='latin-1')
    dataframes_bens.append(cand_df)
    
bens_df = pd.concat(dataframes_bens)
```
Temos então os dois dataframes, que representam todos o candidatos e todas as declarações de bens, Então precisamos agora agregar as declarações. 

Antes de fazermos isso precisamos fazer um tratamento quanto a coluna VR_BEM_CANDIDATO. Como no brasil usamos ',' como separador em números o Pandas entendeu essa coluna como uma coluna de strings. precisamos tratar isso, fazendo um replace na virgula por ponto e por fim convertendo para float. Aqui vamos usar o apply junto a uma [função lambda](https://pythonhelp.wordpress.com/tag/lambda/). 
```python
bens_df = pd.concat(dataframes_bens)
bens_df['VR_BEM_CANDIDATO'] = bens_df['VR_BEM_CANDIDATO'].apply(lambda x: float(x.replace(',', '.')))
```

Por fim, podemos agrupar todas as linhas na tabela bens_df com o mesmo SQ_CANDIDATO e somar VR_BEM_CANDIDATO. Ficaria dessa maneira:

```python
bens_por_candidato = bens_df.groupby(['SQ_CANDIDATO'])['VR_BEM_CANDIDATO'].sum()

candidato_dfs['BENS_AGG_SUM'] = bens_por_candidato
```

Nesse caso, estamos criando uma pandas.Series na primeira linha, que tem como índice o SQ_CANDIDATO e o valor a soma de todos os valores em VR_BEM_CANDIDATO. Na segunda linha atribuímos essa series a uma coluna em ```candidato_dfs```. Dessa forma temos uma nova coluna na tabela de candidatos com a declaração de todos os bens!

Aqui vem nosso primeiro (e até onde eu mexi nesses dados o único que tive) problema com essa base de dados. Aparemente houveram erros na hora de inserir os valores de certos bens declarados. Por exemplo, vamos olhar como esta agora os politicos com mais bens declarados:

<amp-img src="{{ site.baseurl }}assets/images/top_politicos.png" layout="responsive" width="580px" height="407px" alt="" class="mb3"></amp-img>

Note que, por exemplo, o candidato de nome EVERALDO LISBOA DE BRITO tem quase 1.2 Bilhões em bens declarados! e ai cabe sempre um trabalho de olhar a base e até outras fontes para garantir, vamos pegar o nome desse candidato que esta no topo e olhar para a planilha:

<amp-img src="{{ site.baseurl }}assets/images/table_wrong_dataset.png" layout="responsive" width="527px" height="162px" alt="" class="mb3"></amp-img>

Note que a coluna DS_BEM_CANDIDATO descreve o bem que aquele candidato esta declarando, e no caso de 4 dos 5 bens que esse candidato declarou tem o valor na descrição, mas quando vamos para a VR_BEM_CANDIDATO os numeros estão com alguns zero a mais! fazendo com que um Gol de 16 mil reais se torne um **carro de 160 milhões**. Para a nossa analise fica que precisamos limpar a base e mesmo assim não teremos certeza se havera um ou outro bem com valor errado.

Para uma lição maior fica que bases de dados são quase sempre populadas, em algum momento, por humanos, e humanos cometem erros, é bem provavel que esse problema, se já não estiver dando irá dar uma tremenda dor de cabeça a esse candidato.

Beleza, entendido o problema precisamos resolve-lo, ao olhar a base notei que é bem raro um unico bem valer mais que 100 milhões, e todos aqueles que estavam no topo do grafico erroneamente era justamente por bens nessa medida. Apesar de não ser a estrategia a apliquei e o resultado foi esse:

<amp-img src="{{ site.baseurl }}assets/images/top_politicos_filtrado.png" layout="responsive" width="580px" height="407px" alt="" class="mb3"></amp-img>

Como sabemos que esse filtro foi bem efetivo? bom, não podemos olhar um a um todos os bens, mas uma leve pesquisa na internet nos mostra que já fizeram uma noticia com os candidatos com mais bens, e justamente os nomes FERNANDO DE CASTRO MARQUES e OGARI DE CASTRO PACHECO são citados como os mais "bem vividos".

Temos essa lista com os bens de todos os politicos, e nessa mesma tabela temos a relação de partidos de cada um, o que me deu uma ideia super natural de analisar a média de bens dos cadidatos agregando-os por partido.

Essa tarefa é bem facil e basicamente usamos tudo que já sabemos, veja o código e o grafico gerado:
```python
bens_por_partido = candidato_dfs.groupby(['SG_PARTIDO'])['BENS_AGG_SUM'].mean()
bens_por_partido = bens_por_partido.sort_values()
fig, ax = plt.subplots()
x, y = list(bens_por_partido.values), list(bens_por_partido.index)
fig.set_size_inches(18.5, 10.5)
import locale
locale.setlocale(locale.LC_ALL, 'pt_BR.UTF-8')
ax.bar(y, x)
ax.set_xticklabels(y, rotation='vertical')
fig.suptitle('Média de Bens declarados por Deputados por Partido')
fig.savefig('top_partidos.png', bbox_inches='tight')
```

<amp-img src="{{ site.baseurl }}assets/images/top_partidos.png" layout="responsive" width="1100px" height="780px" alt="" class="mb3"></amp-img>

Bom, chegamos a um resultado bacana, tudo que é necessario para chegar nesse ultimo grafico esta nos códigos que adicionei durante o texto mas se preferir tem esse [jupyter notebook](https://gist.github.com/Alexsandr0x/d607ce4e9488f94cd6f1e2688c2ab055) com todo o código usado.

Se quiser dar continuidade ao aprendizado de manipulação de tabelas e visualização de dados com essa tabela sugiro usar de inspiração noticias que estão surgindo que usam de base esses dados e tentar chegar no mesmo resultado como o resultado chegado pelo Jornal Nexo em __[As profissões de quem vai estar na urna em outubro, por partido](https://www.nexojornal.com.br/grafico/2018/09/03/As-profiss%C3%B5es-de-quem-vai-estar-na-urna-em-outubro-por-partido?utm_campaign=Echobox&utm_medium=Social&utm_source=Twitter#Echobox=1536015078)__.