*redis* enables reading zone data from redis database.
this plugin should be located right next to *etcd* in *plugins.cfg*

## syntax

~~~
redis
~~~

redis loads authoritative zones from redis server

Address will default to local redis server (localhost:6379)
~~~
redis {
    address ADDR
    password PWD
    db     REDIS_DATABASE
    prefix PREFIX
    suffix SUFFIX
    connect_timeout TIMEOUT
    read_timeout TIMEOUT
    ttl TTL
}
~~~

* `address` is redis server address to connect in the form of *host:port* or *ip:port*.
* `password` is redis server *auth* key
* `db` specifies the database to select when dialing a connection.
* `connect_timeout` time in ms to wait for redis server to connect
* `read_timeout` time in ms to wait for redis server to respond
* `ttl` default ttl for dns records, 300 if not provided
* `prefix` add PREFIX to all redis keys
* `suffix` add SUFFIX to all redis keys

## examples

~~~ corefile
. {
    redis example.com {
        address localhost:6379
        password foobared
        db 2
        connect_timeout 100
        read_timeout 100
        ttl 360
        prefix _dns:
    }
}
~~~

## reverse zones

reverse zones is not supported yet

## proxy

proxy is not supported yet

## zone format in redis db

### zones

each zone is stored in redis as a hash map with *zone* as key

~~~
redis-cli>KEYS *
1) "example.com."
2) "example.net."
redis-cli>
~~~

### dns RRs 

dns RRs are stored in redis as json strings inside a hash map using address as field key.
*@* is used for zone's own RR values.

#### A

~~~json
{
    "a":{
        "ip" : "1.2.3.4",
        "ttl" : 360
    }
}
~~~

#### AAAA

~~~json
{
    "aaaa":{
        "ip" : "::1",
        "ttl" : 360
    }
}
~~~

#### CNAME

~~~json
{
    "cname":{
        "host" : "x.example.com.",
        "ttl" : 360
    }
}
~~~

#### TXT

~~~json
{
    "txt":{
        "text" : "this is a text",
        "ttl" : 360
    }
}
~~~

#### NS

~~~json
{
    "ns":{
        "host" : "ns1.example.com.",
        "ttl" : 360
    }
}
~~~

#### MX

~~~json
{
    "mx":{
        "host" : "mx1.example.com",
        "priority" : 10,
        "ttl" : 360
    }
}
~~~

#### SRV

~~~json
{
    "srv":{
        "host" : "sip.example.com.",
        "port" : 555,
        "priority" : 10,
        "weight" : 100,
        "ttl" : 360
    }
}
~~~

#### SOA

~~~json
{
    "soa":{
        "ttl" : 100,
        "mbox" : "hostmaster.example.com.",
        "ns" : "ns1.example.com.",
        "refresh" : 44,
        "retry" : 55,
        "expire" : 66
    }
}
~~~

#### CAA

~~~json
{
    "caa":{
        "flag" : 0,
        "tag" : "issue",
        "value" : "letsencrypt.org"
    }
}
~~~

#### example

~~~
$ORIGIN example.net.
 example.net.                 300 IN  SOA   <SOA RDATA>
 example.net.                 300     NS    ns1.example.net.
 example.net.                 300     NS    ns2.example.net.
 *.example.net.               300     TXT   "this is a wildcard"
 *.example.net.               300     MX    10 host1.example.net.
 sub.*.example.net.           300     TXT   "this is not a wildcard"
 host1.example.net.           300     A     5.5.5.5
 _ssh.tcp.host1.example.net.  300     SRV   <SRV RDATA>
 _ssh.tcp.host2.example.net.  300     SRV   <SRV RDATA>
 subdel.example.net.          300     NS    ns1.subdel.example.net.
 subdel.example.net.          300     NS    ns2.subdel.example.net.
 host2.example.net                    CAA   0 issue "letsencrypt.org"
~~~

above zone data should be stored at redis as follow:

~~~
redis-cli> hgetall example.net.
 1) "_ssh._tcp.host1"
 2) "{\"srv\":[{\"ttl\":300, \"target\":\"tcp.example.com.\",\"port\":123,\"priority\":10,\"weight\":100}]}"
 3) "*"
 4) "{\"txt\":[{\"ttl\":300, \"text\":\"this is a wildcard\"}],\"mx\":[{\"ttl\":300, \"host\":\"host1.example.net.\",\"preference\": 10}]}"
 5) "host1"
 6) "{\"a\":[{\"ttl\":300, \"ip\":\"5.5.5.5\"}]}"
 7) "sub.*"
 8) "{\"txt\":[{\"ttl\":300, \"text\":\"this is not a wildcard\"}]}"
 9) "_ssh._tcp.host2"
10) "{\"srv\":[{\"ttl\":300, \"target\":\"tcp.example.com.\",\"port\":123,\"priority\":10,\"weight\":100}]}"
11) "subdel"
12) "{\"ns\":[{\"ttl\":300, \"host\":\"ns1.subdel.example.net.\"},{\"ttl\":300, \"host\":\"ns2.subdel.example.net.\"}]}"
13) "@"
14) "{\"soa\":{\"ttl\":300, \"minttl\":100, \"mbox\":\"hostmaster.example.net.\",\"ns\":\"ns1.example.net.\",\"refresh\":44,\"retry\":55,\"expire\":66},\"ns\":[{\"ttl\":300, \"host\":\"ns1.example.net.\"},{\"ttl\":300, \"host\":\"ns2.example.net.\"}]}"
15) "host2"
16)"{\"caa\":[{\"flag\":0, \"tag\":\"issue\", \"value\":\"letsencrypt.org\"}]}"
redis-cli>
~~~
