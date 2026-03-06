# Vytvorenie lokálneho vývojového prostredia v Docker

Nainštalujte si Docker Engine a skontrolujte, či máte aj príkaz `compose`. Následne si založte adresár, pre lokálne prostredie.

```
/localphpdev
│
├── nginx
│   └── default.conf
│
├── php
│   └── Dockerfile
│
├── src
│   └── index.php
│
└── docker-compose.yml
```

Súbor `nginx/default.conf`

```conf
server {
	listen 80;
	server_name localhost;

	root /var/www/html;
	index index.php index.html;

  location ~* /vendor/ {
    deny all;
    return 403;
  }

	location / {
		try_files $uri $uri/ /index.php$query_string;
	}

	location ~ \.php$ {
		include fastcgi_params;
		fastcgi_pass php:9000;
		fastcgi_index index.php;
		fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
	}

	location ~ /\.ht {
		deny all;
	}
}
```

Súbor `php/Dockerfile` - zostaví kontajner s dodatočne nainštalovanými balíkmi a `composer` správcom knižníc.

```Dockerfile
FROM php:8.3-fpm

RUN docker-php-ext-install pdo pdo_mysql mysqli

RUN apt-get update && apt-get install -y \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    git \
    unzip \
    nano \
    curl \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install gd \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Configure PHP for large file uploads
RUN echo "upload_max_filesize = 500M" >> /usr/local/etc/php/php.ini && \
    echo "post_max_size = 500M" >> /usr/local/etc/php/php.ini

WORKDIR /var/www/html
```

Súbor `docker-compose.yml`, ktorý vytvorí výsledný _LEMP Stack_

```yml
services:
  nginx:
    image: nginx:latest
    container_name: dev_nginx
    ports:
      - "8080:80"
    volumes:
      - ./src:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php
      - db

  php:
    build: ./php
    container_name: dev_php
    volumes:
      - ./src:/var/www/html
    depends_on:
      - db

  db:
    image: mariadb:11
    container_name: dev_db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: app_db
      MYSQL_USER: app_user
      MYSQL_PASSWORD: app_pass
    volumes:
      - db_data:/var/lib/mysql
    ports:
      - "3306:3306"

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: dev_phpmyadmin
    restart: always
    ports:
      - "8081:80"
    environment:
      PMA_HOST: db
      PMA_USER: app_user
      PMA_PASSWORD: app_pass
    depends_on:
      - db

volumes:
  db_data:
```

Adresár `src/` slúži na umiestnenie vami vytvorených PHP skriptov. Tento adresár je koreňovým adresárom pre Nginx server.

## Použitie

Príkaz pre spustenie kontajnerov `docker compose up -d` - pri prvom spustení sa musia stiahnuť image a kontajnery sa nainštalujú. Všetky príkazy sa volajú na úrovni súboru `docker-compose.yml`.

Následne budete mať k dispozícii dve adresy:

- `http://localhost:8080` - tu beží Nginx server a hostuje adresáre a súbory zo `src`.
- `http://localhost:8081` - tu beží PhpMyAdmin rozhranie s automaticky prihláseným používateľom `app_user` a vytvorí databázu `app_db`.

Príkaz zastavenia kontajnerov: `docker compose down -v`.

Príkaz pre overenie `composer` inštalácie - `docker exec -it dev_php composer --version`, by mal vrátiť:

```bash
Composer version 2.9.5 2026-01-29 11:40:53
PHP version 8.3.30 (/usr/local/bin/php)
Run the "diagnose" command to get more detailed diagnostics output.
```

### Inštalácia PHP knižníc

1. Pripojenie sa do spusteného kontajnera `dev_php` a spustenie _shellu_:

```bash
docker exec -it dev_php bash
```

2. Zmení sa prompt na niečo takéto: `root@3e60dcb58592:/var/www/html#` (`3e60dcb58592` je identifikátor kontajnera). Následne môžeme nainštalovať požadované balíky:

```bash
composer require robthree/twofactorauth
```

3. Z promptu kontajnera sa dostaneme príkazom:

```bash
exit
```

Alternatívne môžeme inštalovať aj balíky bez pripojenia do _shell_:

```bash
docker-compose exec dev_php composer require google/apiclient
```

V adresári `src` vzniknú súbory `composer.json` a `composer.lock` a adresár so stiahnutými knižnicami `vendor`.
