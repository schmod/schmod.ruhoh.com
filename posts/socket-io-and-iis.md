---
title: Socket.IO and IIS
date: '2012-10-02'
description:
categories: ['windows','node','code','websockets','socket.io']
---

[WebSockets](https://developer.mozilla.org/en-US/docs/WebSockets) are pretty great.  I won't go into too much detail here, but the low-latency persistent connections that WebScokets provide have a temendous potential for boosting many web applications.  

Unfortunately, browser support for WebSockets is still spotty, although we've had the basic building blocks for providing WebSocket-like functionality for some time now.  [Socket.IO](http://socket.io) patches those building blocks together, and provides WebSocket-like functionality on just about any browser imaginable.  

As it turns out, a few *servers* also don't quite support WebSockets yet.  Long story, but until the Windows world [migrates to IIS 8](http://www.paulbatum.com/2011/09/getting-started-with-websockets-in.html), there's also no WebSocket support in IIS.  Fortunately, the Socket.IO's Node.js-based server component lets us use [AJAX Long Polling](http://en.wikipedia.org/wiki/Comet_%28programming%29#Ajax_with_long_polling) to simulate the same functionality, with WebSocket-like semantics.  It's not perfect, but it works, and be modified to use actual WebSockets with almost no code changes (once you get a server that supports "real" WebSockets).

In any event, if you're stuck with IIS 7.5, you can still use Socket.IO today.  This isn't particualrly well-documented, and there are some [needlessly complicated workarounds](https://github.com/tjanczuk/iisnode/issues/35) floating around, so here's how you should do it:

1. Install [Node](http://nodejs.org/) and [iisnode](https://github.com/tjanczuk/iisnode).  (More on that [here](http://tomasz.janczuk.org/2011/08/hosting-nodejs-applications-in-iis-on.html))
2. In your node application's folder, grab socket.io with npm:
`npm install socket.io`
3. Create your Socket.IO server application, and a client.  Be sure to [configure](https://github.com/LearnBoost/Socket.IO/wiki/Configuring-Socket.IO) Socket.IO with a `resource` parameter on both your client and server.

Here's a barebones example to get you started.  In practice, you'll probably want to use URL rewriting (and [Express](http://expressjs.com/)) so you can also serve up other content.

On the server (`myApp/server.js`):
<pre class"lang-js">
var app = require('http').createServer(handler),
    io = require('socket.io').listen(app);<br>
io.configure(function(){
  io.set('transports',['xhr-polling']); //Use long-polling instead of websockets!
  io.set('resource','/myApp/server.js'); //Where we'll listen for connections.
});<br>
app.listen(process.env.PORT);<br>
function handler(req,res){
  /*
   *  Your code for handling non-websocket requests...
   *  Note that Socket.IO will intercept any requests made
   *  to myApp/server.js without calling this handler. 
   */
}<br>
io.sockets.on('connection', function (client) {
  console.log("New Connection");
  client.on('message',function(message,callback){
    console.log("Message Received");
    client.emit('response',"Got it!");
  }
}
</pre>
And on the client (which can be hosted anywhere):
<pre class"lang-html">
&lt;script src="/path-to-socket.io-client/socket.io.js"&gt;&lt;/script&gt;
&lt;script&gt;
  var socket = io.connect('http://myServer',{resource: 'myApp/server.js'}); //Same resource as the server.
  socket.on('response', function (msg) {
    console.log("Message received:",msg);
  });
  socket.emit('message','hi');
&lt;/script&gt;
</pre>

In practice, you'll probably want to set up URL rewriting to rewrite requests for anything inside `/myApp/*` to `server.js`.  In my case, I rewrite `myApp/socket` to `server.js` in `web.config`, and specify `myApp/socket` as the `resource` parameter on both the client and server.  