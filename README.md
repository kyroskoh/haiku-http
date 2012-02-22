# Multi-tenant runtime for simple HTTP web APIs

Haiku-http provides you with a multi-tenant runtime for hosting simple HTTP web APIs implemented in JavaScript:

- **Sub process multi-tenancy**. You can execute code from multiple tenants within a single process while preserving data integrity of each tenant. You also get a degree of local denial of service prevention: the section about the sandbox below explains features and limitations.
- **Programming model based on node.js**. Programming model for authoring HTTP web APIs is based on a subset of node.js. Implementing an HTTP web API resembles coding the body of the node.js HTTP request handler function. You can currently use HTTP[S], TCP, TLS, and MongoDB. 
- **Easy deployment**. You can host the code for HTTP web APIs wherever it can be accessed with an HTTP call from within the runtime. GitHub or Gist work fine. 

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

Open a browser and navigate to 

```
http://localhost?x-haiku-handler=https://github.com/tjanczuk/haiku-http/blob/master/samples/haikus/hello.js
```

You should see a 'Hello, world!' message, which is the result of executing the haiku-http web API implemented at https://github.com/tjanczuk/haiku-http/blob/master/samples/haikus/hello.js. 

## Samples

You can explore and run more samples from https://github.com/tjanczuk/haiku-http/blob/master/samples/haikus/. Each file contains a haiku-http handler that implements one HTTP web API. These demonstrate making outbound HTTP[S] calls and getting data from a MongoDB database, among others.

## Tests

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
