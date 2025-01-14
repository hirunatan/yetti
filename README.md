# Jetty (10) adapter for Ring

[Ring](https://github.com/ring-clojure/ring) adapter for [Jetty
(10)](https://www.eclipse.org/jetty/), with
HTTP2C and WebSocket support.

This package is as fork of [rj9a][1]. This fork is focused on have a
cleaned API (no need to maintain backward compatibility with previous
versions of [rj9a][1]) and focused on performance.

This adapter is intended to be used under a http proxy for TLS
offloding (such that NGINX or HAPROXY); this is the reason there are
no options for configure SSL/TLS. This is a practical decission to not
maintain code that is almost never used on our own use cases; and if
you really need TLS handled from the JVM you can always use the
[rj9a][1] or the default ring jetty adapter.

It requires JDK >= 11.

[1]: https://github.com/sunng87/ring-jetty9-adapter

Many thanks to @sunng87 for all the work on the original [rj9a][1]
adatper.


## Usage

### Reference

See official [Ring documentation](https://github.com/ring-clojure/ring#documentation).


### Quick Start

On deps.edn:

```clojure
funcool/yetti {:git/tag "v4.0" :git/sha "59ed2a7"
               :git/url "https://github.com/funcool/yetti.git"}
```

In the REPL:

```clojure
(require '[yetti.adapter :as yt])

(-> app
    (yt/server {:http/port 11010})
    (yt/start!))
```


### Server Options

Creates and confgures an instance of jetty server. This is a list of options
that you can provide:

```clojure
:ring/async                    ;; enables the ring 1.6 async handler
:ring/version                  ;; specifies the ring version (only supports 1)

:thread-pool/daemon            ;; use daemon threads (defaults to true)
:thread-pool/max-threads       ;; the max number of threads to use (default 200)
:thread-pool/min-threads       ;; the min number of threads to use (default 5)
:thread-pool/idle-timeout      ;; the max idle time in milliseconds for a thread (default 60000)
:thread-pool/job-queue         ;; the job queue to be used by the Jetty threadpool (default is unbounded)
:thread-pool/instance          ;; specifies the thread pool used for jetty workloads. If you specifies
                               ;; this option, all the other will be ignored.

:http/protocols                ;; a set of enabled protocols #{:h1 :h2c :proxy} (defaults to #{:h1 :h2c})
:http/port                     ;; the port to listen on (defaults to 11010)
:http/host                     ;; the hostname to listen on, defaults to 'localhost'
:http/idle-timeout             ;; the max idle time in ms for a connection (default 200000)

:websocket/idle-timeout        ;; the max idle time in ms for a websocket connection (default 500000)
:websocket/max-text-msg-size   ;; the max text message size in bytes for a websocket connection (default 65536)
:websocket/max-binary-msg-size ;; the max binary message size in bytes for a websocket connection (default 65536)

:jetty/wrap-handler            ;; a wrapper fn that wraps default jetty handler into another, default to
                               ;; the `identity` fn, note that it's not a ring middleware
```

### Ring Async

This adapter also supports the ring (>=1.6) async handlers:

```clojure
(require '[yetti.adapter :as yt])

(defn app
  [request send-response raise-error]
  (send-response {:body "It works!"}))

(-> app
    (yt/server {:http/port 11010 :ring/async true})
    (yt/start!))
```

### HTTP/2 (h2c)

The HTTP2 ClearText (h2c) is enabled by default, so you don't need to
do nothing. If you want to disable it:

```clojure
(yt/server app {:http/port 11010 :http/protocols #{:h1}})
```

The http2 on secure layer (TLS) is not supported, this package is
intended to be used with TLS offloding layer such that NGINX or
HAPROXY.


### WebSocket

Any ring handler can upgrade to websocket protocol, there is an example:

```clojure
(require '[yetti.websocket :as yws])

(defn handler
  [request]
  (if (yws/upgrade-request? request)
    (yws/upgrade request (fn [upgrade-request]
                           {:on-connect (fn [ws])
                            :on-error (fn [ws e])
                            :on-close (fn [ws status-code reason])
                            :on-text (fn [ws text-message])
                            :on-bytes (fn [ws bytes offset len])
                            :on-ping (fn [ws bytebuffer])
                            :on-pong (fn [ws bytebuffer])}))
    {:status 404}))
```

IWebSocket protocol allows you to read and write data on the `ws` value:

- `(yws/send! ws msg)`
- `(yws/send! ws msg callback)`
- `(yws/ping! ws msg)`
- `(yws/ping! ws msg callback)`
- `(yws/pong! ws msg)`
- `(yws/pong! ws msg callback)`
- `(yws/close! ws)`
- `(yws/remote-addr ws)`
- `(yws/idle-timeout! ws timeout)`

Notice that we support different type of msg:

* **byte[]** and **ByteBuffer**: send binary websocket message
* **String**: send text websocket message
* **fn**: custom function you can operate on Jetty's [RemoteEndpoint][3]

[3]: https://www.eclipse.org/jetty/javadoc/jetty-11/org/eclipse/jetty/websocket/api/RemoteEndpoint.html

A callback can also be specified for `send!`, `ping!` or `pong!`:

```clojure
(yws/send! ws msg (fn [err])
;; The `err` is null if operation terminates successfuly
```


## FAQ

### Why don't use jetty 11?

Initially we have tried to use the jetty-11, but we rolled back to
10.x branch because the ring multipart parsing depends on a library
that depends on the servlet api (pre jakarta, javax).

Right now the adapter is 100% compatible with latest servlet API
(exposed under jakarta. package), you just need to rename javax to
jakarta.

And on the end, jetty-11 is the same code as jetty-10 but with javax
renamed to jakarta (see more on [jetty10v11][2]). Until we find a
multipart parsing library that does not depends on javax api we will
use the jetty-10 branch.

[2]: https://webtide.com/jetty-10-and-11-have-arrived

### Why not contribute back to [rj9a][1]?

We already do it when it make sense. But some changes implies breaking
user code and we don't want it. Making a new package, opens the window
to break API and build a new one. This is the one of the main reason
behind this package.

## License

```
Copyright © 2021-Now Andrey Antukh
Copyright © 2013-2021 Sun Ning
Distributed under the Eclipse Public License, the same as Clojure.
```
