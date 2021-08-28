---
title: Docker ubuntu container에서 nodejs install 하기
date: "2021-08-28T08:46:32.015Z"
category: nodejs
draft: false
---


```docker
FROM ubuntu:18.04

# install nodejs
RUN apt-get -qq update
RUN apt-get -qq upgrade --yes 
RUN apt-get -qq install curl --yes
RUN curl -sL https://deb.nodesource.com/setup_14.x | bash -
RUN apt-get -qq install nodejs --yes

WORKDIR /app

COPY ./package.json ./

RUN npm install

COPY . .
```

사실 그냥 **ubuntu**에서 다운받는걸 똑같이 한다.

```yml
version: '3'
services:
  server:
    build: .
    container_name: server_app
    volumes:
      - .:/usr/app/
      - /usr/app/node_modules
    command: npm run start:prod
```
보너스로 docker-compose.yml 파일이다

## 참고
- [How to install latest node inside a docker container](https://askubuntu.com/questions/720784/how-to-install-latest-node-inside-a-docker-container)