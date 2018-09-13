---
layout: post
title: Rede neural para classificar questões de vestibular
date: 2018-09-01 19:35:26
tags: machine learning
author: alexsandro
---

Eu escrevi esse texto com dois objetivos: Mapear tudo que aprendi durante esse projeto e servir para quem quer aprender como fazer algo parecido. Pretendo escrever um texto sobre cada uma das tecnicas mas nesse texto em especifico é mais um resumo.

Nesse pequeno projeto meu objetivo foi desenvolver um algoritmo que pudesse me responder a qual disciplina pertence uma determinada questão, foi uma escolha pensada na quantidade de dados e na facilidade de captura-los. Existem muitos sites gratuitos que compilam questões para vestibulandos de vestibular e concursos públicos e eles foram.

Foi feito para a disciplina de Processamento de Linguagem Natural do curso de Ciência da Computação na Universidade Federal do ABC, disciplina ministrada pelo otimo professor [Jesus Mena Chalco](http://professor.ufabc.edu.br/~jesus.mena) (do qual agradeço muito por me ajudar no escopo do projeto).

Agora chega de bla bla bla vamos ao que interessa!

<h3 id="heading3">O Problema!</h3>

A ideia do projeto era ter um algoritmo que pudesse nos inferir algum dado quando entregue a ele alguma questão de vestibular, inicialmente queria que ele fosse até o sub-topico da disciplina, mas notei que isso seria muito dificil e precisaria de muito mais dados. Me fechei apenas a classificação de 5 disciplinas: __português, matemática, historia, quimica e biologia__


<h3 id="heading3">Os Dados!</h3>

Foi necessario pegar uma base de dados massiva para o teste. A base foi montada através de um web-scrapping de um site de questões de vestibulares e concursos. Para cada disciplina que seria classificada foi capturado 2000 questões diferentes, totalizando um valor de 10000 questões.

Esses numeros são bem grandes e não duvido que um resultado semelhante ou melhor possa ser encontrado com uma base de dados mais humilde. Mas como todo projeto de faculdade muitos pontos não foram pensados corretamente ¯\\\_(ツ)\_/¯.

Desta forma, depois de uma noite inteira o web-crawler me retorno 10000 linhas com "questão", "tema" e "disciplina", fica mais ou menos assim:


|questao|tema|disciplina|
--------|----|----------|
|A  técnica do carbono-14 permite a datação de fósseis pela medição dos valores de emissão beta desse isótopo presente no fóssil. Para um ser em vida, o máximo são 15 emissões beta/(min g). Após a morte, a quantidade de 14C se reduz pela metade a cada 5 730 anos. A prova do carbono 14. Disponível em: http://noticias.terra.com.br. Acesso em: 9 nov, 2013 (adaptado). Considere que um fragmento fóssil de massa igual a 30 g foi encontrado em um sítio arqueológico, e a medição de radiação apresentou 6 750 emissões beta por hora. A idade desse fóssil, em anos, é "|Estrutura Atômica|Quimica|
|Nos últimos anos, a televisão tem passado por uma verdadeira revolução, em termos de qualidade de imagem, som e interatividade com o telespectador. Essa transformação se deve à conversão do sinal analógico para o sinal digital. Entretanto, muitas cidades ainda não contam com essa nova tecnologia. Buscando levar esses benefícios a três cidades, uma emissora de televisão pretende construir uma nova torre de transmissão, que envie sinal às antenas A, B e C, já existentes nessas cidades. As localizações das antenas estão representadas no plano cartesiano: |Geometria Analítica|Matematica|
|Suponha que um pesticida lipossolúvel que se acumula no organismo após ser ingerido tenha sido utilizado durante anos na região do Pantanal, ambiente que tem uma de suas cadeias alimentares representadas no esquema:PLÂNCTON → PULGA-D’ÁGUA → LAMBARI → PIRANHA → TUIUIÚ Um pesquisador avaliou a concentração do pesticida nos tecidos de lambaris da região e obteve um resultado de 6,1 partes por milhão (ppm). Qual será o resultado compatível com a concentração do pesticida (em ppm) nos tecidos dos outros componentes da cadeia alimentar?|Ecologia|Biologia|

<br>
Trabalho de captura feito! Agora precisamos carregar essa planilha em nosso código e fazer alguns tratamentos no texto. Pandas foi usado para carregar para o Python a planilha e foi usado estrategias de processamento de textos como __stopwords__ e __stemmings__.


Esse é o código para carregar a base, aplicar stopwords, stemming e separar a X (questões) do vetor y (disciplinas) fica dessa maneira:

```python
import pandas as pd

from nltk import SnowballStemmer

stemmer = SnowballStemmer("portuguese", ignore_stopwords=True)

stopwords = open('classifiers/stopword.txt', 'r').read().split('\n')

questions_data = pd.read_csv("dataset/crawler/questoes.csv")

# Carrega as questões
X = questions_data.questao

# Aplica steamming na palavra e já filtra por stopwords na mesma linha
X = [[stemmer.stem(w) for w in x.split(' ') if w.lower() not in stopwords] for x in X]

# Nosso y sera um vetor binario onde cada indice representa a tag de uma disciplina
# Ou seja, se a questão for de biologia o y dela sera = [0, 1, 0, 0, 0]
y = [[int(d == 'Portugues'), int(d == 'Biologia'), int(d == 'Matematica'), int(d == 'Quimica'), int(d == 'Historia')] for d in questions_data.disciplina]
```


<h3 id="heading3">A Rede Neural!</h3>

Depois de algumas pesquisas de metodos encontrei um rede neural chamada LSTM (Long Short Term Memory). Que basicamente é uma rede neural convolucional que tenta levar em consideração a dependencia posicional das palavras simulando um tratamento de "tempo" a rede neural. Dessa forma a rede que esta tratando a frase x<sub>t</sub> considera a para x<sub>it-1</sub>, x<sub>t-2</sub>, etc...

O Conceito é complexo e a matematica mais ainda, eu ainda vou estudar a fundo o LSTM mas enquanto isso você pode conhecer mais sobre essa ferramenta matemagica __[aqui](http://adventuresinmachinelearning.com/keras-lstm-tutorial/)__.

No resumo, vamos criar uma rede neural de forma sequencial com a biblioteca __Keras__.

A criação em codigo é super simples e vamos aproveitar o momento para entender cada elemento que compoe ela:

```python
from keras.layers import Embedding, LSTM, Dense
from keras.models import Sequential
from keras.preprocessing.text import Tokenizer
from keras.preprocessing.sequence import pad_sequences

tokenizer = Tokenizer(num_words=5000)
tokenizer.fit_on_texts(X)
sequences = tokenizer.texts_to_sequences(X)
data = pad_sequences(sequences, maxlen=50)

model = Sequential()
model.add(Embedding(5000, 128, input_length=50))
model.add(LSTM(128, dropout=0.2, recurrent_dropout=0.2))
model.add(Dense(5, activation='sigmoid'))
model.compile(loss='binary_crossentropy', optimizer='adam', metrics=['accuracy'])

model.fit(data, np.array(y), validation_split=0.5, epochs=5)

```

Antes de explicar a rede neural, cabe a explicação de mais um pré processamento, que é o Tokenizador e o pad_sequences.

A LSTM, por mais que seja muito usada para processamento de texto, não necessariamente recebe como input texto. Ela deve receber uma matriz numérica. Onde cada numero vai representar uma palavra, cada linha um "tempo".

Para transformarmos palavras em numeros, é necessario usar o Tokenizador, o tokenizador vai fazer duas coisas importantes em nosso código: Criar um dicionario que relacionara um número a uma palavra, e considerar um numero de corte (```num_words```) que irá considerar as ```n``` palavras mais frequentes apenas. Isso reduz consideravelmente o tamanho da nossa entrada.

Mas isso não é o bastante, pense que antes tinhamos uma lista de palarasm agora temos uma lista de numeros. Mas a LSTM recebe uma matriz, onde cada linha é uma frase que contem uma outra lista, dessa vez contendo as palavras (que já estão descritas de maneira númerica graças ao Tokenizer). Nesse caso, usaremos o pad_sequence. Ele irá transformar  

A gente pode visualizar melhor esse código entendendo o fluxo do dado pela rede:

<amp-img src="{{ site.baseurl }}assets/images/lstm.svg" layout="responsive" width="720px" height="240px" alt="" class="mb3"></amp-img>

Agora, vamos caso a caso:



* [Embedding](https://keras.io/layers/embeddings/): É uma camada que torna uma matriz esparsa em uma matriz densa com um certo tamanho, 