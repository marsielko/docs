# Deconstructing WebSockets: Everything you (probably never) wanted to know

This article is all about building a deeper understanding for what WebSockets are, how they came to be and what's actually going on under the hood. Together we'll learn about the WebSocket API and the framing protocol itself, build a bare-bones WebSocket server, and actually parse and generate the raw binary WebSocket frames. At the end, we'll take a look at some existing solutions that you might for handling the client and/or the server side of the WebSocket equation.

- [Aren't we reinventing the wheel here?](#arent-we-reinventing-the-wheel-here)
- [First, a bit of background](#first-a-bit-of-background)
  - [Scripting the web with JavaScript](#scripting-the-web-with-javascript)
  - [The birth of XMLHttpRequest and AJAX](#the-birth-of-xmlhttprequest-and-ajax)
  - [A world of new possibilities](#a-world-of-new-possibilities)
  - [Hatching the real-time web](#hatching-the-real-time-web)
  - [Enter, WebSockets](#enter-websockets)
  - [So what exactly are WebSockets, anyway?](#so-what-exactly-are-websockets-anyway)
    - [A quick note about authentication and authorization](#a-quick-note-about-authentication-and-authorization)
- [Implementing a WebSocket Server (with Node.js)](#implementing-a-websocket-server-with-nodejs)
    - [Set up your project environment](#set-up-your-project-environment)
  - [Getting the ball rolling with HTTP](#getting-the-ball-rolling-with-http)
  - [Ditching HTTP for something more appropriate](#ditching-http-for-something-more-appropriate)
  - [Avoiding funny business](#avoiding-funny-business)
  - [Subprotocols; agreeing upon a shared dialect](#subprotocols-agreeing-upon-a-shared-dialect)
    - [WebSocket Extensions](#websocket-extensions)
  - [The client side: using WebSockets in the browser](#the-client-side-using-websockets-in-the-browser)
    - [Give it a go yourself](#give-it-a-go-yourself)
  - [Generating and parsing WebSocket message frames](#generating-and-parsing-websocket-message-frames)
    - [Alignment of Node.js socket buffers with WebSocket message frames](#alignment-of-nodejs-socket-buffers-with-websocket-message-frames)
    - [Implementing the parser](#implementing-the-parser)
    - [Opcodes: message frames vs control frames](#opcodes-message-frames-vs-control-frames)
    - [Masked frames](#masked-frames)
    - [Constructing a frame buffer for the response message](#constructing-a-frame-buffer-for-the-response-message)
    - [Dispatching the response back to the client](#dispatching-the-response-back-to-the-client)
- [WebSocket libraries you can use right now](#websocket-libraries-you-can-use-right-now)
  - [ws](#ws)
    - [Minimal code samples](#minimal-code-samples)
  - [μWebSockets](#%CE%BCwebsockets)
    - [Minimal code sample](#minimal-code-sample)
  - [faye-websocket](#faye-websocket)
    - [Minimal code samples](#minimal-code-samples)
  - [Socket.io](#socketio)
    - [Minimal code samples](#minimal-code-samples)
  - [SocketCluster](#socketcluster)
    - [Minimal code samples](#minimal-code-samples)
  - [SockJS](#sockjs)
    - [Minimal code samples](#minimal-code-samples)
  - [deepstream server](#deepstream-server)
- [Scaling beyond a single server](#scaling-beyond-a-single-server)
- [Realtime messaging platforms-as-a-service](#realtime-messaging-platforms-as-a-service)
  - [Ably](#ably)
  - [PubNub](#pubnub)
  - [Pusher](#pusher)
  - [deepstreamHub](#deepstreamhub)

## Aren't we reinventing the wheel here?

One of the best - and worst - things about being a developer is the vast quantity of abstractions available to us to get a job done. So many of the problems we need to solve have been encountered by others before us, that often not only is there usually a significant wealth of wisdom already published that we can draw upon, but often there are numerous solutions already implemented, ready to plug directly into whatever we're building.

Existing abstractions are great, but in some cases they can be a crutch, making us complacent and ultimately limiting what we're actually capable of doing. As the old saying goes, "don't reinvent the wheel", and if you just have a job to get done quickly, then existing solutions are great. Plug them in and move on. This isn't always the right approach, however, and in fact it's rarely that simple anyway.

In order to deeply understand something, we almost always have to experience it ourselves, at least to some degree. Human brains are not optimised to store "facts". They're optimised to capture and match on patterns they experience. This is why, when you let go of your aversion to reinventing wheels and learning new things, and decide to have a go at doing something yourself, the depth of your understanding can grow far beyond what you ever expected and empower you to start coming up with ideas and approaches you'd never have otherwise seen.

There are many WebSocket abstractions out there already, and many more higher-level services built on top of them. By understanding the underlying technology yourself though, when you inevitably hit a brick wall as a result of some mismatch between what you're trying to do and what your chosen abstraction offers, you'll have a way of breaking through that wall. Maybe you'll want to contribute fixes and improvements to whatever you're using, or perhaps you'll choose to go deeper and explore entirely different approaches to your problem - approaches that are highly dependant on knowing what's actually going on under the hood.

As I like to parrot whenever I'm given a soapbox:

> &ldquo;The best way to understand something is to do it yourself&rdquo;  
> _- Me_

Also:

> &ldquo;The best tool for a job is the one designed specifically with that job in mind&rdquo;  
> _- Me again_

So let's rip off that hood and see what WebSockets are really all about.

## First, a bit of background

WebSockets have actually only been generally available for a few years. Before they came along, we had still been able to build robust, essentially "real-time" applications for a while, but it was a lot more painful to do so, and moreso the further back in time we look.

The web has been built on the back of the HTTP protocol, which was originally designed entirely as a request-response mechanism. Open a connection, describe what you want, get back a response, then close the connection. This was fine in the early days of the web because, back then, really we were just dealing with a text document and maybe a few additional assets (usually images).

### Scripting the web with JavaScript

In 1995, Netscape Communications hired Brendan Eich with the goal of embedding scripting capabilities into their Netscape Navigator browser, and thus JavaScript was born. Initially JavaScript was kind of weird and couldn't do a whole lot (especially with the extremely limited browser DOM that JavaScript had at its disposal), but it was useful for a few things, such as simple validation of input fields before submitting an HTML form to the server.

Microsoft soon entered the arena with Internet Explorer, which is where the original "browser wars" really began. Both companies were competing with each other to have the best browser, so features and capabilities were inevitably added on a regular basis to both Netscape and Internet Explorer as each company strove to outdo the other.

### The birth of XMLHttpRequest and AJAX

Two of the most significant capabilities that were soon introduced at that time were the ability to embed Java applets into a page, and Microsoft's own offering - ActiveX controls. These were essentially precompiled components that could optionally present an embedded user interface of their own within a web page. More than that though, they allowed a whole host of additional possibilities beyond JavaScript's own (at the time) meagre suite of scripting capabilities.

While there were a few comparable networking capabilities available via Java, the most significant background communication feature first appeared in 1999, with the [Microsoft.XMLHTTP](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/Using_XMLHttpRequest_in_IE6) `ActiveXObject` interface. It was available natively in Internet Explorer 5.0 without installing plugins, it could be instantiated with a single line of JavaScript, and it didn't require any of the usual friction involved when dealing with Java applets. The XMLHTTP object made it possible to silently issue a request to a server and receive a response - all without reloading the page or otherwise interrupting the user's experience. JavaScript code could then decipher the response and construct modifications to the page, enabling a whole host of rich experiences to be integrated into a website. Common early use cases included things like allowing a dropdown box to be populated with options based on a user's prior input, and "instant" validation of username availability while filling out a user registration form.

Example JavaScript code that would have been used to instantiate an `XMLHTTP` object:

```js
// Note: this code only works in old versions of Internet Explorer
var xmlhttp = new ActiveXObject("Microsoft.XMLHTTP");
xmlhttp.open("GET", "/api/example", true);
xmlhttp.onreadystatechange = function () {
  if (xmlhttp.readyState == 4) {
    alert(xmlhttp.responseText);
  }
};
xmlhttp.send(null);
```

XMLHTTP later became the [XMLHttpRequest](https://xhr.spec.whatwg.org/) defacto standard due to its adoption by other browsers. This was about the time that the term "AJAX" was coined, standing for _Asynchronous JavaScript and XML_. The `JSON` standard later came along and made everything better, but the _X_ in _AJAX_ (not to mention the _XML_ in _XMLHttpRequest_) never really went away, despite actual XML-formatted data having largely disappeared from standard messaging payloads.

Comparative example of modern use of the `XMLHttpRequest` object:

```js
const req = new XMLHttpRequest();
req.addEventListener("load", () => console.log(this.responseText));
req.open("GET", "/api/example");
req.send();
```

As you can see, the modern analog is much the same as the original, albeit with a few additions to make things a little more concise.

### A world of new possibilities

In any case, XMLHttpRequest was still following the same HTTP request-response model used to retrieve the original HTML document. There was no real notion of allowing a server to proactively contact the user, or to establish any kind of general two-way connection for more sophisticated use cases. All the while though, JavaScript was gaining new features and capabilities as time went on, and browsers were enhancing the document object model (DOM), leading to greater and greater potential with respect to how JavaScript could be used to enrich the user's experience of interacting with a web page.

As the potential for really rich experiences started to become apparent, developers naturally gravitated toward the ideal of client-server applications implemented directly in the browser. Prior to this, the standard paradigm for anything non-trivial was to build a dedicated software application, package it up with an installer, have the user download the installer and then install it natively on their machine. Needless to say, this was quite a barrier to entry for all but the tech-savvy user. Keeping the application up to date with fixes and enhancements was a challenge all of its own. It is easy to understand, then, how alluring it was to be able to build an application that neither required installation before it could be accessed, nor user training and nagging in order to iterate on the software's implementation.

### Hatching the real-time web

> &ldquo;Innovation is taking two things that already exist and putting them together in a new way.&rdquo;  
> _- [Tom Freston](https://en.wikipedia.org/wiki/Tom_Freston)_

When something appears to be technically possible, and the potential reward is worth the effort, we'll usually go to great lengths to bend what is available into a shape that serves our needs. Thus, developers took XMLHttpRequest and abused it to start emulating real-time back-and-forth communication between the web page and the server. Some of the techniques to do this became commonplace - "standard", even - and eventually started being referred to with the umbrella term _"[Comet](https://en.wikipedia.org/wiki/Comet_(programming))"_.

Probably the most common of these techniques was (is) [_long polling_](https://en.wikipedia.org/wiki/Push_technology#Long_polling). This involves opening an XMLHttpRequest connection to the server and leaving it open until ongoing communication is no longer required. Under normal circumstances when making an HTTP request, the server's response is streamed back to a client via the connection on which the request was made, the intent being to allow a browser to start rendering an HTML page while waiting for the next part of the document to be delivered by the server. With long polling, the effect of leaving an HTTP connection open is that the server can continue to deliver response data for as long as the connection remains open, and there is no technical requirement that the data be in one format or another, or that the request be closed after sending data to the client. The same applies to the HTTP request payload sent by the client. A server may begin delivering its response before the client's request data has arrived in its entirety, and the client is not strictly required to stop sending request data until it chooses to do so. This means that, just as the server may continue delivering response data for the life of the connection, the client may do the same, resulting in a defacto two-way communication stream between server and client.

As enabling as it has been for the web application developer community in the absence of other suitable tools, long polling is tricky to do properly and is fraught with unexpected complications that must be managed. The same goes for other Comet techniques, which are beyond the scope of this article. All in all, long polling is really just a case of MacGuyvering the tools available in order to do something they weren't really designed to do.

A real solution was needed - something which would empower developers with proper TCP/IP socket-style capabilities in a web environment. Such a solution would need to be built _for the web_, and it would need to address all of the concerns that arise when operating in a web environment.

### Enter, WebSockets

Around the middle of 2008, the pain and limitations of using Comet when implementing anything truly robust were being felt particularly keenly by developers [Michael Carter](https://en.wikipedia.org/wiki/Michael_Carter_(entrepreneur)) and [Ian Hickson](https://en.wikipedia.org/wiki/Ian_Hickson). Through collaboration [on IRC](https://krijnhoetmer.nl/irc-logs/whatwg/20080618#l-1145) and [W3C mailing lists](https://lists.w3.org/Archives/Public/public-whatwg-archive/2008Jun/0165.html), they hatched a plan to introduce a new standard for modern real-time, bidirectional communication on the web, and thus [the name _"WebSocket"_ was coined](https://lists.w3.org/Archives/Public/public-whatwg-archive/2008Jun/0186.html).

The idea made its way into the W3C HTML draft standard and, shortly after, Michael Carter wrote [an article introducing the Comet community to the WebSockets.](http://cometdaily.com/2008/07/04/html5-websocket/) In 2010, Google Chrome 4 was the first browser to ship full support for WebSockets, with other browser vendors following suit over the course of the next few years. In 2011, [RFC 6455 - _The WebSocket Protocol_](https://tools.ietf.org/html/rfc6455) - was published to the IETF website.

Today, [all major browsers have full support for WebSockets](https://caniuse.com/#feat=websockets), even including Internet Explorer 10 and 11. In addition, browsers on both iOS and Android have supported WebSockets since 2013, which means that, all in all, the modern landscape for WebSocket support is very healthy. Much of the "internet of things" runs on some version of Android as well, so WebSocket support on other types of devices is reasonably pervasive too, as of 2018.

### So what exactly are WebSockets, anyway?

In a nutshell, WebSockets are a thin transport layer built on top of a device's TCP/IP stack. The intent is to provide what is essentially an as-close-to-raw-as-possible TCP communication layer to web application developers, while adding a few abstractions to eliminate certain friction that would otherwise exist with respect to the way the web works. They also cater for the fact that the web has additional security considerations that must be taken into account in order to protect both consumers and service providers.

You may have heard WebSockets simultaneously referred to both as a "transport" and as a "protocol". The former is more correct, because while they are a protocol in the sense that a strict set of rules for establishing communication and enveloping the transmitted data must be adhered to, the standard does not take any stance regarding how the actual data payload is structured within the outer message envelope. In fact, part of the specification includes the option for the client and server to agree on a protocol with which the transmitted data will be formatted and interpreted. The standard actually refers to these as "subprotocols", in order to avoid issues of ambiguity in the nomenclature. Examples of subprotocols are JSON, XML, MQTT, WAMP, et al, and these can ensure agreement not only about the way the data is structured, but also the way communication must commence, continue and eventually terminate. As long as both parties understand what the protocol entails, anything goes. The WebSocket simply provides a transport layer over which that messaging process can be implemented, which is why most common subprotocols are not exclusive to WebSocket-based communications.

#### A quick note about authentication and authorization

Seeing as WebSockets is a thin layer built on top of TCP/IP, anything beyond the basic handshake and specification for message framing is really something that needs to be handled either on a per-application or per-library basis. [Quoting the RFC](https://tools.ietf.org/html/rfc6455#section-10.5):

> _This protocol doesn't prescribe any particular way that servers can authenticate clients during the WebSocket handshake.  The WebSocket server can use any client authentication mechanism available to a generic HTTP server, such as cookies, HTTP authentication, or TLS authentication._

In a nutshell, use the HTTP-based authentication methods you'd use anyway, or use a subprotocol such as [MQTT](http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html) or [WAMP](https://wamp-proto.org/static/rfc/draft-oberstet-hybi-crossbar-wamp.html), both of which offer approaches for  authentication and authorization.

## Implementing a WebSocket Server (with Node.js)

In this section I'll take you step-by-step through the process of implementing a basic, no-frills WebSocket server. I'll be using Node.js for this (version 10.7 is installed on my machine at the time of this article). Your server may be running Go, .NET, Java, or something else, and the implementation in each of those environments will vary depending upon the HTTP server libraries available in each environment. The actual concepts will be fairly consistent though, as they all follow the same standard specifications for interpreting and constructing HTTP requests and responses, and for parsing and generating data that uses the WebSocket framing protocol. For now though, I'll assume you have at least some familiarity with Node.js.

> _Note: If you've never touched Node.js in your life, perhaps take a break before continuing with the article and find a couple of basic Node.js tutorials to get a feel for running a simple Node.js server on your machine, and to see how to use NPM to install packages, as I'll be assuming at least a little familiarity with these as we continue._

There are a lot of things to consider when writing a WebSocket server, and the intent here is just to demonstrate a starting point. Scalability, performance, connection recovery, robust handling of different edge cases, handling of large messages (e.g. 10kb+), etc. are beyond what I'll cover in this implementation, but are just a few of the things you'll want to think about if you take things further.

On the client side of things, unless you're implementing something custom outside of a browser environment, such as on some kind of custom mobile hardware, you don't really need to do anything special other than use the `WebSocket` class that is built into modern browsers by default. On the server, however, unless you're using a WebSocket server library you've installed, you'll need to handle the HTTP connection WebSocket upgrade handshake yourself. You'll then need read the raw binary data received via your HTTP socket connection and translate it according to the WebSocket framing protocol specification. This is outlined in [Section 5 of RFC 6455](https://tools.ietf.org/html/rfc6455#section-5). Finally, the server will need to construct its own messages according to the same specification, and dispatch them back to the client via the open socket connection.

#### Set up your project environment

In a new folder, make sure you've got a `package.json` file ready, then `npm install node-static`, which help us fast-track serving up your client-side files.

Create the following files:

**server.js:**
```js
const http = require('http');
const static = require('node-static');

const file = new static.Server('./');

const server = http.createServer((req, res) => {
  req.addListener('end', () => file.serve(req, res)).resume();
});

const port = 3210;
server.listen(port, () => console.log(`Server running at http://localhost:${port}`));
```

The `node-static` library is a convenience that takes care of the fiddly process of responding to requests for the static HTML and JavaScript files that will run on the client side in the browser.

**client.js:**
```js
console.log('WebSocket client script will run here.');
```

The above is just a stub so you know your client script has actually loaded correctly; we'll fill it out as we continue with the implementation further below.

**index.html:**
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>WebSocket Example</title>
    <script src="client.js"></script>
  </head>
  <body>
    <h1>WebSocket Example</h1>
    <p>Open your browser's developer tools to see client/server activity.</p>
  </body>
</html>
```

Then, from your terminal/console, run:

```sh
node server.js
```

You should see:
```
Server running at http://localhost:3210
```

Open up your browser. You should see the contents of `index.html`. Open the browser's developer tools and you should see the console output of your `client.js` script:

```
WebSocket client script will run here.
```

### Getting the ball rolling with HTTP

One of the early considerations when defining the WebSocket standard was to ensure that it "play nicely" with the web. This meant recognising that the web is generally addressed using URLs, not IP addresses and port numbers, and that a WebSocket connection should be able to take place with the same initial HTTP-based handshake used for any other type of web request.

Here's what happens in a simple HTTP GET request.

Let's say there's an HTML page hosted at http://www.example.com/index.html. Without getting too deep into [the HTTP protocol](https://tools.ietf.org/html/rfc2616) itself, it is enough to know that a request must start with what is referred to as [_Request-Line_](https://tools.ietf.org/html/rfc2616#section-5.1), followed by a sequence of key-value pair header lines, each telling the server something about what to expect in the subsequent request payload that that will follow the header data, and what it can expect from the client regarding the kinds of responses it will be able to understand.

The very first token in the request is the HTTP _method_, which tells the server the type of operation that the client is attempting with respect to the referenced URL. The `GET` method is used when the client is simply requesting that the server deliver it a copy of the resource that is referenced by the specified URL.

A barebones example of a request header, formatted according to the HTTP RFC, looks like this:

```
GET /index.html HTTP/1.1
Host: www.example.com
```

Having received the request header, the server then formats a response header starting with a [_Status-Line_](https://tools.ietf.org/html/rfc2616#section-6.1), followed by a set of key-value header pairs that provide the client with reciprocal information from the server, with respect to the request that the server is responding to. The _Status-Line_ tells the client the HTTP status code (usually `200` if there were no problems) and provides a brief "reason" text description explaining the status code. Key-value header pairs appear next, followed by the actual data that was requested (unless the status code indicated that the request was unable to be fulfilled for some reason).

```
HTTP/1.1 200 OK
Date: Wed, 1 Aug 2018 16:03:29 GMT
Content-Length: 291
Content-Type: text/html
(additional headers...)

(response payload continues here...)
```

So what's this got to do with WebSockets, you might ask?

### Ditching HTTP for something more appropriate

When making an HTTP request and receiving a response, the actual two-way network communication involved takes place over an active TCP/IP socket. The web URL that was requested in the browser is mapped via the global DNS system to an IP address, and the default port for HTTP requests is 80. This means that though a web URL was entered into the browser, the actual communication occurs via TCP/IP, using an IP address and port combination that looks something like, for example, `123.11.85.9:80`.

As we now know, WebSockets are built on top of the TCP stack as well, which means all we need is a way for the client and the server to jointly agree to hold the socket connection open and repurpose it for ongoing communication. If they do this, then there is no technical reason why they can't continue to use the socket to transmit any kind of arbitrary data, as long as they have both agreed as to how the binary data being sent and received should be interpreted.

To begin the process of repurposing the TCP socket for WebSocket communication, the client can include a standard request header that was invented specifically for this kind of use case:

```
GET /index.html HTTP/1.1
Host: www.example.com
Connection: Upgrade
Upgrade: websocket
```

The `Connection` header tells the server that the client would like to negotiate a change in the way the socket is being used. The accompanying value _`Upgrade`_ indicates that the transport protocol currently in use via TCP should change. Now that the server knows that the client wants to upgrade the protocol currently in use over the active TCP socket, the server knows to look for the corresponding `Upgrade` header, which will tell it which transport protocol the client wants to use for the remaining lifetime of the connection. As soon as the server sees `websocket` as the value of the `Upgrade` header, it knows that a WebSocket handshake process has begun.

_Note that the handshake process (along with everything else) is outlined in detail in [RFC 6455](https://tools.ietf.org/html/rfc6455), if you'd like to go into more detail than is covered in this article._

**Update your server.js code so that it can respond to an HTTP upgrade request:**

```js
server.on('upgrade', (req, socket) => {
  // Make sure that we only handle WebSocket upgrade requests
  if (req.headers['upgrade'] !== 'websocket') {
    socket.end('HTTP/1.1 400 Bad Request');
    return;
  }
  
  // More to come...
});
```

Node.js does most of the heavy lifting with respect to parsing HTTP requests received from the client, as you can see in the code above.All we need to do is listen for the `upgrade` event, then check that the `Upgrade` header is trying to switch to a WebSocket connection, and not something else unexpected. Once we've done that, we can complete the handshake and then move on to the business of sending and receiving WebSocket frame data.

_The fine-grained details of the handshake are outlined in [section 4 of the RFC](https://tools.ietf.org/html/rfc6455#section-4)_.

### Avoiding funny business

The first part of the WebSocket handshake, other than what is described above, involves proving that this is actually a proper WebSocket upgrade handshake, and that the process is not being circumvented or emulated via some kind of intermediate trickery either by the client or perhaps by a proxy server that sits in the middle.

When initiating an upgrade to a WebSocket connection, the client must include a `Sec-WebSocket-Key` header with a value unique to that client. Here's an example:

```
Sec-WebSocket-Key: BOq0IliaPZlnbMHEBYtdjmKIL38=
```

The above is automatic and handled for you if using the `WebSocket` class provided in modern browsers. You need only look for it on the server side and produce a response.

When responding, the server must append the special GUID value `258EAFA5-E914-47DA-95CA-C5AB0DC85B11` to the key, generate a SHA-1 hash of the resultant string, then include it as the base-64-encoded value of a `Sec-WebSocket-Accept` header that it includes in the response:

```
Sec-WebSocket-Accept: 5fXT1W3UfPusBQv/h6c4hnwTJzk=
```

In a Node.js WebSocket server, we could write a function to generate this value like so:

```js
const crypto = require('crypto');

function generateAcceptValue (acceptKey) {
  return crypto
    .createHash('sha1')
    .update(acceptKey + '258EAFA5-E914-47DA-95CA-C5AB0DC85B11', 'binary')
    .digest('base64');
}
```

We'd then need only call this function, passing the value of the `Sec-WebSocket-Key` header as the argument, and set the function return value as the value of the `Sec-WebSocket-Accept` header when sending the response.

To complete the handshake, write the appropriate HTTP response headers to the client socket. A bare-bones response would look something like this:

```
HTTP/1.1 101 Web Socket Protocol Handshake
Upgrade: WebSocket
Connection: Upgrade
Sec-WebSocket-Accept: m9raz0Lr21hfqAitCxWigVwhppA=
```

Update your `server.js` script like so:

```js
const http = require('http');
const crypto = require('crypto');
const static = require('node-static');

const file = new static.Server('./');

const server = http.createServer((req, res) => {
  req.addListener('end', () => file.serve(req, res)).resume();
});

server.on('upgrade', function (req, socket) {
  if (req.headers['upgrade'] !== 'websocket') {
    socket.end('HTTP/1.1 400 Bad Request');
    return;
  }

  // Read the websocket key provided by the client:
  const acceptKey = req.headers['sec-websocket-key'];

  // Generate the response value to use in the response:
  const hash = generateAcceptValue(acceptKey);

  // Write the HTTP response into an array of response lines:
  const responseHeaders = [
    'HTTP/1.1 101 Web Socket Protocol Handshake',
    'Upgrade: WebSocket',
    'Connection: Upgrade',
    `Sec-WebSocket-Accept: ${hash}`
  ];

  // Write the response back to the client socket, being sure to append two
  // additional newlines so that the browser recognises the end of the response
  // header and doesn't continue to wait for more header data:
  socket.write(responseHeaders.join('\r\n') + '\r\n\r\n');
});

// Don't forget the hashing function described earlier:
function generateAcceptValue (acceptKey) {
  return crypto
    .createHash('sha1')
    .update(acceptKey + '258EAFA5-E914-47DA-95CA-C5AB0DC85B11', 'binary')
    .digest('base64');
}
```

We're not actually quite finished with the handshake at this point - there are a couple more things to think about.

### Subprotocols; agreeing upon a shared dialect

The client and server generally need to agree on a compatible strategy with respect to how they format, interpret and organise the data itself, both within a given message, and over time from one message to the next. This is where subprotocols (mentioned earlier) come in. If the client knows that it can deal with one or more specific application-level protocols (such as WAMP, MQTT, etc.), it can include a list of the protocols it understands when making the initial HTTP request. If it does so, the server is then required to either select one of those protocols and include it in a response header, or to otherwise fail the handshake and terminate the connection.

Example subprotocol request header:

```
Sec-WebSocket-Protocol: mqtt, wamp
```

Example reciprocal header issued by the server in the response:

```
Sec-WebSocket-Protocol: wamp
```

Note that the server must select _exactly one protocol_ from the list provided by the client. Selecting more than one would mean that the data in subsequent WebSocket messages cannot be reliably or consistently interpreted by the server. An example of this would be if both `json-ld` and `json-schema` were selected by the server. Both are data formats built on the JSON standard, and there would be numerous edge cases where one could be interpreted as the other, leading to unexpected errors when processing the data. _While admittedly not messaging protocols per se, the example is still applicable._

When both the client and server are implemented to use a common messaging protocol from the outset, the `Sec-WebSocket-Protocol` header can be omitted in the initial request, in which case the server can ignore this step. Subprotocol negotiation is most useful when implementing general-purpose services, infrastructure and tools where there can be no forward guarantee that both the client and server will understand each other once the WebSocket connection has been established.

Standardised names for common protocols [should be registered](https://tools.ietf.org/html/rfc6455#section-11.5) with the [IANA registry for Websocket Subprotocol Names](https://www.iana.org/assignments/websocket/websocket.xml), which, at the time of this article, has 36 names already registered, including `soap`, `xmpp`, `wamp`, `mqtt`, et al. Though the registry is the canonical source for mapping a subprotocol name to its interpretation, the only strict requirement is that the client and server agree upon what their mutually-selected subprotocol actually means, irrespective of whether not it appears in the IANA registry.

As a purely illustrative demonstration of the use of subprotocols in the handshake, let's have the client and server both agree to use JSON-formatted data when communicating. As I mentioned above, nobody's going to be standing around with a gun, forcing the client and server to _only_ use an official subprotocol, or for the subprotocol to be an actual end-to-end messaging protocol. If we just want to agree on `json` as a subprotocol for the purposes of formatting client and server messages in a consistent way, this is a perfectly valid thing to do, and is what we'll be doing here. We'll add this to the server code as a starting point:

Directly after the `const responseHeaders = [ ... ];` statement:

```js
// Read the subprotocol from the client request headers:
const protocol = req.headers['sec-websocket-protocol'];

// If provided, they'll be formatted as a comma-delimited string of protocol
// names that the client supports; we'll need to parse the header value, if
// provided, and see what options the client is offering:
const protocols = !protocol ? [] : protocol.split(',').map(s => s.trim());

// To keep it simple, we'll just see if JSON was an option, and if so, include
// it in the HTTP response:
if (protocols.includes('json')) {
  // Tell the client that we agree to communicate with JSON data
  responseHeaders.push(`Sec-WebSocket-Protocol: json`);
}
```

Note that if the client has requested use of a subprotocol but hasn't provided any that the server is able to support, the server must send a failure response and close the connection. Though we won't bother with that in this implementation, you would need to do so if taking things further yourself.

#### WebSocket Extensions

There is also a header for defining extensions to the way the data payload is encoded and framed, but at the time of this article, only one standardised extension type exists, and it provides a kind of WebSocket-equivalent to gzip compression in messages. Another example of where extensions might come into play is multiplexing - the use of a single socket to interleave multiple concurrent streams of communication.

WebSocket extensions are a somewhat advanced topic, and are really beyond the scope of this article. For now it is enough to know what they are, and how they fit into the picture.

### The client side: using WebSockets in the browser

Before continuing with the server implementation, let's first take a look at the client side of the equation, as it's kind of hard to test the server code without a client to kick things off.

The WebSocket API is defined in the the [WHATWG HTML Living Standard](https://html.spec.whatwg.org/multipage/web-sockets.html#network) and actually pretty trivial to use. To begin, construct a `WebSocket` object:

```js
const ws = new WebSocket('ws://example.org');
```

Note the use of `ws` where you'd normally have the `http` scheme. There's also the option to use `wss` where you'd normally use `https`. These protocols were introduced in tandem with the `WebSocket` specification, and are designed to represent an HTTP connection that includes a request to upgrade the connection to use WebSockets.

Creating the `WebSocket` object isn't going to do a lot by itself. The connection is established asynchronously, so you'll need to listen for the completion of the handshake before sending any messages:

```js
ws.addEventListener('open', () => {
  // Send a message to the WebSocket server
  ws.send('Hello!');
});
```

Another event listener is required in order to receive messages from the server:

```js
ws.addEventListener('message', event => {
  // The `event` object is a typical DOM event object, and the message data sent
  // by the server is stored in the `data` property
  console.log('Received:', event.data);
});
```

There are also `error` and `close` events. WebSockets don't automatically recover when connections are terminated - this is something you need to implement yourself, and is part of the reason why there are many client-side libraries in existence. While the `WebSocket` class is straightforward and easy to use, it really is just a basic building block. Support for different subprotocols or additional features such as messaging channels must be implemented separately.

#### Give it a go yourself

As a convenience, a public echo server for testing WebSockets is hosted by websocket.org. Try the following code in your browser. The echo server receives whatever message you send it and echoes the message data back to the WebSocket from which the message originated.

Enter the following into your `client.js` script and run it, paying attention to the developer console:

```js
// Establish a WebSocket connection to the echo server
const ws = new WebSocket('wss://echo.websocket.org');

// Add a listener that will be triggered when the WebSocket is ready to use
ws.addEventListener('open', () => {
  const message = 'Hello!';
  console.log('Sending:', message);
  // Send the message to the WebSocket server
  ws.send(message);
});

// Add a listener in order to process WebSocket messages received from the server
ws.addEventListener('message', event => {
  // The `event` object is a typical DOM event object, and the message data sent
  // by the server is stored in the `data` property
  console.log('Received:', event.data);
});
```

You should see the "Hello!" message sent to the echo server and then received back in the `message` handler:

```
Sending: Hello!
Received: Hello!
```

Once you've done that, modify the code in preparation for communication with your own WebSocket server:

```js
const ws = new WebSocket('ws://localhost:3210', ['json', 'xml']);

ws.addEventListener('open', () => {
  const data = { message: 'Hello from the client!' }
  const json = JSON.stringify(data);
  ws.send(json);
});

ws.addEventListener('message', event => {
  const data = JSON.parse(event.data);
  console.log(data);
});
```

Notice that I've added an additional parameter during `WebSocket` construction - an array of subprotocols that will be sent via the `Sec-WebSocket-Protocol` request header. As per the server code provided earlier, the server will select `json` from the list and include it in the initial handshake response. The WebSocket API implemented by the browser will automatically fail the connection if the server does not select one of the specified subprotocols. During implementation, try having the server select a subprotocol that is inconsistent with what was received from the client during the initial request. The client should throw an error and terminate the connection immediately.

The above code is the entire client-side script we'll be using! Anything else you want to add is up to you, but the remainder of this implementation will take place on the server side.

### Generating and parsing WebSocket message frames

Once the handshake response has been sent to the client, we can begin sending and receiving messages between the client and server. Take a quick look at [section 5 of the RFC](https://tools.ietf.org/html/rfc6455#section-5) to get a sense of what's involved.

WebSocket messages are delivered in packages called "frames", which begin with a message header, and conclude with the "payload" - the message data for this frame. Large messages may split the data over several frames, in which case you'll need to keep track of what you've received so far, and piece the data together once it has all arrived.

#### Alignment of Node.js socket buffers with WebSocket message frames

Node.js socket data ([I'm talking about `net.Socket`](https://nodejs.org/api/net.html#net_class_net_socket) in this case, not WebSockets) is received in buffered chunks that are split apart with no regard for where your WebSocket frames begin or end! What this means for you is that if your server is receiving large messages fragmented into multiple WebSocket frames, and/or receiving large numbers of messages in rapid succession, _there's no guarantee that each data buffer received by the Node.js socket will align with the start and end of the byte data that makes up a given frame_. As such, as you're parsing each buffer received by the socket, you'll need to keep track of where one frame ends and where the next begins, and you'll need to be sure that you've received all of the bytes of data for a frame _before you can safely consume that frame's data_. It may be that one frame ends partway through the same buffer in which the next frame begins. It also may be that a frame is split across several buffers that will be received in succession.

The following diagram is an exaggerated illustration of the issue. In most cases frames tend to fit inside a buffer, and due to the nature of the way data arrives, you'll often find that a frame will start and end in line with the start and end of the socket buffer, but this can't be relied upon in all cases and must be considered during implementation.

![Diagram of Node.js socket buffers vs websocket frames](websockets-frames-vs-nodejs-socket-buffer.png)

This can take some work to get right. For the basic implementation that follows below, in addition to neglecting performance, scalability and just about everything else, I also have skipped any code for handling large messages or messages split across multiple frames. Doing so requires coordination of multiple data buffers across multiple frames, and would overcomplicate the exercise with logic that is more about Node.js sockets than it is about WebSockets. My purpose here is to demonstrate some of the low-level work required to parse WebSocket frames in pursuit of working two-way communication between a client and a server.

#### Implementing the parser

To get started, update `server.js` code with the following:

```js
socket.on('data', buffer => {
  const message = parseMessage(buffer);
  if (message) {
    // For our convenience, so we can see what the client sent
    console.log(message);
    
    // We'll just send a hardcoded message in this example
    socket.write(constructReply({ message: 'Hello from the server!' }));
  }
  else if (message === null) {
    console.log('WebSocket connection closed by the client.');
  }
});

function constructReply(data) {
  // TODO: Construct a WebSocket frame Node.js socket buffer
}

function parseMessage(buffer) {
  // TODO: Parse the WebSocket frame from the Node.js socket buffer
}
```

We'll start by implementing the `parseMessage` function.

The RFC is actually reasonably easy to follow. The sequence of bytes in a frame is [laid out in the RFC](https://tools.ietf.org/html/rfc6455#section-5.2) using to the following diagram:

[![Diagram of the WebSocket framing protocol according to RFC 6455](websocket-framing-protocol.png)](https://tools.ietf.org/html/rfc6455#section-5.2)

Let's break it down. There are two primary types of frames: message frames and control frames. Message frames carry message data (the "payload"), and control frames are used for other purposes, such as sending a ping or pong, indicating that the connection is about to close, and so forth.

WebSocket frames consists of a header, followed by the message payload, if applicable. The header always starts with two bytes (or 16 bits). The first bit tells us, for message frames, whether this is the last frame of the current message. This will always have the value `1` if your message frames are small (&lt; 126 bytes). Three reserved bits follow (we can generally ignore them), and them four bits for an "opcode", which tells us what sort of frame this is. After that, there is a single bit telling us whether the message payload is "masked" (more on this later), and then seven bits for the number of bytes in the payload. Seven bits isn't a lot though, so if the length is `126`, then we can expect an extra two bytes (16 bits) with the _actual_ payload length. Alternatively, if the length is 127, instead of two bytes, there will be an additional eight bytes to store the length as a 64-bit integer.

![Diagram of the first two bytes of the WebSocket framing protocol](websocket-framing-protocol-first-two-bytes.png)

#### Opcodes: message frames vs control frames

According to the RFC:

```
Opcode:  4 bits

Defines the interpretation of the "Payload data".  If an unknown
opcode is received, the receiving endpoint MUST _Fail the
WebSocket Connection_.  The following values are defined.

*  %x0 denotes a continuation frame
*  %x1 denotes a text frame
*  %x2 denotes a binary frame
*  %x3-7 are reserved for further non-control frames
*  %x8 denotes a connection close
*  %x9 denotes a ping
*  %xA denotes a pong
*  %xB-F are reserved for further control frames
```

Important points:

- A continuation frame is used when a message is split into multiple frames. Our implementation is only dealing with small messages, so we won't handle this case for now, but if you decide to do this yourself, keep in mind that when you encounter a continuation frame, it means that the opcode is the same as for the initial message frame in the sequence, up until the end of the final frame, which you'll identify via the first bit in the message header.
- We generally only care about text frames and binary frames for messages, and in our case, we'll only be supporting text frames. You might use binary frames for images, audio, and so forth.
- You do need to support the `0x8` - the "close" opcode - otherwise you won't know that the client has disconnected.

Let's implement this:

```js
function parseMessage (buffer) {
  const firstByte = buffer.readUInt8(0);

  const isFinalFrame = Boolean((firstByte >>> 7) & 0x1);
  const [reserved1, reserved2, reserved3] = [
    Boolean((firstByte >>> 6) & 0x1),
    Boolean((firstByte >>> 5) & 0x1),
    Boolean((firstByte >>> 4) & 0x1)
  ];
  const opCode = firstByte & 0xF;

  // We can return null to signify that this is a connection termination frame
  if (opCode === 0x8) return null;

  // We only care about text frames from this point onward
  if (opCode !== 0x1) return;

  const secondByte = buffer.readUInt8(1);
  const isMasked = Boolean((secondByte >>> 7) & 0x1);
  
  // Keep track of our current position as we advance through the buffer
  let currentOffset = 2;
  let payloadLength = secondByte & 0x7F;

  if (payloadLength > 125) {
    if (payloadLength === 126) {
      payloadLength = buffer.readUInt16BE(currentOffset);
      currentOffset += 2;
    }
    else { // 127
      // If this has a value, the frame size is ridiculously huge!
      const leftPart = buffer.readUInt32BE(currentOffset);
      const rightPart = buffer.readUInt32BE(currentOffset += 4);

      // Honestly, if the frame length requires 64 bits, you're probably doing it wrong.
      // In Node.js you'll require the BigInt type, or a special library to handle this.
      throw new Error('Large payloads not currently implemented');
    }
  }
}
```

#### Masked frames

Frames sent by the client may be "masked" (and when the client is a browser, this will always be the case). What this means is that the browser will include a special four-byte "masking key" which must be XOR'd against each consecutive four-byte sequence in the message payload, with the result being the actual data. Masking is a mechanism that ensures that message frames retain their integrity, and are not interfered with by third parties, such as intermediate proxy servers.

If the second byte of the initial header starts with the first bit set to `1`, then the masking key will occupy the four bytes that follow the payload length (including the extended payload length, if present).

Start by reading the masking key from the buffer:

```js
let maskingKey;
if (isMasked) {
  maskingKey = buffer.readUInt32BE(currentOffset);
  currentOffset += 4;
}
```

Next, read the data from the buffer:

```js
// Allocate somewhere to store the final message data
const data = Buffer.alloc(payloadLength);

// Only unmask the data if the masking bit was set to 1
if (isMasked) {
  // Loop through the source buffer one byte at a time, keeping track of which
  // byte in the masking key to use in the next XOR calculation
  for (let i = 0, j = 0; i < payloadLength; ++i, j = i % 4) {
    // Extract the correct byte mask from the masking key
    const shift = j === 3 ? 0 : (3 - j) << 3;
    const mask = (shift === 0 ? maskingKey : (maskingKey >>> shift)) & 0xFF;

    // Read a byte from the source buffer
    const source = buffer.readUInt8(currentOffset++);

    // XOR the source byte and write the result to the data buffer
    data.writeUInt8(mask ^ source, i);
  }
}
else {
  // Not masked - we can just read the data as-is
  buffer.copy(data, 0, currentOffset++);
}
```

The `data` buffer should now hold the data sent by the client, which means all we have to do is decode the buffer to a string:

```js
function parseMessage (buffer) {
  // ...all of the above, then:

  const json = data.toString('utf8');
  return JSON.parse(json);
}
```

#### Constructing a frame buffer for the response message

If you were able to successfully get all of the above working, then the response should be fairly trivial, especially because messages sent by the server _should not be masked_, according to the RFC.

To save time, here's the implementation:

```js
function constructReply (data) {
  // Convert the data to JSON and copy it into a buffer
  const json = JSON.stringify(data)
  const jsonByteLength = Buffer.byteLength(json);
  
  // Note: we're not supporting > 65535 byte payloads at this stage
  const lengthByteCount = jsonByteLength < 126 ? 0 : 2;
  const payloadLength = lengthByteCount === 0 ? jsonByteLength : 126;
  const buffer = Buffer.alloc(2 + lengthByteCount + jsonByteLength);

  // Write out the first byte, using opcode `1` to indicate that the message
  // payload contains text data
  buffer.writeUInt8(0b10000001, 0);
  buffer.writeUInt8(payloadLength, 1);

  // Write the length of the JSON payload to the second byte
  let payloadOffset = 2;
  if (lengthByteCount > 0) {
    buffer.writeUInt16BE(jsonByteLength, 2);
    payloadOffset += lengthByteCount;
  }

  // Write the JSON data to the data buffer
  buffer.write(json, payloadOffset);
  
  return buffer;
}
```

#### Dispatching the response back to the client

You should already have the following code from earlier in the article, but I'll repeat it here in order to wrap things up:

```js
socket.on('data', buffer => {
  const message = parseMessage(buffer);
  if (message) {
    // For our convenience, so we can see what the client sent
    console.log(message);
    
    // We'll just send a hardcoded message in this example
    socket.write(constructReply({ message: 'Hello from the server!' }));
  }
  else if (message === null) {
    console.log('WebSocket connection closed by the client.');
  }
});
```

At this point you can run your server from the command line, then open your browser to `localhost` on port `3210`, and, making sure you have the developer tools open, you'll see the browser send a message and the server respond accordingly.

Remember - my implementation is not even remotely complete, nor is it particularly efficient. There are many existing WebSocket server implementations out there already, and we'll take a look at some of them next.

## WebSocket libraries you can use right now

There are two primary classes of WebSocket libraries; those that implement the protocol and leave the rest to the developer, and those that build on top of the protocol with various additional features commonly required by realtime messaging applications, such as restoring lost connections, pub/sub and channels, authentication, authorization, etc. The latter variety often require that their own libraries be used on the client side, rather than just using the raw `WebSocket` API provided by the browser. As such, it becomes important to make sure you're happy with how they work and what they're offering. You may find yourself locked into your chosen solution's way of doing things once it has been integrated into your architecture, and any issues with reliability, performance and extensibility may come back to bite you.

I'll start with a list of those that fall into the first of the above two categories;

### ws

[ws](https://github.com/websockets/ws) is a _"simple to use, blazing fast and thoroughly tested WebSocket client and server for Node.js"_. It is definitely a barebones implementation, designed to do all the hard work of implementing the protocol, however additional features such as connection restoration, pub/sub and so forth, are concerns you'll have to manage yourself.

#### Minimal code samples

**Client (Browser, before bundling):**
```js
const WebSocket = require('ws');

const ws = new WebSocket('ws://www.host.com/path');

ws.on('open', function open() {
  ws.send('something');
});

ws.on('message', function incoming(data) {
  console.log(data);
});
```

**Server (Node.js):**

```js
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', function connection(ws) {
  ws.on('message', function incoming(message) {
    console.log('received: %s', message);
  });

  ws.send('something');
});
```

### μWebSockets

[μWS](https://github.com/uNetworking/uWebSockets) is a drop-in replacement for [ws](#ws), implemented with a particular focus on performance and stability. To the best of my knowledge, μWS is the fastest WebSocket server implementation available by a mile. It's actually used under the hood by [SocketCluster](#socketcluster), which I'll talk about below. 

There has been a little [controversy](https://www.reddit.com/r/node/comments/91kgte/uws_has_been_deprecated/) around μWS recently due to the author having attempted to pull it from NPM for philosophical reasons, but [the latest working version](https://www.npmjs.com/package/uws/v/10.148.1) remains on NPM and can be installed specifying that version explicitly when installing from NPM. That said, the author is working on [a new version](https://github.com/uNetworking/v0.15), with accompanying [node.js bindings](https://github.com/uNetworking/uWebSockets-node) also [in development](https://github.com/uNetworking/uWebSockets-node/issues/2).

#### Minimal code sample

**Server (Node.js):**

```js
var WebSocketServer = require('uws').Server;
var wss = new WebSocketServer({ port: 3000 });
 
function onMessage(message) {
  console.log('received: ' + message);
}
 
wss.on('connection', function(ws) {
  ws.on('message', onMessage);
  ws.send('something');
});
```

### faye-websocket

[faye-websocket](https://github.com/faye/faye-websocket-node) is a standards-compliant WebSocket implementation for both the client and server, and originated from the Ruby-on-Rails community as part of the [Faye project](https://faye.jcoglan.com/). As per the Github, project:

> It does not provide a server itself, but rather makes it easy to handle WebSocket connections within an existing Node application. It does not provide any abstraction other than the standard WebSocket API.

In the sample code below for the server, you can see the similarity to our own implementation earlier in this article. The difference of course is that all the work of handling the connection upgrade and translating messaging frames from inbound socket buffers is handled by the `WebSocket` class provided for you. As with other minimal solutions, this is a no-frills implementation - you'll need to handle application-specific concerns yourself.

#### Minimal code samples

**Client (Browser, before bundling):**

```js
var WebSocket = require('faye-websocket'),
    ws        = new WebSocket.Client('ws://www.example.com/');

ws.on('open', function(event) {
  console.log('open');
  ws.send('Hello, world!');
});

ws.on('message', function(event) {
  console.log('message', event.data);
});

ws.on('close', function(event) {
  console.log('close', event.code, event.reason);
  ws = null;
});
```

**Server (Node.js):**

```js
var WebSocket = require('faye-websocket'),
    http      = require('http');

var server = http.createServer();

server.on('upgrade', function(request, socket, body) {
  if (WebSocket.isWebSocket(request)) {
    var ws = new WebSocket(request, socket, body);
    
    ws.on('message', function(event) {
      ws.send(event.data);
    });
    
    ws.on('close', function(event) {
      console.log('close', event.code, event.reason);
      ws = null;
    });
  }
});

server.listen(8000);
```

### Socket.io

[Socket.io](https://github.com/socketio/socket.io) has been around for a while now; I like to think of it as the "jQuery" of WebSockets. It uses long polling and WebSockets for its transports, by default starting with long polling and then upgrading to WebSockets if available. Given the waning relevance of long polling, Socket.io's main drawcards these days are its [other features](https://stackoverflow.com/questions/38546496/moving-from-socket-io-to-raw-websockets/38546537#38546537), such as restoring dropped connections, automatic support for JSON, and "namespaces", which are essentially isolated messaging channels multiplexed over the same client connection.

Socket.io isn't actually interchangeable with generic WebSockets solutions - either on the server side or on the client side - and attempting to connect to something other than a Socket.io client or server will fail. It has its own additional handshake protocol, and some additional metadata included in each message.

Should you use it? On the plus side, in addition to [the simplicity of getting up and running](https://socket.io/get-started/chat/), it's well-established and there is a wealth of learning material out there if you get stuck. On the other hand, there have been plenty of reports of memory leaks and general performance issues over time, and despite there being [a mountain of issues](https://github.com/socketio/socket.io/issues) in the Github repository, responses are few, and [commits and updates](https://github.com/socketio/socket.io/commits/master) these days seem relatively infrequent. Just like jQuery, I think Socket.io is a product of a bygone era, and for new projects you'd be better off with something a little more modern. jQuery does still have its fans though, so...

#### Minimal code samples

**Client (Browser, before bundling):**

```js
const io = require('socket.io-client');
const socket = io();
socket.emit('chat message', 'Hello there');
```

**Server (Node.js):**

```js
var app = require('express')();
var http = require('http').Server(app);
var io = require('socket.io')(http);

app.get('/', function(req, res){
  res.sendFile(__dirname + '/index.html');
});

io.on('connection', function(socket){
  console.log('a user connected');
});

http.listen(3000, function(){
  console.log('listening on *:3000');
});
```
### SocketCluster

[SocketCluster](https://socketcluster.io) is a full-featured client-server messaging framework built entirely around WebSockets, and uses [μWebSockets](#%CE%BCwebsockets) under the hood. From the website:

> SocketCluster is an open source real-time framework for Node.js. It supports both direct client-server communication and group communication via pub/sub channels. It is designed to easily scale to any number of processes/hosts and is ideal for building chat systems. See SocketCluster Design Patterns for Chat.

Unlike simpler solutions such as Socket-io, SocketCluster requires slightly more installation, but it generally very easy to get up and running.

#### Minimal code samples

These are taken from the [getting started guide](https://socketcluster.io/#!/docs/getting-started) in the [SocketCluster documentation](https://socketcluster.io/#!/docs).

**Client (Browser, before bundling):**

```js
var socket = socketCluster.create();
socket.emit('sampleClientEvent', {message: 'This is an object with a message property'});
```

**Server (Node.js):**

```js
var SocketCluster = require('socketcluster');
var socketCluster = new SocketCluster({
  workers: 1, // Number of worker processes
  brokers: 1, // Number of broker processes
  port: 8000, // The port number on which your server should listen
  appName: 'myapp', // A unique name for your app

  // Switch wsEngine to 'sc-uws' for a MAJOR performance boost (beta)
  wsEngine: 'ws',

  /* A JS file which you can use to configure each of your
   * workers/servers - This is where most of your backend code should go
   */
  workerController: __dirname + '/worker.js',

  /* JS file which you can use to configure each of your
   * brokers - Useful for scaling horizontally across multiple machines (optional)
   */
  brokerController: __dirname + '/broker.js',

  // Whether or not to reboot the worker in case it crashes (defaults to true)
  rebootWorkerOnCrash: true
});
```

### SockJS

[SockJS](http://sockjs.org) has as its primary feature WebSocket _emulation_, which puts it into an increasingly-outdated class of solutions, given that support for WebSockets is pretty pervasive these days. From their client repository:

> SockJS is a browser JavaScript library that provides a WebSocket-like object. SockJS gives you a coherent, cross-browser, Javascript API which creates a low latency, full duplex, cross-domain communication channel between the browser and the web server.

It supports numerous [fallback transports](https://github.com/sockjs/sockjs-client#supported-transports-by-name), giving it a fairly robust suite of support for Comet techniques. Again though, how relevant is this in the modern web landscape?

See _[SockJS: web messaging ain't easy](https://github.com/sockjs/sockjs-client/wiki/%5BArticle%5D-SockJS:-web-messaging-ain%E2%80%99t-easy)_ and _[SockJS: WebSocket emulation done right](https://github.com/sockjs/sockjs-client/wiki/%5BArticle%5D-SockJS:-WebSocket-emulation-done-right)_ for some insight into the philosophy behind the SockJS suite of libraries.

On the server side, check out the [SockJS-node](https://github.com/sockjs/sockjs-node) project.

#### Minimal code samples

**Client (Browser, before bundling):**

```js
var sock = new SockJS('https://mydomain.com/my_prefix');
sock.onopen = function() {
  console.log('open');
  sock.send('test');
};

sock.onmessage = function(e) {
  console.log('message', e.data);
  sock.close();
};

sock.onclose = function() {
  console.log('close');
};
```

**Server (Node.js):**

```js
var http = require('http');
var sockjs = require('sockjs');

var echo = sockjs.createServer({ sockjs_url: 'http://cdn.jsdelivr.net/sockjs/1.0.1/sockjs.min.js' });
echo.on('connection', function(conn) {
  conn.on('data', function(message) {
    conn.write(message);
  });
  conn.on('close', function() {});
});

var server = http.createServer();
echo.installHandlers(server, {prefix:'/echo'});
server.listen(9999, '0.0.0.0');
```

### deepstream server

As per the [deepstream server website](https://deepstreamhub.com/open-source/), _"deepstream is a powerful websocket server that syncs realtime data between browsers, smartphones, backends and the IoT"_. Unlike the other server-based libraries above, deepstream server is its own server application which must be downloaded and installed on your server. Bindings are provided for [integration with different languages and environments](https://deepstreamhub.com/docs/), such as Node.js and Java. As a full-featured server, it comes with a full suite of capabilities, including pub/sub channels, presence, authentication, and more.

DeepStream Server is open source, and the company that makes it makes money through their cloud-based [deepstreamHub](https://deepstreamhub.com) platform-as-a-service offering.

## Scaling beyond a single server

The number of concurrent connections a server can handle is rarely the bottleneck when it comes to server load. Most decent WebSocket servers can support hundreds of thousands or even millions of concurrent connections, but what's the workload required to process and respond to messages once the WebSocket server process has handled receipt of the actual data? Typically there will be all kinds of potential concerns, such as reading and writing to and from a database, integration with a game server, allocation and management of resources for each client, and so forth. As soon as one machine is unable to cope with the workload, you'll need to start adding additional servers, which means now you'll need to start thinking about load-balancing, synchronization of messages among clients connected to different servers, generalised access to client state irrespective of connection lifespan or the specific server that the client is connected to - the list goes on and on.

Such concerns are deserving of an article all of their own, and you'll find plenty on the [Ably blog](https://blog.ably.io) - see in particular Matt O'Riordan's writeup on [some of the complexities he had to deal with when building Ably](https://blog.ably.io/the-story-of-how-ably-came-to-be-1d859f2c9a90).

## Realtime messaging platforms-as-a-service

### Ably

### PubNub

### Pusher

### deepstreamHub