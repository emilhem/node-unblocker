# unblocker

Unblocker was originally a web proxy for evading internet censorship, similar to CGIproxy / PHProxy / Glype but 
written in node.js. It's since morphed into a general-purpose library for proxying and rewriting remote webpages.
 
All data is processed and relayed to the client on the fly without unnecessary buffering, making unblocker one of the 
fastest web proxies available.

[![Build Status](https://travis-ci.org/nfriedly/node-unblocker.png?branch=master)](https://travis-ci.org/nfriedly/node-unblocker)

### The magic part

The script uses "pretty" urls which, besides looking pretty, allow links with relative paths 
to just work without modification. (E.g. `<a href="path/to/file2.html"></a>`) 

In addition to this, links that are relative to the root (E.g. `<a href="/path/to/file2.html"></a>`) 
can be handled without modification by checking the referrer and 307 redirecting them to the proper 
location in the referring site. (Although the proxy does attempt to rewrite these links to avoid the redirect.)

Cookies are proxied by adjusting their path to include the proxy's URL, and a bit of extra work is done to ensure they 
remain intact when switching protocols or subdomains.

### Limitations

Although the proxy works well for standard login forms and even most AJAX content, OAuth login forms and anything that uses 
[postMessage](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage) (Google, Facebook, etc.) are not 
likely to work out of the box. This is not an insurmountable issue, but it's not one that I expect to have fixed in the 
near term. Patches are welcome, including both a general-purpose fix to go into the main library, and site-specific 
fixes to go in the examples folder.

## Running the website on your computer

Requires [node.js](http://nodejs.org/) >= 0.12.
Then [download node-unblocker](https://github.com/nfriedly/node-unblocker/archive/master.zip), 
`cd` into the `examples/nodeunblocker.com/` directory, 
and run `npm install` to set things up. 
Then run `npm start` to start the server. It should spawn a new instance for each CPU 
core you have. 

(Note: running `node app.js` *will not work*. The server code is in the [Gatling](https://npmjs.org/package/gatling) 
package, which the `npm start` command calls automatically.)

## Running the website on heroku/bluemix/modulous/etc

This project should be runnable on a free [Heroku](http://www.heroku.com/) instance without 
modification - just copy the `examples/nodeunblocker.com/` folder to a new git repo and push it.

## Using unblocker as a library in your software

Unblocker exports an [express](http://expressjs.com/)-compatible API, so using in an express application is trivial:

    var express = require('express')
    var Unblocker = require('unblocker');
    var app = express();
    
    // this must be one of the first app.use() calls and must not be on a subdirectory to work properly
    app.use(new Unblocker({prefix: '/proxy/'})); 
    
    app.get('/', function(req, res) {
        //...
    });
    
Usage without express is similarly easy, see [examples/simple/server.js](examples/simple/server.js) for an example.

### Configuration

Unblocker supports the following configuration options, defaults are shown:

```js
{
    prefix: '/proxy/',  // Path that the proxied URLs begin with. '/' is not recommended due to a few edge cases.
    host: null, // Host used in redirects (e.g `example.com` or `localhost:8080`). Default behavior is to determine this from the request headers.
    requestMiddleware: [], // Array functions that perform extra processing on client requests before they are sent to the remote server. API is detailed below.
    responseMiddleware: [], // Array functions that perform extra processing on remote responses before they are sent back to the client. API is detailed below.
    standardMiddleware: true, // Allows you to disable all built-in middleware if you need to perform advanced customization of requests or responses.
    processContentTypes: [ // All  built-in middleware that modifies the content of responses limits itself to these content-types.
        'text/html',
        'application/xml+xhtml',
        'application/xhtml+xml',
        'text/css'
    ]
}
```

#### Custom Middleware

Custom middleware should be a function that accepts a single `data` argument and runs synchronously. 

##### requestMiddleware

Data example:
```js
{
    url: 'http://example.com',
    clientRequest: {request},
    clientResponse: {response},
    headers: {
        //...
    },
    stream: {stream of data for PUT/POST requests, empty stream for other types}
}
```

requestMiddleware may inspect the headers, url, etc. It can modify headers, pipe PUT/POST data through a transform stream, or respond to the request directly.
If you're using express, the request and response objects will have all of the usual express goodies. For example:

```js
function validateRequest(data) {
    if (!data.url.match(/^https?:\/\/en.wikipedia.org\//) {
        data.clientResponse.status(403).send('Wikipedia only.');
    }
}
var config = {
    requestMiddleware: [
        validateRequest
    ]
}
```

If any piece of middleware sends a response, no further middleware is run.

After all requestMiddleware has run, the request is forwarded to the remote server with the (potentially modified) url/headers/stream/etc.

##### responseMiddleware

responseMiddleware receives the same `data` object as the requestMiddleware, but the `headers` and `stream` fields are replaced with those of the remote server's response, and several new fields are added for the remote request and response:

Data example:
```js
{
    url: 'http://example.com',
    clientRequest: {request},
    clientResponse: {response},
    remoteRequest {request},
    remoteResponse: {response},
    contentType: 'text/html',
    headers: {
        //...
    },
    stream: {stream of response data}
}
```

For modifying content, create a new stream and then pipe `data.stream` to it and replace `data.stream` with it:

```js
var through = require('through');

function injectScript(data) {
    if (data.contentType == 'text/html') {
        var myStream = through(function(chunk) {
            this.queue(chunk.replace('</body>', '<script src="/my/script.js"></script></body>'));
        });
        
        data.stream = data.stream.pipe(myStream);
    }
}

var config = {
    responseMiddleware: [
        injectScript
    ]
}
```

See `examples/nodeunblocker.com/app.js` for another example of adding a bit of middleware. Also, see any of the built-in middleware in the `lib/` folder.


##### Built-in Middleware

Most of the internal functionality of the proxy is also implemented as middleware:

* **host**: Corrects the `host` header in outgoing responses
* **referer**: Corrects the `referer` header in outgoing requests
* **cookies**: 
    Fixes the `Path` attribute of set-cookie headers to limit cookies to their "path" on the proxy (e.g. `Path=/proxy/http://example.com/`). 
    Also injects redirects to copy cookies from between protocoles and subdomains on a given domain.
* **hsts**: Removes [Strict-Transport-Security](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security) headers because they can leak to other sites and can break the proxy.
* **redirects**: Rewrites urls in 3xx redirects to ensure they go through the proxy
* **decompress**: Decompresses `Content-Encoding: gzip` responses and also tweaks request headers to ask for either gzip-only or no compression at all. (It will attempt to decompress `deflate` content, but there are some issues, so it does not advertise support for `deflate`.)
* **charsets**: Converts the charset of responses to UTF-8 for safe string processing in node.js. Determines charset from headers or meta tags and rewrites all headers and meta tags in outgoing response.
* **urlPrefixer**: Rewrites URLS of links/images/css/etc. to ensure they go through the proxy
* **metaRobots**: Injects a ROBOTS: NOINDEX, NOFOLLOW meta tag to prevent search engines from crawling the entire web through the proxy.
* **contentLength**: Deletes the content-length header on responses if the body was modified.

Setting the `standardMiddleware` configuration option to `false` disables all built-in middleware, allowing you to selectively enable, configure, and re-order the built-in middleware. 

This configuration would mimic the defaults:

```js

var Unblocker = require('unblocker');

var config = {
    prefix: '/proxy/',
    host: null,
    requestMiddleware: [],
    responseMiddleware: [],
    standardMiddleware: false,  // disables all built-in middleware
    processContentTypes: [
        'text/html',
        'application/xml+xhtml',
        'application/xhtml+xml'
    ]
}

var host = Unblocker.host(config);
var referer = Unblocker.referer(config);
var cookies = Unblocker.cookies(config);
var hsts = Unblocker.hsts(config);
var redirects = Unblocker.redirects(config);
var decompress = Unblocker.decompress(config);
var charsets = Unblocker.charsets(config);
var urlPrefixer = Unblocker.urlPrefixer(config);
var metaRobots = Unblocker.metaRobots(config);
var contentLength = Unblocker.contentLength(config);

config.requestMiddleware = [
    host,
    referer,
    decompress.handleRequest,
    cookies.handleRequest
    // custom requestMiddleware here
];

config.responseMiddleware = [
    hsts,
    redirects,
    decompress.handleResponse,
    charsets,
    urlPrefixer,
    cookies.handleResponse,
    metaRobots,
    // custom responseMiddleware here
    contentLength
];

app.use(new Unblocker(config));
```

## Debugging

Unblocker is fully instrumented with [debug](https://www.npmjs.com/package/debug). 
Enable debugging via environment variables:

    DEBUG=unblocker:* node mycoolapp.js
    
There is also a middleware debugger that adds extra debugging middleware before and after each existing middleware 
function to report on changes. It's included with the default DEBUG activation and may also be selectively enabled:


    DEBUG=unblocker:middleware node mycoolapp.js
    
... or disabled:

    DEBUG=*,-unblocker:middleware node mycoolapp.js

## Todo

* Consider adding compress middleware to compress text-like responses
* Un-prefix urls in GET / POST data
* Inject js to proxy postMessage data and fix origins
* More examples
* Even more tests

## AGPL-3.0 License
This project is released under the terms of the [GNU Affero General Public License version 3](https://www.gnu.org/licenses/agpl-3.0.html). 
All source code is copyright Nathan Friedly.
 
Commercial licensing and support are also available, contact Nathan Friedly (nathan@nfriedly.com) for details.

## Contributors 
* [Nathan Friedly](http://nfriedly.com)
* [Arturo Filastò](https://github.com/hellais)
* [tfMen](https://github.com/tfMen)
* [Emil Hemdal](https://github.com/emilhem)

