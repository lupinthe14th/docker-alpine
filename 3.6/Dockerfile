FROM scratch
#FROM usualoma/ruby-with-therubyracer:2.4.1-alpine

MAINTAINER Hideo Suzuki <lupinthe14th@gmail.com>

ENV ALPINE_ARCH x86_64
ENV ALPINE_VERSION 3.6.2

# http://nl.alpinelinux.org/alpine/v3.6/releases/x86_64/alpine-minirootfs-3.6.2-x86_64.tar.gz
ADD alpine-minirootfs-${ALPINE_VERSION}-${ALPINE_ARCH}.tar.gz /

# skip installing gem documentation
RUN mkdir -p /usr/local/etc \
    && { \
        echo 'install: --no-document'; \
        echo 'update: --no-document'; \
    } >> /usr/local/etc/gemrc

ENV RUBY_MAJOR 2.4
ENV RUBY_VERSION 2.4.1
ENV RUBY_DOWNLOAD_SHA256 4fc8a9992de3e90191de369270ea4b6c1b171b7941743614cc50822ddc1fe654
ENV RUBYGEMS_VERSION 2.6.13

# some of ruby's build scripts are written in ruby
#   we purge system ruby later to make sure our final image uses what we just built
# readline-dev vs libedit-dev: https://bugs.ruby-lang.org/issues/11869 and https://github.com/docker-library/ruby/issues/75
RUN set -ex \
    \
    && apk add --no-cache --virtual .ruby-builddeps \
        autoconf \
        bison \
        bzip2 \
        bzip2-dev \
        ca-certificates \
        coreutils \
        dpkg-dev dpkg \
        gcc \
        gdbm-dev \
        glib-dev \
        libc-dev \
        libffi-dev \
        libressl \
        libressl-dev \
        libxml2-dev \
        libxslt-dev \
        linux-headers \
        make \
        ncurses-dev \
        procps \
        readline-dev \
        ruby \
        tar \
        xz \
        yaml-dev \
        zlib-dev \
    \
    && wget -O ruby.tar.xz "https://cache.ruby-lang.org/pub/ruby/${RUBY_MAJOR%-rc}/ruby-$RUBY_VERSION.tar.xz" \
    && echo "$RUBY_DOWNLOAD_SHA256 *ruby.tar.xz" | sha256sum -c - \
    \
    && mkdir -p /usr/src/ruby \
    && tar -xJf ruby.tar.xz -C /usr/src/ruby --strip-components=1 \
    && rm ruby.tar.xz \
    \
    && cd /usr/src/ruby \
    \
# hack in "ENABLE_PATH_CHECK" disabling to suppress:
#   warning: Insecure world writable dir
    && { \
        echo '#define ENABLE_PATH_CHECK 0'; \
        echo; \
        cat file.c; \
    } > file.c.new \
    && mv file.c.new file.c \
    \
    && autoconf \
    && gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
# the configure script does not detect isnan/isinf as macros
    && export ac_cv_func_isnan=yes ac_cv_func_isinf=yes \
    && ./configure \
        --build="$gnuArch" \
        --disable-install-doc \
        --enable-shared \
    && make -j "$(nproc)" \
    && make install \
    \
    && runDeps="$( \
        scanelf --needed --nobanner --recursive /usr/local \
            | awk '{ gsub(/,/, "\nso:", $2); print "so:" $2 }' \
            | sort -u \
            | xargs -r apk info --installed \
            | sort -u \
    )" \
    && apk add --virtual .ruby-rundeps $runDeps \
        bzip2 \
        ca-certificates \
        libffi-dev \
        libressl-dev \
        procps \
        yaml-dev \
        zlib-dev \
    && apk del .ruby-builddeps \
    && cd / \
    && rm -r /usr/src/ruby \
    \
    && gem update --system "$RUBYGEMS_VERSION"

ENV BUNDLER_VERSION 1.15.4

RUN gem install bundler --version "$BUNDLER_VERSION"

# install things globally, for great justice
# and don't create ".bundle" in all our apps
ENV GEM_HOME /usr/local/bundle
ENV BUNDLE_PATH="$GEM_HOME" \
    BUNDLE_BIN="$GEM_HOME/bin" \
    BUNDLE_SILENCE_ROOT_WARNING=1 \
    BUNDLE_APP_CONFIG="$GEM_HOME"
ENV PATH $BUNDLE_BIN:$PATH
RUN mkdir -p "$GEM_HOME" "$BUNDLE_BIN" \
    && chmod 777 "$GEM_HOME" "$BUNDLE_BIN"

# libv8
ENV LIBV8_BRANCH v3.16.14.19
ENV LIBV8_VERSION 3.16.14.19-x86_64-linux

RUN set -ex \
    \
    && apk add --update --no-cache --virtual .libv8-builddeps \
      make \
      python \
      git \
      bash \
      curl \
      findutils \
      binutils-gold \
      tar \
      linux-headers \
      build-base \
    \
    && git clone -b $LIBV8_BRANCH --recursive git://github.com/cowboyd/libv8.git \
    && cd ./libv8 \
    && sed -i -e 's/Gem::Platform::RUBY/Gem::Platform.local/' libv8.gemspec \
    && gem build --verbose libv8.gemspec \
    && export GYP_DEFINES="$GYP_DEFINES linux_use_bundled_binutils=0 linux_use_bundled_gold=0" \
    && gem install --verbose libv8-$LIBV8_VERSION.gem \
    \
    && apk del --purge .libv8-builddeps \
    \
    && cd ../ \
    && rm -rf ./libv8 \
    && cd /usr/local/bundle/gems/libv8-$LIBV8_VERSION/vendor/ \
    && mkdir -p /tmp/v8 \
    && mv ./v8/out /tmp/v8/. \
    && mv ./v8/include /tmp/v8/. \
    && rm -rf ./v8 ./depot_tools \
    && mv /tmp/v8 . \
    \
    && apk add --update --no-cache libstdc++ \
    && apk add --update --no-cache --virtual=build-dependencies \
      build-base \
    && gem install therubyracer \
    && rm -fr /usr/local/bundle/gems/libv8-$LIBV8_VERSION/vendor \
    && apk del --purge build-dependencies \
    && rm -rf /tmp/* /var/tmp/* /var/cache/apk/*

# baku
ENV RAILS_SERVE_STATIC_FILES=true \
    RAILS_ENV=production \
    NODE_ENV=production

EXPOSE 3000

WORKDIR /baku

RUN apk add --update --no-cache --virtual .ruby-builddeps \
     build-base \
     libxml2-dev \
     libxslt-dev \
     mariadb-dev \
     nodejs \
     sqlite-dev \
     tzdata

COPY Gemfile Gemfile.lock /baku/

RUN bundle install --without test

COPY . /baku

VOLUME /baku/public/system /baku/public/assets /baku/public/packs
