---
{"dg-publish":true,"permalink":"/resources/node-js/node-js-design-patterns-book/","tags":["frameworks/nodejs"]}
---


# 1 The Node.js platform

“Some principles and design patterns literally define the developer experience with the Node.js platform and its ecosystem”

- Small core, having the smallest possible set of funcionalities
- Small modules, node uses the concept of **module** as the fundamental means of structuring code, small in code size and *scope*. Using npm or yarns solves dependency hell and lets packages depend on a high number os small, well-focused dependencies 
- Small surface area, expose a minimal set of functionalities to the outside world, producing an API thats clearer to use and less susceptible to erroneous use 

- Designing simple as opposed to perfect 


## How Node.ks works
- I/O is slow
	- blocking I/O
		- blocks thread execution
		- not being able to handle multiple connections on one thread
		- the traditional way to solve that is to use a separate thread.       (or process)

	- non-blocking I/O
		- the system call always returns without waiting, if no results the function returns a constant indicating there is no data
		- the no efficient and simplest way to handle non-blocking is called *bussy-waiting*, lopping thorugh resources until data is returned 
		- the modern efficient way that os's provide to handle concurrent non-blocking resources is called (*synchronous event demultiplexer* or *event notification interface*).



*bussy-waiting*
```js
resources = [socketA, socketB, fileA]
while (!resources.isEmpty()) {
  for (resource of resources) {
    // try to read
    data = resource.read()
    if (data === NO_DATA_AVAILABLE) {
      // there is no data to read at the moment
      continue
    }
    if (data === RESOURCE_CLOSED) {
      // the resource was closed, remove it from the list
      resources.remove(i)
    } else {
      //some data was received, process it
      consumeData(data)
    }
  }
}
```


#### Event demultiplexing
most modern operating systems provide a native mechanism to handle concurrent non-blocking resources, the **synchronous event demultiplexer** a.k.a. **event notification interface**

multiplexing refers to the method by which multiple signals are combined into one 
demultiplexing refers to the opposite, the signals split again into its original components 

The demultiplexer watches multiple resources and returns a new event when a read or write operation executed completes


*event notification interface pseudocode*
```js
watchedList.add(socketA, FOR_READ) // (1)
watchedList.add(fileB, FOR_READ)

while (events = demultiplexer.watch(watchedList)) { // (2)
  // event loop
	for (event of events) { // (3)
    // This read will never block and will always return data
	data = event.resource.read()
    if (data === RESOURCE_CLOSED) {
      // the resource was closed, remove it from the watched list
      demultiplexer.unwatch(event.resource)
    } else {
      // some actual data was received, process it
      consumeData(data)
    }
  }
}
```

1. resources added to the data structure, associating each one with a specific operation
2. demultiplexer.watch() is synchronous and blocks until any of the watched resources are ready for read, then the event demultiplexer returns from the call and a new set of events is available to be processed
3. each event return is processed, when all events are processed the flow will block again on the event demultiplexer until new events are avaialble, ***THIS IS CALLED THE EVENT LOOP***

With this pattern, we can handle several I/0 operations inside a single thread

![Screenshot 2025-05-03 at 1.26.04 PM.png](/img/user/Screenshot%202025-05-03%20at%201.26.04%20PM.png)

The tasks are spread over time, instead of being spread across multiple threads. This has the clear advantage of minimizing the total idle time of the thread

Having a single thread also has a beneficial impact on the way programmers approach concurrency in general. Throughout the book, you will see how *the absence of in-process race conditions and multiple threads to synchronize allows us to use much simpler concurrency strategies*

## The reactor pattern
We can now introduce the reactor pattern, which is a specialization of the algorithms presented in the previous sections

The main idea behind the reactor pattern is to have a handler associated with each I/O operation, a handler is represented by a callback , the callback will be invoked as soon as an event is produced and processed

![Screenshot 2025-05-03 at 1.32.29 PM.png](/img/user/Screenshot%202025-05-03%20at%201.32.29%20PM.png)

1. the app requests a new operation to the event demultiplexer specifying the handler (handler invoked when completed), submit the request is non blocking, immediately returns control to the app 
2. when operation completes events are pushed to the queue
3. the queue iterates over the items 
4. for each evento the handler is invoked
5. the handler (app code), gives back control to the eventloop
6. when all items are processed the event loop blocks on the event demultiplexer until new events are requested


> *A Node.js application will exit when there are no more pending operations in the event demultiplexer, and no more events to be processed inside the event queue.*


The reactor pattern: *Handles I/O by blocking until new events are available from a set of observed resources, and then reacts by dispatching each event to an associated handler.*



### Libuv, the I/O engine of Node.js

Node.js core team created a native library called **libuv**, with the objective to make Node.js compatible with all the major operating systems and normalize the non-blocking behavior of the different types of resource

### Recipe for Node.js

![Screenshot 2025-05-03 at 1.44.33 PM.png](/img/user/Screenshot%202025-05-03%20at%201.44.33%20PM.png)

# 3 Callbacks and Events

synchronous, series of consecutive steps
asynchronous somme operations are launched and then executed "in the background"

the most basic mechanism to get notified about the completion og an asynchronous operations in node.js is the callback, wich is nothing more thatn a function invoked by the runtime with the result of an asynchronous operation 

You will learn about the ***Observer pattern***, which can be consider a close relative of the ***Callback pattern*** 


### Callback pattern

 Callback: function that is passed as an argument to another function and is invoked with the result when the operation completes, un functional programming this way of propagating the result is call **continuation-passing style**. (CPS)

CPS no tiene que ser asyncrono, solo se refiere a que el resultado es propagado pasandolo a otra funcion en vez de propagarlo directamente a la funcion que se ejecuto.

##### CPS sincrono 
```js 
function addCps (a, b, callback) {
  callback(a + b)
}
console.log('before')
addCps(1, 2, result => console.log(`Result: ${result}`))
console.log('after')”

- before
- Result: 3
- after
```

##### CPS asincrono 
```js 
function addCps (a, b, callback) {
  setTimeout(() => callback(a + b), 100)
}
console.log('before')
addCps(1, 2, result => console.log(`Result: ${result}`))
console.log('after')”

- before
- after
- Result: 3
```


#### Sincrono o asincrono
Al tener funciones que tienen comportamiento sincrono o asincrono hay tecnicas para hacerlas puramente sincronas o asincronas. Esto mitiga un comportamiento muy peligroso, un API inconsistente.

>  Pattern: Always choose a direct style for purely synchronous functions.

Una posible solucion es hacer todo sincrono o asincrono 

todo sincrono 
node js tiene APIs sincronas para las operaciones basicas de I/O, aunque una API sincrona no siempre estara disponible

> Pattern: Use blocking APIs sparingly and only when they don't affect the ability of the application to handle concurrent asynchronous operations.



todo asincrono 

usando *process.nextTick()*, toma un callback como argumento y lo envia al inicio del event queue y el callback es ejecutado en cuanto la operacion que se esta ejecutando le regrese el control al event loop


> Pattern: You can guarantee that a callback is invoked asynchronously by deferring its execution using process.nextTick().

Otra forma de postponer la ejecucion de codigo es usando *setImmedate()*, tiene algunas diferencias a *process.nextTick()*

*process.nextTick()*
Calbacks postpuestas usando esta funcion se llaman **microtasks** y son ejecutadas justo despues de la actual termine, y se ejecutan ANTES del resto de I/O operations que esten pendientes, aso que se ejecuta una funcion puede detener a las I/O operations indefinidamente

*setImmedate()* 
Callbacks postpuestas usando esta funcion se ejecutan DESPUES del resto de I/O operations

El event loop ejecuta callbacks en distintas fases:

- **Timers:** This phase executes callbacks scheduled by `setTimeout()` and `setInterval()`. The event loop checks if the timer's due time has arrived, and if so, executes the corresponding callback.

- **Pending Callbacks:** Executes I/O callbacks deferred from the previous loop iteration. These callbacks are typically for system operations, such as network errors. 

- **Poll:** This phase retrieves new I/O events and executes their callbacks. It also determines how long the event loop should block while waiting for new events. If there are no pending poll events, it will check timers and proceed accordingly. 

- **Check:** Executes callbacks scheduled by `setImmediate()`. This provides a way to execute code immediately after the poll phase completes.

- **Close Callbacks:** Executes callbacks for 'close' events, like when a socket or handle is closed abruptly.

After each phase, the event loop checks for microtasks (promises and `process.nextTick`) and executes them before moving to the next phase. This ensures that microtasks are handled promptly.

#### Conventions 
- The callback comes last
- Any error comes first, is a best practice to always check for the presence of an error, the error must always be of type *Error*

```js
readFile('foo.txt', 'utf8', (err, data) => {
  if(err) {
	handleError(err)
  } else {
    processData(data)
  }
})
```

- Propagating errors, propagatins en funciones sincronas se usa *throw* que causa que el error salte en el call stack hasta que es atrapado, en CPS asincrono el error se propaga pasando el error al siguiente callback 

```js
import { readFile } from 'fs'
function readJSON (filename, callback) {
  readFile(filename, 'utf8', (err, data) => {
    let parsed
    if (err) {
      // propagate the error and exit the current function
      return callback(err)
    }
    try {
      // parse the file contents
      parsed = JSON.parse(data)
    } catch (err) {
      // catch parsing errors
      return callback(err)
    }
    // no errors, propagate just the data
    callback(null, parsed)
  })
}
```

- Notice how we propagate the error received by the readFile() operation. We do not throw it or return it; instead, we just use the callback as if it were any other result.

- Also, notice how we use the try...catch statement to catch any error thrown by JSON.parse(), which is a synchronous function and therefore uses the traditional throw instruction to propagate errors to the caller

- Lastly, if everything went well, callback is invoked with null as the first argument to indicate that there are no errors.

- It's also interesting to note how we refrained from invoking callback from within the try block. *This is because doing so would catch any error thrown from the execution of the callback itself, which is usually not what we want*.


### Observer pattern
The main difference between observer and callback is that the subject can notify multiple observers, while traditional CPS callback will usually propagate it's result to one listener 

> The observer pattern defines an object (called subject) that can notify multiple observers (or listeners) when a change in its state occurs

In traditional object oriented programming you need more things, in Node.js is simpler, the pattern is already built into the core and is available through the EventEmitter class 

The EventEmitter class allows to register one or more functions as listeners.

```js
on(event, listener) // register a new listener
once(event, listener) // register a new listener that is removed after event is emitted the first time
emit(event, [arg1], [...]) // produces a new event 
removeListener(event, listener) // 
```

creating and using eventEmitter 

```js
const EventEmitter = require('node:events');
const emitter = new EventEmitter();
emitter.on('hey', () => {console.log('got an hey event')});
emitter.emit('hey');
```

As with callbacks, the EventEmitter can't just throw an exception when an error condition occurs. Instead, the convention is to emit a special event, called error, and pass an Error object as an argument.

### Making any object observable 
It's more common to see the EventEmitter class used to extend other classes, this enables them to inherit the capabilities of the EventEmitter becoming an observable object 

```js
class FindRegex extends EventEmitter {
  constructor (regex) {
    super()
    this.regex = regex
    this.files = []
  }
  addFile (file) {
    this.files.push(file)
    return this
  }
  find () {
    for (const file of this.files) {
      readFile(file, 'utf8', (err, content) => {
        if (err) {
          return this.emit('error', err)
        }
        this.emit('fileread', file)
        const match = content.match(this.regex)
        if (match) {
          match.forEach(elem => this.emit('found', file, elem))
        }
      })
    }
    return this
  }
}
```

### Memory leaks 
When subscribing to observables it is extremely important to *unsubscribe* and prevent memory leaks 

An EventEmitter has a very simple built-in mechanism for warning the developer about possible memory leaks. When the count of listeners registered to an event exceeds a specific amount (by default, 10), the EventEmitter will produce a warning.

> We can use the convenience method *once(event, listener)* in place of *on(event, listener)* to automatically unregister a listener after the event is received for the first time. However, be advised that if the event we specify is never emitted, then the listener is never released, causing a memory leak.

### Synchronous and asynchronous events 

As with callbacks, events can also be emitted synchronously or asynchronously with respect to the moment the tasks that produce them are triggered. It is crucial that we never mix the two approaches in the same EventEmitter, but even more importantly, we should never emit the same event type using a mix of synchronous and asynchronous code, to avoid producing the same problems described in the Unleashing Zalgo section.

When events are emitted asynchronously, we can register new listeners, even after the task that produces the events is triggered, up until the current stack yields to the event loop. This is because the events are guaranteed not to be fired until the next cycle of the event loop, so we can be sure that we won't miss any events.

The *FindRegex()* class we defined previously emits its events asynchronously after the find() method is invoked. This is why we can register the listeners after the find() method is invoked, without losing any events, as shown

```js
findRegexInstance
  .addFile(...)
  .find()
  .on('found', ...)
```

let's modify the FindRegex class we defined previously and make the find() method synchronous

```js
find () {
  for (const file of this.files) {
    let content
    try {
      content = readFileSync(file, 'utf8')
    } catch (err) {
      this.emit('error', err)
    }
    this.emit('fileread', file)
    const match = content.match(this.regex)
    if (match) {
      match.forEach(elem => this.emit('found', file, elem))
    }
  }
  return this
}
```

Now, let's try to register a listener before we launch the find() task, and then a second listener after that to see what happens:

```js
const findRegexSyncInstance = new FindRegexSync(/hello \w+/)
findRegexSyncInstance
  .addFile('fileA.txt')
  .addFile('fileB.json')
  // this listener is invoked
  .on('found', (file, match) => console.log(`[Before] Matched "${match}"`))
  .find()
  // this listener is never invoked
  .on('found', (file, match) => console.log(`[After] Matched "${match}"`))
```

As expected, the listener that was registered after the invocation of the find() task is never called

> The emission of synchronous events can be deferred with process.nextTick() to guarantee that they are emitted asynchronously.

### Event emitters vs callbacks 

The general differentiating rule is semantic: callbacks should be used when a result must be returned in an asynchronous way, while events should be used when there is a need to communicate that something has happened.

Callbacks have some limitations when it comes to supporting different types of events. In fact, we can still differentiate between multiple events by passing the type as an argument of the callback, or by accepting several callbacks, one for each supported event. However, this can't exactly be considered an elegant API.

The EventEmitter should be used when the same event can occur multiple times, or may not occur at all. A callback, in fact, is expected to be invoked exactly once, whether the operation is successful or not. Having a possibly repeating circumstance should make us think again about the semantic nature of the occurrence, which is more similar to an event that has to be communicated, rather than a result to be returned.

An API that uses callbacks can notify only one particular callback, while using an EventEmitter allows us to register multiple listeners for the same event.

### Combining callbacks and events 

There are some particular circumstances where the EventEmitter can be used in conjunction with a callback. This pattern is extremely powerful as it allows us to pass a result asynchronously using a traditional callback, and at the same time return an EventEmitter, which can be used to provide a more detailed account on the status of an asynchronous process.

One example of this pattern is offered by the glob package (nodejsdp.link/npm-glob)

```js
const eventEmitter = glob(pattern, [options], callback)

glob('data/*.txt',
  (err, files) => {
    if (err) {
      return console.error(err)
    }
    console.log(`All files found: ${JSON.stringify(files)}`)
  })
  .on('match', match => console.log(`Match found: ${match}`))

```

The function takes a pattern as the first argument, a set of options, and a callback that is invoked with the list of all the files matching the provided pattern. At the same time, the function returns an EventEmitter, which provides a more fine-grained report about the state of the search process.

Combining an EventEmitter with traditional callbacks is an elegant way to offer two different approaches to the same API. One approach is usually meant to be simpler and more immediate to use, while the other is targeted at more advanced scenarios.