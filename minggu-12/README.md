
Nama : Yoga Kurnia Subekti  
NIM  : 175410033  
    ################################################################################
    # Docker Compose Drupal 8 full dev stack.
    #
    # Project page:
    #   https://github.com/Mogtofu33/docker-compose-drupal
    # Documentation:
    #   https://github.com/Mogtofu33/docker-compose-drupal/blob/master/README.md
    ################################################################################
version: '3'
services:
  apache:
    image: mogtofu33/apache:latest
    depends_on:
      - php
    ports:
      - "${APACHE_HOST_HTTP_PORT:-80}:80"
      - "${APACHE_HOST_HTTPS_PORT:-443}:443"
      # Web root access, optional.
      - "${APACHE_HOST_ROOT_PORT:-88}:81"
    volumes:
      - ${HOST_WEB_ROOT:-./drupal}:/var/www/localhost
      # Apache configuration with SSL support.
      - ./config/apache/httpd.conf:/etc/apache2/httpd.conf:ro
      - ./config/apache/vhost.conf:/etc/apache2/vhost.conf:ro
      - ./config/apache/conf.d/:/etc/apache2/conf.d/:ro
      - ./config/apache/ssl/:/etc/ssl/apache2/:ro
      # Php FPM socket
      - php-sock:/sock:ro
      ## If PgSQL, ease drush pgsql access.
      - ./config/pgsql/.pg_pass:/home/apache/.pg_pass:ro
    env_file: .env
    restart: unless-stopped
    container_name: ${PROJECT_NAME:-dcd}-apache
  nginx:
    image: nginx:alpine
    depends_on:
      - php
    ports:
      - "${NGINX_HOST_HTTP_PORT:-81}:80"
      # - "${NGINX_HOST_HTTPS_PORT:-444}:443"
    volumes:
      - ${HOST_WEB_ROOT:-./drupal}:/var/www/localhost
      - ./config/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      # Php FPM socket
      - php-sock:/sock:ro
    env_file: .env
    restart: unless-stopped
    container_name: ${PROJECT_NAME:-dcd}-nginx
  php:
    image: mogtofu33/php:${PHP_VERSION:-7.2}
    volumes:
      ## If you have composer set locally, share cache to speed up.
      # - ${HOME:-}/.composer/cache:/var/www/.composer/cache
      - ${HOST_WEB_ROOT:-./drupal}:/var/www/localhost
      - ./config/php/${PHP_VERSION:-7.2}/php.ini:/etc/php7/php.ini:ro
      - ./config/php/${PHP_VERSION:-7.2}/php-fpm.conf:/etc/php7/php-fpm.conf:ro
      - ./config/php/${PHP_VERSION:-7.2}/conf.d/:/etc/php7/conf.d/:ro
      - ./config/php/${PHP_VERSION:-7.2}/php-fpm.d/:/etc/php7/php-fpm.d/:ro
      # Drush 9 config file.
      - ./config/drush/drush.yml:/etc/drush/drush.yml:ro
      # Used by dashboard for accessing tools.
      - ./tools:/tools:ro
      # Share db dump folder.
      - ${HOST_DATABASE_DUMP:-./database/dump}:/dump
      # Php FPM socket
      - php-sock:/sock
      ## If PgSQL, ease drush pgsql access.
      - ./config/pgsql/.pg_pass:/home/apache/.pg_pass:ro
    links:
      # Choose database, uncomment service concerned below.
      - mysql
      - pgsql
      # Choose optionnal services.
      - memcache
      - redis
      - solr
      - mailhog
      # - varnish
    env_file: .env
    restart: unless-stopped
    container_name: ${PROJECT_NAME:-dcd}-php
  mysql:
    image: mariadb:latest
    expose:
      - "3306"
    volumes:
      - ${HOST_DATABASE_DUMP:-./database/dump}:/dump
      # All .sql files will be imported on first start by order.
      - ./database/mysql-init:/docker-entrypoint-initdb.d
      - ./config/mysql:/etc/mysql:ro
    env_file: .env
    restart: unless-stopped
    container_name: ${PROJECT_NAME:-dcd}-mysql
  pgsql:
    image: postgres:alpine
    expose:
      - "5432"
    volumes:
      - ${HOST_DATABASE_DUMP:-./database/dump}:/dump
      # A dump.pg_dump file will be imported from here on first start.
      - ./database/pgsql-init:/docker-entrypoint-initdb.d
      # Add pg_pass to ease drush access.
      - ./config/pgsql/.pg_pass:/home/postgres/.pg_pass
    env_file: .env
    restart: unless-stopped
    container_name: ${PROJECT_NAME:-dcd}-pgsql
  mailhog:
    image: mailhog/mailhog:latest
    expose:
      - "1025"
    ports:
      - "8025:8025"
    restart: unless-stopped
    container_name: ${PROJECT_NAME:-dcd}-mailhog
  memcache:
    image: memcached:alpine
    expose:
      - "11211"
    restart: unless-stopped
    container_name: ${PROJECT_NAME:-dcd}-memcache
  redis:
    image: redis:alpine
    expose:
      - "6379"
    restart: unless-stopped
    container_name: ${PROJECT_NAME:-dcd}-redis
  solr:
    image: mogtofu33/solr:${SOLR_VERSION:-7}
    ports:
      - "8983:8983"
    environment:
      - EXTRA_ACCESS="/solr/drupal"
    restart: unless-stopped
    container_name: ${PROJECT_NAME:-dcd}-solr
    # varnish:
    #   image: wodby/varnish:latest
    #   depends_on:
    #     - apache
    #     # - nginx
    #   ports:
    #     - "6081:6081"
    #   environment:
    #     VARNISH_SECRET: secret
    #     VARNISH_BACKEND_HOST: apache
    #     # VARNISH_BACKEND_HOST: nginx
    #     VARNISH_BACKEND_PORT: 80
    #     VARNISH_CONFIG_PRESET: drupal
    #     VARNISH_PURGE_EXTERNAL_REQUEST_HEADER: X-Real-IP
    #   container_name: ${PROJECT_NAME:-dcd}-varnish
  tools:
    image: nginx:alpine
    depends_on:
      - php
    ports:
      - "${HOST_TOOLS_PORT:-8008}:80"
    volumes:
      - ./tools:/tools:ro
      - ./build/dashboard/default.conf:/etc/nginx/conf.d/default.conf:ro
      # Php FPM socket
      - php-sock:/sock:ro
    env_file: .env
    restart: unless-stopped
    container_name: ${PROJECT_NAME:-dcd}-tools
  dashboard:
    image: mogtofu33/dashboard:latest
    ports:
      - "${HOST_DASHBORAD_PORT:-8181}:5000"
    volumes:
      # Share configs for reading on the dashboard.
      - ./config/apache/vhost.conf:/config/vhost.conf:ro
      - ./config/nginx/default.conf:/config/default.conf:ro
      - ./tools:/tools:ro
      - .env:/config/stack.env:ro
      # Access docker daemon.
      - /var/run/docker.sock:/var/run/docker.sock
    env_file: .env
    restart: unless-stopped
    container_name: ${PROJECT_NAME:-dcd}-dashboard
    ## Full advanced dashboard.
    # portainer:
    #   image: portainer/portainer:latest
    #   ports:
    #     - "9000:9000"
    #   command: --no-auth -H unix:///var/run/docker.sock
    #   volumes:
    #     - /var/run/docker.sock:/var/run/docker.sock
    #   restart: unless-stopped
    #   container_name: ${PROJECT_NAME:-dcd}-portainer

volumes:
  php-sock:
