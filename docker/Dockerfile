# Backend build stage
FROM composer:latest AS backend_build

COPY backend/composer.json /var/www/
RUN composer install -d /var/www/

# Frontend build stage
FROM node:lts-alpine as frontend_build
ARG DOMAIN_NAME

RUN apk add git

COPY frontend /app/
RUN sed -i -r "s|REPLACE_ME|https:\/\/${DOMAIN_NAME}|g"  /app/app/home/startCommands.coffee

WORKDIR /app

RUN npm install
RUN npm install bower -g

RUN bower --allow-root install
RUN npm run-script dist

# Production stage
FROM php:7.4.33-apache

RUN apt-get update
RUN apt-get install -y --no-install-recommends \
      libicu-dev \
      libpq-dev \
      ca-certificates \
      ssl-cert \
      libcurl4-gnutls-dev \
      git \
      unzip \
      libzip-dev

RUN update-ca-certificates
RUN docker-php-ext-install \
      pdo_mysql \
      opcache \
      curl \
      zip

RUN apt-get autoremove \
    && rm -r /var/lib/apt/lists/*

COPY ./docker/apache-conf/000-default.conf /etc/apache2/sites-enabled/000-default.conf
RUN a2enmod rewrite expires deflate ssl cgi alias env headers

ARG GIT_REPO
ARG SITE_ADMIN_NAME
ARG SITE_ADMIN_EMAIL

RUN chown www-data:www-data /var/www
USER www-data
WORKDIR /var/www
RUN mkdir website

COPY --chown=www-data:www-data ./backend website/backend
COPY --chown=www-data:www-data --from=backend_build /var/www/vendor website/backend/vendor
COPY --chown=www-data:www-data --from=frontend_build /app/ website/frontend/

RUN mkdir -p website/backend/hook/hints website/backend/hook/verifications git workingarea

# Get the exercises
COPY --chown=www-data:www-data exercises/ /tmp/exercises/
RUN [ -z "${GIT_REPO}" ] || (rm -rf /tmp/exercises && git clone ${GIT_REPO} /tmp/)
# Copy the hints and verifications into the backend
RUN mkdir /var/www/git/exercises.git

# Create the exercise branches
WORKDIR /var/www/git/exercises.git
COPY docker/init-exercises.sh .
RUN ./init-exercises.sh
RUN rm -rf /tmp/exercises init-exercises.sh

RUN ln -s /var/www/website/backend/hook/hook.php .git/hooks/update
COPY ./docker/git-config .git/config

RUN git checkout main

