FROM 192.168.168.219/lnmp/nginx-php-fpm:5.6
COPY localtime /etc/localtime
RUN echo 'Asia/Shanghai' >/etc/timezone
RUN rm -f /etc/nginx/nginx.conf
COPY nginx.conf /etc/nginx/nginx.conf
RUN rm -f /etc/nginx/www
COPY www /etc/nginx/www
RUN rm -f /etc/nginx/logs
COPY logs /etc/nginx/logs
RUN rm -f /etc/nginx/php.conf
COPY php.conf /etc/nginx/php.conf
RUN rm -f /usr/local/etc/php-fpm.d/www.conf
COPY php-fpm-www.conf /usr/local/etc/php-fpm.d/www.conf
ENTRYPOINT /usr/sbin/nginx -c /etc/nginx/nginx.conf && /usr/local/sbin/php-fpm -c /usr/local/etc/php-fpm.conf
