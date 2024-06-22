<h1 align="center">Sistema de Recomendação utilizando PySpark, MongoDB e FastAPI</h1>

<p>
  O projeto tem como objetivo utilizar uma base de dados já existente, contendo o ID do usuário, o ID do filme, a nota atribuída pelo usuário ao filme e a data e hora em que ele realizou a avaliação em formato timestamp. No projeto, será utilizada uma biblioteca chamada ALS, permitindo treinar uma base de dados para ser usada como modelo de recomendação. Por exemplo, o sistema recomendará filmes semelhantes aos que o usuário assistiu e conseguirá "prever" a nota que ele atribuiria a esses filmes. Para isso, o sistema lerá um arquivo TXT contendo esses dados, onde cada linha representa uma entrada. O sistema fará a leitura dessas linhas e as transformará em colunas. As informações sobre filmes e recomendações de notas serão salvas diretamente no MongoDB. Para a verificação das informações, serão utilizados scripts em Python com a estrutura web do FastAPI, integrando com o banco de dados MongoDB e permitindo consultas de usuários via interface web.
</p>
<p>
  Para o desenvolvimento do projeto, foram instanciados dois clusters no Docker para o funcionamento do sistema: um cluster contendo o servidor do Spark e outro cluster contendo o servidor do banco MongoDB.
</p>
<p align="center">
  <img src="https://github.com/mateusvicentin/pyspark-film-recommendations/assets/31457038/947d111e-d697-4981-9850-4e1964266133" alt="img1">
</p>

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
<h3>Iniciando a Seção do Spark </h3>
<h4>Nessa parte iremos configuração a conexão com o MongoDB, vamos criar um Database chamado <code>Filmes</code> e uma Collection chamada <code>Recomendações</code></h4>

```python
spark = SparkSession\
        .builder\
        .appName("ProjetoRecomendações")\
        .config("spark.mongodb.read.connection.uri", "mongodb://172.17.0.2:27018/filmes.recomendacoes") \
        .config("spark.mongodb.write.connection.uri", "mongodb://172.17.0.2:27018/filmes.recomendacoes") \
        .config('spark.jars.packages',"org.mongodb.spark:mongo-spark-connector_2.12:10.3.0")\
        .getOrCreate()
```


