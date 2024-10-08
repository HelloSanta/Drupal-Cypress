FROM php:8.1-apache-bullseye

# install the PHP extensions we need
RUN set -eux; \
	\
	if command -v a2enmod; then \
		a2enmod rewrite; \
	fi; \
	\
	savedAptMark="$(apt-mark showmanual)"; \
	\
	apt-get update; \
	apt-get install -y --no-install-recommends \
		libfreetype6-dev \
		libjpeg62-turbo-dev \
		libjpeg-dev \
		libpng-dev \
		libwebp-dev \
		libxpm-dev \
		libpq-dev \
		libzip-dev \
		libsodium-dev \
	; \
	\
	docker-php-ext-configure gd \
		--with-freetype \
		--with-jpeg=/usr \
		--with-webp=/usr \
		--with-xpm=/usr \
	; \
	\
	docker-php-ext-install -j "$(nproc)" \
		gd \
		opcache \
		pdo_mysql \
		pdo_pgsql \
		zip \
		bcmath \
		exif \
		sodium \
	; \
	\
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
	apt-mark auto '.*' > /dev/null; \
	apt-mark manual $savedAptMark; \
	ldd "$(php -r 'echo ini_get("extension_dir");')"/*.so \
		| awk '/=>/ { print $3 }' \
		| sort -u \
		| xargs -r dpkg-query -S \
		| cut -d: -f1 \
		| sort -u \
		| xargs -rt apt-mark manual; \
	\
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	rm -rf /var/lib/apt/lists/*

# set recommended PHP.ini settings
# see https://secure.php.net/manual/en/opcache.installation.php
RUN { \
	echo 'opcache.memory_consumption=128'; \
	echo 'opcache.interned_strings_buffer=8'; \
	echo 'opcache.max_accelerated_files=4000'; \
	echo 'opcache.revalidate_freq=60'; \
	echo 'opcache.fast_shutdown=1'; \
	} > /usr/local/etc/php/conf.d/opcache-recommended.ini

# Install drush launcher
RUN php -r "readfile('https://github.com/drush-ops/drush-launcher/releases/latest/download/drush.phar');" > drush \
	&& chmod +x drush \
	&& mv drush /usr/local/bin/drush

# Install Memcached for php 8
RUN apt-get update && apt-get install -y libmemcached-dev zlib1g-dev \
		&& pecl install memcached \
		&& docker-php-ext-enable memcached

# Install mysql-clients && rsync. In order to sync database with the container
RUN apt-get install -y rsync default-mysql-client

# Install nodejs && cypress
RUN curl -s https://deb.nodesource.com/setup_19.x | bash
RUN apt-get update && \
    apt-get install -y \
	nodejs \
	wget \
	libgtk2.0-0 \
	libgtk-3-0 \
	libgbm-dev \
	libnotify-dev \
	libgconf-2-4 \
	libnss3 \
	libxss1 \
	libasound2 \
	libxtst6 \
	xauth \
	xvfb

RUN npm install cypress --save-dev

# Install Chrome browser
RUN apt-get update && \
    apt-get install -y curl apt-transport-https gnupg && \
    curl -sSL https://dl.google.com/linux/linux_signing_key.pub | apt-key add - && \
    echo "deb [arch=amd64] https://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google-chrome.list && \
    apt-get update && \
    apt-get install -y dbus-x11 google-chrome-stable && \
    rm -rf /var/lib/apt/lists/*

# ADD Configuration to the Container
ADD docker_conf/apache2.conf /etc/apache2/apache2.conf
ADD docker_conf/000-default.conf /etc/apache2/sites-available/000-default.conf
ADD docker_conf/php.ini /usr/local/etc/php/

WORKDIR /var/www/html