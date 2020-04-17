# Errori

<!--introduced_in=v4.0.0-->

<!--type=misc-->

Applications running in Node.js will generally experience four categories of errors:

- Errori standard di JavaScript, come ad esempio: 
  - {EvalError} : generato quando una chiamata a `eval()` fallisce.
  - {SyntaxError} : thrown in response to improper JavaScript language syntax.
  - {RangeError} : generato quando un valore non rientra nell'intervallo previsto
  - {ReferenceError} : generato quando si utilizzano variabili non definite
  - {TypeError} : generato quando vengono passati argomenti di tipo errato
  - {URIError}: generato quando una funzione di gestione globale degli URI è usata in modo errato.
- System errors triggered by underlying operating system constraints such as attempting to open a file that does not exist, attempting to send data over a closed socket, etc;
- E gli errori specifici dell'utente causati dal codice applicativo.
- Assertion Errors are a special class of error that can be triggered whenever Node.js detects an exceptional logic violation that should never occur. These are raised typically by the `assert` module.

All JavaScript and System errors raised by Node.js inherit from, or are instances of, the standard JavaScript {Error} class and are guaranteed to provide *at least* the properties available on that class.

## Propagazione e Intercettazione degli Errori

<!--type=misc-->

Node.js supports several mechanisms for propagating and handling errors that occur while an application is running. How these errors are reported and handled depends entirely on the type of Error and the style of the API that is called.

All JavaScript errors are handled as exceptions that *immediately* generate and throw an error using the standard JavaScript `throw` mechanism. These are handled using the [`try / catch` construct](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch) provided by the JavaScript language.

```js
// Viene generato con un ReferenceError perché z è indefinito
prova {
  const m = 1;
  const n = m + z;
} catch (err) {
  // Gestisci l'errore qui.
}
```

Any use of the JavaScript `throw` mechanism will raise an exception that *must* be handled using `try / catch` or the Node.js process will exit immediately.

With few exceptions, *Synchronous* APIs (any blocking method that does not accept a `callback` function, such as [`fs.readFileSync`][]), will use `throw` to report errors.

Gli errori che si verificano all'interno di *API Asincrone* possono essere segnalati in diversi modi:

- Most asynchronous methods that accept a `callback` function will accept an `Error` object passed as the first argument to that function. If that first argument is not `null` and is an instance of `Error`, then an error occurred that should be handled. 
      js
      const fs = require('fs');
      fs.readFile('a file that does not exist', (err, data) => {
        if (err) {
          console.error('There was an error reading the file!', err);
          return;
        }
        // Otherwise handle the data
      });

- When an asynchronous method is called on an object that is an `EventEmitter`, errors can be routed to that object's `'error'` event.
  
  ```js
  const net = require('net');
  const connection = net.connect('localhost');
  
  // Aggiungere un handler di evento 'error' a uno stream:
  connection.on('error', (err) => {
   //Se la connessione è resettata dal server, o se non può
   // connettersi per niente, o in un qualsiasi tipo di errore incontrato dalla
   // connessione, l'errore verrà spedito qui.
    console.error(err);
  });
  
  connection.pipe(process.stdout);
  ```

- A handful of typically asynchronous methods in the Node.js API may still use the `throw` mechanism to raise exceptions that must be handled using `try / catch`. There is no comprehensive list of such methods; please refer to the documentation of each method to determine the appropriate error handling mechanism required.

The use of the `'error'` event mechanism is most common for [stream-based](stream.html) and [event emitter-based](events.html#events_class_eventemitter) APIs, which themselves represent a series of asynchronous operations over time (as opposed to a single operation that may pass or fail).

For *all* `EventEmitter` objects, if an `'error'` event handler is not provided, the error will be thrown, causing the Node.js process to report an unhandled exception and crash unless either: The [`domain`](domain.html) module is used appropriately or a handler has been registered for the [`process.on('uncaughtException')`][] event.

```js
const EventEmitter = require('events');
const ee = new EventEmitter();

setImmediate(() => {
  // Questo causerà il crash del processo perché non è stato aggiunto alcun
  // handler dell’evento 'error'.
  ee.emit('error', new Error('This will crash'));
});
```

Errors generated in this way *cannot* be intercepted using `try / catch` as they are thrown *after* the calling code has already exited.

Developers must refer to the documentation for each method to determine exactly how errors raised by those methods are propagated.

### Error-first callbacks<!--type=misc-->Most asynchronous methods exposed by the Node.js core API follow an idiomatic pattern referred to as an 

*error-first callback* (sometimes referred to as a *Node.js style callback*). With this pattern, a callback function is passed to the method as an argument. When the operation either completes or an error is raised, the callback function is called with the Error object (if any) passed as the first argument. If no error was raised, the first argument will be passed as `null`.

```js
const fs = require('fs');

function errorFirstCallback(err, data) {
  if (err) {
    console.error('There was an error', err);
    return;
  }
  console.log(data);
}

fs.readFile('/some/file/that/does-not-exist', errorFirstCallback);
fs.readFile('/some/file/that/does-exist', errorFirstCallback);
```

The JavaScript `try / catch` mechanism **cannot** be used to intercept errors generated by asynchronous APIs. A common mistake for beginners is to try to use `throw` inside an error-first callback:

```js
//QUESTO NON FUNZIONERÀ:
const fs = require('fs');

try {
  fs.readFile('/some/file/that/does-not-exist', (err, data) => {
    // ipotesi errata: generando qui...
    if (err) {
      throw err;
    }
  });
} catch (err) {
  // Questo non prenderà il throw!
  console.error(err);
}
```

This will not work because the callback function passed to `fs.readFile()` is called asynchronously. By the time the callback has been called, the surrounding code (including the `try { } catch (err) { }` block will have already exited. Throwing an error inside the callback **can crash the Node.js process** in most cases. If [domains](domain.html) are enabled, or a handler has been registered with `process.on('uncaughtException')`, such errors can be intercepted.

## Class: Error<!--type=class-->A generic JavaScript 

`Error` object that does not denote any specific circumstance of why the error occurred. `Error` objects capture a "stack trace" detailing the point in the code at which the `Error` was instantiated, and may provide a text description of the error.

For crypto only, `Error` objects will include the OpenSSL error stack in a separate property called `opensslErrorStack` if it is available when the error is thrown.

All errors generated by Node.js, including all System and JavaScript errors, will either be instances of, or inherit from, the `Error` class.

### nuovo Error(message)

- `message` {string}

Creates a new `Error` object and sets the `error.message` property to the provided text message. If an object is passed as `message`, the text message is generated by calling `message.toString()`. The `error.stack` property will represent the point in the code at which `new Error()` was called. Stack traces are dependent on [V8's stack trace API](https://github.com/v8/v8/wiki/Stack-Trace-API). Stack traces extend only to either (a) the beginning of *synchronous code execution*, or (b) the number of frames given by the property `Error.stackTraceLimit`, whichever is smaller.

### Error.captureStackTrace(targetObject[, constructorOpt])

- `targetObject` {Object}
- `constructorOpt` {Function}

Creates a `.stack` property on `targetObject`, which when accessed returns a string representing the location in the code at which `Error.captureStackTrace()` was called.

```js
const myObject = {};
Error.captureStackTrace(myObject);
myObject.stack;  // simile a `new Error().stack`
```

The first line of the trace will be prefixed with `${myObject.name}: ${myObject.message}`.

L'argomento facoltativo `constructorOpt` accetta una funzione. If given, all frames above `constructorOpt`, including `constructorOpt`, will be omitted from the generated stack trace.

The `constructorOpt` argument is useful for hiding implementation details of error generation from an end user. Ad esempio:

```js
function MyError() {
  Error.captureStackTrace(this, MyError);
}

// Senza passare MyError a captureStackTrace, il MyError
// frame verrebbe mostrato nella proprietà  .Stack. Passando
// il constructor, omettiamo quel frame, e manteniamo tutti i frame inferiori.
new MyError().stack;
```

### Error.stackTraceLimit

- {number}

The `Error.stackTraceLimit` property specifies the number of stack frames collected by a stack trace (whether generated by `new Error().stack` or `Error.captureStackTrace(obj)`).

Il valore predefinito è `10` ma può essere impostato su un qualsiasi numero JavaScript valido. Changes will affect any stack trace captured *after* the value has been changed.

If set to a non-number value, or set to a negative number, stack traces will not capture any frames.

### error.code

- {string}

La proprietà `error.code` è un etichetta di stringa che identifica il tipo de errore. See [Node.js Error Codes](#nodejs-error-codes) for details about specific codes.

### error.message

- {string}

The `error.message` property is the string description of the error as set by calling `new Error(message)`. The `message` passed to the constructor will also appear in the first line of the stack trace of the `Error`, however changing this property after the `Error` object is created *may not* change the first line of the stack trace (for example, when `error.stack` is read before this property is changed).

```js
const err = new Error('The message');
console.error(err.message);
// Stampa: Il messaggio
```

### error.stack

- {string}

The `error.stack` property is a string describing the point in the code at which the `Error` was instantiated.

Per esempio:

```txt
Errore: Continuano ad accadere cose!
   at /home/gbusey/file.js:525:2
at Frobnicator.refrobulate (/home/gbusey/business-logic.js:424:21)
at Actor.&lt;anonymous&gt; (/home/gbusey/actors.js:400:8)
at increaseSynergy (/home/gbusey/actors.js:701:6)
```

The first line is formatted as `<error class name>: <error message>`, and is followed by a series of stack frames (each line beginning with "at "). Each frame describes a call site within the code that lead to the error being generated. V8 attempts to display a name for each function (by variable name, function name, or object method name), but occasionally it will not be able to find a suitable name. If V8 cannot determine a name for the function, only location information will be displayed for that frame. Otherwise, the determined function name will be displayed with location information appended in parentheses.

I frame sono generati solo per le funzioni JavaScript. If, for example, execution synchronously passes through a C++ addon function called `cheetahify` which itself calls a JavaScript function, the frame representing the `cheetahify` call will not be present in the stack traces:

```js
const cheetahify = require('./native-binding.node');

function makeFaster() {
  // cheetahify  chiama speedy *in modo sincrono*.
  cheetahify(function speedy() {
    throw new Error('oh no!');
  });
}

makeFaster();
// genererà:
//   /home/gbusey/file.js:6
//       genera nuovo errore ('oh no!');
//           ^
//   Errore: oh no!
//       at speedy (/home/gbusey/file.js:6:11)
//       at makeFaster (/home/gbusey/file.js:5:3)
//       at Object.&lt;anonymous&gt; (/home/gbusey/file.js:10:1)
//       at Module._compile (module.js:456:26)
//       at Object.Module._extensions..js (module.js:474:10)
//       at Module.load (module.js:356:32)
//       at Function.Module._load (module.js:312:12)
//       at Function.Module.runMain (module.js:497:10)
//       at startup (node.js:119:16)
//       at node.js:906:3
```

Le informazioni di posizione saranno di tipo:

- `native`, se il frame rappresenta una chiamata interna a V8 (come in `[].forEach`).
- `plain-filename.js:line:column`, if the frame represents a call internal to Node.js.
- `/absolute/path/to/file.js:line:column`, if the frame represents a call in a user program, or its dependencies.

The string representing the stack trace is lazily generated when the `error.stack` property is **accessed**.

The number of frames captured by the stack trace is bounded by the smaller of `Error.stackTraceLimit` or the number of available frames on the current event loop tick.

System-level errors are generated as augmented `Error` instances, which are detailed [here](#errors_system_errors).

## Classe: AssertionError

Una sottoclasse di `Error` che indica il fallimento di un assertion. Such errors commonly indicate inequality of actual and expected value.

Per esempio:

```js
assert.strictEqual(1, 2);
// AssertionError [ERR_ASSERTION]: 1 === 2
```

## Classe: RangeError

A subclass of `Error` that indicates that a provided argument was not within the set or range of acceptable values for a function; whether that is a numeric range, or outside the set of options for a given function parameter.

Per esempio:

```js
require('net').connect(-1);
// genera "RangeError: l'opzione "port" dovrebbe essere >= 0 and < 65536: -1"
```

Node.js will generate and throw `RangeError` instances *immediately* as a form of argument validation.

## Classe: ReferenceError

A subclass of `Error` that indicates that an attempt is being made to access a variable that is not defined. Such errors commonly indicate typos in code, or an otherwise broken program.

While client code may generate and propagate these errors, in practice, only V8 will do so.

```js
doesNotExist;
// genera ReferenceError, doesNotExist non è una variabile in questo programma.
```

Unless an application is dynamically generating and running code, `ReferenceError` instances should always be considered a bug in the code or its dependencies.

## Classe: SyntaxError

Una sottoclasse di `Error` che indica che un programma non è un JavaScript valido. These errors may only be generated and propagated as a result of code evaluation. Code evaluation may happen as a result of `eval`, `Function`, `require`, or [vm](vm.html). These errors are almost always indicative of a broken program.

```js
try {
  require('vm').runInThisContext('binary ! isNotOk');
} catch (err) {
  // l'errore sarà del tipo SyntaxError
}
```

`SyntaxError` instances are unrecoverable in the context that created them – they may only be caught by other contexts.

## Classe: TypeError

A subclass of `Error` that indicates that a provided argument is not an allowable type. For example, passing a function to a parameter which expects a string would be considered a TypeError.

```js
require('url').parse(() => { });
// genera TypeError, dato che si aspettava una stringa
```

Node.js will generate and throw `TypeError` instances *immediately* as a form of argument validation.

## Eccezioni vs. Errors<!--type=misc-->A JavaScript exception is a value that is thrown as a result of an invalid operation or as the target of a 

`throw` statement. While it is not required that these values are instances of `Error` or classes which inherit from `Error`, all exceptions thrown by Node.js or the JavaScript runtime *will* be instances of Error.

Alcune eccezione sono *non recuperabili* al livello di JavaScript. Such exceptions will *always* cause the Node.js process to crash. Examples include `assert()` checks or `abort()` calls in the C++ layer.

## Errori di Sistema

System errors are generated when exceptions occur within the program's runtime environment. Typically, these are operational errors that occur when an application violates an operating system constraint such as attempting to read a file that does not exist or when the user does not have sufficient permissions.

System errors are typically generated at the syscall level: an exhaustive list of error codes and their meanings is available by running `man 2 intro` or `man 3 errno` on most Unices; or [online](http://man7.org/linux/man-pages/man3/errno.3.html).

In Node.js, system errors are represented as augmented `Error` objects with added properties.

### Class: System Error

#### error.code

- {string}

The `error.code` property is a string representing the error code, which is typically `E` followed by a sequence of capital letters.

#### error.errno

- {string|number}

La proprietà `error.errno` è un numero o una stringa. The number is a **negative** value which corresponds to the error code defined in [`libuv Error handling`]. See uv-errno.h header file (`deps/uv/include/uv-errno.h` in the Node.js source tree) for details. In case of a string, it is the same as `error.code`.

#### error.syscall

- {string}

La proprietà `error.syscall` è una stringa che descrive il [syscall](http://man7.org/linux/man-pages/man2/syscall.2.html) che ha fallito.

#### error.path

- {string}

When present (e.g. in `fs` or `child_process`), the `error.path` property is a string containing a relevant invalid pathname.

#### error.address

- {string}

When present (e.g. in `net` or `dgram`), the `error.address` property is a string describing the address to which the connection failed.

#### error.port

- {number}

When present (e.g. in `net` or `dgram`), the `error.port` property is a number representing the connection's port that is not available.

### Errori di Sistema Comuni

This list is **not exhaustive**, but enumerates many of the common system errors encountered when writing a Node.js program. An exhaustive list may be found [here](http://man7.org/linux/man-pages/man3/errno.3.html).

- `EACCES` (Permission denied): An attempt was made to access a file in a way forbidden by its file access permissions.

- `EADDRINUSE` (Address already in use): An attempt to bind a server ([`net`][], [`http`][], or [`https`][]) to a local address failed due to another server on the local system already occupying that address.

- `ECONNREFUSED` (Connection refused): No connection could be made because the target machine actively refused it. This usually results from trying to connect to a service that is inactive on the foreign host.

- `ECONNRESET` (Connection reset by peer): A connection was forcibly closed by a peer. This normally results from a loss of the connection on the remote socket due to a timeout or reboot. Commonly encountered via the [`http`][] and [`net`][] modules.

- `EEXIST` (File exists): An existing file was the target of an operation that required that the target not exist.

- `EISDIR` (Is a directory): An operation expected a file, but the given pathname was a directory.

- `EMFILE` (Too many open files in system): Maximum number of [file descriptors](https://en.wikipedia.org/wiki/File_descriptor) allowable on the system has been reached, and requests for another descriptor cannot be fulfilled until at least one has been closed. This is encountered when opening many files at once in parallel, especially on systems (in particular, macOS) where there is a low file descriptor limit for processes. To remedy a low limit, run `ulimit -n 2048` in the same shell that will run the Node.js process.

- `ENOENT` (No such file or directory): Commonly raised by [`fs`][] operations to indicate that a component of the specified pathname does not exist — no entity (file or directory) could be found by the given path.

- `ENOTDIR` (Not a directory): A component of the given pathname existed, but was not a directory as expected. Di solito generata da [`fs.readdir`][].

- `ENOTEMPTY` (Directory not empty): A directory with entries was the target of an operation that requires an empty directory — usually [`fs.unlink`][].

- `EPERM` (Operation not permitted): An attempt was made to perform an operation that requires elevated privileges.

- `EPIPE` (Broken pipe): A write on a pipe, socket, or FIFO for which there is no process to read the data. Commonly encountered at the [`net`][] and [`http`][] layers, indicative that the remote side of the stream being written to has been closed.

- `ETIMEDOUT` (Operation timed out): A connect or send request failed because the connected party did not properly respond after a period of time. Usually encountered by [`http`][] or [`net`][] — often a sign that a `socket.end()` was not properly called.

<a id="nodejs-error-codes"></a>

## Codici Errore Node.js

<a id="ERR_ARG_NOT_ITERABLE"></a>

### ERR_ARG_NOT_ITERABLE

An iterable argument (i.e. a value that works with `for...of` loops) was required, but not provided to a Node.js API.

<a id="ERR_ASYNC_CALLBACK"></a>

### ERR_ASYNC_CALLBACK

An attempt was made to register something that is not a function as an `AsyncHooks` callback.

<a id="ERR_ASYNC_TYPE"></a>

### ERR_ASYNC_TYPE

Il tipo di risorsa asincrona non è valido. Note that users are also able to define their own types if using the public embedder API.

<a id="ERR_ENCODING_INVALID_ENCODED_DATA"></a>

### ERR_ENCODING_INVALID_ENCODED_DATA

Data provided to `util.TextDecoder()` API was invalid according to the encoding provided.

<a id="ERR_ENCODING_NOT_SUPPORTED"></a>

### ERR_ENCODING_NOT_SUPPORTED

Encoding provided to `util.TextDecoder()` API was not one of the [WHATWG Supported Encodings](util.md#whatwg-supported-encodings).

<a id="ERR_FALSY_VALUE_REJECTION"></a>

### ERR_FALSY_VALUE_REJECTION

A `Promise` that was callbackified via `util.callbackify()` was rejected with a falsy value.

<a id="ERR_HTTP_HEADERS_SENT"></a>

### ERR_HTTP_HEADERS_SENT

È stato effettuato un tentativo di aggiungere ulteriori intestazioni dopo che le intestazioni erano già state inviate.

<a id="ERR_HTTP_INVALID_CHAR"></a>

### ERR_HTTP_INVALID_CHAR

An invalid character was found in an HTTP response status message (reason phrase).

<a id="ERR_HTTP_INVALID_STATUS_CODE"></a>

### ERR_HTTP_INVALID_STATUS_CODE

Il codice di stato non rientrava nel normale intervallo di codici di stato (100-999).

<a id="ERR_HTTP_TRAILER_INVALID"></a>

### ERR_HTTP_TRAILER_INVALID

The `Trailer` header was set even though the transfer encoding does not support that.

<a id="ERR_HTTP2_ALREADY_SHUTDOWN"></a>

### ERR_HTTP2_ALREADY_SHUTDOWN

Occurs with multiple attempts to shutdown an HTTP/2 session.

<a id="ERR_HTTP2_ALTSVC_INVALID_ORIGIN"></a>

### ERR_HTTP2_ALTSVC_INVALID_ORIGIN

I frame HTTP/2 ALTSVC richiedono un'origine valida.

<a id="ERR_HTTP2_ALTSVC_LENGTH"></a>

### ERR_HTTP2_ALTSVC_INVALID_ORIGIN

I frame HTTP/2 ALTSVC sono limitati a un massimo di 16,382 payload byte.

<a id="ERR_HTTP2_CONNECT_AUTHORITY"></a>

### ERR_HTTP2_CONNECT_AUTHORITY

For HTTP/2 requests using the `CONNECT` method, the `:authority` pseudo-header is required.

<a id="ERR_HTTP2_CONNECT_PATH"></a>

### ERR_HTTP2_CONNECT_PATH

For HTTP/2 requests using the `CONNECT` method, the `:path` pseudo-header is forbidden.

<a id="ERR_HTTP2_CONNECT_SCHEME"></a>

### ERR_HTTP2_CONNECT_SCHEME

For HTTP/2 requests using the `CONNECT` method, the `:scheme` pseudo-header is forbidden.

<a id="ERR_HTTP2_FRAME_ERROR"></a>

### ERR_HTTP2_FRAME_ERROR

A failure occurred sending an individual frame on the HTTP/2 session.

<a id="ERR_HTTP2_GOAWAY_SESSION"></a>

### ERR_HTTP2_GOAWAY_SESSION

New HTTP/2 Streams may not be opened after the `Http2Session` has received a `GOAWAY` frame from the connected peer.

<a id="ERR_HTTP2_HEADER_REQUIRED"></a>

### ERR_HTTP2_HEADER_REQUIRED

A required header was missing in an HTTP/2 message.

<a id="ERR_HTTP2_HEADER_SINGLE_VALUE"></a>

### ERR_HTTP2_HEADER_SINGLE_VALUE

Multiple values were provided for an HTTP/2 header field that was required to have only a single value.

<a id="ERR_HTTP2_HEADERS_AFTER_RESPOND"></a>

### ERR_HTTP2_HEADERS_AFTER_RESPOND

Dopo che una risposta HTTP/2 era stata iniziata è stata specificata un’ulteriore intestazione.

<a id="ERR_HTTP2_HEADERS_OBJECT"></a>

### ERR_HTTP2_HEADERS_OBJECT

An HTTP/2 Headers Object was expected.

<a id="ERR_HTTP2_HEADERS_SENT"></a>

### ERR_HTTP2_HEADERS_SENT

C'è stato un tentativo di inviare molteplici intestazioni di risposta.

<a id="ERR_HTTP2_INFO_HEADERS_AFTER_RESPOND"></a>

### ERR_HTTP2_INFO_HEADERS_AFTER_RESPOND

HTTP/2 Informational headers must only be sent *prior* to calling the `Http2Stream.prototype.respond()` method.

<a id="ERR_HTTP2_INFO_STATUS_NOT_ALLOWED"></a>

### ERR_HTTP2_INFO_STATUS_NOT_ALLOWED

Informational HTTP status codes (`1xx`) may not be set as the response status code on HTTP/2 responses.

<a id="ERR_HTTP2_INVALID_CONNECTION_HEADERS"></a>

### ERR_HTTP2_INVALID_CONNECTION_HEADERS

HTTP/1 connection specific headers are forbidden to be used in HTTP/2 requests and responses.

<a id="ERR_HTTP2_INVALID_HEADER_VALUE"></a>

### ERR_HTTP2_INVALID_HEADER_VALUE

È stato specificato un valore di intestazione HTTP/2 non valido.

<a id="ERR_HTTP2_INVALID_INFO_STATUS"></a>

### ERR_HTTP2_INVALID_INFO_STATUS

È stato specificato un codice di stato informativo HTTP non valido. Informational status codes must be an integer between `100` and `199` (inclusive).

<a id="ERR_HTTP2_INVALID_ORIGIN"></a>

### ERR_HTTP2_INVALID_ORIGIN

HTTP/2 `ORIGIN` frames require a valid origin.

<a id="ERR_HTTP2_INVALID_PACKED_SETTINGS_LENGTH"></a>

### ERR_HTTP2_INVALID_PACKED_SETTINGS_LENGTH

Input `Buffer` and `Uint8Array` instances passed to the `http2.getUnpackedSettings()` API must have a length that is a multiple of six.

<a id="ERR_HTTP2_INVALID_PSEUDOHEADER"></a>

### ERR_HTTP2_INVALID_PSEUDOHEADER

Only valid HTTP/2 pseudoheaders (`:status`, `:path`, `:authority`, `:scheme`, and `:method`) may be used.

<a id="ERR_HTTP2_INVALID_SESSION"></a>

### ERR_HTTP2_INVALID_SESSION

An action was performed on an `Http2Session` object that had already been destroyed.

<a id="ERR_HTTP2_INVALID_SETTING_VALUE"></a>

### ERR_HTTP2_INVALID_SETTING_VALUE

È stato specificato un valore non valido per un'impostazione HTTP/2.

<a id="ERR_HTTP2_INVALID_STREAM"></a>

### ERR_HTTP2_INVALID_STREAM

Un'operazione è stata eseguita su uno stream che era già stato distrutto.

<a id="ERR_HTTP2_MAX_PENDING_SETTINGS_ACK"></a>

### ERR_HTTP2_MAX_PENDING_SETTINGS_ACK

Whenever an HTTP/2 `SETTINGS` frame is sent to a connected peer, the peer is required to send an acknowledgment that it has received and applied the new `SETTINGS`. By default, a maximum number of unacknowledged `SETTINGS` frames may be sent at any given time. This error code is used when that limit has been reached.

<a id="ERR_HTTP2_NESTED_PUSH"></a>

### ERR_HTTP2_NESTED_PUSH

An attempt was made to initiate a new push stream from within a push stream. Nested push streams are not permitted.

<a id="ERR_HTTP2_NO_SOCKET_MANIPULATION"></a>

### ERR_HTTP2_NO_SOCKET_MANIPULATION

An attempt was made to directly manipulate (read, write, pause, resume, etc.) a socket attached to an `Http2Session`.

<a id="ERR_HTTP2_ORIGIN_LENGTH"></a>

### ERR_HTTP2_ORIGIN_LENGTH

HTTP/2 `ORIGIN` frames are limited to a length of 16382 bytes.

<a id="ERR_HTTP2_OUT_OF_STREAMS"></a>

### ERR_HTTP2_OUT_OF_STREAMS

The number of streams created on a single HTTP/2 session reached the maximum limit.

<a id="ERR_HTTP2_PAYLOAD_FORBIDDEN"></a>

### ERR_HTTP2_PAYLOAD_FORBIDDEN

A message payload was specified for an HTTP response code for which a payload is forbidden.

<a id="ERR_HTTP2_PING_CANCEL"></a>

### ERR_HTTP2_PING_CANCEL

Un ping HTTP/2 è stato cancellato.

<a id="ERR_HTTP2_PING_LENGTH"></a>

### ERR_HTTP2_PING_LENGTH

I payload dei ping HTTP/2 devono avere esattamente 8 byte di lunghezza.

<a id="ERR_HTTP2_PSEUDOHEADER_NOT_ALLOWED"></a>

### ERR_HTTP2_PSEUDOHEADER_NOT_ALLOWED

Una pseudo-intestazione HTTP/2 è stata usata in modo inappropriato. Pseudo-headers are header key names that begin with the `:` prefix.

<a id="ERR_HTTP2_PUSH_DISABLED"></a>

### ERR_HTTP2_PUSH_DISABLED

An attempt was made to create a push stream, which had been disabled by the client.

<a id="ERR_HTTP2_SEND_FILE"></a>

### ERR_HTTP2_SEND_FILE

An attempt was made to use the `Http2Stream.prototype.responseWithFile()` API to send something other than a regular file.

<a id="ERR_HTTP2_SESSION_ERROR"></a>

### ERR_HTTP2_SESSION_ERROR

La `Http2Session` si è chiusa con un codice di errore diverso da zero.

<a id="ERR_HTTP2_SETTINGS_CANCEL"></a>

### ERR_HTTP2_SETTINGS_CANCEL

The `Http2Session` settings canceled.

<a id="ERR_HTTP2_SOCKET_BOUND"></a>

### ERR_HTTP2_SOCKET_BOUND

An attempt was made to connect a `Http2Session` object to a `net.Socket` or `tls.TLSSocket` that had already been bound to another `Http2Session` object.

<a id="ERR_HTTP2_SOCKET_UNBOUND"></a>

### ERR_HTTP2_SOCKET_UNBOUND

An attempt was made to use the `socket` property of an `Http2Session` that has already been closed.

<a id="ERR_HTTP2_STATUS_101"></a>

### ERR_HTTP2_STATUS_101

L'uso del Codice di stato informativo `101` è proibito in HTTP/2.

<a id="ERR_HTTP2_STATUS_INVALID"></a>

### ERR_HTTP2_STATUS_INVALID

È stato specificato un codice di stato HTTP invalido. Status codes must be an integer between `100` and `599` (inclusive).

<a id="ERR_HTTP2_STREAM_CANCEL"></a>

### ERR_HTTP2_STREAM_CANCEL

An `Http2Stream` was destroyed before any data was transmitted to the connected peer.

<a id="ERR_HTTP2_STREAM_ERROR"></a>

### ERR_HTTP2_STREAM_ERROR

È stato specificato un codice di errore diverso da zero in un frame `RST_STREAM`.

<a id="ERR_HTTP2_STREAM_SELF_DEPENDENCY"></a>

### ERR_HTTP2_STREAM_SELF_DEPENDENCY

When setting the priority for an HTTP/2 stream, the stream may be marked as a dependency for a parent stream. This error code is used when an attempt is made to mark a stream and dependent of itself.

<a id="ERR_HTTP2_TRAILERS_ALREADY_SENT"></a>

### ERR_HTTP2_TRAILERS_ALREADY_SENT

Le intestazioni del trailing sono già state inviate sul `Http2Stream`.

<a id="ERR_HTTP2_TRAILERS_NOT_READY"></a>

### ERR_HTTP2_TRAILERS_NOT_READY

The `http2stream.sendTrailers()` method cannot be called until after the `'wantTrailers'` event is emitted on an `Http2Stream` object. The `'wantTrailers'` event will only be emitted if the `waitForTrailers` option is set for the `Http2Stream`.

<a id="ERR_HTTP2_UNSUPPORTED_PROTOCOL"></a>

### ERR_HTTP2_UNSUPPORTED_PROTOCOL

`http2.connect()` was passed a URL that uses any protocol other than `http:` or `https:`.

<a id="ERR_INDEX_OUT_OF_RANGE"></a>

### ERR_HTTP2_UNSUPPORTED_PROTOCOL

Un dato indice non rientrava nell'intervallo accettato (ad esempio, offset negativi).

<a id="ERR_INVALID_ARG_TYPE"></a>

### ERR_INVALID_ARG_TYPE

Un argomento di tipo errato è stato passato a un'API Node.js.

<a id="ERR_INVALID_ASYNC_ID"></a>

### ERR_INVALID_ASYNC_ID

Un `asyncld` o `triggerAsyncld`non valido è stato passato utilizzando `AsyncHooks`. An id less than -1 should never happen.

<a id="ERR_INVALID_CALLBACK"></a>

### ERR_INVALID_CALLBACK

È stata richiesta una funzione di callback ma non era stata fornita a un'API Node.js.

<a id="ERR_INVALID_FILE_URL_HOST"></a>

### ERR_INVALID_FILE_URL_HOST

A Node.js API that consumes `file:` URLs (such as certain functions in the [`fs`][] module) encountered a file URL with an incompatible host. This situation can only occur on Unix-like systems where only `localhost` or an empty host is supported.

<a id="ERR_INVALID_FILE_URL_PATH"></a>

### ERR_INVALID_FILE_URL_PATH

A Node.js API that consumes `file:` URLs (such as certain functions in the [`fs`][] module) encountered a file URL with an incompatible path. The exact semantics for determining whether a path can be used is platform-dependent.

<a id="ERR_INVALID_HANDLE_TYPE"></a>

### ERR_INVALID_HANDLE_TYPE

An attempt was made to send an unsupported "handle" over an IPC communication channel to a child process. See [`subprocess.send()`] and [`process.send()`] for more information.

<a id="ERR_INVALID_OPT_VALUE"></a>

### ERR_INVALID_OPT_VALUE

Un valore imprevisto o non valido è stato passato in un oggetto di un opzione.

<a id="ERR_INVALID_PERFORMANCE_MARK"></a>

### ERR_INVALID_PERFORMANCE_MARK

While using the Performance Timing API (`perf_hooks`), a performance mark is invalid.

<a id="ERR_INVALID_PROTOCOL"></a>

### ERR_INVALID_PROTOCOL

È stato passato un `options.protocol` non valido.

<a id="ERR_INVALID_SYNC_FORK_INPUT"></a>

### ERR_INVALID_SYNC_FORK_INPUT

A `Buffer`, `Uint8Array` or `string` was provided as stdio input to a synchronous fork. See the documentation for the [`child_process`](child_process.html) module for more information.

<a id="ERR_INVALID_THIS"></a>

### ERR_INVALID_THIS

Una funzione API Node.js è stata chiamata con un valore `this` non compatibile.

Esempio:

```js
const { URLSearchParams } = require('url');
const urlSearchParams = new URLSearchParams('foo=bar&baz=new');

const buf = Buffer.alloc(1);
urlSearchParams.has.call(buf, 'foo');
// Throws a TypeError with code 'ERR_INVALID_THIS'
```

<a id="ERR_INVALID_TUPLE"></a>

### ERR_INVALID_TUPLE

An element in the `iterable` provided to the [WHATWG](url.html#url_the_whatwg_url_api) [`URLSearchParams` constructor][`new URLSearchParams(iterable)`] did not represent a `[name, value]` tuple – that is, if an element is not iterable, or does not consist of exactly two elements.

<a id="ERR_INVALID_URL"></a>

### ERR_INVALID_URL

An invalid URL was passed to the [WHATWG](url.html#url_the_whatwg_url_api) [`URL` constructor][`new URL(input)`] to be parsed. The thrown error object typically has an additional property `'input'` that contains the URL that failed to parse.

<a id="ERR_INVALID_URL_SCHEME"></a>

### ERR_INVALID_URL_SCHEME

An attempt was made to use a URL of an incompatible scheme (protocol) for a specific purpose. It is only used in the [WHATWG URL API](url.html#url_the_whatwg_url_api) support in the [`fs`][] module (which only accepts URLs with `'file'` scheme), but may be used in other Node.js APIs as well in the future.

<a id="ERR_IPC_CHANNEL_CLOSED"></a>

### ERR_IPC_CHANNEL_CLOSED

È stato effettuato un tentativo di utilizzare un canale di comunicazione IPC che era già stata chiuso.

<a id="ERR_IPC_DISCONNECTED"></a>

### ERR_IPC_DISCONNECTED

An attempt was made to disconnect an IPC communication channel that was already disconnected. See the documentation for the [`child_process`](child_process.html) module for more information.

<a id="ERR_IPC_ONE_PIPE"></a>

### ERR_IPC_ONE_PIPE

An attempt was made to create a child Node.js process using more than one IPC communication channel. See the documentation for the [`child_process`](child_process.html) module for more information.

<a id="ERR_IPC_SYNC_FORK"></a>

### ERR_IPC_SYNC_FORK

An attempt was made to open an IPC communication channel with a synchronously forked Node.js process. See the documentation for the [`child_process`](child_process.html) module for more information.

<a id="ERR_MISSING_ARGS"></a>

### ERR_METHOD_NOT_IMPLEMENTED

Un argomento necessario di un API Node.js non è stato passato. This is only used for strict compliance with the API specification (which in some cases may accept `func(undefined)` but not `func()`). In most native Node.js APIs, `func(undefined)` and `func()` are treated identically, and the [`ERR_INVALID_ARG_TYPE`][] error code may be used instead.

<a id="ERR_MISSING_DYNAMIC_INSTANTIATE_HOOK"></a>

### ERR_MISSING_DYNAMIC_INSTANTIATE_HOOK

> Stability: 1 - Experimental

Used when an \[ES6 module\]\[\] loader hook specifies `format: 'dynamic` but does not provide a `dynamicInstantiate` hook.

<a id="ERR_MISSING_MODULE"></a>

### ERR_MISSING_MODULE

> Stability: 1 - Experimental

Used when an \[ES6 module\]\[\] cannot be resolved.

<a id="ERR_MODULE_RESOLUTION_LEGACY"></a>

### ERR_MODULE_RESOLUTION_LEGACY

> Stabilità: 1 - Sperimentale

Used when a failure occurred resolving imports in an \[ES6 module\]\[\].

<a id="ERR_MULTIPLE_CALLBACK"></a>

### ERR_MULTIPLE_CALLBACK

Un callback è stato chiamato più di una volta.

*Note*: A callback is almost always meant to only be called once as the query can either be fulfilled or rejected but not both at the same time. The latter would be possible by calling a callback more than once.

<a id="ERR_NAPI_CONS_FUNCTION"></a>

### ERR_NAPI_CONS_FUNCTION

Durante l'utilizzo di `N-API`, un constructor passato non era una funzione.

<a id="ERR_NAPI_CONS_PROTOTYPE_OBJECT"></a>

### ERR_NAPI_CONS_PROTOTYPE_OBJECT

While using `N-API`, `Constructor.prototype` was not an object.

<a id="ERR_NAPI_INVALID_DATAVIEW_ARGS"></a>

### ERR_NAPI_INVALID_DATAVIEW_ARGS

While calling `napi_create_dataview()`, a given `offset` was outside the bounds of the dataview or `offset + length` was larger than a length of given `buffer`.

<a id="ERR_NAPI_INVALID_TYPEDARRAY_ALIGNMENT"></a>

### ERR_NAPI_INVALID_TYPEDARRAY_ALIGNMENT

While calling `napi_create_typedarray()`, the provided `offset` was not a multiple of the element size.

<a id="ERR_NAPI_INVALID_TYPEDARRAY_LENGTH"></a>

### ERR_NAPI_INVALID_TYPEDARRAY_LENGTH

Durante la chiamata a `napi_create_typedarray()`, `(length * size_of_element) +
byte_offset` era più grande della lunghezza del `buffer` fornito.

<a id="ERR_NAPI_TSFN_CALL_JS"></a>

### ERR_NAPI_TSFN_CALL_JS

An error occurred while invoking the JavaScript portion of the thread-safe function.

<a id="ERR_NAPI_TSFN_GET_UNDEFINED"></a>

### ERR_NAPI_TSFN_GET_UNDEFINED

An error occurred while attempting to retrieve the JavaScript `undefined` value.

<a id="ERR_NAPI_TSFN_START_IDLE_LOOP"></a>

### ERR_NAPI_TSFN_START_IDLE_LOOP

On the main thread, values are removed from the queue associated with the thread-safe function in an idle loop. This error indicates that an error has occurred when attemping to start the loop.

<a id="ERR_NAPI_TSFN_STOP_IDLE_LOOP"></a>

### ERR_NAPI_TSFN_STOP_IDLE_LOOP

Once no more items are left in the queue, the idle loop must be suspended. This error indicates that the idle loop has failed to stop.

<a id="ERR_NO_ICU"></a>

### ERR_NO_ICU

An attempt was made to use features that require [ICU](intl.html#intl_internationalization_support), but Node.js was not compiled with ICU support.

<a id="ERR_SOCKET_ALREADY_BOUND"></a>

### ERR_SOCKET_ALREADY_BOUND

È stato effettuato un tentativo di associare un socket che è già stato associato.

<a id="ERR_SOCKET_BAD_PORT"></a>

### ERR_SOCKET_BAD_PORT

An API function expecting a port > 0 and < 65536 received an invalid value.

<a id="ERR_SOCKET_BAD_TYPE"></a>

### ERR_SOCKET_BAD_TYPE

An API function expecting a socket type (`udp4` or `udp6`) received an invalid value.

<a id="ERR_SOCKET_CANNOT_SEND"></a>

### ERR_SOCKET_CANNOT_SEND

I dati potrebbero essere inviati su un socket.

<a id="ERR_SOCKET_CLOSED"></a>

### ERR_SOCKET_CLOSED

È stato effettuato un tentativo di operare su un socket già chiuso.

<a id="ERR_SOCKET_DGRAM_NOT_RUNNING"></a>

### ERR_SOCKET_DGRAM_NOT_RUNNING

È stata effettuata una chiamata e il sottosistema UDP non era in esecuzione.

<a id="ERR_STDERR_CLOSE"></a>

### ERR_STDERR_CLOSE<!-- YAML
removed: v8.16.0
changes:

  - version: v8.16.0
    pr-url: https://github.com/nodejs/node/pull/23053
    description: Rather than emitting an error, `process.stderr.end()` now
                 only closes the stream side but not the underlying resource,
                 making this error obsolete.
-->An attempt was made to close the 

`process.stderr` stream. By design, Node.js does not allow `stdout` or `stderr` streams to be closed by user code.

<a id="ERR_STDOUT_CLOSE"></a>

### ERR_STDOUT_CLOSE

<!-- YAML
removed: v8.16.0
changes:

  - version: v8.16.0
    pr-url: https://github.com/nodejs/node/pull/23053
    description: Rather than emitting an error, `process.stderr.end()` now
                 only closes the stream side but not the underlying resource,
                 making this error obsolete.
-->

È stato effettuato un tentativo di chiudere lo stream `process.stdout`. By design, Node.js does not allow `stdout` or `stderr` streams to be closed by user code.

<a id="ERR_TLS_CERT_ALTNAME_INVALID"></a>

### ERR_TLS_CERT_ALTNAME_INVALID

While using TLS, the hostname/IP of the peer did not match any of the subjectAltNames in its certificate.

<a id="ERR_TLS_DH_PARAM_SIZE"></a>

### ERR_TLS_DH_PARAM_SIZE

While using TLS, the parameter offered for the Diffie-Hellman (`DH`) key-agreement protocol is too small. By default, the key length must be greater than or equal to 1024 bits to avoid vulnerabilities, even though it is strongly recommended to use 2048 bits or larger for stronger security.

<a id="ERR_TLS_HANDSHAKE_TIMEOUT"></a>

### ERR_TLS_HANDSHAKE_TIMEOUT

Un handshake TLS/SSL è scaduto. In this case, the server must also abort the connection.

<a id="ERR_TLS_RENEGOTIATION_FAILED"></a>

### ERR_TLS_RENEGOTIATION_FAILED

A TLS renegotiation request has failed in a non-specific way.

<a id="ERR_TLS_REQUIRED_SERVER_NAME"></a>

### ERR_TLS_REQUIRED_SERVER_NAME

While using TLS, the `server.addContext()` method was called without providing a hostname in the first parameter.

<a id="ERR_TLS_SESSION_ATTACK"></a>

### ERR_TLS_SESSION_ATTACK

An excessive amount of TLS renegotiations is detected, which is a potential vector for denial-of-service attacks.

<a id="ERR_TRANSFORM_ALREADY_TRANSFORMING"></a>

### ERR_TRANSFORM_ALREADY_TRANSFORMING

A Transform stream finished while it was still transforming.

<a id="ERR_TRANSFORM_WITH_LENGTH_0"></a>

### ERR_TRANSFORM_WITH_LENGTH_0

A Transform stream finished with data still in the write buffer.

<a id="ERR_UNKNOWN_SIGNAL"></a>

### ERR_UNKNOWN_SIGNAL

An invalid or unknown process signal was passed to an API expecting a valid signal (such as [`subprocess.kill()`][]).

<a id="ERR_UNKNOWN_STDIN_TYPE"></a>

### ERR_UNKNOWN_STDIN_TYPE

An attempt was made to launch a Node.js process with an unknown `stdin` file type. This error is usually an indication of a bug within Node.js itself, although it is possible for user code to trigger it.

<a id="ERR_UNKNOWN_STREAM_TYPE"></a>

### ERR_UNKNOWN_STREAM_TYPE

An attempt was made to launch a Node.js process with an unknown `stdout` or `stderr` file type. This error is usually an indication of a bug within Node.js itself, although it is possible for user code to trigger it.

<a id="ERR_V8BREAKITERATOR"></a>

### ERR_V8BREAKITERATOR

The V8 BreakIterator API was used but the full ICU data set is not installed.

<a id="ERR_VALID_PERFORMANCE_ENTRY_TYPE"></a>

### ERR_VALID_PERFORMANCE_ENTRY_TYPE

While using the Performance Timing API (`perf_hooks`), no valid performance entry types were found.

<a id="ERR_VALUE_OUT_OF_RANGE"></a>

### ERR_VALUE_OUT_OF_RANGE

Un determinato valore è fuori dal range accettato.