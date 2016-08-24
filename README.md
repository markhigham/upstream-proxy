
Upstream Proxy
==============

Virtual host your apps: Upstream Proxy routes incoming requests - based on their host header - to Node.js apps in the backend.

### It works with all kinds of web traffic...

* HTTP
* TLS (HTTPS)
* HTTP/2
* WebSocket
* Server-Sent Events

### ...and is optimized for

* ease of use *(by providing a clean API)*
* speed *(by keeping things lean and simple :)*

Plug it to an NGINX upstream (via TCP port or IPC socket), connect it to any other service - or expose it directly to the internet (port 80 and 443). 

In the future there will be [examples](https://github.com/nodexo/upstream-proxy/tree/master/examples) covering most use cases.


Installation
------------

    npm install upstream-proxy


Usage
-----
Basic example:
```javascript
  const upstreamProxy = require('upstream-proxy');

  let myConfig = {
    frontend_connectors: [
      {
        host_headers: ['127.0.0.1', 'localhost'],
        target: 'local-website'
      }
    ],
    backend_connectors: [
      { 
          name: 'local-website',
          endpoints: {
              tcp: { host: '127.0.0.1', port: 3001 }
          }
      }
    ]
  }

  let proxy = new upstreamProxy(myConfig).listen(3000).start();
```


API Methods
------------
- [start()](#start)
- [stop()](#stop)
- [getStatus()](#getstatus)
- [setConfig(obj)](#setconfigobj)
- [getConfig()](#getconfig)
- [getRoutes()](#getroutes)
- [setCallbacks(obj)](#setcallbacksobj)
- [getCallbacks()](#getcallbacks)
- [disconnectClients(str)](#disconnectclientsstr)
- [disconnectAllClients()](#disconnectallclients)


### start()

Starts the proxy.

Once created the proxy listens for connections immediately.   
If nothing is configured yet, each request returns a "503 Service Unavailable" (see [getStatus](#getStatus)).


### stop()

Stops the proxy.

New requests will be answered with "503 Service Unavailable".  
Existing connections are NOT affected, to disconnect clients use [disconnectAllClients](#disconnectAllClients).


### getStatus()

Gets current status.

Possible return values
- active: proxy routes incoming requests
- passive: each request returns a "503 Service Unavailable"


Example:

```js
let proxyStatus = proxy.getStatus();

console.log(proxyStatus);
// active
```


### setConfig(obj)

Sets the configuration and generates the routing map.

- Complete overwrite of current config and routes
- Uses only "frontend_connectors" and "backend_connectors" properties of the passed object
- Generates property "created" with the current unix timestamp

Existing connections are not affected: To disconnect clients after a configuration change use [disconnectClients](#disconnectClients).

Example:

```js
const upstreamProxy = require('upstream-proxy');

let myConfig = {
  "frontend_connectors": [
    {
      "host_headers": [ "127.0.0.1", "localhost" ],
      "target": "local-website"
    }
  ],
  "backend_connectors": [
    {
      "name": "local-website",
      "endpoints": { 
        "tcp": { "host": "127.0.0.1", "port": 3001 }
      }
    }
  ]
}

let proxy = new upstreamProxy().listen(3000);

proxy = setConfig(myConfig);
// OK
```

### getConfig()

Gets the current configuration.

Example:

```js
let liveConfig = proxy.getConfig();

console.log( JSON.stringify(liveConfig, null, 2) );
/*
{
  "created": 1472018433024,
  "frontend_connectors": [
    {
      "host_headers": [
        "127.0.0.1",
        "localhost"
      ],
      "target": "local-website"
    }
  ],
  "backend_connectors": [
    {
      "name": "local-website",
      "endpoints": {
        "tcp": {
          "host": "127.0.0.1",
          "port": 3001
        }
      }
    }
  ]
}
*/
```

### getRoutes()

Gets current routes.

Example:

```js
let liveRoutes = proxy.getRoutes();

console.log( JSON.stringify([...liveRoutes], null, 2) );
/*
[
  [
    "127.0.0.1",
    {
      "tcp": {
        "host": "127.0.0.1",
        "port": 3001
      }
    }
  ],
  [
    "localhost",
    {
      "tcp": {
        "host": "127.0.0.1",
        "port": 3001
      }
    }
  ]
]
*/
```

### setCallbacks(obj)

Sets callbacks (hooks) for individual error handling.

Example:

```js
let error503 = (socket, host_header) => {
  //...
}

let myCallbacks = {
  503: error503
}

proxy = setCallbacks(myCallbacks);
// OK
```

### getCallbacks()

Gets current callbacks.

Example:

```js
let liveCallbacks = proxy.getCallbacks();

console.log( JSON.stringify(liveCallbacks, null, 2) );
/*
{
  503: error503
}
*/
```

### disconnectClients(str)

Disconnects clients for the specified host name, returns the number of terminated connections.  
*Only affects the exact host name, "localhost" DOES NOT disconnect clients for "127.0.0.1"*

Example:

```js
proxy = disconnectClients("localhost");
// 7
```

### disconnectAllClients()

Disconnects all clients, returns the number of terminated connections.

Example:

```js
proxy = disconnectAllClients();
// 102
```


Further Information
-------------------

### Serving the web

In a classic [NGINX](https://www.nginx.com/resources/wiki/) or [Apache](https://httpd.apache.org/) web server configuration you would use [virtual hosting](https://en.wikipedia.org/wiki/Virtual_Hosting) to host multiple apps/sites on a single IP address. But this is complicated and troublesome to configure/automate.

Proxying requests through Upstream Proxy to multiple Node.js processes is my favorite route to go, because it allows me to...
* write apps that are simple to develop and test - because they're stand alone. All they need to know for proper operation is an endpoint for listening.
* do hassle-free configuration of routes
* apply monitoring and clustering to my apps via process manager (e.g. [PM2](http://pm2.keymetrics.io/)) - avoid using a web server for it!
* intentionally reload or restart single apps

Upstream Proxy is purpose built: It takes the place of a regular web server when you don't need it for anything else except proxying requests.
If you want to plug it into an NGINX upstream for routing requests you are fine too - the configuration is much simpler and you get **Javascript truly all the way down**.

FYI: There is no reason you can't use Upstream Proxy as a gateway to non-Node.js apps or services.



### TLS (HTTPS) and HTTP/2

[SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) is supported for detecting the hostname of a secure request.
[Today all modern browsers support SNI.](http://caniuse.com/#feat=sni)

Upstream Proxy does not need to know about your certificates. It peeks at the TLS/SNI extension headers (which are not encrypted) and forwards the encrypted data as-is. There is no need to read or modify the encrypted payload.


### WebSocket(s)

They just work!

The initial negotation of for a [WebSocket](https://en.wikipedia.org/wiki/WebSocket) is done via HTTP(S), so Upstream Proxy routes it normally. Once a socket has been established, data is passively/transparently piped between the upstream and downstream sockets, including all WebSocket negotiation and data.

The same applies to [Server-Sent Events](https://en.wikipedia.org/wiki/Server-sent_events).


Benchmarks
----------

There are none published yet (internally I have done a lot of testing...).

If you create one and want to share, I would be happy to include some.
