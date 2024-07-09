# Dockerfile pour de nombreux langages de programmation

## Contents

- [React](#dockerfile-pour-react)
- [NodeJS](#dockerfile-pour-nodejs)
- [Python](#dockerfile-pour-python)
- [Golang](#dockerfile-pour-golang)
- [Java Spring Boot](#dockerfile-pour-java-spring-boot)
- [Java Quarkus](#dockerfile-pour-java-quarkus)
- [ASP.NET Core](#dockerfile-pour-aspnet-core)
- [Ruby](#dockerfile-pour-ruby-on-rails)
- [Rust](#dockerfile-pour-rust)
- [PHP Laravel](#dockerfile-pour-php-laravel)
- [Dart](#dockerfile-pour-dart)
- [R Studio](#dockerfile-pour-r-studio)
- [Contact](#contact)

## Dockerfile pour React

Normal:

```Dockerfile
FROM node:20-alpine as build

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:stable-alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Avec pnpm:

```Dockerfile
FROM node:20-alpine as build

RUN npm install -g pnpm

WORKDIR /app
COPY package*.json ./
RUN CI=true pnpm install
COPY . .
RUN pnpm build

FROM nginx:stable-alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Nginx Config:

```
# Détecte automatiquement un bon nombre de processus à exécuter.
worker_processes auto;

# Fournit le contexte du fichier de configuration dans lequel sont spécifiées les directives qui affectent le traitement des connexions.
events {
    # Définit le nombre maximum de connexions simultanées pouvant être ouvertes par un processus de travail.
    worker_connections 8000;
    # Demande d'accepter plusieurs connexions à la fois.
    multi_accept on;
}


http {
    # Quelles heures inclure.
    include       /etc/nginx/mime.types;
    # Quel est celui par défaut.
    default_type  application/octet-stream;

    # Définit le chemin, le format et la configuration pour une écriture de log en mémoire tampon.
    log_format compression '$remote_addr - $remote_user [$time_local] '
        '"$request" $status $upstream_addr '
        '"$http_referer" "$http_user_agent"';

    server {
        # Écouter sur le port 80
        listen 80;
        # Enregistrer les logs ici
        access_log /var/log/nginx/access.log compression;

        # Root ici.
        root /usr/share/nginx/html;
        # Quel fichier serveur comme index
        index index.html index.htm;

        location / {
            # Essayez d'abord de servir la demande en tant que fichier, puis
            # Comme répertoire, puis revenez à la redirection vers index.html
            try_files $uri $uri/ /index.html;
        }

        # Médias: images, icons, video, audio, HTC
        location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc)$ {
          expires 1M;
          access_log off;
          add_header Cache-Control "public";
        }

        # Javascript et CSS fichiers
        location ~* \.(?:css|js)$ {
            try_files $uri =404;
            expires 1y;
            access_log off;
            add_header Cache-Control "public";
        }

        # Toute route contenant une extension de fichier (par exemple /devicesfile.js)
        location ~ ^.+\..+$ {
            try_files $uri =404;
        }
    }
}
```

## Dockerfile pour NodeJS

ExpressJS:

```Dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci
COPY . .

CMD ["node", "index.js"]
```

Installer node-gyp sur la version Node Alpine:

```Dockerfile
FROM node:20-alpine

RUN apk add --no-cache \
    make \
    gcc \
    g++ \
    python3 \
    pkgconfig \
    pixman-dev \
    cairo-dev \
    pango-dev \
    libjpeg-turbo-dev \
    giflib-dev

WORKDIR /app

COPY package*.json ./
RUN npm install
COPY . .

CMD ["node", "index.js"]
```

Si vous essayez d'utiliser le package Sharp avec NodeJS mais rencontrez des erreurs:

- `Impossible de charger le module "sharp" à l'aide du runtime linuxmusl-x64`
- `sharp: Erreur d'installation: Version invalide: 1.2.4_git20230717`

Réparer en modifiant `FROM node:20-alpine` à `FROM node:20-buster-slim`:

```Dockerfile
FROM node:20-buster-slim

WORKDIR /app

COPY package*.json ./
RUN npm ci
COPY . .

CMD ["node", "index.js"]
```

NestJS Framework:

```Dockerfile
FROM node:20-alpine as build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY --chown=node:node . .
RUN npm run build

# L'exécution de `npm ci` supprime le répertoire node_modules existant et le passage de --only=production garantit que seules les dépendances de production sont installées. Cela garantit que le répertoire node_modules est aussi optimisé que possible
RUN npm ci --only=production && npm cache clean --force

USER node

FROM node:20-alpine

WORKDIR /app

COPY --from=build --chown=node:node /app/package*.json ./
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist

CMD ["node", "dist/main.js"]
```

## Dockerfile pour Python

Normal:

```Dockerfile
FROM python:3.9-slim-bullseye

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt

COPY . .

CMD ["python3", "app.py"]
```

Avec Flask ou Django, vous devez exécuter sur l'hôte `0.0.0.0`.

Flask:

```Dockerfile
FROM python:3.9-slim-bullseye

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt

COPY . .

CMD [ "python3", "-m" , "flask", "run", "--host=0.0.0.0"]
```

Django:

```Dockerfile
FROM python:3.9-slim-bullseye

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt

COPY . .

CMD ["python3", "manage.py", "runserver", "0.0.0.0:8000"]
```

Avec Poetry, gestion des packages python comme npm dans Node:

- `pyproject.toml` semblable à `package.json` dans Node
- `poetry.lock` semblable à `package-lock.json` dans Node

```Dockerfile
FROM python:3.9-slim-bullseye as builder
RUN pip install poetry

WORKDIR /app
COPY poetry.lock pyproject.toml ./
RUN poetry install

FROM python:3.9-slim-bullseye as base
WORKDIR /app

COPY --from=builder /app /app

ENV PATH="/app/.venv/bin:$PATH"
CMD ["python", "-m", "app.py"]
```

## Dockerfile pour Golang

Normal:

```Dockerfile
FROM golang:1.20-alpine AS build

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download && go mod verify
COPY . .

# Construit l'application comme une application liée statiquement, pour lui permettre de s'exécuter sur Alpine.
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o run .

# Déplacer le binaire vers « l'image finale » pour le rendre plus petit.
FROM alpine
WORKDIR /app
COPY --from=build /build/run .
CMD ["/app/run"]
```

With private repo:

```Dockerfile
FROM golang:1.20-alpine AS build

# Installer git et openssh.
RUN apt update && apt upgrade -y && \
    apt install -y git make openssh-client

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download && go mod verify
COPY . .

# Construit l'application comme une application liée statiquement, pour lui permettre de s'exécuter sur Alpin.
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o run .

# Déplacer le binaire vers « l'image finale » pour le rendre plus petit.
FROM alpine
WORKDIR /app
COPY --from=build /build/run .
CMD ["/app/run"]
```

## Dockerfile pour Java Spring Boot

```Dockerfile
FROM eclipse-temurin:17-jdk-focal as build

WORKDIR /build

COPY .mvn/ ./.mvn
COPY mvnw pom.xml  ./
RUN sed -i 's/\r$//' mvnw
RUN ./mvnw dependency:go-offline

COPY . .
RUN sed -i 's/\r$//' mvnw
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
COPY --from=build /build/target/*.jar run.jar
ENTRYPOINT ["java", "-jar", "/app/run.jar"]
```

## Dockerfile pour Java Quarkus

```Dockerfile
FROM maven:3.8.4-openjdk-17 AS build

WORKDIR /build
COPY ./pom.xml ./pom.xml
COPY ./settings.xml /root/.m2/settings.xml
RUN mvn dependency:go-offline -B

COPY src src
ARG QUARKUS_PROFILE
RUN mvn package -Dquarkus.profile=${QUARKUS_PROFILE}

FROM registry.access.redhat.com/ubi8/ubi-minimal:8.4

ARG JAVA_PACKAGE=java-17-openjdk-headless
ARG RUN_JAVA_VERSION=1.3.8
ENV LANG='en_US.UTF-8' LANGUAGE='en_US:en'

# Installer Java et le script run-java
RUN microdnf install curl ca-certificates wget ${JAVA_PACKAGE} \
  && microdnf update \
  && microdnf clean all \
  && mkdir /deployments \
  && chown 1001 /deployments \
  && chmod "g+rwX" /deployments \
  && chown 1001:root /deployments \
  && curl https://repo1.maven.org/maven2/io/fabric8/run-java-sh/${RUN_JAVA_VERSION}/run-java-sh-${RUN_JAVA_VERSION}-sh.sh -o /deployments/run-java.sh \
  && chown 1001 /deployments/run-java.sh \
  && chmod 540 /deployments/run-java.sh \
  && echo "securerandom.source=file:/dev/urandom" >> /etc/alternatives/jre/conf/security/java.security

# Nous créons quatre couches distinctes, donc en cas de modifications d'application, les couches de bibliothèque peuvent être réutilisées.
COPY --from=build /build/target/quarkus-app/lib/ /deployments/lib/
COPY --from=build /build/target/quarkus-app/*.jar /deployments/
COPY --from=build /build/target/quarkus-app/app/ /deployments/app/
COPY --from=build /build/target/quarkus-app/quarkus/ /deployments/quarkus/

USER 1001

ENTRYPOINT [ "/deployments/run-java.sh" ]
```

## Dockerfile pour ASP.NET Core

Normal:

```Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

WORKDIR /build

# Copier csproj et restaurer en tant que couches distinctes.
COPY *.csproj .
RUN dotnet restore

# Copier et publier des applications et des bibliothèques.
COPY . .
RUN dotnet publish --no-restore -o app

FROM mcr.microsoft.com/dotnet/aspnet:8.0

WORKDIR /app

COPY --from=build /build/app .

ENTRYPOINT ["./aspnetapp"]
```

Alpine version:

```Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build

WORKDIR /build

# Copier csproj et restaurer en tant que couches distinctes.
COPY *.csproj .
RUN dotnet restore

# Copier et publier des applications et des bibliothèques.
COPY . .
RUN dotnet publish --no-restore -o app

FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine

WORKDIR /app

COPY --from=build /build/app .

ENTRYPOINT ["./aspnetapp"]
```

## Dockerfile pour Ruby on Rails

Without assets:

```Dockerfile
FROM ruby:3.2-slim-bullseye

# Installer les dépendances système requises à la fois au moment de l'exécution et au moment de la construction.
RUN apt-get update && apt-get install -y \
    build-essential \
    # Exemple de dépendances système qui nécessitent "gem install pg".
    libpq-dev

COPY Gemfile Gemfile.lock ./

# Installation (hors dépendances de développement/test).
RUN gem install bundler && \
  bundle config set without "development test" && \
  bundle install

COPY . .

CMD ["rails", "server", "-b", "0.0.0.0"]
```

Avec des actifs:

```Dockerfile
FROM ruby:3.2-slim-bullseye

# Installer les dépendances système requises à la fois au moment de l'exécution et au moment de la construction
RUN apt-get update && apt-get install -y \
    build-essential \
    # Exemple de dépendances système qui nécessitent "gem install pg".
    libpq-dev \
    nodejs \
    yarn

COPY Gemfile Gemfile.lock ./

# Installation (hors dépendances de développement/test).
RUN gem install bundler && \
  bundle config set without "development test" && \
  bundle install

COPY package.json yarn.lock ./
RUN yarn install

COPY . .

# Installer des actifs.
RUN RAILS_ENV=production SECRET_KEY_BASE=assets bundle exec rails assets:precompile

CMD ["rails", "server", "-b", "0.0.0.0"]
```

**Note** - Sur MacOS M Chip, vous devrez peut-être ajouter un indicateur `--platform=linux/amd64` lors de la construction:

```
docker build . -t rubyonrails-app --platform=linux/amd64
```

## Dockerfile pour Rust

Normal:

```Dockerfile
FROM rust:1.70.0-slim-bullseye AS build

# Voir le nom de l'application dans Cargo.toml.
ARG APP_NAME=devopsvn

WORKDIR /build

COPY Cargo.lock Cargo.toml ./
RUN mkdir src \
    && echo "// dummy file" > src/lib.rs \
    && cargo build --release

COPY src src
RUN cargo build --locked --release
RUN cp ./target/release/$APP_NAME /bin/server

FROM debian:bullseye-slim AS final
COPY --from=build /bin/server /bin/
ENV ROCKET_ADDRESS=0.0.0.0
CMD ["/bin/server"]
```

Avec utilisateur non privilégié:

```Dockerfile
FROM rust:1.70.0-slim-bullseye AS build

# Voir le nom de l'application dans Cargo.toml.
ARG APP_NAME=devopsvn

WORKDIR /build

COPY Cargo.lock Cargo.toml ./
RUN mkdir src \
    && echo "// dummy file" > src/lib.rs \
    && cargo build --release

COPY src src
RUN cargo build --locked --release
RUN cp ./target/release/$APP_NAME /bin/server

FROM debian:bullseye-slim AS final

RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "10001" \
    appuser
USER appuser

COPY --from=build /bin/server /bin/
ENV ROCKET_ADDRESS=0.0.0.0
CMD ["/bin/server"]
```

## Dockerfile pour PHP Laravel

Normal:

```Dockerfile
FROM php:8.2-fpm

ARG user
ARG uid

# Installer les dépendances du système.
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    supervisor \
    nginx \
    build-essential \
    openssl

RUN docker-php-ext-install gd pdo pdo_mysql sockets

# Obtenez le dernier compositeur.
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Créer un utilisateur système pour exécuter les commandes Composer et Artisan.
RUN useradd -G www-data,root -u $uid -d /home/$user $user
RUN mkdir -p /home/$user/.composer && \
    chown -R $user:$user /home/$user

WORKDIR /var/www

# Si vous avez besoin de corriger SSL.
COPY ./openssl.cnf /etc/ssl/openssl.cnf

COPY composer.json composer.lock ./
RUN composer install --no-dev --optimize-autoloader --no-scripts

COPY . .

RUN chown -R $uid:$uid /var/www

# Copie de configuration du superviseur.
COPY ./supervisord.conf /etc/supervisord.conf

# Exécuter le superviseur.
CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisord.conf"]
```

supervisord.conf:

```
[supervisord]
user=root
nodaemon=true
logfile=/dev/stdout
logfile_maxbytes=0
pidfile=/var/run/supervisord.pid
loglevel = INFO

[program:php-fpm]
command = /usr/local/sbin/php-fpm
autostart=true
autorestart=true
priority=5
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0

[program:nginx]
command=/usr/sbin/nginx -g "daemon off;"
autostart=true
autorestart=true
priority=10
stdout_events_enabled=true
stderr_events_enabled=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
```

openssl.cnf:

```
openssl_conf = openssl_init

[openssl_init]
ssl_conf = ssl_sect

[ssl_sect]
system_default = system_default_sect

[system_default_sect]
Options = UnsafeLegacyRenegotiation
```

Si vous souhaitez ajouter une extension supplémentaire, pour exemple MongoDB, créez un fichier php.ini:

```
extension=mongodb.so
```

Mise à jour du Dockerfile:

```
...
WORKDIR /var/www

# If you need to fix ssl
COPY ./openssl.cnf /etc/ssl/openssl.cnf
# If you need add extension
COPY ./php.ini /usr/local/etc/php/php.ini
...
```

## Dockerfile pour Dart

```Dockerfile
FROM dart AS build

WORKDIR /build

COPY pubspec.* /build
RUN dart pub get --no-precompile

COPY . .
RUN dart compile exe app.dart -o run

FROM debian:bullseye-slim

WORKDIR /build

COPY --from=build /build/run /app/run
CMD ["/app/run"]
```

## Dockerfile pour R Studio

Pilote SQL Server:

```Dockerfile
FROM rocker/rstudio

RUN apt-get update && apt-get install -y \
    curl \
    apt-transport-https \
    tdsodbc \
    libsqliteodbc \
    gnupg \
    unixodbc \
    unixodbc-dev \
    ## clean up
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/ \
    && rm -rf /tmp/downloaded_packages/ /tmp/*.rds

RUN curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - \
 && curl https://packages.microsoft.com/config/debian/9/prod.list > /etc/apt/sources.list.d/mssql-release.list \
 && apt-get update \
 && ACCEPT_EULA=Y apt-get install --yes --no-install-recommends msodbcsql17 msodbcsql18 mssql-tools18 \
 && install2.r odbc \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/* \
 && rm -rf /tmp/*

RUN Rscript -e 'install.packages(c("DBI","odbc"))'
```

Pilote MYSQL:

```Dockerfile
FROM rocker/rstudio

RUN apt-get update && apt-get install -y \
    curl \
    apt-transport-https \
    tdsodbc \
    libsqliteodbc \
    gnupg \
    unixodbc \
    unixodbc-dev \
    ## clean up
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/ \
    && rm -rf /tmp/downloaded_packages/ /tmp/*.rds

RUN wget https://downloads.mysql.com/archives/get/p/10/file/mysql-connector-odbc-8.0.19-linux-debian9-x86-64bit.tar.gz \
 && tar xvf mysql-connector-odbc-8.0.19-linux-debian9-x86-64bit.tar.gz \
 && cp mysql-connector-odbc-8.0.19-linux-debian9-x86-64bit/bin/* /usr/local/bin \
 && cp mysql-connector-odbc-8.0.19-linux-debian9-x86-64bit/lib/* /usr/local/lib \
 && sudo apt-get update \
 && apt-get install --yes libodbc1 odbcinst1debian2 \
 && chmod 777 /usr/local/lib/libmy*

RUN myodbc-installer -a -d -n "MySQL ODBC 8.0 Driver" -t "Driver=/usr/local/lib/libmyodbc8w.so" \
 && myodbc-installer -a -d -n "MySQL ODBC 8.0" -t "Driver=/usr/local/lib/libmyodbc8a.so"

RUN Rscript -e 'install.packages(c("DBI","odbc"))'
```

## Contact

Besoin d'une mise à jour ? Contactez-moi sur [LinkedIn](https://www.linkedin.com/in/minopy/).
