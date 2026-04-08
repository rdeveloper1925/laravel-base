# Override via build-arg (e.g. mirror when Docker Hub auth returns 504)
ARG PHP_BASE_IMAGE=php:8.4-fpm

# Stage 1: Build environment and Composer dependencies
FROM ${PHP_BASE_IMAGE} AS builder

# Install system dependencies and PHP extensions required for Laravel + MySQL
# Some dependencies are required for PHP extensions only in the build stage
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    unzip \
    libonig-dev \
    libssl-dev \
    libxml2-dev \
    libcurl4-openssl-dev \
    libicu-dev \
    libzip-dev \
    && docker-php-ext-install -j$(nproc) \
    pdo_mysql \
    intl \
    zip \
    bcmath \
    soap \
    && pecl install redis \
    && docker-php-ext-enable redis \
    && apt-get autoremove -y && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

WORKDIR /var/www

# Copy application code (composer scripts need artisan and app paths)
COPY . /var/www

# APP_KEY is required for package:discover during composer install
RUN cp .env.example .env \
    && php -r '$k = base64_encode(random_bytes(32)); $e = file_get_contents(".env"); $e = preg_replace("/^APP_KEY=.*/m", "APP_KEY=base64:".$k, $e); file_put_contents(".env", $e);'

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer \
    && composer install --no-dev --optimize-autoloader --no-interaction --no-progress --prefer-dist

RUN rm -f .env

# Build Vite assets (Wayfinder plugin runs `php artisan wayfinder:generate` during `npm run build`)
FROM builder AS assets

RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl \
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y --no-install-recommends nodejs \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /var/www

RUN cp .env.example .env \
    && php -r '$k = base64_encode(random_bytes(32)); $e = file_get_contents(".env"); $e = preg_replace("/^APP_KEY=.*/m", "APP_KEY=base64:".$k, $e); file_put_contents(".env", $e);'

RUN npm ci && npm run build && rm -f .env

# Stage 2: Production environment
FROM ${PHP_BASE_IMAGE} AS production

RUN apt-get update && apt-get install -y --no-install-recommends \
    libicu-dev \
    libzip-dev \
    libfcgi-bin \
    nano \
    procps \
    && apt-get autoremove -y && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

RUN curl -o /usr/local/bin/php-fpm-healthcheck \
    https://raw.githubusercontent.com/renatomefi/php-fpm-healthcheck/master/php-fpm-healthcheck \
    && chmod +x /usr/local/bin/php-fpm-healthcheck

COPY ./docker/production/php-fpm/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

COPY ./storage /var/www/storage-init

COPY --from=builder /usr/local/lib/php/extensions/ /usr/local/lib/php/extensions/
COPY --from=builder /usr/local/etc/php/conf.d/ /usr/local/etc/php/conf.d/
COPY --from=builder /usr/local/bin/docker-php-ext-* /usr/local/bin/

RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

COPY ./docker/common/php-fpm/conf.d/*.conf /usr/local/etc/php-fpm.d/

COPY --from=builder /var/www /var/www

COPY --from=assets /var/www/public/build /var/www/public/build

WORKDIR /var/www

RUN chown -R www-data:www-data /var/www

USER www-data

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]

EXPOSE 9000
CMD ["php-fpm"]
