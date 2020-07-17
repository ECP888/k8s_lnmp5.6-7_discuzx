FROM  php:5.6-fpm-alpine
ADD repositories /etc/apk/repositories
ADD default.conf /
ADD index.html /
ADD run.sh /
ADD php.ini /usr/local/etc/php/
RUN apk update && apk add nginx && \
    apk add m4 autoconf make gcc g++ linux-headers && \
    docker-php-ext-install pdo_mysql opcache mysqli && \
    mkdir /run/nginx && \
    mv /default.conf /etc/nginx/conf.d && \
    mv /index.html /var/www/html && \
    touch /run/nginx/nginx.pid && \
    chmod 755 /run.sh && \
    apk del m4 autoconf make gcc g++ linux-headers

EXPOSE 80
EXPOSE 9000

ENTRYPOINT ["/run.sh"]
