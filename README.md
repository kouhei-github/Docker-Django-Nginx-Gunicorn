# Docker-Django-Nginx-Gunicorn
Dockerで構築したコンテナ上で、  
Nginx経由でDjangoのプロジェクトが見れるようにした。

下記コマンドでダウンロードしてください
```shell
$ git clone https://github.com/kouhei-github/Docker-Django-Nginx-Gunicorn.git
```

---

## Usage(使い方)
dockerを立ち上げる
```shell
$ docker compose up -d
```

Dockerfileを修正した際など、Imageをbuildし直すコマンド
```shell
$ docker compose build
```

dockerコンテナの容量が大部分を占め、一旦削除したいとき
```shell
$ docker prune -a
```

---

## Docker関連の説明

###  I. 大前提
1. Dockerfileや関連ファイルはdockerディレクトリの中に格納
2. Djangoのコードはdjangoという名前でコンテナのコードとマウントさせている


**ディレクトリ構造**
```
├── docker
│   ├── api
│   │   ├── Dockerfile
│   │   └── requirements.txt
│   ├── front
│   │   └── Dockerfile
│   ├── mysql
│   │   ├── my.cnf
│   └── nginx
│       └── conf.d
│           └── default.conf
├── django
├── nuxt 
└── docker-compose.yml
```

### II. DjangoのDockerfile
Djangoを動かすための環境を用意する。

```dockerfile
# 使用するPythonイメージを記述
FROM python:3.9-slim-buster

# 環境変数の設定
ENV PYTHONIOENCODING utf-8
ENV TZ="Asia/Tokyo"
ENV LANG=C.UTF-8
ENV LANGUAGE=en_US:en

# ワーキングディレクトリを指定
WORKDIR /var/www/html/pdf-to-csv

# requirements.txtをコピー
COPY ./docker/api/requirements.txt /tmp/requirements.txt

# mysqlclientをインストールする際に必要なツールのインストール
RUN apt-get update && \
    apt-get -y install gcc libmariadb-dev

# requirements.txtをインストール
RUN pip install -r /tmp/requirements.txt
```
**Dockerfileの中身の補足**  
MySQLを使用する際はgccコンパイラとlibmariadb-devのインストールが必要になる。
```dockerfile
# mysqlclientをインストールする際に必要なツールのインストール
RUN apt-get update && \
    apt-get -y install gcc libmariadb-dev
```

### III. DockerでMySQLを使用するための設定
下記設定はなくても良いがDBのデータを永続化させたい場合は下記ディレクトリが必須。  
なぜならdocker-composeでDBの永続化先を./docker/mysqlに指定しているから   
./docker/mysql/my.cnf
```shell
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci

[client]
default-character-set=utf8mb4
```

### IV. DockerでNginxを使用するための設定
Dockerfileを作成しても良いがImageを指定して、
defaukt.confをコンテナにコピーするだけなのでdocker-compose.ymlに書くことにし。

./docker/nginx/conf.d/default.conf
```shell
upstream django_app {
    server django:8000;
}

server {

    listen 80;

    location / {
        proxy_pass http://django_app;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
        autoindex on;
        alias /django/static/;
    }

    location /media/ {
        autoindex on;
        alias /django/media/;
    }
}
```

**下記二箇所は注意**   
upstreamの名前を任意のものに変更し、serverをservice名:ポート番号にする
```shell
upstream django_app {
    # docker-compose.ymlのdjangoが動いているservice名:ポート番号
    server django:8000;
}
```
proxy_passを[http://upstream名](http://upstream名)にする
```shell
location / {
    # upstreamの名前を http://hogehoge; にする
    proxy_pass http://django_app;
        :
        : 省略
        :
}
```

djangoのstaticファイルをNginxのコンテナにマウントしているので、そのパスと合わせる
```shell
location /static/ {
    autoindex on;
    # マウントしているstaticディレクトリのパス
    alias /django/static/;
}

location /media/ {
    autoindex on;
    # マウントしているmediaディレクトリのパス
    alias /django/media/;
}
```


### V. .envファイルの作成
./.env
```dotenv
USERNAME=test-USERNAME
USERPASS=test-USERPASS
DATABASE=test-DATABASE
ROOTPASS=test-ROOTPASS
```
.envファイルの中身をdocker-compose.ymlで使用する際は、   
下記の書き方で取り出せる
```shell
${USERNAME}
${USERPASS}
${DATABASE}
${ROOTPASS}
```


### VI. docker-compose.ymlについて
```yaml
version: "3"
services:
  django:
    build:
      context: .
      dockerfile: ./docker/api/Dockerfile
    volumes:
      - ./django:/var/www/html/pdf-to-csv
      - ./django/static:/var/www/html/pdf-to-csv/static
      - ./django/media:/var/www/html/pdf-to-csv/media
    command: gunicorn config.wsgi:application --bind 0.0.0.0:8000
#    command: python3 -u manage.py runserver 0.0.0.0:8000

    ports:
      - "8000:8000"
    tty: true
    depends_on:
      - db

  nginx:
    image: nginx
    container_name: nginx
    volumes:
      - ./django/static:/django/static
      - ./django/media:/django/media
      - ./docker/nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf
    tty: true
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - django

  # MySQL
  db:
    image: mysql:5.7
    container_name: test-mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${ROOTPASS}
      MYSQL_DATABASE: ${DATABASE}
      MYSQL_USER: ${USERNAME}
      MYSQL_PASSWORD: ${USERPASS}
      TZ: 'Asia/Tokyo'
    command: mysqld --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    volumes:
    - ./docker/mysql/data:/var/lib/mysql
    - ./docker/mysql/my.cnf:/etc/mysql/conf.d/my.cnf
    - ./docker/mysql/sql:/docker-entrypoint-initdb.d
    ports:
    - "3306:3306"

  # phpMyAdmin
  phpmyadmin:
    container_name: test-phpmyadmin
    image: phpmyadmin/phpmyadmin
    environment:
    - PMA_ARBITRARY=1
    - PMA_HOSTS=test-mysql
    - PMA_USER=root
    - PMA_PASSWORD=${ROOTPASS}
    ports:
    - "8080:80"

```

**Nginxを使用することからgunicorn経由で呼び出す**   
下記コマンドではなく
```shell
$ python3 -u manage.py runserver 0.0.0.0:8000
```
gunicornでDjangoのアプリを起動する日長がある
```shell
gunicorn config.wsgi:application --bind 0.0.0.0:8000
```

### Django側の設定
djangoのアプリケーションのソースコードにDBの接続情報を載せたくないので.envファイルを作成する   
```dotenv
# MySQL
# DATABASE_URL=mysql://USERNAME:USERPASS@docker-compose.ymlのDBのコンテナ名:3306/DATABASE
DATABASE_URL=mysql://dreamer:jamnuvu)$hna@09ss@db:3306/tuchiya-pdf-to-csv
SECRET_KEY=-シークレットキー
DEBUG=True
ALLOWED_HOSTS =[127.0.0.1]
DOMAIN_ROOT=/var/www/html/pdf-to-csv/

SERVICE_ACCOUNT_JSON=secret.json

# EMAIL
EMAIL_HOST_USER=test@testt.com
EMAIL_HOST_PASSWORD=hogehoge
```

---