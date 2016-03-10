## Conversations about enchannel and Jupyter messaging

The following is a rough transcript between two developers, one familiar with Jupyter (J) and one who is just getting started (F). 

> F: I'm playing with `ick` now to get a better understanding of the channels. After this I'll construct some feedback on how we can do the architecture that uses them.

> J: I'm really happy you're diving into grokking the channels

> F: reading the `ick` source now

> J: There are ways to hack with that stuff without having to build a full cli either.
> Of note: `ick` is not a serious project - it was built to explore the channels outside the notebook
> to help us implement every other feature
> We're doing some crazy things in there as a result
> including the inline images

> F: looking at enchannel too
> I’m going to get familiar with these two tonight

> J: ok cool, that's the most important thing in all of this. There is nothing else to Jupyter than being a messaging protocol here.

... later ...


> F: Ok so the subjects come from sockets

> J: yep

> F: And they come from the ipython kernel
> F: Not Python itself

> J: yeah, basically a driver

> F: IPython is a driver against Python

> J: which is how/why we have the ability to select and use JS, R, Python 2 vs Python 3

> F: That has emitting sockets, that are hooked into the channels

> J: yep

> F: Where does zmq sit? In between the kernel and the observables?

> J: yeah

> F: Ok this makes way more sense now

> J: 

```
[kernel][zmq pub] < ---- > [zmq sub][observable] (frontend)
```

> F: I thought enchannel replaced the Jupyter kernel on top of Python

> J: nah, just an abstraction. makes it so that picture can look like this:

```
[kernel][zmq pub] < -- > [zmq sub][socketio] < -- > [socketioclient][observable]  (frontend)
```

> F: So I noticed that ick maintains scope, like a repl

> J: yeah

> F: When you start a kernel, if you execute var a = 5;
> then "a", it's going to output 5

> J: yep. Same thing for cells in the notebook

> F: So each document has its own kernel

> J: yes!

> F: Now when you edit the first cell, does that blow away the existing variable scope?

> J: As soon as you run the first cell, those are the variables defined

> F: Better phrased as, are variables hoisted from previous runs

> J: nah, because the notebook is just capturing what went in and out. It doesn't have the kernel state

> F: So it's sequential

> J: intended to be, though you can run cells in any order

> F: So editing the first cell will update the last cell or you'd manually run to see new results

> J: You would have to manutally run. It won't change the last cell for you.

> F: Ok

> J: the flow tends to be you go back to edit some function or whatever then run your computation down below.
> Function in one cell, the call in another cell

> F: Didn't realize how much of that was handled at the kernel level

> J: yeah, everything hinges on the comms with these kernels

> F: So if you have 6 cells, update the first and run the 5th.
> Are 2-4 taken into account when running the 5th?

> J: nope, what you choose to run is up to you

> F: Ok, so it operates on the executed scope

> J: yeah. This is the reason people request a run-all action in jupyter/notebook

> F: Wow, I can't believe I didn't have clarity on this. Sorry :(

> J: It's ok! I think it was a bit much to go over early on and we were still carving things out. This is definitely all good, I'm glad to have the back and forth on speccing it out.

> F: Ok I'm gonna continue reviewing the socket specs on jupyter. Happy to understand how all the pieces fit together.

> J: The hard part is that this is one of my domain areas, so I've probably tended toward thinking people picked it up right away.

> F: One question, what's heartbeat?

> J: Heh, so that's yet another socket, that we're not even using. It's basically an echo channel:

```
socket.emit('hey')

... later

recv --> 'hey'

```

> F: Ah ok

> J: It's solely to see if the kernel is still alive. Not all of these features of the kernels make sense in the notebook.

> F: And shell is execute and iopub is result ?

> J: roughly speaking, yes

> F: What's stdin all about

> J: In our context, our frontend has to handle stdin requests. The kernel would send a message requesting stdin and we'd respond to it by prompting the user and sending it on. That may mean inline in the outputs

```
In [3]: input('name: ')
name: Kyle
Out[3]: 'Kyle'
```

> F: Oh ok. On run you get a prompt?  

> J: Yeah

> F: In browser terms

> J: On CLI it'd be a stdin

> F: Ok. This is all becoming wildly clear

> J: also a little crazy right?

> F: Interestingly architected. Seems to be well thought out.

> J: Works well for Python and Julia. Not sure how well this really works for others. Time will tell.

> F: JavaScript seems alright

> J: Yeah I use the JS kernel from n-riesco

> F: First thing I tried "let a = 5". Womp womp. Writing my cool kid js.

> J: There's a babel kernel from Nico too

> F: Nice

> J: I'll bet you'd like to know why there are separate sockets, instead of all-in-one.

> F: Shoot. I get the control socket one but why the rest?

> J: OK, cool. You covered the first. This doesn't apply for our notebook here, though it's neat to think about. For IOPub, you can have many listeners. They can only subscribe and not send messages. This means you can make little displays of the last messages that came back. In my case, I use this to run a terminal with html output in a separate window. That sidecar doesn't have to send back messages, just listen in to display my plots. Generally speaking, they're broken up so that you get non-blocking operations between each. For example, stdin shouldn't block receiving other requests

> F: Figured blocking was a consideration. Is iopub set up that way to also prevent inbound messaging?

> J: Well, it was put together for the feature of having a broadcast of messages. The nature of the zmq socket type dictates that you can only `SUB` to the IOPub. zmq is lovely because it's purely message based. Forces you to work with an Actor model.

> F: How exactly does zmq work

> J: That... is a longer conversation than you really want tonight. If you want to either blow your mind or collapse in despair over all the tech, read http://zguide.zeromq.org/
> It's the core messaging library to a lot of distributed systems
> The Zero in Zero MQ is that there isn't a broker (a hub) for all the messages to pass through
> I've actually got some sample node code I wrote for this using our zmq-prebuilt bindings
> https://github.com/rgbkrk/zmq-prebuilt-example
 
```
// Publisher
var zmq = require('zmq-prebuilt')
  , sock = zmq.socket('pub');

sock.bindSync('tcp://127.0.0.1:3000');
console.log('Publisher bound to port 3000');

setInterval(function(){
  console.log('sending a multipart message envelope');
  sock.send(['kitty cats', 'meow!']);
}, 500);
```

> F: Oh this is THAT low level huh

> J: Yeah

> F: Oh ok

> J:

```
// Subscriber
var zmq = require('zmq-prebuilt')
  , sock = zmq.socket('sub');

sock.connect('tcp://127.0.0.1:3000');
sock.subscribe('kitty cats');
console.log('Subscriber connected to port 3000');

sock.on('message', function(topic, message) {
  console.log('received a message related to:', topic.toString(), 'containing message:', message.toString());
});
```

> J: This is how we're able to communicate across 30+ languages

> F: So it's like socket.io but at a systems level

> J: yeah

> F: (To hilariously oversimplify and expose my web bias)

> J: haha, it's ok. We are currently pairing to learn all this together. You'll have to trust me on saying that you can pick the zmq up quick. It really is about as simple as socket.io makes things. If you pick up the book "Node.js the Right Way", they lead you straight to using zmq.

> F: Assuming those bindings are written in c?

> J: yeah

> F: How do bindings between Python and zmq work?

> J: About the same. All bindings implement a protocol underneath (ZMTP), which they rely on libzmq for (or implement in that higher level language). `pyzmq` is probably the best of the higher level bindings and is maintained by quite a few though mostly by Min RK, one of the Jupyter/IPython leads... who is also part of nteract.

> F: Awesome. Ok I'm gonna keep studying this stuff, thanks for clearing stuff up for me!

> J: yeah no problem. keep firing when you got 'em

> F: Yessir
