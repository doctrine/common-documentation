Doctrine  Caching
=================

The Doctrine Common caching library was born from a need in the
`Doctrine2 ORM <http://www.doctrine-project.org/projects/orm>`_ to
allow caching of result sets. The library is independent and can be
used in your own libraries to caching.

Introduction
------------

Doctrine caching provides a very simple interface on top of which
several out of the box implementations are provided:

- ApcCache (requires ext/apc)
- ArrayCache (in memory, lifetime of the request)
- FilesystemCache (not optimal for high concurrency)
- MemcacheCache (requires ext/memcache)
- MemcachedCache (requires ext/memcached)
- PhpFileCache (not optimal for high concurrency)
- RedisCache.php (requires ext/phpredis)
- WinCacheCache.php (requires ext/wincache)
- XcacheCache.php (requires ext/xcache)
- ZendDataCache.php  (requires Zend Server Platform)

.. code-block :: php

    <?php

    $cache = new \Doctrine\Common\Cache\ArrayCache();
    $id = $cache->fetch("some key");
    if (!$id) {
        $id = do_something();
        $cache->save("some key", $id);
    }
