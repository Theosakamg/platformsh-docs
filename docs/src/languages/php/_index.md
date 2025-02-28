---
title: "PHP"
description: PHP is a popular scripting language designed especially for the web. It currently powers over 80% of websites.
layout: single
---

## Supported versions

| **Grid** | **Dedicated** |
|----------------------------------|---------------|
|  {{< image-versions image="php" status="supported" environment="grid" >}} | {{< image-versions image="php" status="supported" environment="dedicated" >}} |


Note that as of PHP 7.1 we use the Zend Thread Safe (ZTS) version of PHP.

To specify a PHP container, use the `type` property in your `.platform.app.yaml`.

{{< readFile file="src/registry/images/examples/full/php.app.yaml" highlight="yaml" >}}

## Deprecated versions

The following versions are available but are not receiving security updates from upstream, so their use is not recommended. They will be removed at some point in the future.

| **Grid** | **Dedicated** |
|----------------------------------|---------------|
|  {{< image-versions image="php" status="deprecated" environment="grid" >}} | {{< image-versions image="php" status="deprecated" environment="dedicated" >}} |

## Support libraries

While it is possible to read the environment directly from your application, it is generally easier and more robust to use the [`platformsh/config-reader`](https://github.com/platformsh/config-reader-php) Composer library which handles decoding of service credential information for you.

## Alternate start commands

PHP is most commonly run in a CGI mode, using PHP-FPM. That is the default on Platform.sh. However, you can also start alternative processes if desired, such as if you're running an Async PHP daemon, a thread-based worker process, etc. To do so, simply specify an alternative start command in `platform.app.yaml`, similar to the following:

```yaml
web:
    commands:
        start: php run.php
    upstream:
            socket_family: tcp
            protocol: http
```

The above configuration will execute the `run.php` script in the application root when the container starts using the PHP-CLI SAPI, just before the deploy hook runs, but will *not* launch PHP-FPM. It will also tell the front-controller (Nginx) to connect to your application via a TCP socket, which will be specified in the `PORT` environment variable. Note that the start command _must_ run in the foreground.

If not specified, the effective default start command varies by PHP version:

* On PHP 5.x, it's `/usr/sbin/php5-fpm`.
* On PHP 7.0, it's `/usr/sbin/php-fpm7.0`.
* On PHP 7.1, it's `/usr/sbin/php-fpm7.1-zts`.
* On PHP 7.2, it's `/usr/sbin/php-fpm7.2-zts`.
* On PHP 7.3, it's `/usr/sbin/php-fpm7.3-zts`.
* On PHP 7.4, it's `/usr/sbin/php-fpm7.4-zts`.

While you can call it manually that is generally not necessary. Note that PHP-FPM cannot run simultaneously along with another persistent process (such as ReactPHP or Amp). If you need both they will have to run in separate containers.

## Expanded dependencies

In addition to the standard `dependencies` format, it is also possible to specify alternative repositories for use by Composer.  The standard format like so:

```yaml
dependencies:
    php:
        "platformsh/client": "dev-master"
```

is equivalent to `composer require platform/client dev-master`.  However, you can also specify explicit `require` and `repositories` blocks:

```yaml
dependencies:
    php:
        require:
            "platformsh/client": "dev-master"
        repositories:
            - type: vcs
              url: "git@github.com:platformsh/platformsh-client-php.git"
```

That would install `platformsh/client` from the alternate repository specified, as a global dependency.  That is, it is equivalent to the following `composer.json` file:

```json
{
    "repositories": [
        {
            "type": "vcs",
            "url":  "git@github.com:platformsh/platformsh-client-php.git"
        }
    ],
    "require": {
        "platformsh/client": "dev-master"
    }
}
```

That allows you to install a forked version of a global dependency from a custom repository.

## Build flavor

PHP images use the `composer` build flavor by default, which will run `composer --no-ansi --no-interaction install --no-progress --prefer-dist --optimize-autoloader` if a `composer.json` file is detected.

Note that by default, all PHP containers include the latest Composer 1.x release. If you wish to use Composer 2.x, add it as a `dependency` (see the section below).

```yaml
dependencies:
    php:
        composer/composer: '^2'
```

You will still see a message in the build output warning you about the availability of a new Composer version; that is the pre-packaged Composer 1 running to download Composer 2.  You can safely ignore it.  As Composer 2 is considerably more performant than Composer 1 we strongly recommend upgrading unless your application has a Composer plugin dependency that has not yet been updated.

`drupal` will run `drush make` automatically in one of a few different ways.  See the [Drupal 7](/frameworks/drupal7/_index.md) documentation for more details.  There is no reason to use this build mode except for Drupal 7.

## Opcache preloading

PHP 7.4 introduced a new feature called Opcache Preloading, which allows you to load selected files into shared memory when PHP-FPM starts.  That means functions and classes in those files are always available and do not need to be autoloaded, at the cost of any changes to those files requiring a PHP-FPM restart.  Since PHP-FPM restarts anyway when a new deploy happens this feature is a major win on Platform.sh, and we recommend using it aggressively.

To enable preloading, add a `php.ini` value that specifies a preload script.  Any [`php.ini` mechanism](/languages/php/ini.md) will work, but using a variable in `.platform.app.yaml` is the recommended approach:

```yaml
variables:
    php:
        opcache.preload: 'preload.php'
```

The `opcache.preload` value is evaluated as a file path relative to the application root (where `.platform.app.yaml` is), and it may be any PHP script that calls `opcache_compile_file()`.  The following example will preload all `.php` files anywhere in the `vendor` directory:

```php
<?php
$directory = new RecursiveDirectoryIterator(getenv('PLATFORM_APP_DIR') . '/vendor');
$iterator = new RecursiveIteratorIterator($directory);
$regex = new RegexIterator($iterator, '/^.+\.php$/i', RecursiveRegexIterator::GET_MATCH);

foreach ($regex as $key => $file) {
    // This is the important part!
    opcache_compile_file($file[0]);
}
```

{{< note >}}
Preloading all `.php` files may not be optimal for your application, and may even introduce errors.  Your application framework may provide recommendations or a pre-made presload script to use instead.  Determining an optimal preloading strategy is the user's responsibility.
{{< /note >}}

## FFI

PHP 7.4 introduced support for Foreign Function Interfaces (FFI), which allows user-space code to bridge to existing C-ABI-compatible libraries.  FFI is fully supported on Platform.sh.

Note: FFI is only intended for advanced use cases, and is rarely a net win for routine web requests.  Use with caution.

There are a few steps to leveraging FFI:

1. Enable the FFI extension in `.platform.app.yaml`:

    ```yaml
    runtime:
        extensions:
            - ffi
   ```

2. Specify a [preload file](#opcache-preloading) in which you can call `FFI::load()`.  Using `FFI::load()` in preload will be considerably faster than loading the linked library on each request or script run.

3. Ensure the library is available locally, but not in a web-accessible directory.  `.so` files may included in your repository, downloaded i your build hook, or compiled in your build hook.  If compiling C code, `gcc` is available by default.  If compiling Rust code, you can download the [Rust compiler in the build hook](https://doc.rust-lang.org/stable/book/ch01-01-installation.html).

4. For running FFI from the command line, you will need to enable the opcache for command line scripts in addition to the preloader.  The standard pattern for the command would be `php -d opcache.preload="your-preload-script.php" -d opcache.enable_cli=true your-cli-script.php`.

A working [FFI example](https://github.com/platformsh-examples/php-ffi) is available online for both C and Rust.

## Debug PHP-FPM

If you want to inspect what's going on with PHP-FPM, you can install this [small CLI](https://github.com/wizaplace/php-fpm-status-cli):

```yaml
dependencies:
    php:
        wizaplace/php-fpm-status-cli: "^1.0"
```

Then when you are connected to your project over SSH, you can run:

```shell
$ php-fpm-status --socket=unix://$SOCKET --path=/-/status --full
```

## Accessing services

To access various [services](/configuration/services/_index.md) with PHP, see the following examples.  The individual service pages have more information on configuring each service.

{{< codetabs >}}

---
title=Elasticsearch
file=static/files/fetch/examples/php/elasticsearch
highlight=php
markdownify=false
---

<--->

---
title=Memcached
file=static/files/fetch/examples/php/memcached
highlight=php
markdownify=false
---

<--->

---
title=MongoDB
file=static/files/fetch/examples/php/mongodb
highlight=php
markdownify=false
---

<--->

---
title=MySQL
file=static/files/fetch/examples/php/mysql
highlight=php
markdownify=false
---

<--->

---
title=PostgreSQL
file=static/files/fetch/examples/php/postgresql
highlight=php
markdownify=false
---

<--->

---
title=RabbitMQ
file=static/files/fetch/examples/php/rabbitmq
highlight=php
markdownify=false
---

<--->

---
title=Redis
file=static/files/fetch/examples/php/redis
highlight=php
markdownify=false
---

<--->

---
title=Solr
file=static/files/fetch/examples/php/solr
highlight=php
markdownify=false
---

{{< /codetabs >}}


## Runtime configuration

It is possible to change the PHP-FPM runtime configuration via the `runtime` block on your `.platform.app.yaml`. The PHP-FPM options below are configurable:

* `request_terminate_timeout` - The timeout for serving a single request after which the PHP-FPM worker process will be killed.  That is separate from the PHP runtime's `max_execution_time` ini option, which is preferred.  This option may be used if the PHP process is dying without cleaning up properly and causing the FPM process to hang.

    ```yaml
    runtime:
        request_terminate_timeout: 300
    ```

## Project templates

A number of project templates for major PHP applications are available on GitHub. Not all of them are proactively maintained but all can be used as a starting point or reference for building your own website or web application.

{{< repolist lang="php" >}}
