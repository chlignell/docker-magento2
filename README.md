# Magento2 (Varnish + PHP7 + Redis + SSL) cluster ready docker-compose infrastructure

## Infrastructure overview
* Container 1: MariaDB
* Container 2: Redis (for Magento's cache)
* Container 3: Apache 2.4 + PHP 7 (modphp)
* Container 4: Cron
* Container 5: Varnish 4.1
* Container 6: Redis (for autodiscovery cluster nodes)
* Container 7: Nginx SSL terminator

### Why a separate cron container?
First of all containers should be (as far as possible) single process, but the most important thing is that (if someday we'll be able to deploy this infrastructure in production) we may need a cluster of apache+php containers but a single cron container running.

Plus, with this separation, in the context of a docker swarm, you may be able in the future to separare resources allocated to the cron container from the rest of the infrastructure.

## Downloading Magento2
```
composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition magento2
```

## Do you want sample data?
Execute (from the host machine):
```
cd magento2
php bin/magento sampledata:deploy
composer update
```
You can lunch the same commmands from within the container, it's actually the same thing

## Starting all docker containers
```
docker-compose up -d
```
The fist time you run this command it's gonna take some time to download all the required images from docker hub.

## Install Magento2

open your browser to the address:
```
http://magento2.docker/
```
and use the wizard to install Magento2.  
For database configuration use hostname dockermagento2_db_1 and username/password/dbname you have in your docker-compose.xml file, defaults are:
- MYSQL_USER=magento2
- MYSQL_PASSWORD=magento2
- MYSQL_DATABASE=magento2

## Deploy static files
```
docker exec -it dockermagento2_apache_1 bash
php bin/magento dev:source-theme:deploy
php bin/magento setup:static-content:deploy
```

## Enable Redis for Magento's cache
open magento2/app/etc/env.php and add these lines:
```php
'cache' => array(
  'frontend' => array(
    'default' => array(
      'backend' => 'Cm_Cache_Backend_Redis',
      'backend_options' => array(
        'server' => 'cache',
        'port' => '6379',
        'persistent' => '', // Specify a unique string like "cache-db0" to enable persistent connections.
        'database' => '0',
        'password' => '',
        'force_standalone' => '0', // 0 for phpredis, 1 for standalone PHP
        'connect_retries' => '1', // Reduces errors due to random connection failures
        'read_timeout' => '10', // Set read timeout duration
        'automatic_cleaning_factor' => '0', // Disabled by default
        'compress_data' => '1', // 0-9 for compression level, recommended: 0 or 1
        'compress_tags' => '1', // 0-9 for compression level, recommended: 0 or 1
        'compress_threshold' => '20480', // Strings below this size will not be compressed
        'compression_lib' => 'gzip', // Supports gzip, lzf and snappy,
        'use_lua' => '0' // Lua scripts should be used for some operations
      )
    ),
    'page_cache' => array(
      'backend' => 'Cm_Cache_Backend_Redis',
      'backend_options' => array(
        'server' => 'cache',
        'port' => '6379',
        'persistent' => '', // Specify a unique string like "cache-db0" to enable persistent connections.
        'database' => '1', // Separate database 1 to keep FPC separately
        'password' => '',
        'force_standalone' => '0', // 0 for phpredis, 1 for standalone PHP
        'connect_retries' => '1', // Reduces errors due to random connection failures
        'lifetimelimit' => '57600', // 16 hours of lifetime for cache record
        'compress_data' => '0' // DISABLE compression for EE FPC since it already uses compression
      )
    )
  )
),
```
and delete all Magento's cache with
```
rm -rf magento2/var/cache/*
```
from now on the var/cache directory should stay empty cause all the caches should be stored in Redis.

## Enable Varnish
Varnish Full Page Cache should already be enabled out of the box (we startup Varnish with the default VCL file generated by Magento2) but you could anyway go to "stores -> configuration -> advanced -> system -> full page cache" and:
* select Varnish in the "caching application" combobox
* type "apache" in both "access list" and "backend host" fields
* type 80 in the "backend port" field
* save

## Enable SSL Support
Add this line to magento2/.htaccess
```
SetEnvIf X-Forwarded-Proto https HTTPS=on
```
Then you can configure Magento as you wish to support secure urls.

If you need to generate new self signed certificates use this command
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt
```
then you can mount them into the nginx-ssl container using the "volumes" instruction in the docker-compose.xml file. Same thing goes if you need to use custom nginx configurations (you can mount them into /etc/nginx/conf.d). Check the source code of https://github.com/fballiano/docker-nginx-ssl-for-magento2 to better understand where are the configuration stored inside the image/container.

## Scaling apache containers
If you need more horsepower you can
```
docker-compose scale apache=X
```
where X is the number of apache containers you want.

The cron container will check how many apache containers we have (broadcast/discovery service is stored on the redis_clusterdata container) and will update Varnish's VCL.

You can start your system with just one apache container, then scale it afterward, autodiscovery will reconfigure the load balancing on the fly.

Also, the cron container (which updates Varnish's VCL) sets a "probe" to "/fb_host_probe.txt" every 5 seconds, if 1 fails (container has been shut down) the container is considered sick.

## Tested on:
* Docker for Mac 17

## TODO
* migrate to alpine/linuxkit based containers
* optimize everything for docker swarm
* sessions on redis?
* DB clustering?
