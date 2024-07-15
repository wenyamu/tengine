FROM neilpang/acme.sh:latest

ENV TENGINE_VERSION 3.1.0

RUN rm -rf /var/cache/apk/* && \
    rm -rf /tmp/*

ENV CONFIG "\
        --prefix=/etc/nginx \
        --sbin-path=/usr/sbin/nginx \
        --modules-path=/usr/lib/nginx/modules \
        --conf-path=/etc/nginx/nginx.conf \
        --error-log-path=/var/log/nginx/error.log \
        --http-log-path=/var/log/nginx/access.log \
        --pid-path=/var/run/nginx.pid \
        --lock-path=/var/run/nginx.lock \
        --http-client-body-temp-path=/var/cache/nginx/client_temp \
        --http-proxy-temp-path=/var/cache/nginx/proxy_temp \
        --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
        --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
        --http-scgi-temp-path=/var/cache/nginx/scgi_temp \
        --user=www-data \
        --group=www-data \
        --with-http_ssl_module \
        --with-http_realip_module \
        --with-http_addition_module \
        --with-http_sub_module \
        --with-http_dav_module \
        --with-http_flv_module \
        --with-http_mp4_module \
        --with-http_gunzip_module \
        --with-http_gzip_static_module \
        --with-http_random_index_module \
        --with-http_secure_link_module \
        --with-http_stub_status_module \
        --with-http_auth_request_module \
        --with-threads \
        --with-stream \
        --with-stream_ssl_module \
        --with-stream_ssl_preread_module \
        --with-stream_realip_module \
        --with-http_slice_module \
        --with-mail \
        --with-mail_ssl_module \
        --with-compat \
        --with-file-aio \
        --with-http_v2_module \
        --with-ipv6 \
        --with-http_xslt_module=dynamic \
        --with-http_image_filter_module=dynamic \
        --with-http_geoip_module=dynamic \
        --with-stream_geoip_module=dynamic \
        --add-module=modules/ngx_http_upstream_check_module \
        --add-module=modules/ngx_http_concat_module \
        --add-module=modules/dynamic-upstream-module-src-0.1.6 \
        --add-module=modules/ngx_http_upstream_session_sticky_module \
        "
RUN     addgroup -S nginx \
        && adduser -u 101 -D -S -h /var/cache/nginx -s /sbin/nologin -G nginx nginx \
        && adduser -u 82 -D -S -G www-data www-data \
        && apk add --no-cache --virtual .build-deps \
                gcc \
                libc-dev \
                make \
                openssl-dev \
                pcre-dev \
                zlib-dev \
                linux-headers \
                curl \
                libxslt-dev \
                gd-dev \
                geoip-dev \
        && curl -L "https://github.com/alibaba/tengine/archive/$TENGINE_VERSION.tar.gz" -o tengine.tar.gz \
        && mkdir -p /usr/src \
        && tar -zxC /usr/src -f tengine.tar.gz \
        && rm tengine.tar.gz \
        && cd /usr/src/tengine-$TENGINE_VERSION \
        && curl -L "https://github.com/nginx-with-docker/dynamic-upstream-module-src/archive/refs/tags/v0.1.6.tar.gz" -o more.tar.gz \
        && tar -zxC /usr/src/tengine-$TENGINE_VERSION/modules -f more.tar.gz \
        && rm  more.tar.gz \
        && ./configure $CONFIG \
        && make -j$(getconf _NPROCESSORS_ONLN) \
        && make install \
        && rm -rf /etc/nginx/html/ \
        && mkdir /etc/nginx/conf.d/ \
        && mkdir -p /usr/share/nginx/html/ \
        && install -m644 html/index.html /usr/share/nginx/html/ \
        && install -m644 html/50x.html /usr/share/nginx/html/ \
        && ln -s /usr/lib/nginx/modules /etc/nginx/modules \
        && strip /usr/sbin/nginx* \
        && strip /usr/lib/nginx/modules/*.so \
        && rm -rf /usr/src/tengine-$TENGINE_VERSION \
        \
        # Bring in gettext so we can get `envsubst`, then throw
        # the rest away. To do this, we need to install `gettext`
        # then move `envsubst` out of the way so `gettext` can
        # be deleted completely, then move `envsubst` back.
        && apk add --no-cache --virtual .gettext gettext \
        && mv /usr/bin/envsubst /tmp/ \
        \
        && runDeps="$( \
                scanelf --needed --nobanner --format '%n#p' /usr/sbin/nginx /usr/lib/nginx/modules/*.so /tmp/envsubst \
                        | tr ',' '\n' \
                        | sort -u \
                        | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
        )" \
        && apk add --no-cache --virtual .nginx-rundeps $runDeps \
        && apk del .build-deps \
        && apk del .gettext \
        && mv /tmp/envsubst /usr/local/bin/ \
        \
        # Bring in tzdata so users could set the timezones through the environment
        # variables
        && apk add --no-cache tzdata \
        # Bring in curl and ca-certificates to make registering on DNS SD easier
        && apk add --no-cache curl ca-certificates \
        # forward request and error logs to docker log collector
        && ln -sf /dev/stdout /var/log/nginx/access.log \
        && ln -sf /dev/stderr /var/log/nginx/error.log \
        && apk add --no-cache supervisor \
        && mkdir /var/log/supervisor \
        && touch /var/log/supervisor/supervisord.log

# nginx
ADD ./nginx/nginx.conf /etc/nginx/
ADD ./nginx/default.conf /etc/nginx/conf.d/

# supervisor
ADD ./supervisor/supervisord.conf /etc/supervisor/
ADD ./supervisor/services /etc/supervisor/conf.d/

EXPOSE 80 443

COPY entry.sh /entry.sh
RUN chmod +x /entry.sh
ENTRYPOINT ["/entry.sh"]
