FROM resin/raspberry-pi-alpine:3.6

# ensure www-data user exists
RUN set -x \
	&& addgroup -g 82 -S www-data \
	&& adduser -u 82 -D -S -G www-data www-data
# 82 is the standard uid/gid for "www-data" in Alpine
# http://git.alpinelinux.org/cgit/aports/tree/main/apache2/apache2.pre-install?h=v3.3.2
# http://git.alpinelinux.org/cgit/aports/tree/main/lighttpd/lighttpd.pre-install?h=v3.3.2
# http://git.alpinelinux.org/cgit/aports/tree/main/nginx-initscripts/nginx-initscripts.pre-install?h=v3.3.2

ENV HTTPD_PREFIX /usr/local/apache2
ENV PATH $HTTPD_PREFIX/bin:$PATH
RUN mkdir -p "$HTTPD_PREFIX" \
	&& chown www-data:www-data "$HTTPD_PREFIX"
WORKDIR $HTTPD_PREFIX

ENV HTTPD_VERSION 2.4.27
ENV HTTPD_SHA1 699e4e917e8fb5fd7d0ce7e009f8256ed02ec6fc

# https://issues.apache.org/jira/browse/INFRA-8753?focusedCommentId=14735394#comment-14735394
ENV HTTPD_BZ2_URL https://www.apache.org/dyn/closer.cgi?action=download&filename=httpd/httpd-$HTTPD_VERSION.tar.bz2
# not all the mirrors actually carry the .asc files :'(
ENV HTTPD_ASC_URL https://www.apache.org/dist/httpd/httpd-$HTTPD_VERSION.tar.bz2.asc

# if the version is outdated, we have to pull from the archive :/
ENV HTTPD_BZ2_FALLBACK_URL https://archive.apache.org/dist/httpd/httpd-$HTTPD_VERSION.tar.bz2
ENV HTTPD_ASC_FALLBACK_URL https://archive.apache.org/dist/httpd/httpd-$HTTPD_VERSION.tar.bz2.asc

# see https://httpd.apache.org/docs/2.4/install.html#requirements
RUN set -x \
	&& runDeps=' \
		apr-dev \
		apr-util-dev \
		apr-util-ldap \
		perl \
	' \
	&& apk add --no-cache --virtual .build-deps \
		$runDeps \
		ca-certificates \
		coreutils \
		dpkg-dev dpkg \
		gcc \
		gnupg \
		libc-dev \
		# mod_session_crypto
		libressl \
		libressl-dev \
		# mod_proxy_html mod_xml2enc
		libxml2-dev \
		# mod_lua
		lua-dev \
		make \
		# mod_http2
		nghttp2-dev \
		pcre-dev \
		tar \
		# mod_deflate
		zlib-dev \
	\
	&& { \
		wget -O httpd.tar.bz2 "$HTTPD_BZ2_URL" \
		|| wget -O httpd.tar.bz2 "$HTTPD_BZ2_FALLBACK_URL" \
	; } \
	&& echo "$HTTPD_SHA1 *httpd.tar.bz2" | sha1sum -c - \
# see https://httpd.apache.org/download.cgi#verify
	&& { \
		wget -O httpd.tar.bz2.asc "$HTTPD_ASC_URL" \
		|| wget -O httpd.tar.bz2.asc "$HTTPD_ASC_FALLBACK_URL" \
	; } \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --keyserver ha.pool.sks-keyservers.net --recv-keys A93D62ECC3C8EA12DB220EC934EA76E6791485A8 \
	&& gpg --batch --verify httpd.tar.bz2.asc httpd.tar.bz2 \
	&& rm -rf "$GNUPGHOME" httpd.tar.bz2.asc \
	\
	&& mkdir -p src \
	&& tar -xf httpd.tar.bz2 -C src --strip-components=1 \
	&& rm httpd.tar.bz2 \
	&& cd src \
	\
# https://bz.apache.org/bugzilla/show_bug.cgi?id=61184
# https://github.com/docker-library/httpd/pull/68
	&& wget -O libressl.patch 'https://git.alpinelinux.org/cgit/aports/plain/main/apache2/libressl.patch?id=d7292029f25a131a0f15ebc3bc2300e75f4c131a' \
	&& patch -p1 < libressl.patch \
	&& rm libressl.patch \
	\
	&& gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
	&& ./configure \
		--build="$gnuArch" \
		--prefix="$HTTPD_PREFIX" \
		--enable-mods-shared=reallyall \
	&& make -j "$(nproc)" \
	&& make install \
	\
	&& cd .. \
	&& rm -r src man manual \
	\
	&& sed -ri \
		-e 's!^(\s*CustomLog)\s+\S+!\1 /proc/self/fd/1!g' \
		-e 's!^(\s*ErrorLog)\s+\S+!\1 /proc/self/fd/2!g' \
		"$HTTPD_PREFIX/conf/httpd.conf" \
	\
	&& runDeps="$runDeps $( \
		scanelf --needed --nobanner --recursive /usr/local \
			| awk '{ gsub(/,/, "\nso:", $2); print "so:" $2 }' \
			| sort -u \
			| xargs -r apk info --installed \
			| sort -u \
	)" \
	&& apk add --virtual .httpd-rundeps $runDeps \
	&& apk del .build-deps

COPY httpd-foreground /usr/local/bin/

EXPOSE 80
CMD ["httpd-foreground"]
