---
layout: post
title:  "Debugando Python dockerizado no PyCharm"
author: Rauan
date:   2024-01-21
tags: debug docker python tutorial
---

### `print()` my beloved

Eu passei um bom tempo usando `print()` para debugar código Python que estava rodando em containers por não saber
como usar o debugger de maneira adequada (dã). Acho que descobri.

Como eu utilizo o <a href="https://www.jetbrains.com/pycharm/" target="_blank">PyCharm</a> (e talvez você devesse também),
esse tutorial não cobre <a href="https://code.visualstudio.com/" target="_blank">VS Code</a> ou
<a href="https://docs.python.org/3/library/pdb.html" target="_blank">pdb</a> puro.

Sigamos.

### Caso simples: Uma app boba

Digamos que você tem uma web app _Flask_ super complexa:

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    my_return = "Hello, World!"
    return my_return

if __name__ == '__main__':
    app.run(host="0.0.0.0")
```

Dockerfile:

```dockerfile
FROM python:3.10
WORKDIR /
COPY . .
RUN pip3 install -r requirements.txt
CMD [ "python3", "./main.py"]
```
docker-compose.yml:
```yaml
version: "3.5"
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "5000:5000"
```

* <small>Vou utilizar um _docker-compose.yml_ por ser útil para mapear portas e para adicionar outros serviços (ex.: bd),
como veremos adiante.</small>

Bom, se tentarmos debugar utilizando o interpretador que está aí no seu _venv_ (pf esteja usando um venv), vai funcionar,
porém, não estaríamos debugando a aplicação rodando no container e sim localmente. 
Temos que especificar um interpretador que esteja dentro do container que será debugado.

Add novo interpretador "Docker Compose"

<img src="/images/pycharm-docker-debug/new_interpreter-1.gif" style="display:block;margin:0 auto">

* <small> Se preferir: menu > settings > project > python interpreter > add new interpreter </small>

Especifique o caminho do seu _docker-compose.yml_. 
Depois, escolha o nome do que deseja debugar, especificando o mesmo nome que foi utilizado no
_docker-compose.yml_. No nosso caso é _app_. Clique `next`. 

<img src="/images/pycharm-docker-debug/new_interpreter-2.png" style="display:block;margin:0 auto">

* <small> Poderiam haver outros serviços como bancos de dados ou caches que não seriam alvo de debug </small>

Por último, escolhemos o interpretador. _System Interpreter_ > `/usr/local/bin/python3`, clique `create`.

<img src="/images/pycharm-docker-debug/new_interpreter-3.png" style="display:block;margin:0 auto">

* <small>Deixamos na opção _System Interpreter_ pois não tem necessidade de usar um venv dentro do container.</small>

Ok, interpretador criado!

O próximo passo é criar uma configuração de _run/debug_, para rodar nosso programa utilizando o interpretador que acabamos
de criar. Láá em cima, do lado da <a href="https://www.google.com/search?q=quantas+pernas+tem+uma+barata" target="_blank">baratinha</a>:

<img src="/images/pycharm-docker-debug/run-debug-config.gif" style="display:block;margin:0 auto">

<br>
Clique no "+" e crie um novo perfil "Python"

Escolha o interpretador que foi criado e especifique o _entrypoint_ da sua aplicação. Nesse caso é o próprio `main.py`,
com o tipo "script". Clique `Apply`

<img src="/images/pycharm-docker-debug/run-debug-config.png" style="display:block;margin:0 auto">
* <small>Poderia ser um perfil "Flask", mas para ser mais genérico vamos de "Python"</small>

🤗Pronto🤗

Agora, coloque seus breakpoints e clique na baratinha para debugar.

### Caso menos simples: Uma app boba + um banco de dados

Bom, digamos que existam outros serviços adjacentes à sua aplicação, como, por exemplo, um banco de dados.

docker-compose.yml:
```yaml
version: "3.5"
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "5000:5000"
  mongo:
    extends:
      file: mongo.yml
      service: mongo
    volumes:
      - mongo_data:/data/db

volumes:
  mongo_data:
```

mongo.yml:
```yaml
version: "3.5"

services:
  mongo:
    image: mongo:5
    command: mongod --port 27017
    restart: always
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
```

Nesse caso, se seguirmos os passos da seção anterior, nosso banco de dados nunca será executado, pois o interpretador configurado só
diz respeito à aplicação em si (_app_), e a configuração de _run/debug_ também, e apenas executa nosso programa a partir de um
_entrypoint_ pré-estabelecido (_main.py_).

Criaremos então outra configuração de _run/debug_. Assim como antes, abra a aba de _run/debug_, e dessa vez,
clique no "+" e escolha uma config do tipo "Docker Compose".

<img src="/images/pycharm-docker-debug/run-debug-docker-compose.png" style="display:block;margin:0 auto">

Especifique o caminho do arquivo "docker-compose.yml".
Note, no campo "Services" está escrito: "Leave empty to run all services" (Deixe em branco para rodar todos os serviços).

<img src="/images/pycharm-docker-debug/run-debug-docker-compose-2.png" style="display:block;margin:0 auto">

Ou seja, podemos deixar o campo em branco para que essa config de _run/debug_ rode os dois serviços (_app_ e _mongo_), ou,
especificar apenas o serviço _mongo_, uma vez que não podemos "debugar" a _app_ utilizando essa config do tipo "Docker Compose".
Para debugar nossa _app_ de fato, utilizando breakpoints e tudo mais, é necessário utilizar a config criada anteriormente
que especifica o interpretador Dockerizado (criado lá no início) e o entrypoint (main.py nesse caso).

Porém, não é problema deixarmos essa nova config de _run/debug_ iniciar os dois serviços e depois executarmos
a config criada anteriormente para então debugar nossa aplicação. Por isso, pode deixar em branco, ou especificar apenas
_mongo_ na aba "Services".

Ok, agora é só rodar (clicando na setinha do lado da barata) essa nova config para subir o mongo + app (ou só mongo), e depois rodar a config que criamos 
anteriormente para então debugar nossa aplicação. 
<small>Lembre de usar a baratinha</small>

* <small>A app acaba nem chamando o mongo... mas poderia.</small>

### Debugando testes com _Pytest_

Se você precisa debugar seus testes, crie uma nova config de _run/debug_ do tipo
"Pytest". Escolha o interpretador criado anteriormente (aquele do container), e, se você estiver utilizando `pytest-cov` 
no seu projeto, no campo "Additional Arguments" coloque <a href="https://github.com/pytest-dev/pytest-cov/issues/131" target="_blank">`--no-cov`</a>.
Pronto, agora pode colocar seus breakpoints nos testes e debugar feliz da vida.

<img src="/images/pycharm-docker-debug/pytest-debug.png" style="display:block;margin:0 auto">

### Como estou dirigindo?

Se você tem alguma sugestão, dica, ou encontrou algum erro, pode deixar nos comentários abaixo. (:

bye bye