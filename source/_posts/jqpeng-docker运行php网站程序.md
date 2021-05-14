---
title: docker运行php网站程序
tags: ["php","docker","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-01-03 09:07
---
文章作者:jqpeng
原文链接: [docker运行php网站程序](https://www.cnblogs.com/xiaoqi/p/docker-php.html)

有一个之前的php网站程序需要迁移到K8S，简单调研了下。

## 基础镜像

官方提供了诸如php:7.1-apache的基础镜像，但是确认必要的扩展，例如gd，当然官方提供了`docker-php-ext-install`命令，可以用来安装需要的扩展。但是每次构建都重新安装非常费时，最好的办法是构建一个包含必要扩展的基础镜像。


    FROM php:7.1-apache
    ENV PORT 80
    EXPOSE 80
    
    RUN buildDeps=" \
            default-libmysqlclient-dev \
            libbz2-dev \
            libsasl2-dev \
        " \
        runtimeDeps=" \
            curl \
            git \
            libfreetype6-dev \
            libicu-dev \
            libjpeg-dev \
            libmcrypt-dev \
            libpng-dev \
            libpq-dev \
            libxml2-dev \
        " \
        sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list  \
        && apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y $buildDeps $runtimeDeps \
        && docker-php-ext-install bcmath bz2 calendar iconv intl mbstring mcrypt mysqli opcache pdo_mysql pdo_pgsql pgsql soap zip \
        && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
        && docker-php-ext-install gd \
        && apt-get purge -y --auto-remove $buildDeps \
        && rm -r /var/lib/apt/lists/* 
    
    ENV PATH=$PATH:/root/composer/vendor/bin COMPOSER_ALLOW_SUPERUSER=1
    


然后构建基础镜像


    docker build -t common/php:7.1-apache .


PS： 更多的php镜像，查看 [https://github.com/chialab/docker-php](https://github.com/chialab/docker-php)

## 使用基础镜像

Dockerfile应用刚构建好的基础镜像：


    FROM common/php:7.1-apache
    ENV PORT 80
    EXPOSE 80
    COPY . /var/www/html
    RUN usermod -u 1000 www-data; \a2enmod rewrite; \chown -R www-data:www-data /var/www/html


构建即可：


    docker build -t common/zhifou:v0.0.12 .


