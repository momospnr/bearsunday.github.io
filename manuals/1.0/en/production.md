---
layout: docs-en
title: Production
category: Manual
permalink: /manuals/1.0/en/production.html
---

(This chapter is not yet updated for the BEAR.Package 1.10+)

# Production

In this section, we will cover how to setup the cache and the system for the production environment.


## Context

`prod` is the context for production.
Cache is used for root object `$app`, annotation reader, etc.

```php
<?php
require dirname(__DIR__) . '/autoload.php';
exit((require dirname(__DIR__) . '/bootstrap.php')('prod-api-app'));
```

## Refresh $app

**IMPORTANT**

**In production, You need to regenerate $app cache in each deploy.**

To regenerate the `$app` cache, You **change the timestamp of the `src/` directory**. The BEAR.Sunday framework recognise it, then it re-generate `$app` and whole DI/AOP files under `tmp/` directory.

## ProdModule

Set `ProdModule` for the application in `src/Module/ProdModule.php` to customize the bindings for production and to allow HTTP OPTIONS methods.

```php
<?php
namespace Polidog\Todo\Module;

use BEAR\Package\Context\ProdModule as PackageProdModule;
use BEAR\QueryRepository\CacheVersionModule;
use BEAR\Resource\Module\OptionsMethodModule;
use BEAR\Package\AbstractAppModule;

class ProdModule extends AbstractModule
{
    /**
     * {@inheritdoc}
     */
    protected function configure()
    {
        $this->install(new PackageProdModule);       // default setting (recommended)
        $this->install(new OptionsMethodModule);     // Enable HTTP OPTIONS method in production
        $this->install(new CacheVersionModule('1')); // Specify version number of resource cache
    }
}
```
## Cache

There are two kinds of caches: a local cache that does not share between multiple Web servers and a shared cache that shares. The local cache is used for unchanged cachesafter deploy, such as annotations. The shared cache is used to store the resource state.

Both caches are by default chain cache of [APCu + file cache]. APCu is used preferentially and the write is done to both storages.

### Resource Cache

In order to configure multiple web servers, it is necessary to set shared cache storage. Install ([Memcached] (http://php.net/manual/en/book.memcached.php) or [Redis] (https://redis.io/)) module.

### Memcached

```php
<?php
namespace BEAR\HelloWorld\Module;

use BEAR\QueryRepository\StorageMemcachedModule;
use BEAR\Package\Context\ProdModule as PackageProdModule;
use BEAR\Package\AbstractAppModule;
use Ray\Di\Scope;

class ProdModule extends AbstractModule
{
    /**
     * {@inheritdoc}
     */
    protected function configure()
    {
        // memcache
        // {host}:{port}:{weight},...
        $memcachedServers = 'mem1.domain.com:11211:33,mem2.domain.com:11211:67';
        $this->install(new StorageMemcachedModule(memcachedServers);

        // Install default ProdModule
        $this->install(new PackageProdModule);
    }
}
```

### Redis

```php?start_inline
// redis
$redisServer = 'localhost:6379'; // {host}:{port}
$this->install(new StorageRedisModule($redisServer);
```

When using storage other than the above, create a new module with reference to each module.

### Set cache expiration (TTL)

To change the default TTL,Install `StorageExpiryModule`.


```php?start_inline
// Cache time
$short = 60;
$medium = 3600;
$long = 24 * 3600;
$this->install(new StorageExpiryModule($short, $medium, $long);
```
### Specify the cache version

Resource cache incompatibility is lost In the case of deploy, change the cache version.

```
$this->install(new CacheVersionModule($cacheVersion));
```

If you want to destroy the resource cache every time you deploy, assign time and random value to `$ cacheVersion`. (This statement is invoked only once after deploy.)

## Deploy

### ⚠️ Avoid overwriting updates

Overwriting a running project folder with `rsync` or the like has a risk of inconsistency between the resource cache and automatic generation class file created in　`tmp/` and the actual class. On heavily loaded sites, it is possible that multiple jobs such as cache creation and opcode creation are executed at the same time, exceeding the performance capacity of the site.

Set up in a different directory and switch it (by symlink) if the setup is OK.

### 👍🏻 Compilation recommended

When setting up, you can warm up the project using the `vendor/bin/bear.compile` script. The compilation script prepares all static cache files such as dynamically created files and annotations for DI / AOP in advance.

Since injection is done in all classes, there is no problem of DI error at runtime. In addition, although `.env` generally contains credential information such as API key and password, all contents are imported into PHP file and can be deleted after compilation. Compilation makes the deployment faster and safer. The optimized autoload.php file and preload.php are also output.

**Execution at the console**

```
vendor/bin/bear.compile 'Polidog\Todo' prod-html-app /path/to/prject
```

### autoload.php

It will output an autoload.php file optimized for `{project_path}/autoload.php`, which is much faster than the `vendor/autoload.php` dumped by `composer dumpa-autoload --optimize`.

### preload.php

You need to specify `opcache.preload` and `opcache.preload` in `php.ini` to enable preload, a feature supported in PHP 7.4 but not recommended for `7.4.0`-`7.4.2`.

Example)

```
opcache.preload=/path/to/project/preload.php
opcache.preload_user=www-data
```

Note: Please refer to bechmark for performance information.

## Deploy

Deployer's [BEAR.Sunday recipe](https://github.com/bearsunday/deploy) is convenient and safe to use. Consider using the other server configuration tool as well as referring or running the Deployer script. Since Deployer generates a project directory each time, you do not have to worry about regenerating `$app`.
