ARG PHP_VERSION
FROM php:${PHP_VERSION}

## Diretório da aplicação
ARG APP_DIR=/var/www/html

## Versão da Lib do Redis para PHP
ARG REDIS_LIB_VERSION=5.3.7

### apt-utils é um extensão de recursos do gerenciador de pacotes APT
RUN apt-get update -y && apt-get install -y --no-install-recommends \
    apt-utils \
    supervisor \
    curl \
    wget

RUN apt-get install -y locales locales-all
ENV LC_ALL en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US.UTF-8

# dependências recomendadas de desenvolvido para ambiente linux
RUN apt-get update && apt-get install -y \
    zlib1g-dev \
    libzip-dev \
    libpng-dev \
    libpq-dev \
    libxml2-dev \
    unzip \
    build-essential \
    libaio1

# Install Oracle Instantclient
RUN mkdir /usr/lib/oracle
RUN mkdir /usr/lib/oracle/21.1
RUN mkdir /usr/lib/oracle/21.1/client64

ADD ./docker/oracle/instantclient-basic-linux.x64-21.1.0.0.0.zip ./
ADD ./docker/oracle/instantclient-sdk-linux.x64-21.1.0.0.0.zip ./

RUN unzip ./instantclient-basic-linux.x64-21.1.0.0.0.zip -d /usr/lib/oracle/21.1/client64
RUN unzip ./instantclient-sdk-linux.x64-21.1.0.0.0.zip -d /usr/lib/oracle/21.1/client64

RUN cp -R /usr/lib/oracle/21.1/client64/instantclient_21_1 lib

RUN echo /usr/lib/oracle/21.1/client64/lib > /etc/ld.so.conf.d/oracle.conf

RUN ldconfig

ENV LD_LIBRARY_PATH /usr/lib/oracle/21.1/client64/instantclient_21_1

# Install Oracle extensions
RUN pecl channel-update pecl.php.net

RUN docker-php-ext-configure pdo_oci --with-pdo-oci=instantclient,/usr/lib/oracle/21.1/client64/instantclient_21_1
RUN docker-php-ext-install pdo_oci
RUN docker-php-ext-enable pdo_oci

# Install PHP our compiled oci8 extensions.
RUN docker-php-ext-configure oci8 --with-oci8=instantclient,/usr/lib/oracle/21.1/client64/instantclient_21_1 \
    && docker-php-ext-install -j$(nproc) oci8 \
    && docker-php-ext-enable oci8

# habilita instalação do Redis
RUN pecl install redis-${REDIS_LIB_VERSION} \
    && docker-php-ext-enable redis

RUN docker-php-ext-install zip iconv simplexml pcntl gd fileinfo

# Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

COPY ./docker/supervisord/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
#COPY ./docker/supervisord/conf /etc/supervisord.d/

COPY ./docker/php/extra-php.ini "$PHP_INI_DIR/99_extra.ini"
COPY ./docker/php/extra-php-fpm.conf /etc/php8/php-fpm.d/www.conf

WORKDIR $APP_DIR
RUN cd $APP_DIR
RUN chown www-data:www-data $APP_DIR

COPY --chown=www-data:www-data ./app $APP_DIR
#RUN composer install --no-interaction

RUN apt-get install nginx -y
RUN rm -rf /etc/nginx/sites-enabled/* && rm -rf /etc/nginx/sites-available/*
COPY ./docker/nginx/sites.conf /etc/nginx/sites-enabled/default.conf
COPY ./docker/nginx/error.html /var/www/html/error.html

#COPY ./docker/nginx/ssl/server.crt /var/www/server.crt
#COPY ./docker/nginx/ssl/server.key /var/www/server.key

RUN apt-get clean && rm -rf /var/lib/apt/lists/*
# RUN apt update -y && apt install nano git -y

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf"]
