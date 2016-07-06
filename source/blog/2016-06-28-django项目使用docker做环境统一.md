# Django项目如何使用Docker搭建环境

## 背景

公司的项目前后端是分离的，前端使用React作为技术栈，后端使用Django做完web开发框架，
前端的同学调试代码的时候需要启动后端的项目，后端的项目往往需要一堆环境依赖，例如数据库，
Redis缓存，Python的库，每次搭建环境更新环境对于前端的同学来说都是一场噩梦，比如安装数据库，
编译Python的库等等都会遇到很多问题，为了提高效率，更好的统一开发环境，所以使用了Docker来做
这件事。

## 达到的效果

## 流程

1.  安装Docker环境
2.  编写开发环境需要的Dockerfile
3.  编写docker-compose.yml定义好启动环境需要的依赖
4.  启动环境
5.  搭建私有的Docker Registry
6.  将自定义的docker image push到私有的registry
7.  常用操作

### 安装Docker环境

这个不多说，大家参照官网安装就行

### 自定义开发环境所需要的Dockerfile

    # 使用国内的源加快速度
    FROM daocloud.io/library/python:2.7.11
    ENV PYTHONUNBUFFERED 1
    RUN sed -i 's/http:\/\/httpredir\.debian\.org\/debian\//http:\/\/mirrors\.163\.com\/debian\//g' /etc/apt/sources.list
    RUN apt-get update && apt-get install -y gcc g++ python-software-properties libpq-dev git libmysqlclient-dev build-essential
    RUN mkdir /code
    WORKDIR /code
    ADD . /code/
    # 使用豆瓣源加快速度
    RUN pip install -r requirements/dev.txt -i http://pypi.douban.com/simple/ --trusted-host pypi.douban.com

### 编写docker-compose.yml

    # mysql
    # username admin password root
    mysql:
      image: daocloud.io/library/mysql:5.5.44
      environment:
        - MYSQL_ROOT_PASSWORD=root
        - MYSQL_DATABASE=owl
        - MYSQL_ALLOW_EMPTY_PASSWORD=yes
      volumes:
        - ./conf:/etc/mysql/conf.d

    # redis
    # password root
    redis:
      image: redis:latest
      command: redis-server --requirepass root

    # mongo
    mongo:
      image: daocloud.io/library/mongo:3.2.7

    # web
    web:
      image: burnish/owl:latest
      command: python manage.py runserver 0.0.0.0:8888
      volumes:
        - .:/code
      ports:
        - "8888:8888"
      environment:
        - PYTHONPATH=/code
      links:
        - mysql:mysql
        - redis:redis
        - mongo:mongo

### 启动环境

    docker-compose up

### 搭建私有的Docker Registry

这个过程内容较长，不在这篇文章描述，后续会专门写一篇关于如何搭建环境的教程,
搭建好的私有docker registry域名是burnsh

### 将build完成的docker image push到私有registry

    # 登陆
    docker login burnish

    # 打tag
    docker tag image burnish/owl:latest

    # push
    docker push burnish/owl:latest

### 常用操作和场景

-   1. 初始化项目和app

        # 进入docker
        docker-compose web run --rm /bin/bash
        django-admin.py startapp app_xxx

        # 命令解释

-   2. 数据库生成migration和migrate

        docker-compose web run --rm python manage.py makemigrations
        docker-compose web run --rm python manage.py migrate

        # 命令解释

-   3. 安装新的python 包

    安装

        docker-compose web run pip install xxxx
        docker commit container_id -a "runforever" -c "add new xxxx python lib" burnish/owl:latest
        docker push burnish/owl:latest

    更新

        docker pull burnish/owl:latest
        docker-compose up

-   4. pdb调试

        docker-compose run --service-ports --rm web

    <!-- more -->
