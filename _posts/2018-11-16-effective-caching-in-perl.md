---
layout: post
title:  "Effective Caching in Perl"
date:   2018-11-16 00:00:00 +1100
description: "Implementing a caching layer in a Perl application using some of CPAN's rock-solid modules"
---
Caching strategies, cache invalidation, and everything else caching related is hard to get right, especially as systems become more and more liberal in the data that is cached, and more variable in the retention rules.

This isn’t a guide on caching and caching patterns, but sheds light on some of the excellent options that are available in the Perl ecosystem.

## Problem

I’m going to use a totally useless web service as an example; MD5-as-a-Service. All it does is take a word, calculate the MD5 sum, and returns the checksum in a JSON response.

*[edit] Note: This is not a realistic web service; it’s just an example, purely for the purpose of the blog post. Calculating MD5 checksums seemed more fun than sleeping for 5 seconds.*

```perl
#!/usr/bin/env perl

use warnings;
use strict;

use Digest::MD5 qw(md5_hex);
use Mojolicious::Lite;

post '/:word' => sub {
	my ($c) = @_;
	my $word = $c->param('word');

	my $digest = md5_hex($word);

	$c->render(json => {
		word   => $word,
		digest => $digest,
	} );
};

app->start;
```

The service can be served out of [Starman](https://metacpan.org/pod/Starman), which is a pre-forking web server, defaulting to 5 workers.

```
$ starman --listen :5000 -a app.psgi
2018/11/13-20:32:40 Starman::Server (type Net::Server::PreFork) starting! pid(92029)
Resolved [*]:5000 to [0.0.0.0]:5000, IPv4
Binding to TCP port 5000 on host 0.0.0.0 with IPv4
Setting gid to "20 20 20 504 401 12 61 79 80 81 98 33 100 204 395 398 399"
```

And requests can be made with a simple curl command.

```
$ curl -X POST http://localhost:5000/foo
{"digest":"acbd18db4cc2f85cedef654fccc4a4d8","word":"foo"}
```

Simple.

This works for a while. But then, after a period of time, it seems MD5-as-a-Service has gotten popular, and too many precious CPU cycles are being wasted calculating the same checksums over and over again.

## Local Caching

The service only lives on one server at the moment, so some sort of local cache sounds like a good idea. The first tool to grab from the CPAN toolbox is [Cache::FastMmap](https://metacpan.org/pod/Cache::FastMmap).

It’s fairly simple to add.

```perl
#!/usr/bin/env perl

use warnings;
use strict;

use Cache::FastMmap;
use Digest::MD5 qw(md5_hex);
use Mojolicious::Lite;

my $CACHE = Cache::FastMmap->new(
	share_file => '/tmp/md5-perl-caching',
	cache_size => '10m',
);

post '/:word' => sub {
	my ($c) = @_;
	my $word = $c->param('word');

	if (my $digest = $CACHE->get($word)) {
		$c->render(json => {
			from_cache => 1,
			word	   => $word,
			digest	 => $digest,
		} );
	}
	else {
		my $digest = md5_hex($word);
		$CACHE->set($word, $digest);

		$c->render(json => {
			from_cache => 0,
			word	   => $word,
			digest	 => $digest,
		} );
	}
};

app->start;
```

First, it checks if the requested value is in the cache. If it is, it serves the value out of the cache back to the client. Otherwise, it calculates the checksum requested, stores it in the cache, and *then* serves the value back to the client.

This is referred to as the Cache-Aside pattern.

I’ve added an extra key in the JSON response, purely to see whether or not the value came from the cache.

<pre>
$ curl -X POST http://localhost:5000/foo
{"digest":"acbd18db4cc2f85cedef654fccc4a4d8",<strong>"from_cache":0</strong>,"word":"foo"}
$ curl -X POST http://localhost:5000/foo
{"digest":"acbd18db4cc2f85cedef654fccc4a4d8",<strong>"from_cache":1</strong>,"word":"foo"}
</pre>

Excellent!

The best part is that even though Starman is a pre-forked web server, Cache::FastMmap was designed to share the cache between many processes.
> A shared memory cache through an mmap’ed file. It’s core is written in C for performance. It uses fcntl locking to ensure multiple processes can safely access the cache at the same time. It uses a basic LRU algorithm to keep the most used entries in the cache.

When it comes time to tweak the details of the cache to get more performance out of the module, the documentation explains all of the knobs that can be tuned for all of the other caching nerds out there.

## Expiration

The code above initialised the cache with a size of 10MB. If the cache exceeds 10MB, it will expire entries based on a LRU algorithm (as mentioned above in the docs).

That might make sense for the kind of data being cached in this service - because MD5 checksums don’t change no matter how much time passes - but when a system is caching values that *can* change, e.g. values out of a database that represent an organic value, expiring cache items based on a unit of time makes sense.

A simple way to do this for *all items* in the cache is at initialisation.

```perl
my $CACHE = Cache::FastMmap->new(
	expire_time => '3s',
);

$CACHE->set(foo => 'bar');
```

This sets the expiry time at 3 seconds. The value 10m can be used for 10 minutes, 1h for 1 hour, etc.

However, if the cache is storing many different things with different expiration requirements, the expiry can be specified with the call to set.

```perl
$CACHE->set(foo => 'bar', '3s');
```

Alternatively, items can be removed explicitly.

```perl
$CACHE->remove('foo');
```

## A Side Note

If data in the cache goes stale, it’s important to expire the cached data. Other than expiring cached data based on a unit of time, there are a couple of other simple strategies for expiring stale data:

1. When the origin data is updated, delete the old data from the cache, or set its expiry time so that it expires immediately. This will cause the next request to read the data to refresh the cached value, whenever that may happen.

1. When the origin data is updated, the client code will also update the cached value, rather than relying on the web service (or whatever is reading the data) to refresh the cached data. This is a good strategy if there are potentially many readers of the data that may attempt to refresh the cached data at the same time, resulting in the same database query from many different servers at the same time, otherwise known as a [Thundering Herd](https://en.wikipedia.org/wiki/Thundering_herd_problem) problem.

And, for now at least, we’ve saved MD5-as-a-Service; it can continue receiving requests for the same words over and over again, and it will only ever calculate checksums for words it either hasn’t seen recently, or ever at all.

## Remote Caching

Uh oh. It’s getting close to Christmas, and, in the mad Christmas panic, MD5-as-a-Service is seeing many more requests than usual. The server has been given more CPUs and more memory, but it’s still not holding on. It’s time to scale out to a couple more servers.

Moments later, a couple more instances are acquired, a load balancer is thrown in front of them, and everything is fine again.

But there’s a problem. The service is starting to recompute MD5 sums for the same words again. This happens because the cache is local to each server.

## Redis

There are many kinds of remote caching servers. [Redis](https://redis.io/) will be the weapon of choice for this task.

Azure offers [Azure Cache for Redis](https://azure.microsoft.com/en-au/services/cache/), a hosted and fully managed Redis service. AWS has the [Elasticache](https://aws.amazon.com/elasticache/) service, which offers both Redis *and* memcached. And Google Cloud has [MemoryStore](https://cloud.google.com/memorystore/), which is a managed Redis service.

## Cache Handling Interface (CHI)

[CHI](https://metacpan.org/pod/CHI) is an awesome module for building and using caching modules. It’s a facade from which any caching module can integrate with.
> CHI provides a unified caching API, designed to assist a developer in persisting data for a specified period of time.
> The CHI interface is implemented by driver classes that support fetching, storing and clearing of data. Driver classes exist or will exist for the gamut of storage backends available to Perl, such as memory, plain files, memory mapped files, memcached, and DBI.

That way you only need to remember one API, and you can use any CHI driver. Obviously, different caching mechanisms have different quirks, but the code remains the same.

There are a few Redis caching modules, and a couple of them provide CHI drivers. I’ve had good experiences with [CHI::Driver::Redis](https://metacpan.org/pod/CHI::Driver::Redis).

Integrating it is still pretty simple. The only thing that changes is the initialisation of the cache object. Everything else remains the same.

```perl
#!/usr/bin/env perl

use warnings;
use strict;

use CHI;
use Digest::MD5 qw(md5_hex);
use Mojolicious::Lite;

my $CACHE = CHI->new(
	driver	=> 'Redis',
	namespace => 'md5_as_a_service',
	server	=> 'cache_server:6379',
	debug	 => 0,
	l1_cache  => {
		driver	 => 'FastMmap',
		share_file => "/tmp/md5-perl-caching",
		cache_size => '10m',
	},
);

post '/:word' => sub {
	my ($c) = @_;
	my $word = $c->param('word');

	if (my $digest = $CACHE->get($word)) {
		$c->render(json => {
			from_cache => 1,
			word	   => $word,
			digest	 => $digest,
		} );
	}
	else {
		my $digest = md5_hex($word);
		$CACHE->set($word, $digest);

		$c->render(json => {
			from_cache => 0,
			word	   => $word,
			digest	 => $digest,
		} );
	}
};

app->start;
```

What’s the `l1_cache` thing and why is it referencing the FastMmap driver? Well, a super handy feature of the CHI module is that it can use a level 1 cache, which will be checked *before* querying the main cache driver. And the L1 cache is simply another CHI driver.

The driver being used, FastMmap, refers to the CHI::Driver::FastMmap module, which uses Cache::FastMmap under the hood.
> On a get, the L1 cache is checked first - if a valid value exists, it is returned. Otherwise, the primary cache is checked - if a valid value exists, it is returned, and the value is placed in the L1 cache with the same expiration time. In this way, items fetched most frequently from the primary cache will tend to be in the L1 cache.
> set operations are distributed to both the primary and L1 cache.

This means the service gains all the benefits of using the local cache (lower latency) but it also gains the ability to use the remote Redis cache, so that it can share the cached results from previous requests from other service instances.

To test this out, I’ve setup two services running with a single Redis instance.

1. The first instance is listening on port 5000

1. The second instance is listening on port 5001

1. The Redis instance is listening on the default Redis port 6379

When I query both of the services, the first request finds nothing in the cache, but the second request to the other instance *does* find something in the cache.

<pre>
$ curl -X POST http://<strong>localhost:5000</strong>/foo
{"digest":"acbd18db4cc2f85cedef654fccc4a4d8",<strong>"from_cache":0</strong>,"word":"foo"}
$ curl -X POST http://<strong>localhost:5001</strong>/foo
{"digest":"acbd18db4cc2f85cedef654fccc4a4d8",<strong>"from_cache":1</strong>,"word":"foo"}
</pre>

And Redis does, in fact, have the value we see:

<pre>
$ redis-cli
127.0.0.1:6379> <strong>get md5_as_a_service||foo</strong>
"\xa9,\xee[\xff\xff\xff\xff\xff\xff\xff\xff\x00\x01<strong>acbd18db4cc2f85cedef654fccc4a4d8</strong>"
</pre>

A couple of things to note:

1. The `md5_as_a_service||` prefix is not a Redis-specific thing, but is how CHI::Driver::Redis implements namespaces, because Redis doesn’t have namespaces.

1. The extra bytes in front of the checksum is a [pack](http://perldoc.perl.org/functions/pack.html)ed string made up of internal CHI-related data, including the creation time and the expire time.

Finally! MD5-as-a-Service is ready in time for the Christmas rush!

There are a variety of caching options in Perl. The CHI module offers a very simple interface to caching, with many, very solid driver options; Cache::FastMmap, Redis, and their CHI drivers have been very dependable modules in my experience.

CHI also offers more fine-grained control, for when it’s time to get more complicated. [The CHI docs](https://metacpan.org/pod/CHI) are very well laid out, and are filled with useful examples.

## Resources

* [https://github.com/ltriant/md5-as-a-service](https://github.com/ltriant/md5-as-a-service) - MD5-as-a-Service example code and Docker files

* [https://codeahoy.com/2017/08/11/caching-strategies-and-how-to-choose-the-right-one/](https://codeahoy.com/2017/08/11/caching-strategies-and-how-to-choose-the-right-one/)
