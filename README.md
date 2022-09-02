Caddy Module: http.handlers.cache
================================

This is a distributed HTTP cache module for Caddy based on [Souin](https://github.com/darkweak/souin) cache.  Since it is hosted under github.com/caddyserver it is the "official" cache handler.   

## Features

 * Supports both local and distributed storage options:  [Badger](https://dgraph.io/badger) and [Olric](https://olric.io/).
 * Supports multiple caches, which can be conditionally invoked using [Caddy's Matchers](https://caddyserver.com/docs/caddyfile/matchers#syntax).
 * REST API to purge the cache and list stored resources.
 * Prometheus API.
 * [RFC 7234](https://httpwg.org/specs/rfc7234.html) compliant HTTP Cache.
 * Sets [the `Cache-Status` HTTP Response Header](https://httpwg.org/http-extensions/draft-ietf-httpbis-cache-header.html)
 * CDN interface provided by [Souin](https://github.com/Darkweak/Souin) . 

 
## Installation
The easiest way to install it would be to first install caddy, and then add in the plug in with the command

caddy add-package  github.com/caddyserver/cache-handler

It worked for me, but as of September 2022,  when you type 

`caddy`

this is listed as an experimental command,  so the other way to install it is with the xcaddy command. 

xcaddy build v2.5.2  --with github.com/caddyserver/cache-handler

Here you can read more  [ about using xcaddy](https://github.com/caddyserver/xcaddy). 

But just remember that when you use your operating system tools to install caddy, they may also create a service.  So if you use the xcaddy tool to create the executable, copy it onto the existing executable, but also completely stop and restart the service. 

## Configuration

There is not much point in using caching to serve files, the operating system is already pretty good at caching.   It may even be discouraged.  Usually caching is used to serve from a reverse proxy server.  So here is the simplest reverse proxy caddyfile with caching enabled.  

```
{
  order cache before reverse_proxy 

}

your-domain.com {
  cache {
    ttl 10s
  }
  reverse_proxy http://localhost:8084/
}
```
Remember to change the domain name and port number. 
Once you get the basics working it should be reasonably easy to add in the other options. [Here is a longer example](https://github.com/caddyserver/cache-handler/blob/master/Caddyfile).
And remember to actually test that caching is improving your performance. 


Here is the full configuration.
```
{
    order cache before rewrite
    log {
        level debug
    }
    cache {
        allowed_http_verbs GET POST PATCH
        api {
            basepath /some-basepath
            prometheus
            souin {
                basepath /souin-changed-endpoint-path
            }
        }
        badger {
            path the_path_to_a_file.json
        }
        cdn {
            api_key XXXX
            dynamic
            email darkweak@protonmail.com
            hostname domain.com
            network your_network
            provider fastly
            strategy soft
            service_id 123456_id
            zone_id anywhere_zone
        }
        headers Content-Type Authorization
        log_level debug
        olric {
            url url_to_your_cluster:3320
            path the_path_to_a_file.yaml
            configuration {
                # Your badger configuration here
            }
        }
        regex {
            exclude /test2.*
        }
        stale 200s
        ttl 1000s
        default_cache_control no-store
    }
}

:4443
respond "Hello World!"

@match path /test1*
@match2 path /test2*
@matchdefault path /default
@souin-api path /souin-api*

cache @match {
    ttl 5s
    badger {
        path /tmp/badger/first-match
        configuration {
            # Required value
            ValueDir <string>

            # Optional
            SyncWrites <bool>
            NumVersionsToKeep <int>
            ReadOnly <bool>
            Compression <int>
            InMemory <bool>
            MetricsEnabled <bool>
            MemTableSize <int>
            BaseTableSize <int>
            BaseLevelSize <int>
            LevelSizeMultiplier <int>
            TableSizeMultiplier <int>
            MaxLevels <int>
            VLogPercentile <float>
            ValueThreshold <int>
            NumMemtables <int>
            BlockSize <int>
            BloomFalsePositive <float>
            BlockCacheSize <int>
            IndexCacheSize <int>
            NumLevelZeroTables <int>
            NumLevelZeroTablesStall <int>
            ValueLogFileSize <int>
            ValueLogMaxEntries <int>
            NumCompactors <int>
            CompactL0OnClose <bool>
            LmaxCompaction <bool>
            ZSTDCompressionLevel <int>
            VerifyValueChecksum <bool>
            EncryptionKey <string>
            EncryptionKey <Duration>
            BypassLockGuard <bool>
            ChecksumVerificationMode <int>
            DetectConflicts <bool>
            NamespaceOffset <int>
        }
    }
}

cache @match2 {
    ttl 50s
    badger {
        path /tmp/badger/second-match
        configuration {
            ValueDir match2
            ValueLogFileSize 16777216
            MemTableSize 4194304
            ValueThreshold 524288
            BypassLockGuard true
        }
    }
    headers Authorization
    default_cache_control "public, max-age=86400"
}

cache @matchdefault {
    ttl 5s
    badger {
        path /tmp/badger/default-match
        configuration {
            ValueDir default
            ValueLogFileSize 16777216
            MemTableSize 4194304
            ValueThreshold 524288
            BypassLockGuard true
        }
    }
}

cache @souin-api {}
```
What does these directives mean?  
|  Key                      |  Description                                                                                                                                 |  Value example                                                                                                          |
|:--------------------------|:---------------------------------------------------------------------------------------------------------------------------------------------|:------------------------------------------------------------------------------------------------------------------------|
| `allowed_http_verbs`      | The HTTP verbs allowed to be cached                                                                                                          | `GET POST PATCH`<br/><br/>`(default: GET HEAD)`                                                                         |
| `api`                     | The cache-handler API cache management                                                                                                       |                                                                                                                         |
| `api.basepath`            | BasePath for all APIs to avoid conflicts                                                                                                     | `/your-non-conflict-route`<br/><br/>`(default: /souin-api)`                                                             |
| `api.prometheus`          | Enable the Prometheus metrics                                                                                                                |                                                                                                                         |
| `api.souin.basepath`      | Souin API basepath                                                                                                                           | `/another-souin-api-route`<br/><br/>`(default: /souin)`                                                                 |
| `badger`                  | Configure the Badger cache storage                                                                                                           |                                                                                                                         |
| `badger.path`             | Configure Badger with a file                                                                                                                 | `/anywhere/badger_configuration.json`                                                                                   |
| `badger.configuration`    | Configure Badger directly in the Caddyfile or your JSON caddy configuration                                                                  | [See the Badger configuration for the options](https://dgraph.io/docs/badger/get-started/)                              |
| `cdn`                     | The CDN management, if you use any cdn to proxy your requests Souin will handle that                                                         |                                                                                                                         |
| `cdn.provider`            | The provider placed before Souin                                                                                                             | `akamai`<br/><br/>`fastly`<br/><br/>`souin`                                                                             |
| `cdn.api_key`             | The api key used to access to the provider                                                                                                   | `XXXX`                                                                                                                  |
| `cdn.dynamic`             | Enable the dynamic keys returned by your backend application                                                                                 | `(default: false)`                                                                                                      |
| `cdn.email`               | The api key used to access to the provider if required, depending the provider                                                               | `XXXX`                                                                                                                  |
| `cdn.hostname`            | The hostname if required, depending the provider                                                                                             | `domain.com`                                                                                                            |
| `cdn.network`             | The network if required, depending the provider                                                                                              | `your_network`                                                                                                          |
| `cdn.strategy`            | The strategy to use to purge the cdn cache, soft will keep the content as a stale resource                                                   | `hard`<br/><br/>`(default: soft)`                                                                                       |
| `cdn.service_id`          | The service id if required, depending the provider                                                                                           | `123456_id`                                                                                                             |
| `cdn.zone_id`             | The zone id if required, depending the provider                                                                                              | `anywhere_zone`                                                                                                         |
| `default_cache_control`   | Set the default value of `Cache-Control` response header if not set by upstream (Souin treats empty `Cache-Control` as `public` if omitted)  | `no-store`                                                                                                              |
| `headers`                 | List of headers to include to the cache                                                                                                      | `Authorization Content-Type X-Additional-Header`                                                                        |
| `olric`                   | Configure the Olric cache storage                                                                                                            |                                                                                                                         |
| `olric.path`              | Configure Olric with a file                                                                                                                  | `/anywhere/badger_configuration.json`                                                                                   |
| `olric.configuration`     | Configure Olric directly in the Caddyfile or your JSON caddy configuration                                                                   | [See the Badger configuration for the options](https://github.com/buraksezer/olric/blob/master/cmd/olricd/olricd.yaml/) |
| `regex.exclude`           | The regex used to prevent paths being cached                                                                                                 | `^[A-z]+.*$`                                                                                                            |
| `stale`                   | The stale duration                                                                                                                           | `25m`                                                                                                                   |
| `ttl`                     | The TTL duration                                                                                                                             | `120s`                                                                                                                  |
| `log_level`               | The log level                                                                                                                                | `One of DEBUG, INFO, WARN, ERROR, DPANIC, PANIC, FATAL it's case insensitive`                                           |

Other resources
---------------
You can find an example for the [Caddyfile](Caddyfile) or the [JSON file](configuration.json).  
See the [Souin](https://github.com/darkweak/souin) configuration for the full configuration, and its associated [Caddyfile](https://github.com/darkweak/souin/blob/master/plugins/caddy/Caddyfile)  

## TODO

* [ ] Improve the API and add relevant endpoints
