---
title: 'Windows is not Unix'
date: '2012-09-25'
description:
categories: ['windows','php','node','code','cli']
---

The command line doesn't get enough love.  

Almost every developer leans heavily on the command line, but one can't help shake the feeling that the console is slowly drifting toward irrelevance and neglect.  As less and less of our data can be expressed in terms of plaintext, the core Unix toolset also drifts toward irrelevance.

That's not to say these tools aren't useful.  However, these tools are failing to keep pace with the evolution of computing.  One only needs to look at [Mosh](http://mosh.mit.edu/#techinfo)'s heroic attempt to fix SSH to see just how broken (and neglected) our consoles have become.  The perfectionists at Apple even managed to break `cat` [two years ago](https://profiles.google.com/110440139189906861022/buzz/LGNWMv1LFoJ), and still haven't fixed it.  Nobody would accept these kinds of bugs in a GUI application, so why do we put up with them in the console?  

If that's the state of the Unix CLI, I'm sure you can imagine what things are like on Windows...

At the moment, I'm developing a web interface for a script that performs a few post-processing tasks on our video files so that they're in the right format for our CDN.  This isn't a particularly challenging task, and indeed, the synchronous version of this script only requires about six lines of code.

However, it's generally bad form for web scripts to take 10 minutes to execute, and even worse form for a process to disappear into the background/abyss with no way of monitoring it.  So, we launch the process in the background, and periodically monitor/poll its output.

Luckily, asynchronous I/O is currently a very hot topic, and we have a lot of great libraries for making this task as simple as possible.  [Node.js](http://nodejs.org/) leads the pack, but Node's influence is swiftly spreading to other languages.  [React](http://nodephp.org/) implements the Observer Pattern and much of Node's API in PHP.  

Modern PHP actually has a rather nice syntax, and React makes it dead-simple to open up a process, and pipe its output straight into a WebSocket connection (provided by React's cousin, [Ratchet](http://socketo.me/).  Once we've got our event loop set up, and Ratchet's WebSocket server is running, we can do this with just a few lines of code:<pre class="lang-php">
use React\Stream\Stream as Stream;
$ping = new Stream(popen("ping -t 127.0.0.1",'r'),$server->loop);
$ping->on('data',function($data) use ($websocket){
    $websocket->send($data);
});</pre>

Pretty neat, right?  From the `data` callback, I can listen to my external program's output in the background, provide updates on its status/activity, and trigger some further actions once it finishes up (based on its exit code).

Unfortunately, the devil is in the details.  Asynchronous I/O is a relatively new concept for Windows, and almost nobody implements it correctly.  [Including PHP](https://bugs.php.net/bug.php?id=47918).  

While the above code works great with many external commands, the `fread()` call that React uses to poll the process's output will block until some data has been written to the buffer.  Worse still, because React implements a single-threaded event loop, this `fread()` call will block the _entire_ application until it reads some data.

We first noticed this with the [Adobe F4V Post-Processor](http://www.adobe.com/products/flash-media-server-family/tool-downloads.html) (`f4vpp`).  Spawning `f4vpp` using the method described above blocks the entire event loop for the 5 minutes that it takes to run.  `f4vpp`'s welcome message doesn't even make it to the Websocket until the whole thing finishes running.  If we decide to run `ping` simultaneously, its output also gets suppressed until `f4vpp` finishes up.

Maybe the haters were right.  Maybe PHP just wasn't built to live in the modern world.  [Node](http://nodejs.org) has asynchronicity built in from the core, and its backend I/O library, [libuv](https://github.com/joyent/libuv), claims to properly asynchronous I/O properly on Windows.  

Let's implement the same code on Node:<pre class"lang-js">
var spawn = require('child_process').spawn,
    ping = spawn('ping',['-t','127.0.0.1']);
ping.stdout.setEncoding('utf8');
ping.stdout.on('data',function(data){
	websocket.emit('message',{'stdout': data});
});</pre>

Look familiar?  As before, `ping`'s output gets sent to our client via a WebSocket connection.  

However, our old friend, `f4vpp` shows the same problems as it did before.  Node's `data` event doesn't fire until the entire application has finished running.  Unlike PHP, however, the rest of the application keeps churning along.

What's going on here?  Why can I see `f4vpp`'s output when I run it in the terminal, but that same output is invisible to Node and PHP?  Let's try redirecting `f4vpp`'s output to a file, and examine what happens to that file.

In one terminal, I run `f4vpp -v -i in.f4v -o out.mp4 > o.log`, and in another, I follow the file with `tail -f o.log`.  Just like we observed in Node, nothing appears in our `stdout.log` file until `f4vpp` has finished. (By the way, you can get a win32-native `tail` from [gow](https://github.com/bmatzelle/gow/wiki))

It turns out, this is all Adobe's fault.  `f4vpp` writes its output to `stdout`, but never flushes that stream's buffer.  If the application isn't explicitly calling `fflush()`, it's up to the OS to decide when to flush the output buffer.  Usually, this happens [in one of three ways](http://www.pixelbeat.org/programming/stdio_buffering/):
* When the buffer is full (usually 4KiB).  `f4vpp` doesn't write a lot to the console, so this never happens.
* When the process exits.  This is why Node and PHP are able to see any output _at all_ from `f4vpp`.
* When a newline character is written to the stream, *and* `stdout` is a terminal.  This is why `f4vpp` displays its output normally when we run it interactively from the terminal.

Where does this leave us?  We don't control Adobe's source code, and can't modify their application to improve its behavior.  On some Linux-based systems, we could use the [`stdbuf`](http://www.pixelbeat.org/programming/stdio_buffering/stdbuf-man.html) command to force `f4vpp` to give us unbuffered or line-buffered output.  Sadly, no equivalent command exists on Windows systems.  

Alternatively, we can use [expect](http://expect.sourceforge.net/) to interact with the external process, and unbuffer its output.  It's a bit of a kludge, but expect's [`unbuffer`](http://www.linuxcommand.org/man_pages/unbuffer1.html) script does work on ActiveState's [win32 port of expect](http://www.activestate.com/activetcl/expect) (which is part of their TCL distribution).  The `unbuffer` script isn't bundled with ActiveState's Expect, so you'll have to [download that separately](http://expect.cvs.sourceforge.net/viewvc/expect/expect/example/unbuffer?revision=5.34).

Repeating our previous `tail -f` experiment with `tclsh unbuffer f4vpp....` works like we'd expect it to, with similar (good!) results on Node and PHP.  Unfortunately, this is all very cumbersome, and I'm not sure I'd recommend it for production code.  

Theoretically, we can compile this into a self-contained executable.  Unfortunately, ActiveState's Expect isn't open-source, and the other win32 Expect distributions are ancient.  Perhaps some C++ developer will come to the rescue and fix this for us...