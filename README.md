# Multi-tenant runtime for simple HTTP web APIs

Haiku-http provides you with a multi-tenant runtime for hosting simple HTTP web APIs implemented in JavaScript:

- **Sub process multi-tenancy**. You can execute code from multiple tenants within a single process while preserving data integrity of each tenant. You also get a degree of local denial of service prevention: the section about the sandbox below explains features and limitations.
- **Programming model based on node.js**. Programming model for authoring HTTP web APIs is based on a subset of node.js. Implementing an HTTP web API resembles coding the body of the node.js HTTP request handler function. You can currently use HTTP[S], TCP, TLS, and MongoDB. 
- **Easy deployment**. You can host the code for HTTP web APIs wherever it can be accessed with an HTTP call from within the runtime. GitHub or Gist work fine. 

## History, motivation, and goals

Read more about these aspects at [http://tomasz.janczuk.org](http://tomasz.janczuk.org). 

## Prerequisites

- Windows, MacOS, or *nix (tested on Windows 7 & 2008 Server, MacOS Lion, Ubuntu 11.10)
- [node.js v0.7.0 or greater](http://nodejs.org/dist/)

## Getting started

Install haiku-http:

```
npm install haiku-http
```

Start the haiku-http runtime (default settings require ports 80 and 443 to be available):

```
sudo node node_modules/haiku-http/src/haiku-http.js
```

(in an environment with an HTTP proxy, one must specify the proxy address using the `-x` parameter, e.g. `sudo node node_modules/haiku-http/src/haiku-http.js -x itgproxy:80`)

Open a browser and navigate to 

```
http://localhost?x-haiku-handler=https://github.com/tjanczuk/haiku-http/blob/master/samples/haikus/hello.js
```

You should see a 'Hello, world!' message, which is the result of executing the haiku-http web API implemented at https://github.com/tjanczuk/haiku-http/blob/master/samples/haikus/hello.js. 

## Samples

You can explore and run more samples from https://github.com/tjanczuk/haiku-http/blob/master/samples/haikus/. Each file contains a haiku-http handler that implements one HTTP web API. These demonstrate making outbound HTTP[S] calls and getting data from a MongoDB database, among others.

## Using haiku-http runtime

The haiku-http runtime is an HTTP and HTTPS server. It accepts any HTTP[S] request sent to it and returns a response generated by the haiku-http handler associated with that request. The caller controls the processing of the request with parameters attached to the HTTP request. Each named parameter can be provided either in the URL query string or as an HTTP request header. 

### x-haiku-handler (required)

The haiku-http handler to be used for processing the request must be specified by the caller by passing a URL of the handler in the `x-haiku-handler` parameter of the HTTP request. The haiku-http runtime will obtain the handler code by issuing an HTTP GET against the URL. It will then create a sanboxed execution environment for the handler code and pass the HTTP request to it for processing. The response generated by the handler is returned to the original caller. 

The simplest way to specify a haiku-http handler is to use URL query paramaters, e.g.:

```
http://localhost?x-haiku-handler=https://github.com/tjanczuk/haiku-http/blob/master/samples/haikus/hello.js
```

### x-haiku-console (optional)

The caller may use the `x-haiku-console` parameter to get access to the diagnostic information written out to the console  by the haiku-http handler (e.g. using console.log). Following values of the paramater are allowed:

- `none` (default) - console calls made by the haiku-http handler are allowed but ignored.
- `body` - console output is captured and returned in the entity body of the HTTP response instead of the entity body generated by the haiku-http handler. Only console output generated until the call to `res.end()` is returned, all subsequent console output is discarded. 
- `header` - console output is captured and returned in the HTTP response header `x-haiku-console`. Entity body of the HTTP response as well as other HTTP response headers remain unaffected. Only console output generated until an explicit or implicit call to `res.writeHead()` is returned, all subsequent console output is discarded.
- `trailer` - console output is captured and retuend in the HTTP response trailer `x-haiku-console`. Entity body of the HTTP response as well as other HTTP response headers remain unaffected. Only consle output generated until the call to `res.end()` is returned, all subsequent console output is discarded. Note that the debugging tools built into most browsers do not show HTTP trailers. One way to inspect the value of the HTTP trailer is to use `curl --trace -`. 

The simplest way to obtain the console output generated by the handler is to request it in the HTTP response body, e.g. (multiple lines for readability only):

```
http://localhost?x-haiku-handler=https://github.com/tjanczuk/haiku-http/blob/master/samples/haikus/console.js
                 &x-haiku-console=body
```

For more examples of the various modes of accessing the console, check out the [console.js](https://github.com/tjanczuk/haiku-http/blob/master/samples/haikus/console.js) sample. 

## Writing haiku-http handlers

The haiku-http handler corresponds to the body of the HTTP reqeust callback function in node.js. 

For example, given the following node.js application:

```
require('http').createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World\n');
}).listen(1337, "127.0.0.1");
```

a corresponding haiku-http handler would be:

```
res.writeHead(200, {'Content-Type': 'text/plain'});
res.end('Hello World\n');
```

Within the haiku-http handler, the following globals are available:

- `req` and `res` represent the HTTP request and response objects normally passed to the node.js request handler. Both objects are sandboxed. Check the section about sandboxing for details.  
- `setTimeout`, `clearTimeout`, `setInterval`, `clearInterval` work the same as in node.js.  
- `require` allows loading a node.js module from the subset available in the haiku-http sandbox environment. These include node.js core modules `http`, `https`, `net`, `tls`, `crypto`, `url`, `buffer`, `stream`, `events`, `util`, as well as third party modules `request`, and `mongodb`. All modules are sandboxed to only offer client side APIs (e.g. one can make outbound HTTP calls or TCP connections, but not establish a listener). Check the section about sandboxing for more details. 

Unlike in node.js, no global, in-memory state created by a haiku-http handler is preserved between subsequent calls.  

## The haiku-http runtime model and configuration

The haiku-http runtime behavior can be configured using command line parameters passed to the haiku-http.js application:

```
Usage: node ./haiku-http.js

Options:
  -w, --workers            Number of worker processes                                              [default: 16]
  -p, --port               HTTP listen port                                                        [default: 80]
  -s, --sslport            HTTPS listen port                                                       [default: 443]
  -c, --cert               Server certificate for SSL                                              [default: "cert.pem"]
  -k, --key                Private key for SSL                                                     [default: "key.pem"]
  -x, --proxy              HTTP proxy in host:port format for outgoing requests                    [default: ""]
  -i, --maxsize            Maximum size of a handler in bytes                                      [default: "16384"]
  -t, --maxtime            Maximum clock time in milliseconds for handler execution                [default: "5000"]
  -r, --maxrequests        Number of requests before process recycle. Zero for no recycling.       [default: "1"]
  -a, --keepaliveTimout    Maximum time in milliseconds to receive keepalive response from worker  [default: "5000"]
  -v, --keepaliveInterval  Interval between keepalive requests                                     [default: "5000"]
  -l, --maxConsole         Maximum console buffer in bytes. Zero for unlimited.                    [default: "4096"]
  -d, --debug              Enable debug mode                                  
```

Haiku-http uses node.js cluster functionality to load balance HTTP traffic across several worker processes. The `-w` parameter controls the number of worker processes in a cluster. When a worker process exits, the master will create a new instance to replace it. 

Recycling of a worker process can be forced after a specific number of HTTP requests has been processed by that worker. The number of HTTP requests is specified with the `-r` option, and defaults to 1. This means that by default OS processes are used to support haiku-http handler isolation, which is secure but not very efficient. Increasing that number (or disabling recycling altogether by setting the value to 0) increases the opportunity for rouge handlers to attempt local denial of service attacks (e.g. by blocking the node.js event loop), but also greatly improves performace of the system in case handlers are well behaved. Choosing the right value depends on the level of trust you put in the haiku-http handlers your installation runs. 

There are several limits that can be applied to the execution of the haiku-http handlers. First, the `-i` option controls the maximum size in bytes of the haiku-http handler, and is meant to limit the exposure of the runtime to attacks that cause downloading large amounts of data over the network. The `-t` option enforces a clock time limit on the execution of the handler (the time the handler takes to call the `res.end()` method). If the handler does not finish within this time limit, an HTTP error response is returned to the caller. Lastly, the `-l` option controls how much of the console output will be buffered in memory per each handler execution. 

The runtime cannot prevent runaway handlers (e.g. a handler that performs blocking operations) but it can detect such rouge handlers using a keepalive mechanism. At the interval specified by the `-v` option, the runtime will send a challange to the worker process. If the worker process event loop is blocked, and it therefore does not respond to the challange with the timeout specified with the `-a` option, the worker process is considered runaway, and is killed by the runtime. Subsequently the killed worker is replaced with a fresh instance, so the runtime recovers back to the original state. All requests that were active in that worker process are abnormally terminated, which means the number of reqeusts specified by the `-r` option is the upper limit of the exposure of the system to an execution of a rouge handler.

The `-p` and `-s` options control the listen TCP ports for HTTP and HTTPS traffic, respectively. The `-c` and `-k` options specify the file name of the X.509 server certificate and corresponding private key, both in PEM format. 

The `-x` option allows you to specify the hostname and port number of an HTTP proxy to be used for obtaining haiku-http handler code from the URL specified by the `x-haiku-handler` parameter of an HTTP request. This is required in environments where a direct internet connection is not available. The proxy setting only applies to the haiku-http runtime behavior, not the behavior of the actual handler code. 

Lastly, the `-d` option cause any console output generated by the handler to be written out to stdout of the haiku-http.js process, in addition to processing it following client's disposition made with the `x-haiku-console` parameter of the HTTP request. 

## The sandbox and isolation

The sandbox used by the haiku-http runtime is essential in providing data isolation between haiku-http handlers. The principles behind the sandbox are following:

- The sandbox limits available APIs to restrict data storage options to locations that do not rely on the operating system security mechanisms associated with the permissions of the haiku-http runtime process. As such handlers can exchange data with outside systems using HTTP and TCP, manipulate data in a MongoDB database, but they cannot otherwise access any data on the local machine or the network using permissions with which the http-haiku runtime executes. In particular, the `fs` module is not available. 
- The sandbox completely isolates in-memory data of a handler instance from other handler instances. This is accomplished by running each handler in its own V8 script context using a fresh instance of a global object, by disabling node.js module caching, and by preventing untusted code from accessing runtime's trusted core with the use of the ECMA script 5 'strict mode' when evaluating handler code. 
- The sandbox prevents handlers from creating new HTTP or TCP listeners, hence ensuring that invocation of any handler is subject to the sandboxing rules. In particular, rouge handlers cannot intercept requests that target other handlers. 

The haiku-http runtime provides a good level of data isolation at sub-process density using the sandboxing mechanisms above. However, running multiple handlers within the same process leaves room for a range of local denial of service attacks due to the current limits of the V8/node.js platform. You can read more about this aspect at [http://tomasz.janczuk.org](http://tomasz.janczuk.org).   

## Running tests

To run tests, first install the [mocha](https://github.com/visionmedia/mocha) test framework:

```
sudo npm install -g mocha
```

Then start the haiku-http runtime:

```
sudo node node_modules/haiku-http/src/haiku-http.js
```

As well as a local HTTP server that will serve the haiku-http handlers included in the repo:

```
node node_modules/haiku-http/samples/server.js
```

You are now ready to run tests:

```
mocha
```

If you experience errors, first make sure that test prerequisities are all met:

```
mocha -R spec test/0*
```
