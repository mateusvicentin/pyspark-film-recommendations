<h1 align="center">Sistema de Recomendação de Filmes utilizando Docker, PySpark, MongoDB e FastAPI</h1>

<p>
  O projeto tem como objetivo utilizar uma base de dados já existente, contendo o ID do usuário, o ID do filme, a nota atribuída pelo usuário ao filme e a data e hora em que ele realizou a avaliação em formato timestamp. No projeto, será utilizada uma biblioteca chamada ALS, permitindo treinar uma base de dados para ser usada como modelo de recomendação. Por exemplo, o sistema recomendará filmes semelhantes aos que o usuário assistiu e conseguirá "prever" a nota que ele atribuiria a esses filmes. Para isso, o sistema lerá um arquivo TXT contendo esses dados, onde cada linha representa uma entrada. O sistema fará a leitura dessas linhas e as transformará em colunas. As informações sobre filmes e recomendações de notas serão salvas diretamente no MongoDB. Para a verificação das informações, serão utilizados scripts em Python com a estrutura web do FastAPI, integrando com o banco de dados MongoDB e permitindo consultas de usuários via interface web.
</p>
<p>
  Para o desenvolvimento do projeto, foram instanciados dois clusters no Docker para o funcionamento do sistema: um cluster contendo o servidor do Spark e outro cluster contendo o servidor do banco MongoDB.
</p>
<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/947d111e-d697-4981-9850-4e1964266133" alt="img1">
</p>

<h1 align="center">PySpark</h1>

<h2>Iniciando com o PySpark</h2>
<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/5f9225fb-abb3-46cc-9d48-4129a7bd8961" alt="img2">
</p>
<h4>Acessando o serviço do Spark através do link <a href="http://localhost:8888/">http://localhost:8888/</a>, foi criado um novo Notebook para a inicialização do projeto.</h4>
<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/db9ecf36-1800-43f6-afac-fc55c67b6aee" alt="img3">
</p>

```python
from __future__ import print_function

import sys
if sys.version >= '3':
    long = int

from pyspark.sql import SparkSession
from pyspark.ml.evaluation import RegressionEvaluator
from pyspark.ml.recommendation import ALS
from pyspark.sql import Row
```
<h3>Inicializando a Sessão do Spark</h3>
<h4>Nesta seção, configuraremos a conexão com o MongoDB. Vamos criar um banco de dados chamado <code>Filmes</code> e uma coleção chamada <code>Recomendações</code>.</h4>

```python
spark = SparkSession\
        .builder\
        .appName("ProjetoRecomendações")\
        .config("spark.mongodb.read.connection.uri", "mongodb://172.17.0.2:27018/filmes.recomendacoes") \
        .config("spark.mongodb.write.connection.uri", "mongodb://172.17.0.2:27018/filmes.recomendacoes") \
        .config('spark.jars.packages',"org.mongodb.spark:mongo-spark-connector_2.12:10.3.0")\
        .getOrCreate()
```

<h3>Lendo o Arquivo que Contém os Dados</h3>
<h4>O arquivo TXT que será lido é o <code>sample_movielens_ratings.txt</code>. Lembrando que as informações estão dispostas em linhas, sendo necessário transformar esses dados em colunas. </h4>

<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/af941a82-cddc-4ab7-8e93-fbebf40b739b" alt="img4">
</p>

```python
lines = spark.read.text("sample_movielens_ratings.txt").rdd
parts = lines.map(lambda row: row.value.split("::"))
ratingsRDD = parts.map(lambda p: Row(userId=int(p[0]), movieId=int(p[1]),
                                     rating=float(p[2]), timestamp=long(p[3])))
ratings = spark.createDataFrame(ratingsRDD.collect())
ratings.show()
```
<h4>Foram criadas quatro colunas, nomeadas como <code>userId</code>, <code>movieId</code>, <code>rating</code> e <code>timestamp</code>.</h4>

<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/6caac443-ed83-4124-be8d-f3730b50f01d" alt="img5">
</p>

<h3>Treinando os Dados</h3>
<h4>Nesta seção, os dados serão divididos em duas partes: dados para treino e dados para teste. Serão utilizados 80% dos dados para treinamento e 20% para testes. Para isso, foram criadas duas dependências chamadas <code>training</code> e <code>test</code>. A coluna <code>rating</code> será utilizada para esse tipo de teste.</h4>

```python
(training, test) = ratings.randomSplit([0.8, 0.2])
als = ALS(maxIter=5, regParam=0.01, userCol="userId", itemCol="movieId", ratingCol="rating",
              coldStartStrategy="drop")
model = als.fit(training)
```
<h4>Vou calcular a margem de erro da nota para verificar como o modelo identificará o valor da nota a ser considerado.</h4>

```python
predictions = model.transform(test)
evaluator = RegressionEvaluator(metricName="rmse", labelCol="rating",
                                    predictionCol="prediction")
rmse = evaluator.evaluate(predictions)
print("Root-mean-square error = " + str(rmse))
```
<h4>Nesse caso, a margem de erro da nota prevista que o usuário atribuiria ao filme que será recomendado é de 1.5.</h4>
<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/c80ed7eb-892d-41ca-840c-ce2834d5421a" alt="img5">
</p>

<h3>Mostrando o userId, os filmes recomendados e a previsão da nota</h3>
<h4>Mostrando apenas um dos ID para Analise</h4>
<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/0a1a6566-15d3-4011-beb9-8b6e7b962add" alt="img6">
</p>

```python
userRecs = model.recommendForAllUsers(10)
userRecs.show(1, truncate=False)
```
<h1 align="center">MongoDB</h1>
<h2>Salvando os dados no MongoDB</h2>
<h4>Vamos salvar o userID e dois arrays de filmes e notas.</h4>

```python
userRecs.select(userRecs["userId"], \
                userRecs["recommendations"]["movieId"].alias("movieId"),\
userRecs["recommendations"]["rating"].cast('array<double>').alias("rating")).\
    write.format("mongodb").mode("append").save()
```
<h4>Os dados gerados no MongoDB são o userId, uma lista com 10 filmes recomendados (movieId) e a lista de notas para esses 10 filmes.</h4>

<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/49e21230-a37a-42bf-92e6-dea8f3e9189a" alt="img7">
</p>

<h2>Verificando um dos Arquivos no MongoDB</h2>
<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/edd45f7a-9f19-4e1e-ad21-2f8784304f34" alt="img8">
</p>

<h1 align="center">FastAPI</h1>
<h2>Utilizando o FastAPI</h2>
<h4>Vamos utilizar 2 scripts em Python, um chamado <code>mongo</code> e outro chamado <code>main</code>, além do arquivo <code>requirements.txt</code> que contém as bibliotecas usadas no Python.</h4>

<h3>mongo.py</h3>
<h4>O mongo vamos realizar a conexão com o banco de dados do mongodb, passando o nome do banco que no caso é <code>filmes</code> e a collection que é <code>recomendacoes</code></h4>
<h4>Vou chamar a função <code>inicia_conexão()</code> e passar um usuário para pesquisa para verificar se a comunicação está funcionando.</h4>

```python
from pymongo import MongoClient

def inicia_conexao():

    client = MongoClient('localhost', 27018)
    db = client['filmes']
    col = db['recomendacoes']
    return col

def consulta_recomendacoes(usuario, conexao):

    recomendacoes = list(conexao.find({"userId": usuario}))
    list_rec = []
    for rec in recomendacoes:
        list_rec.append((rec['movieId'],rec['rating']))

    return {'Recomendações': list_rec}

conn = inicia_conexao()
res = list(conn.find({"userId": 21 }))
print(res)
```
<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/0c5e85d8-1cd6-4625-b168-59bfe9671912" alt="img9">
</p>

<h3>main.py</h3>
<h4>É responsável por conectar com o FastAPI e fazer as consultas. Irei criar uma função GET para realizar a mesma consulta pelo ID, porém dessa vez através da interface web.</h4>
<h4>Vamos iniciar o servidor executando o código e acessar pelo link <a href="http://localhost:8001/docs/">http://localhost:8001/docs/</a>.</h4>

```python
from fastapi import FastAPI
from typing import Dict
import uvicorn
from bd import mongo

app = FastAPI()
conexao = mongo.inicia_conexao()

@app.get("/rec/v2/{usuario}")
def consulta_rec(usuario: int):
    return {"usuario": usuario, "resultado_recs": mongo.consulta_recomendacoes(usuario, conexao)}

if __name__ == "__main__":
    uvicorn.run(app, host='localhost', port=8001)
```
<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/c2a332b0-14e9-4857-b984-28d7c88c36cc" alt="img10">
</p>
<h3>Apos acessar</h3>
<h4>São informados os requests criados. No caso, tenho apenas um que irá retornar o usuário, além dos arrays de filmes recomendados e notas.</h4>
<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/913a2123-8e9a-4c37-90ee-71dcbda7febc" alt="img11">
</p>
<h4>Vamos realizar a consulta pelo ID do usuário. Neste caso, irei passar o ID 20.</h4>
<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/1a0f430b-c4e7-43c3-92c6-e70661b98809" alt="img12">
</p>
<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/6d47f447-1fdb-4722-907f-7e8f481e637d" alt="img13">
</p>
<h3>Retorno da consulta</h3>
<h4>No caso, são retornados os IDs dos filmes recomendados ao usuário e a nota prevista que ele irá avaliar para aquele filme.</h4>
<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/1a29f1cc-b5be-41ca-b30a-51ebf8fd5f3f" alt="img14">
</p>
<h2>Consultando mais de um usuário e retornando apenas o ID do filme recomendado</h2>

```python
@app.get("/rec/v3/{usuarios}")
def consulta_rec(usuarios: str) -> Dict[str, Dict[int, list]]:
    usuarios_list = [int(usuario) for usuario in usuarios.split(",")]
    resultados = {}
    for usuario in usuarios_list:
        recomendacoes = list(conexao.find({"userId": usuario}))
        list_rec = [rec['movieId'] for rec in recomendacoes]
        resultados[f"Recomendações para o usuário {usuario}"] = list_rec

    return resultados
```

<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/945fa9c2-83a7-4fa6-bbdb-f614bddc8aae" alt="img15">
</p>
<h4>Vou utilizar o <code>/rec/v3/{usuarios}</code> que foi o que criei para esse tipo de consulta.</h4>

<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/f4ffe2aa-ea7b-4327-babf-e7bc5cfc9287" alt="img16">
</p>

<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/d63088d2-f7e4-4dc6-96b0-6e91e02dd060" alt="img17">
</p>

<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/fe3269bf-144d-4dce-84a3-520dfc6a8924" alt="img18">
</p>
<h4>Nesse método, será retornado apenas o ID do usuário e os filmes que serão recomendados a ele.</h4>



