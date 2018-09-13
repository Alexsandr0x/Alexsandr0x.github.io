---
layout: post
title: Requests + BeautifulSoup para captura de dados web!
date: 2018-09-01 19:35:26
tags: web-crawling
author: alexsandro
---

Dentro do mercado de trabalho ou em projetos é comum a necessidade de criarmos nossa própria base de dados de algum assunto para depois fazermos alguma análise. Isso pode acontecer com qualquer tipo de site que não nos proporcione uma API aberta.

Dessa forma podemos aprender conceitos simples de raspagem de dados usando Python simulando um problema desses. Nesse caso vamos supor um seguinte problema:

Vamos supor que você queira entender características de um certo cantor ou banda, qualquer informação que você possa usar para comparar dois músicos diferentes. Nós podemos compará-los por estilos, 'plays' no Spotify, 'views' no Youtube. Porém se quisermos ir mais fundo e entender as letras desses músicos, para analisar sentimento ou similaridade de palavras entre músicos, precisamos montar uma base de dados com essas letras. E é isso que vamos fazer!

__No final deste texto vamos sair com um script que dado um músico ele nos retorna todas as letras de suas músicas!__

Ok! então precisamos de uma fonte de dados para capturar essas letras. Para alguns musicos internacionais existem APIs que nos trazem esse dado que você pode __[ver aqui](https://www.programmableweb.com/category/lyrics/apis?category=20278)__ mas há uma falta de músicos brasileiros e como queremos treinar a raspagem de dados vamos nos forçar a capturar esses dados de um site de forma mais “bruta”, o escolhido foi o __[Letras.com](https://www.letras.com)__.

Para isso, vamos tentar entender o alvo de nossa captura, se acessarmos, por exemplo, a página de algum artista e analisar o HTML vamos conseguir notar padrões que podem ser interessantes para gente.

Vamos acessar a pagina do musico [Rincon Sapência](https://www.letras.com/rincon-sapiencia/) como exemplo. Logo de início vemos o nome do músico, mais tocadas e __todas as músicas__, sendo este parte da página uma lista com, como o próprio nome diz, todas as músicas daquele cantor. agora é só analisar o HTML. Inspecionando o elemento dessa pequena parte:


<amp-img src="{{ site.baseurl }}assets/images/letras_todas_as_musicas.png" layout="responsive" width="648px" height="164px" alt="" class="mb3"></amp-img>

Encontramos uma estrutura parecida com esta:

```html
<div class="cnt-list--alp">
	<ul class="cnt-list"> 
		<li><a id="A" href="/rincon-sapiencia/a-coisa-ta-preta/">A Coisa Tá Preta</a></li>
		<li><a href="/rincon-sapiencia/a-noite-e-nossa/">A Noite é Nossa</a></li>
		<li><a href="/rincon-sapiencia/a-volta-pra-casa/">A Volta pra Casa</a></li>
		<li><a href="/rincon-sapiencia/afro-rep/">Afro Rep</a></li>
		<li><a href="/rincon-sapiencia/amores-as-escuras/">Amores às Escuras</a></li>
		<li><a href="/rincon-sapiencia/andar-com-fe/">Andar Com Fé</a></li>
		<li><a href="/rincon-sapiencia/rea-de-conforto/">Área de Conforto</a></li>
	</ul>
</div>
```

Nesse bloco de HTML já entendemos que uma vez na página do músico conseguimos o __nome da música__ e um __link__ para cada música. Esse será nosso primeiro trabalho.

Vamos fazer agora, em Python, exatamente o que fizemos aqui: Acessar a página de um músico e listar as suas músicas.

Se a primeira coisa que vamos fazer é __acessar uma página__ precisamos então usar a biblioteca __requests__ do python para a tal, uma requisição get, com a URL do que queremos pegar deve ser o bastante:

```python
import requests

def get_all_musics(musician):
	all_musics = []
	
	# Url que iremos fazer a raspagem, nesse caso, a url 
	# base do Letras + o nome do musico que queremos trabalhar.
	url = "https://www.letras.com/{}".format(musician)
	
	# Requisição web para a nossa url ja montada
	# isso no retornara um resultado http
	response = requests.get(url)

	# Desse resultado, podemos pegar o seu conteudo, que sera uma lista de bytes
	# representando o nosso html
	html = response.content

	return all_musics
```

Do jeito que as coisas estão o código ainda retorna apenas uma lista vazia. Ainda precisamos popular a lista ```all_musics``` com o conteúdo dos elementos que havíamos observado no HTML do site.

Como já temos o HTML basta trabalharmos nele, vamos usar a biblioteca __[BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)__. Ela vai facilitar filtrarmos elementos por id e classe sem precisarmos aprender seletores CSS ou XPath (ainda reservo um post só para isso).

Para instalar use pipy:

```bash
pip install beautifulsoup4
```

Agora vamos transformar o conteúdo de nossa requisição em um objeto que possa ser filtrado para acharmos a url e o nome da musica.

```python
import requests
from bs4 import BeautifulSoup

def get_all_musics(musician):
	all_musics = []
	
	# Url que iremos fazer a raspagem, nesse caso, a url 
	# base do Letras + o nome do musico que queremos trabalhar.
	url = "https://www.letras.com/{}".format(musician)
	
	# Requisição web para a nossa url ja montada
	# isso no retornara um resultado http
	response = requests.get(url)

	# Desse resultado, podemos pegar o seu conteudo, que sera uma lista de bytes
	# representando o nosso html
	html = response.content

	# faz o 'parsing' para um objeto em que possamos trabalhar
	soup = BeautifulSoup(html, 'html.parser')

	# captura o elemento div que tenha a classe 'cnt-list--alp'
	all_songs = soup.find('div', {'class': 'cnt-list--alp'})

	# Do elemento armazenado em all_songs, procura em seus filhos por
	# elementos de tag 'li' (aqui chegamos nas linhas da lista)
	song_list = all_songs.find_all('li')

	return all_musics
```

Ainda não retornamos nada, mas se quisermos retornar o ```song_list``` teremos algo parecido com isso:

```
[
	<li><a href="/rincon-sapiencia/a-coisa-ta-preta/" id="A">A Coisa Tá Preta</a></li>,
	 <li><a href="/rincon-sapiencia/a-noite-e-nossa/">A Noite é Nossa</a></li>,
	 <li><a href="/rincon-sapiencia/a-volta-pra-casa/">A Volta pra Casa</a></li>,
	 <li><a href="/rincon-sapiencia/afro-rep/">Afro Rep</a></li>,
	 <li><a href="/rincon-sapiencia/amores-as-escuras/">Amores às Escuras</a></li>,
	 <li><a href="/rincon-sapiencia/andar-com-fe/">Andar Com Fé</a></li>,
	 <li><a href="/rincon-sapiencia/rea-de-conforto/">Área de Conforto</a></li>,
	 <li><a href="/rincon-sapiencia/autentico-x-excentrico-x-incomparavel/">Autêntico X Excêntrico X Incomparável</a></li>,
	 <li><a href="/rincon-sapiencia/batidao/" id="B">Batidão</a></li>,
	 <li><a href="/rincon-sapiencia/bencao/">Benção (part. William Magalhães)</a></li>,
	 <li><a href="/rincon-sapiencia/coisas-de-brasil/" id="C">Coisas de Brasil</a></li>,
	 [...]
```

Isso tudo é uma lista com todas as músicas presentes naquela página, agora só precisamos extrair a url e o nome da musica. Para isso continuaremos usando o BeautifulSoup, adicionamos ao código anterior as linhas:

```python
	for song_elem in song_list:
		# Note que o href esta dentro da tag 'a', que esta dentro do nosso item capturado 'li'
		link = song_elem.find('a')['href']
		name = song_elem.find('a').text
		# Concatena com a url base
		link = 'https://www.letras.com' + link

		song = {
			"url": link,
			"name": name
		}

		all_musics.append(song)
```

Dessa forma temos agora na lista ```all_musics``` um dicionário com nome e link de cada música! Conseguimos com poucas linhas de código já fazermos algo interessante, mas ainda não é nosso objetivo final, nosso próximo passo é pegar essa lista e para cada música capturar a letra dessa música acessando a url.


<amp-img src="{{ site.baseurl }}assets/images/letras_letra_da_musica.png" layout="responsive" width="553px" height="408px" alt="" class="mb3"></amp-img>

Inspecionando o elemento que armazena a letra temos isso:

```html
<div class="cnt-letra p402_premium" style="font-size: 19px;">
	<article>
		<p>Salve!
		<br>OK!<br>
		Rincon Sapiência, conhecido também como Manicongo, certo?
		<br>Quando alguém fala que eu não sou um MC acima da média, eu falo<br>
		(Ahn? Ahn? Ahn? Ahn?)<br>
		Eu não entendo nada, pai!<br>
		(Ahn? Ahn? Ahn? Ahn?)<br>
		A cultura do MC ainda vive, certo? Se depender de mim<br>
		Vam'bora!</p>
		<p>Meu verso é livre, ninguém me cancela
		<br>Tipo Mandela saindo da cela
		<br>Minhas linha voando cheia de cerol
		<br>E dá dó das cabeça quando rela nela
		<br>Partiu para o baile, fugiu da balela
		<br>Batemos tambores, eles panela
		<br>Roubamos a cena, não tem canivete
		<br>As patty derrete que nem muçarela
		<br>Quente que nem a chapinha no crespo
		<br>Não, crespos estão se armando
		<br>Faço questão de botar no meu texto<br>
```

Dessa forma, conseguimos fazer um script parecido com o anterior, porém agora filtrando pela classe ```cnt-letra``` e pegando todo o conteúdo de texto nele contido, a gente pode fazer isso dessa forma:

```python
def get_lyric_from_url(url):
	response = requests.get(url)
	soup = BeautifulSoup(response.content, 'lxml')
	lyric = soup.find('div', 'cnt-letra').getText(separator=' ')

	return lyric
```

Se rodarmos o método ```get_lyric_from_url``` para qualquer url de musica do Letras.com já teremos, em formato de string, a letra desta música.

Agora já temos um método que captura todas as músicas de um determinado músico e um outro método que para o link dessa musica captura sua letra. Para não me alongar muito esse post fica por aqui. Eu pretendo ainda usar esse código para trabalhar com algum texto sobre processamento de linguagem natural e visualização de dados mas isso fica para o futuro.

Esse é o código final de todo o nosso trabalho, com o bônus que ainda gera um arquivo json armazenado tudo que foi capturado!

```python
import json
import requests
from bs4 import BeautifulSoup

def get_all_musics(musician):
	all_musics = []
	
	# Url que iremos fazer a raspagem, nesse caso, a url 
	# base do Letras + o nome do musico que queremos trabalhar.
	url = "https://www.letras.com/{}".format(musician)
	
	# Requisição web para a nossa url ja montada
	# isso no retornara um resultado http
	response = requests.get(url)

	# Desse resultado, podemos pegar o seu conteudo, que sera uma lista de bytes
	# representando o nosso html
	html = response.content

	# faz o 'parsing' para um objeto em que possamos trabalhar
	soup = BeautifulSoup(html, 'html.parser')

	# captura o elemento div que tenha a classe 'cnt-list--alp'
	all_songs = soup.find('div', {'class': 'cnt-list--alp'})

	# Do elemento armazenado em all_songs, procura em seus filhos por
	# elementos de tag 'li' (aqui chegamos nas linhas da lista)
	song_list = all_songs.find_all('li')

	for song_elem in song_list:
		# Note que o href esta dentro da tag 'a', que esta dentro do nosso item capturado 'li'
		link = song_elem.find('a')['href']
		name = song_elem.find('a').text
		# Concatena com a url base
		link = 'https://www.letras.com' + link

		song = {
			"url": link,
			"name": name
		}

		all_musics.append(song)

	return all_musics

def get_lyric_from_url(url):
	response = requests.get(url)
	soup = BeautifulSoup(response.content, 'lxml')
	lyric = soup.find('div', 'cnt-letra').getText(separator=' ')

	return lyric

musician_code = 'rincon-sapiencia'

songs = get_all_musics(musician_code)

for song in songs:
	lyric = get_lyric_from_url(x[0]['url'])
	song['lyric'] = lyric

with open('songs_{}.json'.format(musician_code), 'w') as file_o:
	file_o.write(json.dumps(songs))
```