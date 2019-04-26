---
layout: post
title:  "Concurrency, Perl and Web Services, Oh My!"
date:   2017-07-28 00:00:00
description: "Building concurrent web services in Perl"
---
The majority of the systems I develop - both in my spare time and at work - are heavily IO-based; connecting various web services, databases and files on disk together in some fashion. In those particular systems, I’d be stupid not to promote a concurrency-based solution.

Lately I’ve spent a lot of time developing small web services (or microservices or whatever it’s called this year), both as brand new systems and also as a way of shaving off parts of large monoliths into smaller chunks to auto-scale.

There are many ways to design the architecture for this kind of system and its subsystems, and often there are instances where pre-forking a bunch of worker processes to handle requests is either too resource-hungry or just not appropriate for the work it’s doing.

I want to write an example web service, but I’m sick of seeing the same, “Hello World”-esque web services that can be written in a dozen lines of code that no way represent any web service that anyone ever has written. I want to write an application that can actually benefit from a concurrent solution and semi-resembles a real-world thing. So I’ve got an example:

1. HTTP-based web service

1. Runs in a single process

1. Accepts a domain name, and returns the geographic locations of the domain’s mail servers, in JSON format

To satisfy the first two criteria, a [Plack/PSGI](http://plackperl.org/) application running out of either [Twiggy](https://metacpan.org/pod/Twiggy) or [Feersum](https://metacpan.org/pod/Feersum) should do just fine.

In order to satisfy the last point (i.e. the actual functionality), the app needs to perform a few steps:

1. Retrieve the mail servers of the domain via a DNS lookup. [AnyEvent::DNS](https://metacpan.org/pod/AnyEvent::DNS) can do this.

1. For each of the mail servers, resolve the IP addresses via another DNS lookup. AnyEvent::DNS to the rescue again.

1. For each of the IP addresses, I’m going to use the [IP Vigilante](https://www.ipvigilante.com/) API to retrieve the geographic location data. There are no modules on CPAN for the IP Vigilante service, so I’ll need to write something. [AnyEvent::HTTP](https://metacpan.org/pod/AnyEvent::HTTP) would work just fine here, but lately I prefer to use [Mojo::UserAgent](https://metacpan.org/pod/Mojo::UserAgent) where possible, because it’s much more versatile, e.g. providing a proper request/response object for us, and handling JSON responses.

There’s a fairly straight-forward sequence of operations to perform, so I’m going to use a [Promises](https://metacpan.org/pod/Promises)-based approach. I’ve found that this makes concurrent Perl code much easier to follow, especially when other developers need to jump in and understand and maintain it. There’s no real reason for why I’ve settled on Promises over [Futures](https://metacpan.org/pod/Future) (or any other implementation of these two patterns); either will do just fine.

Firstly, I need a function that can lookup the MX records for a single domain and return an arrayref of addresses (via a promise).

```perl
sub lookup_mx {
	my ($domain) = @_;
	AE::log trace => "lookup_mx($domain)";

	my $d = Promises::deferred;

	AnyEvent::DNS::mx $domain, sub {
		my (@addrs) = @_;

		if (@addrs) {
			$d->resolve(\@addrs);
			return;
		}

		$d->reject("unable to perform MX lookup of $domain");
	};

	return $d->promise;
}
```

This is actually a pretty boring function to look at.

Next, I need a function that can resolve a domain name to an IP address (or addresses).

```perl
sub resolve_addr {
	my ($domain) = @_;
	AE::log trace => "resolve_addr($domain)";

	my $d = Promises::deferred;

	AnyEvent::DNS::a $domain, sub {
		my (@addrs) = @_;

		if (@addrs) {
			$d->resolve(\@addrs);
			return;
		}

		$d->reject("unable to resolve $domain");
	};

	return $d->promise;
}
```

This is also a pretty boring function.

Now I need a function that can perform a lookup to the IP Vigilante service for a single IP address and return an arrayref containing the continent, country and city for which it resides.

```perl
my $ua = Mojo::UserAgent->new->max_redirects(5);

sub ipvigilante {
	my ($address) = @_;
	AE::log trace => "ipvigilante($address)";

	my $d = Promises::deferred;
	my $url = sprintf "https://ipvigilante.com/json/%s", $address;

	$ua->get($url, sub {
		my ($ua, $tx) = @_;
		if ($tx->res->is_success) {
			my $json = $tx->res->json;
			my $rv = [
				$json->{data}->{continent_name},
				$json->{data}->{country_name},
				$json->{data}->{city_name},
			];
			$d->resolve($rv);
			return;
		}
		$d->reject( $tx->res->error );
	} );

	return $d->promise;
}
```

This function is slightly more interesting - it receives a JSON response from IP Vigilante - but, in the end, is still fairly boring, since Mojo::UserAgent handles all of it for us.

The next function will need to take an arrayref of IP addresses, and collate the IP Vigilante data into a hashref, for which the keys will be the IP addresses and the values will be the IP Vigilante information from the previous function.

```perl
sub get_ip_informations {
	my ($ips) = @_;

	my $d = Promises::deferred;

	my %rv;
	Promises::collect( map {
			my $ip = $_;
			ipvigilante($ip)
				->then( sub {
					my ($ip_info) = @_;
					$rv{$ip} = $ip_info;
				} )
			} @$ips )
		->then( sub { $d->resolve(\%rv) } )
		->catch( sub { $d->reject(@_) } );

	return $d->promise;
}
```

This is the first note-worthy function, and it’s still not that big of a function. The call to the `ipvigilante()->then()` chain will return a new promise, and we have used map and the `Promises::collect()` function to collate the results of multiple promises. This means that if we are trying to get the IP information for 10 addresses, the map will return 10 promises, and for this function to return a result, we need the response from all 10 promises. The entire batch executes concurrently and only runs as slow as the slowest IP Vigilante lookup. Yay concurrency!

Lastly, I need a function that will take an arrayref of domain names, resolve each domain to its IP address(es) and get the IP Vigilante information for each IP address (via the previous function) and return it as a hashref.

```perl
sub get_mx_informations {
	my ($addrs) = @_;

	my $d = Promises::deferred;

	my %rv;
	Promises::collect( map {
				my $mx = $_;
				resolve_addr($mx)
					->then( sub { get_ip_informations($_[0]) } )
					->then( sub { $rv{$mx} = $_[0] } );
			} @$addrs )
		->then( sub { $d->resolve(\%rv) } )
		->catch( sub { $d->reject(@_) } );

	return $d->promise;
}
```

This function is basically the bulk of the application.

I feel like these last two functions shouldn’t be necessary and they rub me the wrong way a little, as they’re essentially just for-loops where the inside of the loop has already been put into another function, but for the purposes of maintainability and testability, I kept them.

The beauty about all of the code written so far is that because Promises, AnyEvent and Mojo all integrate with the lower-level [EV](https://metacpan.org/pod/EV) event loop, and in some cases with each other, everything works together. This makes it simple to mix and match your favorite libraries that were originally written for different frameworks.

The whole thing just needs to be wrapped in a Plack/PSGI application.

```perl
my $app = sub {
	my ($env) = @_;
	my $request = Plack::Request->new($env);

	if ($request->method ne 'GET') {
		return [ 400, [], [] ];
	}

	(my $domain = $request->path_info) =~ s{^/}{};

	if (not $domain) {
		return [
			400,
			[ 'Content-Type' => 'application/json' ],
			[ Mojo::JSON::encode_json( { error => 'domain required' } ) ]
		];
	}

	return sub {
		my ($responder) = @_;
		my $response = Plack::Response->new;

		lookup_mx($domain)
			->then( sub { get_mx_informations($_[0]) } )
			->then( sub {
					my ($mx_informations) = @_;
					$response->status(200);
					return { $domain => $mx_informations };
				} )
			->catch( sub {
					my ($error) = @_;
					$response->status(400);
					return { error => $error };
				} )
			->finally( sub {
					my ($json) = @_;
					$response->headers( [
						'Content-Type' => 'application/json'
					] );
					$response->body( Mojo::JSON::encode_json($json) );
					$responder->( $response->finalize )
				} );
	}
};
```

I’m going to use [Carton](https://metacpan.org/pod/Carton) to handle and bundle the dependencies. This step isn’t absolutely necessary, but when deploying Perl applications across many machines in a production environment, it’s a solid tool for keeping things consistent across the board. Not having a solution for this is a *massive* headache once many different pieces of code have been deployed again and again for a few years. The [Carton FAQ](https://metacpan.org/pod/Carton::Doc::FAQ) has a good rundown of its use-cases. I now need to declare my immediate dependencies in a new file - for Carton to consume - cpanfile.

```perl
requires 'Plack';
requires 'Feersum';
requires 'AnyEvent';
requires 'IO::Socket::SSL';
requires 'Mojolicious';
requires 'Promises';
```

I’m not tied down to specific versions of any of these modules.

The last step is to - with the help of carton - install the dependencies, which will also generate a snapshot file with all my dependencies’ dependicies, and then run the server.

    $ carton install
    $ carton exec -- feersum --listen :5000 -a mx.psgi

… and in another shell …

    $ curl -s http://localhost:5000/nab.com.au
    {
      "nab.com.au": {
        "cust23659-2-in.mailcontrol.com": {
          "116.50.58.190": [
            "Oceania",
            "Australia",
            null
          ]
        },
        "cust23659-1-in.mailcontrol.com": {
          "116.50.59.190": [
            "Asia",
            "India",
            null
          ]
        }
      }
    }

The [full code is available](https://gist.github.com/ltriant/85850694c7234722ff4d9d16900f3d56) on github.

This is beyond the original scope of this post, but there’s still a lot more to do. The application is just barely in an acceptable state. There are a number of extra steps before this can/should be deployed to production, for which I may write follow-up posts:

1. Unit tests. The application and functions should be moved into its own package in order to have unit tests written against it. I’ve had great success using [Plack::Test](https://metacpan.org/pod/Plack::Test) to test Plack applications and [Test::MockObject::Extends](https://metacpan.org/pod/Test::MockObject::Extends) to mock functions that would perform network calls, so that I don’t require a working internet connection to run unit tests.

1. Logging. Self-explanatory (I hope).

1. Rate limiting the ipvigilante.com API requests. I don’t want the service to inundate IP Vigilante with tons of connections/requests at the same time.

1. Dealing with ipvigilante.com failures. The circuitbreaker pattern will help the service remain stable and not constantly hit a remote service which is having an outage.

1. Caching. IP addresses aren’t likely to move geographic locations very often (if at all), so caching the IP Vigilante responses will be of great benefit. Either a simple local cache with [Cache::FastMmap](https://metacpan.org/pod/Cache::FastMmap), or perhaps with a remote cache in [Cache::Memcached](https://metacpan.org/pod/Cache::Memcached), if I end up with a cluster of servers - which are an auto-scaling group - and I want a centralised cache for all hosts to use.

1. Monitoring. How long do DNS lookups take? How long do ipvigilante.com API requests take? How often do they fail? When they fail, do they fail fast or do they timeout after 5 minutes of waiting?

1. There’s probably more...
