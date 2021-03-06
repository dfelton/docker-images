FROM alpine

ENV HAPROXY_BRANCH 2.0
ENV HAPROXY_MINOR 2.0.8
ENV HAPROXY_MD5 745fe3f6625bb0f49380c2b6c6b350a7
ENV HAPROXY_SRC_URL http://www.haproxy.org/download

ENV HAPROXY_UID haproxy
ENV HAPROXY_GID haproxy

RUN apk add --no-cache --virtual build-deps gcc libc-dev \
        linux-headers lua5.3-dev make openssl openssl-dev pcre2-dev tar \
        zlib-dev curl shadow && \
    curl -sfSL "$HAPROXY_SRC_URL/$HAPROXY_BRANCH/src/haproxy-$HAPROXY_MINOR.tar.gz" -o haproxy.tar.gz && \
    echo "$HAPROXY_MD5  haproxy.tar.gz" | md5sum -c - && \
    groupadd "$HAPROXY_GID" && \
    useradd -g "$HAPROXY_GID" "$HAPROXY_UID" && \
    mkdir -p /tmp/haproxy && \
    tar -xzf haproxy.tar.gz -C /tmp/haproxy --strip-components=1 && \
    rm -f haproxy.tar.gz && \
    make -C /tmp/haproxy TARGET=linux-glibc CPU=generic USE_PCRE2=1 USE_REGPARM=1 USE_OPENSSL=1 \
                            USE_ZLIB=1 USE_TFO=1 USE_LINUX_TPROXY=1 USE_CRYPT_H=1 USE_GETADDRINFO=1 \
                            USE_LUA=1 LUA_LIB=/usr/lib/lua5.3 LUA_INC=/usr/include/lua5.3 \
                            all install-bin install-man && \
    ln -s /usr/local/sbin/haproxy /usr/sbin/haproxy && \
    mkdir -p /var/lib/haproxy && \
    mkdir -p /etc/haproxy/ssl && \
    chown "$HAPROXY_UID:$HAPROXY_GID" /var/lib/haproxy && \
    rm -rf /tmp/haproxy && \
    apk del build-deps
RUN apk add --no-cache openssl zlib lua5.3-libs pcre2 socat syslog-ng && \
    rm -f /var/cache/apk/*

ADD ./cli /usr/bin/cli
ADD ./entrypoint.sh /entrypoint
RUN chmod +x /entrypoint /usr/bin/cli

EXPOSE 80 443

ENTRYPOINT ["/entrypoint"]
