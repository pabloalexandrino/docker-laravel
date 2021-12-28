## First stage. Copy project files and run composer
FROM composer:2 as composer_stage

RUN rm -rf /var/www && mkdir -p /var/www/html
WORKDIR /var/www/html

COPY composer.json composer.lock symfony.lock .env ./
COPY public public/

RUN composer install --ignore-platform-reqs --prefer-dist --no-scripts --no-progress --no-suggest --no-interaction --no-dev --no-autoloader

RUN composer dump-autoload --optimize --apcu --no-dev

COPY bin bin/
COPY config config/
COPY src src/

RUN composer run-script $NODEV post-install-cmd; \
    chmod +x bin/console;

## Second stage. Build NPM dependencies

FROM node:12 as npm_builder

COPY --from=composer_stage /var/www/html /var/www/html

WORKDIR /var/www/html
COPY yarn.lock package.json webpack.config.js ./
COPY assets ./assets

RUN yarn install
RUN yarn encore prod

# I'm using yarn here, but using npm would be similar, depending on how your project is setup

# RUN npm install
# RUN npm run build # if necessary and the command exists in your project

## Third stage, mostly copied from your original Dockerfile

FROM php:7.4-fpm

# Arguments defined in docker-compose.yml
ARG user
ARG uid

# Install system dependencies
RUN apt-get update && apt-get install -y \
    nodejs \
    npm \
    git \
    libpng-dev \
    zlib1g-dev \
    libxml2-dev \
    libzip-dev \
    libonig-dev \
    zip \
    curl \
    unzip \
    && docker-php-ext-configure gd \
    && docker-php-ext-install -j$(nproc) gd \
    && docker-php-ext-install pdo_mysql \
    && docker-php-ext-install mysqli \
    && docker-php-ext-install zip \
    && docker-php-ext-configure intl \
    && docker-php-ext-install intl \
    && docker-php-source delete

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install PHP extensions
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# Get latest Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Create system user to run Composer and Artisan Commands
RUN useradd -G www-data,root -u $uid -d /home/$user $user
RUN mkdir -p /home/$user/.composer && \
    chown -R $user:$user /home/$user 

# Set working directory
WORKDIR /var/www

USER $user

