# DemoAPI
Qiita記事のソースコード。NginxとAsp.Netをdocker-composeで連携する

# はじめに
DockerでNginxを使用したり、DockerでAsp.netで使用するというものはよくWeb検索すれば見つかるのですが、DockerでNginxとAsp.netを連携させたものは少なかったので、試行錯誤しながら作成してみました。
とりあえず動作確認できるところまで行ったので備忘録もかねて記事にまとめてみました。

# 環境
Visual Studio 2022
Windows OS11
Docker Desktop

# 構成イメージ
だいたいこんな感じで構成していきます
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/3f172b78-36a4-dc4f-8834-0a391d532dde.png)

# ASP.NET Core Web API作成
## APIプロジェクトを作成します
DemoAPIという名前で作ります。
Visual Sdudioで新しいプロジェクトの作成をします。
テンプレートとしてASP.NET.Core Web APIを選択します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/0c3e9543-d078-b7dd-5446-e1afe6d63ed4.png)
設定の留意点としては以下の通り
HTTPSは今回は使用しない(Dockerネットワーク内のみで使用するので)
Dockerを有効にする
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/a0dfa5ad-f78d-2a78-8a29-5e35a8cd24c7.png)

## DemoControllerの作成
初期設定ではWeatherForecastControllerが作成されてSwagger画面が表示されて、動作確認がしにくいので、別にコントローラーを作成します
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/6a50d0e5-4e8e-6331-9033-6b8eac97e137.png)
APIを選択して、読み取り/書き込みアクションがあるAPIコントローラーを選択します。名前はDemoControllerとします。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/0271a52a-8a84-3700-9a72-1155c7ce1a57.png)
作成したコントローラーは、確認しやすいようにちょっと内容を書き換えておきます。(記事用に分かりやすくするためです)
```C#
[HttpGet]
public IEnumerable<string> Get()
{
    return new string[] { "デモです", "DemoAPIです" };
}
```

## DemoAPIのDockerfileの変更
初期作成時は以下のようになっています。
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["DemoAPI/DemoAPI.csproj", "DemoAPI/"]
RUN dotnet restore "DemoAPI/DemoAPI.csproj"
COPY . .
WORKDIR "/src/DemoAPI"
RUN dotnet build "DemoAPI.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "DemoAPI.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "DemoAPI.dll"]
```
この中でEXPOSE 80 に対する変更と、ENV行追加を行います。
変更後は以下の通りです
```dockerfile
#See https://aka.ms/containerfastmode to understand how Visual Studio uses this Dockerfile to build your images for faster debugging.

FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
EXPOSE 5001　　# ここが変更箇所 80 -> 5000

ENV ASPNETCORE_URLS=http://+:5001   # これが追加箇所

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["DemoAPI/DemoAPI.csproj", "DemoAPI/"]
RUN dotnet restore "DemoAPI/DemoAPI.csproj"
COPY . .
WORKDIR "/src/DemoAPI"
RUN dotnet build "DemoAPI.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "DemoAPI.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "DemoAPI.dll"]
```

# Nginxの作成
ソリューションから新しいソリューションフォルダを作成します。フォルダ名はNginxとしておきます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/8fde7b68-9480-c906-8aeb-7d0be0571f2d.png)
ここで注意なのですが、プロジェクトを作成した場合は、そのプロジェクト名でフォルダが作成されるのですが、ソリューションフォルダを作成しても、実際のディレクトリにはフォルダは作成されません。
そのため、フォルダ分けをきちんとしたい場合は、以下のようにフォルダを直接作成するのをお勧めします。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/50667b2b-0ea6-db53-ec64-e97227ee8743.png)


## demo.conf作成
作成したフォルダの中にdemo.confを作ります。
これは現時点ではテンプレートがないので、フォルダ内に作成して、それをVisual Studioから既存の項目の追加をします。中身は空っぽのデータです。(このあたりの作業はVisual Sudio Codeのほうが分かりやすいですね。Visual Studioのソリューションフォルダはちょっと特殊でとっつきにくく感じます)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/dc10bd59-a845-4fb7-0957-25a610c5cf50.png)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/6e5f3423-6ca7-37f6-e03b-e6438487def4.png)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/d54fa832-263b-9438-c7b7-9644190046c1.png)

## domo.conf
以下のように記述します。
後述しますが、default.confを書き換えるのでこの記述なります。
web検索すると、http{}とか、events{}など記載しているものもありますが、default.confの場合はエラーになるようです。
```conf
server {
    listen        80;
    server_name   default_server;

    root /var/www/html;
    index index.html;
    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/demo/ {
        proxy_pass         http://demoapi:5001;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection keep-alive;
        proxy_set_header   Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

## Nginxにindex.html作成
普通にNginxを起動するとデフォルト画面になるので、せっかくですから、自分で作成したページが表示するようにしましょう。
Nginxにindex.htmlを作成します。既存項目の追加という作業でいきます。
```html
<!DOCTYPE html>
<html lang="ja" xmlns="http://www.w3.org/1999/xhtml">
<head>
    <meta charset="utf-8" />
    <title>デモページ</title>
</head>
<body>
    <h2>表示できた～</h2>
</body>
</html>
```

## Nginx用のDockerfile作成
さきほどのdemo.confを作成したのと同じように、フォルダにDockerfileを作成して、既存の項目で追加します。
Dockerfileに以下のように記述します
```dockerfile
FROM nginx:latest
COPY ./Nginx/demo.conf /etc/nginx/conf.d/default.conf
COPY ./Nginx/index.html /var/www/html/index.html
```
nginx:latestにしていますが、nginx:alpineでも構いません。

# docker-composeの作成
ここまででDockerについては出来上がっているので、次はdocker-composeで連携させていきます。
## DemoAPIからdocker-compose作成
Visual Sudioによる自動作成をなるべく使っていきたいので、DemoAPIプロジェクトを右クリックして、追加⇒コンテナーおーすけとレーターのサポート...を選択します。LinuxをターゲットOSとします。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/17b10bf5-92d4-fc91-6f33-3c5674fb0aef.png)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/f0a05be2-eaca-0ef9-70b0-242035cee73f.png)
作成するとdocker-composeプロジェクトが作成されて、docker-compose.ymlとdocker-compose.override.ymlが作られていると思います。

## docker-compose.yml
初期値は以下のようになっています
```dockerfile
version: '3.4'

services:
  demoapi:
    image: ${DOCKER_REGISTRY-}demoapi
    build:
      context: .
      dockerfile: DemoAPI/Dockerfile
```
apiだけなので、ここにnginxの設定を追加します
```dockerfile
version: '3.4'

services:
  demoapi:
    image: ${DOCKER_REGISTRY-}demoapi
    build:
      context: .
      dockerfile: DemoAPI/Dockerfile
  # 以下が追加した部分
  nginx:
    image: ${DOCKER_REGISTRY-}nginx
    build:
      context: .
      dockerfile: Nginx/Dockerfile
```

## docker-compose.override.yml
docker-compsoe.ymlだけでもいいのですが、Visual Studioで作成した場合、このoverrideで内容の上書きができるようになっています。
使い方はいろいろですが、Visual Studioとしてはdocker-compose.ymlでdockerイメージとビルドが記述され、環境等がoverrideに記述されているようです。
最初に作った時はこのようになっているかと思います。
```dockerfile
version: '3.4'

services:
  demoapi:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
    ports:
      - "80"
```
これを以下のように変更します。分かりやすくするために本当に必要なものだけにしました。それぞれの設定・環境に合わせて、追加していくとよいと思います。
```dockerfile
version: '3.4'

services:
  demoapi:
    expose:
      - "5001"
    networks:
      - front-back
  nginx:
    ports:
     - 80:80 
    networks:
      - front-back

networks:
  front-back:
    driver: bridge
```
大切なのは、demoapiのportsをexposeにしているところです。portsは外部との通信で使用されますが、exposeにすることで、Docker内部だけでの通信となり、セキュリティが上がります。これをすることで、httpsではなくhttpでの内部通信のみでよいことになるので、最初にAPIを作成したときにhttpsを選択しなかったわけです。
実際にセキュリティをきちんとするためには、外部と通信するnginxをssl化してhttps通信をするように変更する必要がありますが、これについてはまた後日記事にしようと思います。
あと、ポートでexposeは"5001"、portsは80:80としていて、" " で囲んでいたりいなかったりという風にしていますが、これはどちらでもOKです。
" " で囲むのは64以下の場合に確か必要だったような記憶があります。違ってたらご指摘ください。

# 起動
ここまでで設定など問題なければビルドしてdocker-composeで起動します。
自分が起動したときに一部エラーが起きました。
内容はDockerfileにこの記事用に 「# ここが変更箇所」と書いたままにしていて、それがエラーの原因になりました。
こういったコメントや、スペースが入っているとエラーになるので、注意が必要です。
あと、おそらくこのまま実行するとこんなエラーが出るかもしれません。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/5c83b9ae-57b4-ad78-6b4a-0e0a022ef9b4.png)
これはDemoAPIが初期状態ではSwaggerを表示させようとするのですが、内部ポート(5001)だけにしているのでエラーとなります。
これが表示されても、Swagger使えないと言っているだけなのでOKで次に進んでも問題ありませんが、気になるので出ないようにします。

## ポートの警告を出さないようにする
docker-composeプロジェクトを右クリックしてプロパティを表示させます。
おそらく以下の項目があると思いますので、これを削除してください。
```xml
<DockerServiceUrl>{Scheme}://localhost:{ServicePort}/swagger</DockerServiceUrl>
```

## http: //localhost表示結果
正しく起動できていればこんな画面が表示されます
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/c52041d0-ad55-5ca3-8bbc-2a1d7453bd2e.png)

## http: //localhost/api/demo表示結果
api/demoで以下のように表示されれば成功です
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/956246/8d747a40-43ec-fa97-2af8-11cd3f26e9df.png)


