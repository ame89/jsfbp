# jsfbp

"Classical" FBP "green thread" implementation written in JavaScript, using Node-Fibers - https://github.com/laverdet/node-fibers .  

JSFBP takes advantage of JavaScript's concept of functions as first-degree objects to allow applications to be built using "green threads".  JSFBP makes use of a "Future Events Queue" which supports the green threads, and provides quite good performance (see below) - the JavaScript events queue is only used for JavaScript asynchronous functions, as before.

# General

Test cases so far:

- `fbptest01` - 3 processes:
    - `sender` (generates ascending numeric values)
    - `copier` (copies)
    - `recvr`  (displays incoming values to console)

![JSFBP](https://github.com/jpaulm/jsfbp/blob/master/docs/JSFBP.png "Simple Test Network")

- `fbptest02` - `sender` replaced with `reader`
- `fbptest03` - `sender` and `reader` both feeding into `copier.IN`
- `fbptest04` - `sender` feeding `repl` which sends 3 copies of input IP (as specified in network), each copy going to a separate element of array port `OUT`; all 3 copies then feeding into `recvr.IN`
- `fbptest05` - Two copies of `reader` running concurrently, one feeds direct to `rrmerge` ("round robin" merge) input port element 0; other one into `copier` and then into `rrmerge` input port element 1; from `rrmerge.OUT` to `recvr.IN` 
- `fbptest06` - The output streams of the `repl` (in `fbptest04`) are fed to the input array port of `rrmerge`, and from its `OUT` to `recvr.IN`
- `fbptest07` - Creates a deadlock condition - the status of each Process is displayed
- `fbptest08` - reads text, reverses it twice and outputs it
- `fbptest09` - `copier` in `fbptest01` is replaced with a version of `copier` which terminates prematurely and closes its input port, bringing the network down (ungracefully!)
- `fbptest10` -  `copier` in `fbptest01` is replaced with a non-looping version of `copier`
- `fbptest11` -  Load balancer (`lbal`) feeding 3 instances of a random delay component (`randdelay`)
  
![Fbptest11](https://github.com/jpaulm/jsfbp/blob/master/docs/Fbptest11.png "Diagram of fbptest11 above")

- `fbptest12` -  `reader OUT -> IN copier OUT -> IN writer`
- `fbptest13` -  Simple network to demonstrate functioning of random delay component (`randdelay`)
- `fbptest14` -  Network demonstrating parallelism using two instances of `reader` and two fixed delay components (`delay`)
- `fbptestvl` -  Volume test (see below): `sender` -> `copier` -> `discard` 
 
WebSockets 
- `fbptestws` -  Schematic web socket server (simple Process shown can be replaced by any structure of Processes, provided interfaces are adhered to)
 
![Fbptestws](https://github.com/jpaulm/jsfbp/blob/master/docs/Fbptestws.png "Diagram of fbptestws above")
 
Some of these have tracing set on, depending on what testing was being done when they were promoted!

These tests (except for `fbptestws`) can be run sequentially by running `fbptests.bat`.
 
# Components

- `concat`  - concatenates all the streams that are sent to its array input port (size determined in network definition) 
- `copier`  - copies its input stream to its output stream
- `copier_closing` - forces close of input port after 20 IPs
- `copier_nonlooper` - same as `copier`, except that it is written as a non-looper (it has been modified to call the FBP services from lower in the process's stack)
- `discard` - discard (drop) all incoming IPs
- `lbal`    - load balancer - sends output to output port array element with smallest number of IPs in transit
- `randdelay` - sends incoming IPs to output port after random number of millisecs (between 0 and 400)
- `reader`  - does an asynchronous read on the file specified by its FILE IIP 
- `recvr`   - receives its incoming stream and displays the contents on the console 
- `repl`    - replicates the incoming IPs to the streams specified by an array output port (it does not handle tree structures)
- `reverse` - reverses the string contained in each incoming IP
- `rrmerge` - "round robin" merge 
- `sender`  - sends as many IPs to its output port as are specified by its COUNT IIP (each just contains the current count)
- `writer`  - does an asynchronous write to the file specified by its FILE IIP
  
- `wsrecv`  - general web socket "receive" component for web socket server - outputs substream 
- `wsresp`  - general web socket "respond" component sending data from web socket server to client - takes substream as input
- `wssimproc` - "simulated" processing for web socket server - actually just outputs 3 names

 
# API

## For normal users

1. Get access to JSFBP: `var fbp = require('fbp')`
2. Create a new network: `var network = new fbp.Network();`
3. Define your network:
 - Add processes: `network.defProc(...)`
 - Connect output ports to input ports: `network.connect(...)`
 - Specify IIPs: `network.initialize(...)`
4. Create a new runtime: `var fiberRuntime = new fbp.FiberRuntime();`
5. Run it!
 
```
network.run(fiberRuntime, {trace: true/false}, function success() {
    console.log("Finished!");
  });
```
 Activating `trace` can be desired in debugging scenarios.

## For component developers

Component headers:
`'use strict';`

In most cases you do not need to *require()* any JSFBP-related scripts or libraries as a component developer. Everything you need is injected into the component's function as its context `this` (the process object) and as a parameter (the runtime object).
Some utility functions are stored in `core/utils.js`. Import them if you really need them.
You should generally refrain from accessing runtime-related code (e.g. Fibers) to ensure the greatest compatibility.

Component services

- `var ip = this.createIP(contents);` - create an IP containing `contents`
- `var ip = this.createIPBracket(IP.OPEN|IP.CLOSE[, contents])` - create an open or close bracket IP
- **Be sure** to include IP: `var IP = require('IP')` to gain access to the IP constants.
- `this.dropIP(ip);` - drop IP
  
- `var inport = this.openInputPort('IN');` - create InputPort variable  
- `var array = this.openInputPortArray('IN');` - create input array port array
- `var outport = this.openOutputPort('OUT');` - create OutputPort variable 
- `var array = this.openOutputPortArray('OUT');` - create output array port array   
  
- `var ip = inport.receive();` - returns null if end of stream 
- `var ip = array[i].receive();` - receive to element of port array
- `outport.send(ip);` - returns -1 if send unable to deliver
- `array[i].send(ip);` - send from element of port array
- `inport.close();` - close input port (or array port element)
  
-  `runtime.runAsyncCallback()` - used when doing asynchronous I/O in component; when using this function, include `runtime` in component header, e.g. `module.exports = function xxx(runtime) { ...` 
   
  Example:
```
runtime.runAsyncCallback(function (done) {
  // your asynchronous
  ...
  // call done (possibly asynchronously) when you're done!
  done();
});
```
-  `Utils.getElementWithSmallestBacklog(array);` - used by `lbal` - not for general use 
- **Be sure** to include Utils: `var Utils = require('core/utils')`.

# Install & Run

1. Install node.js - see http://nodejs.org/download/  .  Node 12.0 leads to compatibility problems with Fiber, see [here](https://gist.github.com/ComFreek/c341bacfaae3aca887df) how to use Node 11.16 on a per-project basis. Alternatively, you can also globally install an older Node version.

2. Clone or download this project

3. Run `npm install` in the project directory

4. Run `node examples/fbptestxx.js`, where `fbptestxx` is any of the tests listed above. If tracing is desired, change the value of the `trace` variable at the bottom of `fbptestxx.js` to `true`. 

5. All these tests can be run sequentially by running `examples/fbptests.bat`, or by running `examples/fbptests.sh` under `bash`.

# Testing with Mocha

The folder called `test` contains a number of Mocha tests.

- Run `npm install` (in case you haven't already done so) and `npm test`
- Alternatively, you can directly execute `node.exe node_modules/mocha/bin/mocha --recursive --require test/test_helper.js` in case you need to adjust the path to Node's binary or pass further parameters to Mocha.

# Testing Web Socket Server

Run `node examples\websocket\fbptestws.js`, which is a simple web socket server.  It responds to any request by returning 3 names.

`examples\websocket\chat1.html` is intended as a simple client for testing with `fbptestws.js`. If Firefox doesn't work for you, Chrome should work.

Just enter any string into the input field, and click on `Send`, and it will return the strings: 

- Server: Frankie Tomatto
- Server: Joe Fresh
- Server: Aunt Jemima

Click on the `Stop WS` button, and the network will come down.

Tracing
---

Here is a sample section of the trace output for `fbptest08.js`:
```
recvr recv OK: externally to the processes. These black box processes can be rec
onnected endlessly
data: externally to the processes. These black box processes can be reconnected
endlessly
recvr IP dropped with: externally to the processes. These black box processes ca
n be reconnected endlessly
recvr recv from recvr.IN
Yield/return: state of future events queue:
- reverse2 - status: ACTIVE
---
---
reverse2 send OK
reverse2 IP dropped with:  si PBF .yllanretni degnahc eb ot gnivah tuohtiw snoit
acilppa tnereffid mrof ot
reverse2 recv from reverse2.IN
reverse2 recv OK: .detneiro-tnenopmoc yllarutan suht
reverse2 send to reverse2.OUT: thus naturally component-oriented.
Yield/return: state of future events queue:
- recvr - status: ACTIVE
---
---
recvr recv OK: to form different applications without having to be changed inter
nally. FBP is
```

Performance
---

The volume test case (`fbptestvl`) with 100,000,000 IPs running through three processes took 164 seconds, on my machine which has 4 AMD Phenom(tm) II X4 925 processors,.  Since there are two connections, giving a total of 200,000,000 send/receive pairs, this works out to approx. 0.82 microsecs per send/receive pair. Of course, as it is JavaScript, this test only uses 1 core intensively, although there is some matching activity on the other cores (why...?!)

