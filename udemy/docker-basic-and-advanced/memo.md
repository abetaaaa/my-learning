# 初心者OK！Docker入門＋応用：ゼロからでも実務で使えるスキルが身に付ける
講座リンク：https://www.udemy.com/course/ok-docker/?couponCode=24T3FS41524  
セクション11用リポジトリ：https://github.com/abetaaaa/docker-simple-cicd-demo


## Section6: Dockerfileでカスタムイメージをつくる

### Part33: レイヤー構造とは？
Docker Imageはレイヤ構造を持っている。
Dockerfileの一行がレイヤ一層に対応している。

### Part34: イメージの容量を小さくする
レイヤ構造が少なくなるようなDockerfileの書き方をするとimageの容量が小さくなる。

```
$ docker image ls
REPOSITORY                     TAG       IMAGE ID       CREATED              SIZE
<none>                         <none>    df9b453fc8d7   20 seconds ago       201MB
<none>                         <none>    e09d1befba13   About a minute ago   202MB
```

### Part35: 意識すべきレイヤー構造の特徴
レイヤーを追加すると、イメージの変更差分がレイヤーとして保存される。
すでにビルド済みのところはキャッシュされている。  
レイヤーを分けると変更が加わった際にビルドの時間が短縮できる。  
容量をとるかビルド時間をとるかは開発のフェーズによって使い分ける必要がある。

### Part36: ENVで環境変数を設定する
ENVで環境変数が設定できる。
```
# 書式
ENV [環境変数名]=[値] # 推奨
ENV [環境変数名] [値]
```

```Dockerfile
FROM ubuntu:20.04

ENV hello="Hello World"
ENV hoge=HOGEHOGE

CMD ["bash"]
```
```
$ echo $hello
Hello World

$ echo hoge
HOGEHOGE
```

### Part37: ARGで任意の変数を扱う
ARGで変数設定ができる。
```
# 書式
ARG [キー]=[デフォルト値] # デフォルト値は空欄にしておいてビルド時に与えることもできる
```

```Dockerfile
FROM ubuntu:20.04

ARG message
RUN echo $message > message.txt

CMD ["cat", "message.txt"]
```

```
$ docker image build .
$ docker run -it [ContainerId]

```
```
$ docker image build --build-arg message="Hello Message" . 
$ docker run -it [ContainerId]
Hello Message
```


### Part38: ARGとENVの使いわけ
| ARG                               | ENV                                          |
| --------------------------------- | -------------------------------------------- |
| イメージ作成時のみ有効な一時的変数 | イメージ作成時とコンテナ実行時両方で有効な変数 |

- ARGはDockerfileの中に限って有効なので、イメージ内でしか使用しない変数の場合は意図しないエラーを引き起こさないようにするためにARGを使う。
- コンテナの中でもその変数を使う場合はENVを使用する。

```Dockerfile
FROM ubuntu:20.04

ARG message_arg="Message Arg"
ENV message_env="Message Env"

RUN echo $message_arg > message_arg.txt
RUN echo $message_env > message_env.txt

CMD ["bash"]
```

```
$ docker image build .
$ docker container run -it [ContainerId]
root# cat message_arg.txt
Message Arg
root# cat mesasge_env.txt
Message Env
root# echo $message_env
Message Env
root# echo $message_arg

```

### Part39: WORKDIRで作業ディレクトリを変更する
WORKDIRでRUNやCMDの作業ディレクトリを指定できる。

```Dockerfile
FROM ubuntu:20.04

RUN touch 1.txt
WORKDIR /app/my_dir
RUN touch 2.txt
WORKDIR ..
RUN touch 3.txt

CMD ["bash"]
```

```
$ docker run -it [ContainerId]
root:/app# ls
3.txt  my_dir
root:/app# ls my_dir/
2.txt
root:/app# cd ..
root# ls
1.txt ~~~
```

## Section7: マルチステージビルドを使いこなす

### Part40: マルチステージビルドとは？

1つのDockerfileでステージごとの環境を分離し、必要なイメージを作成することが可能

### Part41: マルチステージビルドでイメージサイズを削減する

C言語のコンパイルにはgccが必要だが、実行には不要。そのため、コンパイル時と実行時でステージを分けることで容量削減が可能。

```Dockerfile
FROM gcc:12.2.0

WORKDIR /app
COPY ./hello.c .
RUN gcc hello.c
CMD ["./a.out"]
```

```
$ docker image build -t single-image .
$ docker container run single-image
Hello World
```

`--from=0`は0番目のステージからコピーをするという指示。  
ASでステージに名前を付けてそれを指定することも可能。

```Dockerfile
# stage 0
FROM gcc:12.2.0 AS compiler
WORKDIR /app
COPY ./hello.c .
RUN gcc hello.c

# stage 1
FROM ubuntu:20.04
WORKDIR /app
COPY --from=compiler /app/a.out .
CMD ["./a.out"]
```

```
$ docker image build -t multi-image .
$ docker container run multi-image
Hello WOrld
```

サイズ比較
```
$ docker image ls
REPOSITORY                     TAG       IMAGE ID       CREATED              SIZE
multi-image                    latest    dd9a78acf0ff   About a minute ago   72.8MB
single-image                   latest    3d29531408a8   6 minutes ago        1.27GB
```


### Part42: マルチステージビルドで複数環境向けのイメージの管理を楽にする

共通部分+{開発環境用/本番環境用}という構成の場合、開発環境と本番環境で分けることも可能だが、管理が煩雑になる。  
マルチステージビルドなら1つのDockerfileで管理可能。

```Dockerfile
FROM ubuntu:20.04 AS base
RUN apt update
CMD ["sh", "-c", "echo My name is $my_name"]

FROM base AS development
ENV my_name=TEST

FROM base AS production
ENV my_name=Bob
```

開発環境のビルド
```
$ docker image build --target development
$ docker container run --rm [ContainerId]
My name is TEST
```

本番環境のビルド
```
$ docker image build --target production
$ docker container run --rm [ContainerId]
My name is Bob
```


## Section10: Docker Composeで複数コンテナを扱う

### Part66: APIサーバーのイメージを作成する

[spring initializr](https://start.spring.io/)で資材をダウンロード  
![](assets\image\section10_66_00.png)

```Dockerfile
FROM gradle:7

WORKDIR /api
COPY . .

CMD [ "./gradlew", "bootRun" ]
```

```
$ docker image build -t api-img .
$ docker container run -p 8080:8080 --rm api-img
```

http://localhost:8080/api/hello?lang=ja にアクセスすると以下の画面が表示される。

![](assets\image\section10_66_01.png)


### Part67: Docker ComposeでAPIサーバーを起動する

Part66のコマンドと同等のdocker-compose.ymlを作成する。

```yml
version: "3.9"
services:
  api:
    build: ./api
    ports:
      - 8080:8080
```
起動コマンド
```
docker compose up
```

### Part68: Docker ComposeでDBを起動する

docker-compose.ymlにDBの設定を追記
```yml
version: "3.9"
services:
  api:
    build: ./api
    ports:
      - 8080:8080
  db:
    image: postgres:15
    ports:
      - 5432:5432
    environment:
      - POSTGRES_PASSWORD=mypassword
      - POSTGRES_USER=postgres
      - POSTGRES_DB=appdb
    volumes:
      - db-storage:/var/lib/postgresql/data
      - ./db/initdb:/docker-entrypoint-initdb.d

volumes:
  db-storage:
```

DBのみ起動
```
docker compose up db
```

DBサーバーの中に入り、データを確認
```
$ docker compose exec db bash
# psql -U postgres
postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------       
 appdb     | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +       
           |          |          |            |            |            |                 | postgres=CTc/postgres        
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +       
           |          |          |            |            |            |                 | postgres=CTc/postgres        
(4 rows)

postgres=# \c appdb
You are now connected to database "appdb" as user "postgres".
appdb=# \dt
           List of relations
 Schema |   Name    | Type  |  Owner
--------+-----------+-------+----------
 public | greetings | table | postgres
(1 row)

appdb=# select * from greetings;
 id | lang |    text
----+------+------------
  1 | ja   | こんにちは
  2 | en   | Hello
(2 rows)
```

## Section11: 実務を想定したDockerを活用したWebアプリケーション開発を体験する

### Part71: ソフトウェア開発の流れを理解する

以下を繰り返しながら開発を進める。(※)ではDockerを使用する。
- 実装開発
    - 開発環境構築(※)
    - 開発(※)
    - コミット
- CI
    - ビルド(※)
    - テスト(※)
- CD
    - デプロイ(※)


### Part72: 作成するアプリの全体像

アプリの全体像
![](assets\image\section11_architecture.png)

詳細
![](assets\image\section11_architecture_task.png)

CI/CD
![](assets\image\section11_architecture_cicd.png)

### Part77: アプリの動作を確認する

docker-simple-cicd-demoディレクトリで以下を実行
```
$ docker compose up -d
$ docker container ls
CONTAINER ID   IMAGE                         COMMAND                   CREATED          STATUS          PORTS                    NAMES
979d67c61212   docker-simple-cicd-demo-web   "docker-entrypoint.s…"   21 seconds ago   Up 16 seconds   0.0.0.0:3000->3000/tcp   web
b4a1b832b983   docker-simple-cicd-demo-api   "/__cacert_entrypoin…"   21 seconds ago   Up 17 seconds   0.0.0.0:8080->8080/tcp   api
```

appのコンテナに入りJavaのコンパイルを実行
```
$ docker compose exec api bash
# cd /workspace
# ./gradlew bootRun
```

webも同様に実行
```
$ docker compose exec web bash
# cd /workspace
# npm install
# npm run start

~~~
Compiled successfully!

You can now view my-app in the browser.

  Local:            http://localhost:3000
  On Your Network:  http://172.20.0.3:3000

Note that the development build is not optimized.
To create a production build, use npm run build.

webpack compiled successfully
No issues found.
~~~
```

http://localhost:3000 にアクセスすると以下の画面が表示される。

![](assets\image\section11_77_00.png)

http://localhost:8080/api/hello にアクセスすると以下の画面が表示される。

![](assets\image\section11_77_01.png)

画面のCall APIボタンを押下すると、HTML文が表示される。F12を押下しNetwork>Headersを確認すると、Request URLが 
http://localhost:3000/undefined/hello となっており、狙いの http://localhost:8080/api/hello になっていない。
![alt text](assets\image\section11_77_02.png)

docker-simple-cicd-demo/web/src/App.tsxを確認すると、11行目が以下のようになっている。
```tsx
const response = await fetch(`${process.env.REACT_APP_API_SERVER}/hello`, {mode: 'cors'})
```
ここで`REACT_APP_API_SERVER`が空のためエラーが起きている。  
よってdocker-compose.ymlで環境変数を設定する。

```yml
  web:
    container_name: web
    build: ./web
    ports:
      - 3000:3000
    environment:
      - REACT_APP_API_SERVER=http://localhost:8080/api
    tty: true
    volumes:
      - ./web:/workspace:cached
    depends_on:
      - api
```

再起動する。
```
$ docker compose down
$ docker compose up -d
```
```
$ docker compose exec api bash
# cd /workspace
# ./gradlew bootRun
```
```
$ docker compose exec web bash
# cd workspace
# npm run start
```

ブラウザをリロードしCall APIを再度押下するとHello World!!が表示される。
![](assets\image\section11_77_03.png)

### Part93: GitHub Actionsの設定（デプロイ編）

docker composeにおけるdocker-compose.ymlと同じような働きをするのが.aws/task-definition.yml  
.github/workflows/main.yml で task-definition.yml の"<image>"を書き換えている。