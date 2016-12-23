
It is an Nginx server Docker on the tiny, efficient and secure Alpine linux for 
[official WordPress image](https://hub.docker.com/_/wordpress/) PHP-FPM variant
and a proxy server for any other service like Redmine.

The Nginx configuration in this image is based on raulr's great project nginx-wordpress and the guidelines given by the [Wordpress Codex](https://codex.wordpress.org/Nginx).

### How to Use This Image From Command Line

    $ docker run --name nginx --link wordpress:wordpress --volumes-from wordpress -d -p 80:80 repassyl/nginx-alpine-wp-and-proxy

#### Environment Variables

* `POST_MAX_SIZE`: Sets max size of post data allowed. Also affects file uploads (defaults to `64m`).
* `BEHIND_PROXY`: Set to `true` if this container is behind a reverse proxy (defaults to `false` unless `VIRTUAL_HOST` environment variable is set).
* `REAL_IP_HEADER`: Defines the request header that will be used to obtain the real client IP when `BEHIND_PROXY` is set to `true` (defaults to `X-Forwarded-For`).
* `REAL_IP_FROM`: Defines trusted addresses to obtain the real client IP when `BEHIND_PROXY` is set to `true` (defaults to `172.17.0.0/16`).

### How to use it via [`docker-compose`](https://github.com/docker/compose)

Example `docker-compose.yml`:

```yaml
wordpress:
  image: wordpress:4-php7.0-fpm-alpine
  links:
    - db:mysql

nginx:
  image: repassyl/nginx-alpine-wp-and-proxy
  links:
   - wordpress
  volumes_from:
   - wordpress
  ports:
   - "80:80"
  environment:
    POST_MAX_SIZE: 128m

db:
  image: mariadb
  environment:
    MYSQL_ROOT_PASSWORD: example
```

Run `docker-compose up -d`, wait for it to initialize completely, and visit `http://host-ip`.

### Example for use it as a proxy and a Wordpress webserver

nginx_redmine_proxy.conf (in same directory as docker-composer.yml):

```yaml

server {
        listen 80;
        listen [::]:80;

        server_name redmine.proability.hu;
        location / {
            proxy_pass         http://redmine:3000/;
            proxy_redirect     off;

            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }

}
```

Example `docker-compose.yml`:

```yaml
version: '2'

services:
  redmine:
    image: redmine:3.3
    ports:
      - 3000:3000
    environment:
      REDMINE_DB_MYSQL: db
      REDMINE_DB_PASSWORD: password123
    depends_on:
      - db
    restart: always
    volumes:
      - ./redmine_files:/usr/src/redmine/files
      - ./configuration.yml:/usr/src/redmine/config/configuration.yml

  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: password123
      MYSQL_DATABASE: redmine
    restart: always

  nginx:
    image: repassyl/nginx-alpine-wp-and-proxy
    links:
      - wordpress
    volumes_from:
      - wordpress
    volumes:
      - ./nginx_redmine_proxy.conf:/etc/nginx/conf.d/nginx_redmine_proxy.conf
    depends_on:
      - redmine
    ports:
      - "80:80"
    environment:
      POST_MAX_SIZE: 128m
      # BEHIND_PROXY: 'true'
    restart: always

  wordpress:
    depends_on:
      - db
    image: wordpress:4-php7.0-fpm-alpine
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_PASSWORD: password123
      WORDPRESS_DB_NAME: example_db
    volumes:
      - ./wordpress_files_on_host:/var/www/html
```









# nginx-alpine-wp-and-proxy
