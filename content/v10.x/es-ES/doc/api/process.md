# Proceso

<!-- introduced_in=v0.10.0 -->

<!-- type=global -->

El objeto `process` es un `global` que proporciona información y control sobre el proceso actual de Node.js. Como un global, siempre está disponible para aplicaciones de Node.js sin utilizar `require()`.

## Eventos del Proceso

El objeto `process` es una instancia de [`EventEmitter`][].

### Evento: 'beforeExit'

<!-- YAML
added: v0.11.12
-->

El evento `'beforeExit'` es emitido cuando Node.js vacía su bucle de evento y no tiene un trabajo adicional que programar. Normalmente, el proceso Node.js se cerrará cuando no haya ningún trabajo programado, pero un listener registrado en el evento `'beforeExit'` puede hacer llamadas asíncronas y así causar que el proceso Node.js continúe.

La función callback del oyente es invocada con el valor de [`process.exitCode`][] pasado como el único argumento.

El evento `'beforeExit'` *no* es emitido para condiciones que causen la terminación explícita, como lo es llamar a [`process.exit()`][] o excepciones no detectadas.

El `'beforeExit'` *no* debe ser utilizado como una alternativa al evento `'exit'` a menos que la intención sea programar un trabajo adicional.

### Evento: 'disconnect'

<!-- YAML
added: v0.7.7
-->

Si el proceso Node.js es generado con un canal IPC (ver la documentación de [Proceso Secundario](child_process.html) y de [Cluster](cluster.html)), el evento `'disconnect'` será emitido cuando el canal IPC esté cerrado.

### Evento: 'exit'

<!-- YAML
added: v0.1.7
-->

* `code` {integer}

El evento `'exit'` es emitido cuando el proceso Node.js está por cerrarse como un resultado de:

* El método `process.exit()` es llamado explícitamente;
* El bucle del evento Node.js ya no tiene ningún trabajo adicional a realizar.

No hay manera de prevenir la salida del bucle del evento en este punto, y una vez que todos los oyentes de `'exit'` hayan terminado de ejecutar el proceso Node.js, se terminarán.

La función callback del oyente es invocada con el código de salida especificado por la propiedad [`process.exitCode`][] o el argumento `exitCode` pasado al método [`process.exit()`].

```js
process.on('exit', (code) => {
  console.log(`About to exit with code: ${code}`);
});
```

Las funciones del oyente sólo **deben** realizar operaciones **sincrónicas**. El proceso Node.js se cerrará inmediatamente después de llamar a los oyentes del evento `'exit'`, causando que se abandone cualquier trabajo adicional que todavía se encuentre en cola en el bucle del evento. En el siguiente ejemplo, el timeout (tiempo de espera) nunca ocurrirá:

```js
process.on('exit', (code) => {
  setTimeout(() => {
    console.log('This will not run');
  }, 0);
});
```

### Evento: 'message'

<!-- YAML
added: v0.5.10
-->

* `message` { Object | boolean | number | string | null } a parsed JSON object or a serializable primitive value.
* `sendHandle` {net.Server|net.Socket} un objeto [`net.Server`][] o [`net.Socket`][] o indefinido.

Si el proceso Node.js es generado con un canal IPC (vea la documentación de [Proceso Secundario](child_process.html) y de [Cluster](cluster.html)), el evento `'message'` es emitido cada vez que el proceso secundario recibe un mensaje enviado por el proceso primario, utilizando [`childprocess.send()`][].

El mensaje pasa a través de la serialización y análisis. El mensaje resultante podría no ser el mismo enviado originalmente.

### Event: 'multipleResolves'

<!-- YAML
added: v10.12.0
-->

* `type` {string} The error type. One of `'resolve'` or `'reject'`.
* `promise` {Promise} The promise that resolved or rejected more than once.
* `value` {any} The value with which the promise was either resolved or rejected after the original resolve.

The `'multipleResolves'` event is emitted whenever a `Promise` has been either:

* Resolved more than once.
* Rejected more than once.
* Rejected after resolve.
* Resolved after reject.

This is useful for tracking errors in an application while using the promise constructor. Otherwise such mistakes are silently swallowed due to being in a dead zone.

It is recommended to end the process on such errors, since the process could be in an undefined state. While using the promise constructor make sure that it is guaranteed to trigger the `resolve()` or `reject()` functions exactly once per call and never call both functions in the same call.

```js
process.on('multipleResolves', (type, promise, reason) => {
  console.error(type, promise, reason);
  setImmediate(() => process.exit(1));
});

async function main() {
  try {
    return await new Promise((resolve, reject) => {
      resolve('First call');
      resolve('Swallowed resolve');
      reject(new Error('Swallowed reject'));
    });
  } catch {
    throw new Error('Failed');
  }
}

main().then(console.log);
// resolve: Promise { 'First call' } 'Swallowed resolve'
// reject: Promise { 'First call' } Error: Swallowed reject
//     at Promise (*)
//     at new Promise (<anonymous>)
//     at main (*)
// First call
```

### Evento: 'rejectionHandled'

<!-- YAML
added: v1.4.1
-->

* `promise` {Promise} La promesa manejada tarde.

El evento `'rejectionHandled'` es emitido cuando una `Promise` ha sido rechazada y se adjuntó un manejador de errores (utilizando [`promise.catch()`][], por ejemplo) después de un giro del bucle de evento de Node.js.

El objeto `Promise` habría sido emitido previamente en un evento `'unhandledRejection'`, pero durante el curso del proceso, ganó un manejador de rechazo.

No hay noción de un nivel superior para una cadena `Promise` en el cual los rechazos pueden ser controlados siempre. Ser inherentemente asíncrono en naturaleza, un rechazo de una `Promise` puede ser manejado en un punto futuro del tiempo — posiblemente más tarde que el giro del bucle de evento que el evento `'unhandledRejection'` sea emitido.

Otra manera de decir esto es que, a diferencia del código asincrónico donde hay una lista de excepciones sin manejar que está en constante crecimiento, con las Promises (Promesas), puede haber una lista creciente y decreciente de rechazos no controlados.

En el código sincrónico, el evento `'uncaughtException'` es emitido cuando la lista de excepciones no controladas crece.

En el código asincrónico, el evento `'unhandledRejection'` es emitido cuando la lista de rechazos no controlados crece, y el evento `'rejectionHandled'` es emitido cuando la lista de rechazos no controlados decrece.

```js
const unhandledRejections = new Map();
process.on('unhandledRejection', (reason, promise) => {
  unhandledRejections.set(promise, reason);
});
process.on('rejectionHandled', (promise) => {
  unhandledRejections.delete(promise);
});
```

En este ejemplo, el `unhandledRejections` `Map` crecerá y decrecerá con el tiempo, reflejando rechazos que inician sin ser manejados y pasen a ser manejados. Es posible registrar dichos errores en un registro de errores, de manera periódica (que es probablemente lo más adecuado para aplicaciones de larga ejecución) o al salir del proceso (que es posiblemente lo más conveniente para scripts).

### Evento: 'uncaughtException'

<!-- YAML
added: v0.1.18
changes:

  - version: v10.17.0
    pr-url: https://github.com/nodejs/node/pull/26599
    description: Added the `origin` argument.
-->

* `err` {Error} The uncaught exception.
* `origin` {string} Indicates if the exception originates from an unhandled rejection or from synchronous errors. Can either be `'uncaughtException'` or `'unhandledRejection'`.

El evento `'uncaughtException'` es emitido cuando una excepción no capturada de JavaScript vuelve al bucle de eventos. By default, Node.js handles such exceptions by printing the stack trace to `stderr` and exiting with code 1, overriding any previously set [`process.exitCode`][]. Añadir un manejador para el evento `'uncaughtException'` anula este comportamiento predeterminado. Alternatively, change the [`process.exitCode`][] in the `'uncaughtException'` handler which will result in the process exiting with the provided exit code. Otherwise, in the presence of such handler the process will exit with 0.

```js
process.on('uncaughtException', (err, origin) => {
  fs.writeSync(
    process.stderr.fd,
    `Caught exception: ${err}\n` +
    `Exception origin: ${origin}`
  );
});

setTimeout(() => {
  console.log('This will still run.');
}, 500);

// Intentionally cause an exception, but don't catch it.
nonexistentFunc();
console.log('This will not run.');
```

#### Advertencia: Usando `'uncaughtException'` correctamente

Note que `'uncaughtException'` es un mecanismo crudo para el manejo de excepciones destinadas a ser usada sólo como último recurso. El evento *no debe* ser usado como un equivalente a `On Error Resume Next`. Excepciones no controladas intrínsecamente significan que una aplicación está en un estado no definido. Intentar reanudar el código de la aplicación sin recuperarse apropiadamente de la excepción puede causar problemas adicionales impredecibles e imprevisibles.

Las excepciones arrojadas desde dentro del manejador de eventos no serán capturadas. En su lugar, el proceso se cerrará con un código de salida distinto de cero y se imprimirá el stack trace. Esto es para evitar una recursión infinita.

Intentar reanudar normalmente luego de una excepción no capturada puede ser similar a desconectar la energía cuando se está actualizando una computadora — nueve de diez veces no ocurre nada - pero la décima vez, el sistema se corrompe.

El uso correcto de `'uncaughtException'` es para realizar la limpieza sincrónica de recursos asignados (por ejemplo, descriptores de archivos, manejadores, etc.) antes de finalizar el proceso. **No es seguro reanudar el funcionamiento normal después de `'uncaughtException'`.**

Para reiniciar de manera confiable una aplicación que ha fallado, haya o no emitido un `'uncaughtException'`, se debe utilizar un monitor que se ejecute en un proceso externo para detectar fallas de la aplicación y recuperar o reiniciar de ser necesario.

### Evento: 'unhandledRejection'

<!-- YAML
added: v1.4.1
changes:

  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/8217
    description: Not handling `Promise` rejections is deprecated.
  - version: v6.6.0
    pr-url: https://github.com/nodejs/node/pull/8223
    description: Unhandled `Promise` rejections will now emit
                 a process warning.
-->

* `reason` {Error|any} El objeto con el cual la promesa fue rechazada (típicamente un objeto [`Error`][]).
* `promise` {Promise} The rejected promise.

El evento `'unhandledRejection'` es emitido cada vez que una `Promise` es rechazada y no hay un manejador de error adjunto a la promesa dentro de un turno del bucle de evento. Al programar con Promises, las excepciones son encapsuladas como "promesas rechazadas". Los rechazos pueden ser capturados y manejados utilizando [`promise.catch()`][], y son propagados a través de una cadena `Promise`. El evento `'unhandledRejection'` es útil para detectar y hacer seguimiento de promesas que fueron rechazadas y cuyos rechazos todavía no han sido manejados.

```js
process.on('unhandledRejection', (reason, promise) => {
  console.log('Unhandled Rejection at:', promise, 'reason:', reason);
  // Application specific logging, throwing an error, or other logic here
});

somePromise.then((res) => {
  return reportToUser(JSON.pasre(res)); // note the typo (`pasre`)
}); // no `.catch()` or `.then()`
```

Lo siguiente también desencadenará al evento `'unhandledRejection'` a ser emitido:

```js
function SomeResource() {
  // Inicialmente establece el estado de carga a una promesa rechazada
  this.loaded = Promise.reject(new Error('Resource not yet loaded!'));
}

const resource = new SomeResource();
// sin .catch o .then en resource.loaded por al menos un turno
```

En este caso de ejemplo, es posible rastrear el rechazo como un error de desarrollador, como sería típicamente el caso para otros eventos `'unhandledRejection'`. Para abordar dichas fallas, un manejador [`.catch(() => { })`][`promise.catch()`] no operacional puede ser adjuntado a `resource.loaded`, lo cual prevendría que se emitiera el evento `'unhandledRejection'`.

### Evento: 'warning'

<!-- YAML
added: v6.0.0
-->

* `warning` {Error} Las propiedades clave de la advertencia son: 
  * `name` {string} El nombre de la advertencia. **Predeterminado:** `'Warning'`.
  * `message` {string} Una descripción de la advertencia proporcionada por el sistema.
  * `stack` {string} Un stack trace a la ubicación en el código donde se emitió la advertencia.

El evento `'warning'` es emitido cada vez que Node.js emite una advertencia del proceso.

Una advertencia de proceso es similar a un error en el sentido de que describe condiciones excepcionales que son traídas a la atención del usuario. Sin embargo, las advertencias no son parte del flujo de manejo de errores de Node.js y JavaScript normal. Node.js puede emitir advertencias cada vez que detecte malas prácticas de programación que pueden conducir a un rendimiento de la aplicación sub-óptimo, bugs o vulnerabilidades de seguridad.

```js
process.on('warning', (warning) => {
  console.warn(warning.name);    // Imprime el nombre de la advertencia
  console.warn(warning.message); // Imprime el mensaje de la advertencia
  console.warn(warning.stack);   // Imprime el stack trace
});
```

Por defecto, Node.js imprimirá advertencias de proceso en `stderr`. La opción de línea de comando `--no-warnings` puede ser usada para suprimir la salida de la consola predeterminada, pero el evento `'warning'` aún será emitido por el objeto `process`.

El siguiente ejemplo ilustra la advertencia que se imprime en `stderr` cuando se han añadido muchos oyentes a un evento:

```txt
$ node
> events.defaultMaxListeners = 1;
> process.on('foo', () => {});
> process.on('foo', () => {});
> (node:38638) MaxListenersExceededWarning: Possible EventEmitter memory leak
detected. 2 foo listeners added. Use emitter.setMaxListeners() to increase limit
```

En cambio, el siguiente ejemplo desactiva la salida de advertencia predeterminada y añade un manejador personalizado al evento `'warning'`:

```txt
$ node --no-warnings
> const p = process.on('warning', (warning) => console.warn('Do not do that!'));
> events.defaultMaxListeners = 1;
> process.on('foo', () => {});
> process.on('foo', () => {});
> Do not do that!
```

La opción de línea de comando `--trace-warnings` puede ser usada para hacer que la salida de la consola predeterminada para advertencias incluya el stack trace completo de la advertencia.

El arranque de Node.js utilizando la bandera de línea de comando `--throw-deprecation` causará que se arrojen como excepciones advertencias de desaprobación personalizadas.

El uso de la bandera de línea de comando `--trace-deprecation` causa que la desaprobación personalizada se imprima en `stderr`, en conjunto con el stack trace.

El uso de la bandera de línea de comando `--no-deprecation` suprimirá todos los informes de la desaprobación personalizada.

Las banderas de línea de comando `*-deprecation` sólo afectan a las advertencias que utilizan el nombre `'DeprecationWarning'`.

#### Emisión de advertencias personalizadas

Vea el método [`process.emitWarning()`](#process_process_emitwarning_warning_type_code_ctor) para emitir advertencias personalizadas o específicas de aplicación.

### Eventos de Señal

<!--type=event-->

<!--name=SIGINT, SIGHUP, etc.-->

Los eventos de señal serán emitidos cuando el proceso Node.js reciba una señal. Por favor, diríjase a signal(7) para obtener un listado de nombres de señales POSIX estándares, como `'SIGINT'`, `'SIGHUP'`, etc.

El manejador de señal recibirá el nombre de la señal (`'SIGINT'`, `'SIGTERM'`, etc.) como primer argumento.

El nombre de cada evento será el nombre común en mayúscula para la señal (p. ej., `'SIGINT'` para las señales `SIGINT`).

```js
// Comience leyendo desde stdin, así el proceso no se cierra.
process.stdin.resume();

process.on('SIGINT', () => {
  console.log('Received SIGINT. Press Control-D to exit.');
});

// Usando una función simple para manejar múltiples señales
function handle(signal) {
  console.log(`Received ${signal}`);
}

process.on('SIGINT', handle);
process.on('SIGTERM', handle);
```

* `'SIGUSR1'` está reservado por Node.js para iniciar el [depurador](debugger.html). Es posible instalar un oyente, pero el hacerlo podría interferir con el depurador.
* `'SIGTERM'` y `'SIGINT'` tienen manejadores predeterminados en plataformas distintas a Windows, que reinician el modo terminal antes de salir con el código `128 + signal number`. Si una de estas señales tiene un oyente instalado, su comportamiento predeterminado será eliminado (Node.js ya no se cerrará).
* `'SIGPIPE'` es ignorado por defecto. Puede tener un oyente instalado.
* `'SIGHUP'` is generated on Windows when the console window is closed, and on other platforms under various similar conditions. See signal(7). Puede tener un oyente instalado, sin embargo, Node.js será cerrado incondicionalmente por Windows unos 10 segundos después. En plataformas distintas de Windows, el comportamiento por defecto de `SIGHUP` es cerrar Node.js, pero una vez que se haya instalado un oyente, su comportamiento por defecto será eliminado.
* `'SIGTERM'` no está soportado en Windows, puede ser escuchado.
* `'SIGINT'` desde el terminal está soportado en todas las plataformas y puede ser generado usualmente con `<Ctrl>+C` (aunque esto puede ser configurable). No es generado cuando está habilitado el modo terminal sin procesar.
* `'SIGBREAK'` es enviado en Windows cuando se presiona `<Ctrl>+<Break>`. En plataformas distintas de Windows, se puede escuchar, pero no hay manera de enviarlo o generarlo.
* `'SIGWINCH'` es enviado cuando la consola ha sido redimensionada. En Windows, esto solo ocurrirá al escribir en la consola cuando se mueva el cursor, o cuando se use un tty legible en modo raw.
* `'SIGKILL'` no puede tener un oyente instalado, esto cerrará incondicionalmente Node.js en todas las plataformas.
* `'SIGSTOP'` no puede tener un oyente instalado.
* `'SIGBUS'`, `'SIGFPE'`, `'SIGSEGV'` y `'SIGILL'`, cuando no se levantan artificialmente utilizando kill(2), intrínsicamente abandonan el proceso en un estado en el cual no es seguro intentar llamar a oyentes JS. Hacer esto podría llevar al proceso a colgarse en un bucle infinito, ya que los oyentes adjuntos que utilizan `process.on()` son llamados asincrónicamente, y, por lo tanto, son incapaces de corregir el problema subyaciente.

Windows no soporta el envío de señales, pero Node.js ofrece una emulación con [`process.kill()`][] y [`subprocess.kill()`][]. El envío de la señal `0` puede ser usado para probar la existencia de un proceso. El envío de `SIGINT`, `SIGTERM`, y `SIGKILL` causa la terminación incondicional del proceso objetivo.

## process.abort()

<!-- YAML
added: v0.7.0
-->

El método `process.abort()` causa que el proceso Node.js se cierre inmediatamente y genere un archivo core.

This feature is not available in [`Worker`][] threads.

## process.allowedNodeEnvironmentFlags

<!-- YAML
added: v10.10.0
-->

* {Set}

The `process.allowedNodeEnvironmentFlags` property is a special, read-only `Set` of flags allowable within the [`NODE_OPTIONS`][] environment variable.

`process.allowedNodeEnvironmentFlags` extends `Set`, but overrides `Set.prototype.has` to recognize several different possible flag representations. `process.allowedNodeEnvironmentFlags.has()` will return `true` in the following cases:

* Flags may omit leading single (`-`) or double (`--`) dashes; e.g., `inspect-brk` for `--inspect-brk`, or `r` for `-r`.
* Flags passed through to V8 (as listed in `--v8-options`) may replace one or more *non-leading* dashes for an underscore, or vice-versa; e.g., `--perf_basic_prof`, `--perf-basic-prof`, `--perf_basic-prof`, etc.
* Flags may contain one or more equals (`=`) characters; all characters after and including the first equals will be ignored; e.g., `--stack-trace-limit=100`.
* Flags *must* be allowable within [`NODE_OPTIONS`][].

When iterating over `process.allowedNodeEnvironmentFlags`, flags will appear only *once*; each will begin with one or more dashes. Flags passed through to V8 will contain underscores instead of non-leading dashes:

```js
process.allowedNodeEnvironmentFlags.forEach((flag) => {
  // -r
  // --inspect-brk
  // --abort_on_uncaught_exception
  // ...
});
```

The methods `add()`, `clear()`, and `delete()` of `process.allowedNodeEnvironmentFlags` do nothing, and will fail silently.

If Node.js was compiled *without* [`NODE_OPTIONS`][] support (shown in [`process.config`][]), `process.allowedNodeEnvironmentFlags` will contain what *would have* been allowable.

## process.arch

<!-- YAML
added: v0.5.0
-->

* {string}

La propiedad `process.arch` devuelve una string que identifica la arquitectura del CPU del sistema operativo para el cual se compiló el binario de Node.js.

Los posibles valores actuales son: `'arm'`, `'arm64'`, `'ia32'`, `'mips'`, `'mipsel'`, `'ppc'`, `'ppc64'`, `'s390'`, `'s390x'`, `'x32'` y `'x64'`.

```js
console.log(`This processor architecture is ${process.arch}`);
```

## process.argv

<!-- YAML
added: v0.1.27
-->

* {string[]}

La propiedad `process.argv` devuelve un array que contiene los argumentos de línea de comando pasados cuando se inició el proceso Node.js. El primer elemento será [`process.execPath`]. Vea `process.argv0` si se necesita acceso al valor original de `argv[0]`. El segundo elemento será la ruta para el archivo de JavaScript siendo ejecutado. Los elementos restantes serán argumentos de línea de comando adicional cualesquiera.

Por ejemplo, asumiendo el siguiente script para `process-args.js`:

```js
// imprimir process.argv
process.argv.forEach((val, index) => {
  console.log(`${index}: ${val}`);
});
```

Iniciar el proceso Node.js como:

```console
$ node process-args.js one two=three four
```

Generaría la salida:

```text
0: /usr/local/bin/node
1: /Users/mjr/work/node/process-args.js
2: one
3: two=three
4: four
```

## process.argv0

<!-- YAML
added: v6.4.0
-->

* {string}

La propiedad `process.argv0` almacena una copia de sólo lectura del valor original de `argv[0]` pasado cuando Node.js inicia.

```console
$ bash -c 'exec -a customArgv0 ./node'
> process.argv[0]
'/Volumes/code/external/node/out/Release/node'
> process.argv0
'customArgv0'
```

## process.channel

<!-- YAML
added: v7.1.0
-->

* {Object}

Si el proceso Node.js fue generado con un canal IPC (vea la documentación del [Proceso Secundario](child_process.html)), la propiedad `process.channel` es una referencia al canal IPC. No existe un canal IPC, esta propiedad es `undefined`.

## process.chdir(directory)

<!-- YAML
added: v0.1.17
-->

* `directory` {string}

El método `process.chdir()` cambia el directorio actualmente operativo del proceso Node.js o arroja una excepción si se falla al hacerlo (por ejemplo, si el `directory` especificado no existe).

```js
console.log(`Starting directory: ${process.cwd()}`);
try {
  process.chdir('/tmp');
  console.log(`New directory: ${process.cwd()}`);
} catch (err) {
  console.error(`chdir: ${err}`);
}
```

This feature is not available in [`Worker`][] threads.

## process.config

<!-- YAML
added: v0.7.7
-->

* {Object}

La propiedad `process.config` devuelve un `Object` que contiene la representación de JavaScript de las opciones de configuración usadas para compilar el actual Node.js ejecutable. Esto es igual que el archivo `config.gypi` que fue producido al correr el script `./configure`.

Un ejemplo de la salida posible luce así:

<!-- eslint-skip -->

```js
{
  target_defaults:
   { cflags: [],
     default_configuration: 'Release',
     defines: [],
     include_dirs: [],
     libraries: [] },
  variables:
   {
     host_arch: 'x64',
     node_install_npm: 'true',
     node_prefix: '',
     node_shared_cares: 'false',
     node_shared_http_parser: 'false',
     node_shared_libuv: 'false',
     node_shared_zlib: 'false',
     node_use_dtrace: 'false',
     node_use_openssl: 'true',
     node_shared_openssl: 'false',
     strict_aliasing: 'true',
     target_arch: 'x64',
     v8_use_snapshot: 'true'
   }
}
```

La propiedad `process.config` **no** es de sólo lectura y existen módulos en el ecosistema que son conocidos por extender, modificar o remplazar completamente el valor de `process.config`.

## process.connected

<!-- YAML
added: v0.7.2
-->

* {boolean}

Si el proceso Node.js es generado con un canal IPC (vea la documentación del [Proceso Secundario](child_process.html) y de [Cluster](cluster.html)), la propiedad `process.connected` devolverá `true` siempre que el canal IPC esté conectado, y devolverá `false` después de que se llame a `process.disconnect()`.

Una vez que `process.connected` sea `false`, ya no será posible enviar mensajes por el canal IPC usando `process.send()`.

## process.cpuUsage([previousValue])

<!-- YAML
added: v6.1.0
-->

* `previousValue` {Object} Un valor de devolución previo a llamar a `process.cpuUsage()`
* Devuelve: {Object} * `user` {integer} * `system` {integer}

El método `process.cpuUsage()` devuelve al usuario y al CPU del sistema el tiempo de uso del proceso actual, en un objeto con las propiedades `user` y `system`, cuyos valores son valores de microsegundos (millonésima de segundo). Estos valores miden el tiempo empleado en usuario y en código de sistema respectivamente, y pueden llegar a ser mayores que el tiempo real transcurrido si múltiples núcleos de CPU están realizando el trabajo para este proceso.

El resultado de una llamada previa a `process.cpuUsage()` puede ser pasado como el argumento a la función, para obtener una lectura diff.

```js
const startUsage = process.cpuUsage();
// { user: 38579, system: 6986 }

// gire el CPU por 500 milisegundos
const now = Date.now();
while (Date.now() - now < 500);

console.log(process.cpuUsage(startUsage));
// { user: 514883, system: 11226 }
```

## process.cwd()

<!-- YAML
added: v0.1.8
-->

* Devuelve: {string}

El método `process.cwd()` devuelve el directorio de trabajo actual del proceso Node.js.

```js
console.log(`Current directory: ${process.cwd()}`);
```

## process.debugPort

<!-- YAML
added: v0.7.2
-->

* {number}

El puerto usado por el depurador de Node.js cuando está habilitado.

```js
process.debugPort = 5858;
```

## process.disconnect()

<!-- YAML
added: v0.7.2
-->

Si el proceso Node.js es generado con un canal IPC (vea la documentación del [Proceso Secundario](child_process.html) y de [Cluster](cluster.html)), el método `process.disconnect()` cerrará el canal IPC para el proceso primario, permitiendo que el proceso secundario se cierre con gracia una vez no haya ninguna otra conexión que lo mantenga con vida.

El efecto de llamar a `process.disconnect()` es el mismo que el de llamar a [`ChildProcess.disconnect()`][] del proceso primario.

Si el proceso Node.js no fue generado con un canal IPC, `process.disconnect()` será `undefined`.

## process.dlopen(module, filename[, flags])

<!-- YAML
added: v0.1.16
changes:

  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/12794
    description: Added support for the `flags` argument.
-->

* `module` {Object}
* `filename` {string}
* `flags` {os.constants.dlopen} **Predeterminado:** `os.constants.dlopen.RTLD_LAZY`

El método `process.dlopen()` permite cargar dinámicamente objetos compartidos. Es usado principalmente por `require()` para cargar Addons de C++, y no debería utilizarse directamente, a excepción de casos especiales. En otras palabras, [`require()`][] debe preferirse sobre `process.dlopen()`, a menos que hayan razones específicas.

El argumento `flags` es un entero que permite especificar el comportamiento de dlopen. Vea la documentación de [`os.constants.dlopen`][] para más detalles.

Si hay razones específicas para usar `process.dlopen()` (por ejemplo, para especificar banderas dlopen), suele ser útil utilizar [`require.resolve()`][] para buscar la ruta del módulo.

Una desventaja importante al llamar a `process.dlopen()` es que la instancia `module` debe ser pasada. Las funciones exportadas por el Addon de C++ serán accesibles a través de `module.exports`.

El siguiente ejemplo muestra cómo cargar un Addon de C++, nombrado como `binding`, que exporta una función `foo`. All the symbols will be loaded before the call returns, by passing the `RTLD_NOW` constant. En este ejemplo se asume que la constante está disponible.

```js
const os = require('os');
process.dlopen(module, require.resolve('binding'),
               os.constants.dlopen.RTLD_NOW);
module.exports.foo();
```

## process.emitWarning(warning[, options])

<!-- YAML
added: v8.0.0
-->

* `warning` {string|Error} La advertencia a emitir.
* `options` {Object} 
  * `type` {string} Cuando `warning` es una `String`, `type` es el nombre a usar para el *tipo* de advertencia que se emite. **Predeterminado:** `'Warning'`.
  * `code` {string} Un identificador único para la instancia de la advertencia que se emite.
  * `ctor` {Function} Cuando `warning` es una `String`, `ctor` es una función opcional usada para limitar el stack trace generado. **Predeterminado:** `process.emitWarning`.
  * `detail` {string} Texto adicional a incluir con el error.

El método `process.emitWarning()` puede usarse para emitir advertencias personalizadas o de aplicación específica del proceso. Estas pueden ser escuchadas añadiendo un manejador al evento [`'warning'`](#process_event_warning).

```js
// Emitir una advertencia con un código y un detalle adicional.
process.emitWarning('Something happened!', {
  code: 'MY_WARNING',
  detail: 'This is some additional information'
});
// Emite:
// (node:56338) [MY_WARNING] Advertencia: Something happened!
// Esta es una información adicional
```

En este ejemplo, un objeto `Error` es generado internamente por `process.emitWarning()` y pasado a través del manejador [`'warning'`](#process_event_warning).

```js
process.on('warning', (warning) => {
  console.warn(warning.name);    // 'Warning'
  console.warn(warning.message); // 'Something happened!'
  console.warn(warning.code);    // 'MY_WARNING'
  console.warn(warning.stack);   // Stack trace
  console.warn(warning.detail);  // 'This is some additional information'
});
```

If `warning` is passed as an `Error` object, the `options` argument is ignored.

## process.emitWarning(warning\[, type[, code]\]\[, ctor\])

<!-- YAML
added: v6.0.0
-->

* `warning` {string|Error} La advertencia a emitir.
* `type` {string} Cuando `warning` es una `String`, `type` es el nombre a usar para el *tipo* de advertencia que se emite. **Predeterminado:** `'Warning'`.
* `code` {string} Un identificador único para la instancia de la advertencia que se emite.
* `ctor` {Function} Cuando `warning` es una `String`, `ctor` es una función opcional usada para limitar el stack trace generado. **Predeterminado:** `process.emitWarning`.

El método `process.emitWarning()` puede usarse para emitir advertencias personalizadas o de aplicación específica del proceso. Estas pueden ser escuchadas añadiendo un manejador al evento [`'warning'`](#process_event_warning).

```js
// Emitir una advertencia utilizando una string.
process.emitWarning('Something happened!');
// Emite: (node: 56338) Advertencia: Something happened!
```

```js
// Emitir una advertencia utilizando una string y un tipo.
process.emitWarning('Something Happened!', 'CustomWarning');
// Emite: (node:56338) CustomWarning: Something Happened!
```

```js
process.emitWarning('Something happened!', 'CustomWarning', 'WARN001');
// Emite: (node:56338) [WARN001] CustomWarning: Something happened!
```

En cada uno de los ejemplos anteriores, un objeto `Error` es generado internamente por `process.emitWarning()` y pasado a través del manejador [`'warning'`](#process_event_warning).

```js
process.on('warning', (warning) => {
  console.warn(warning.name);
  console.warn(warning.message);
  console.warn(warning.code);
  console.warn(warning.stack);
});
```

Si `warning` es pasada como un objeto `Error`, será pasada a través del manejador del evento `'warning'` sin modificar (y los argumentos `type`, `code` y `ctor` serán ignorados):

```js
// Emitir una advertencia utilizando un objeto Error.
const myWarning = new Error('Something happened!');
// Utilizar la propiedad de nombre Error para especificar el nombre del tipo
myWarning.name = 'CustomWarning';
myWarning.code = 'WARN001';

process.emitWarning(myWarning);
// Emite: (node:56338) [WARN001] CustomWarning: Something happened!
```

Se arroja un `TypeError` si `warning` es algo distinto a una string o a un objeto `Error`.

Note que aunque las advertencias del proceso utilizan objetos `Error`, el mecanismo de advertencia del proceso **no** es un remplazo para los mecanismos normales para el manejo de errores.

Se implementa el siguiente manejo adicional si el `type` de advertencia es `'DeprecationWarning'`:

* Si la bandera de línea de comando `--throw-deprecation` es utilizada, se arroja la advertencia de desaprobación como una excepción en lugar de ser emitida como un evento.
* Si la bandera de línea de comando `--no-deprecation` es utilizada, se suprime la advertencia de desaprobación.
* Si la bandera de línea de comando `--trace-deprecation` es utilizada, se imprime la advertencia de desaprobación en `stderr` en conjunto con el stack trace completo.

### Evitar advertencias duplicadas

Como buena práctica, las advertencias deben ser emitidas una sola vez por proceso. Para hacerlo, se recomienda poner la `emitWarning()` detrás de una bandera booleana simple como se ilustra en el siguiente ejemplo:

```js
function emitMyWarning() {
  if (!emitMyWarning.warned) {
    emitMyWarning.warned = true;
    process.emitWarning('Only warn once!');
  }
}
emitMyWarning();
// Emite: (node: 56339) Warning: Only warn once!
emitMyWarning();
// No emite nada
```

## process.env

<!-- YAML
added: v0.1.27
changes:

  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18990
    description: Implicit conversion of variable value to string is deprecated.
-->

* {Object}

La propiedad `process.env` devuelve un objeto que contiene el entorno del usuario. Vea environ(7).

Un ejemplo de este objeto luce así:

<!-- eslint-skip -->

```js
{
  TERM: 'xterm-256color',
  SHELL: '/usr/local/bin/bash',
  USER: 'maciej',
  PATH: '~/.bin/:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin',
  PWD: '/Users/maciej',
  EDITOR: 'vim',
  SHLVL: '1',
  HOME: '/Users/maciej',
  LOGNAME: 'maciej',
  _: '/usr/local/bin/node'
}
```

Es posible modificar este objeto, pero dichas modificaciones no serán reflejadas afuera del proceso Node.js. En otras palabras, el siguiente ejemplo no funcionaría:

```console
$ node -e 'process.env.foo = "bar"' && echo $foo
```

Mientras que el siguiente sí:

```js
process.env.foo = 'bar';
console.log(process.env.foo);
```

Asignar una propiedad en `process.env` convertirá implícitamente el valor a una string. **Este comportamiento es desaprobado.** Las futuras versiones de Node.js podrían arrojar un error cuando el valor no es una string, un número o un booleano.

```js
process.env.test = null;
console.log(process.env.test);
// => 'null'
process.env.test = undefined;
console.log(process.env.test);
// => 'undefined'
```

Utilice `delete` para eliminar una propiedad de `process.env`.

```js
process.env.TEST = 1;
delete process.env.TEST;
console.log(process.env.TEST);
// => undefined
```

En sistemas operativos Windows, las variables de entorno no distinguen entre minúscula y mayúscula.

```js
process.env.TEST = 1;
console.log(process.env.test);
// => 1
```

`process.env` is read-only in [`Worker`][] threads.

## process.execArgv

<!-- YAML
added: v0.7.7
-->

* {string[]}

La propiedad `process.execArgv` devuelve el conjunto de opciones de línea de comandos específicas de Node.js pasadas cuando el proceso Node.js fue lanzado. Estas opciones no aparecen en el arreglo devuelto por la propiedad [`process.argv`][], y no incluyen el ejecutable de Node.js, el nombre del script, o cualquier opción que sigua al nombre del script. Estas opciones son útiles para generar procesos secundarios con el mismo entorno de ejecución que el proceso primario.

```console
$ node --harmony script.js --version
```

Resulta en `process.execArgv`:

<!-- eslint-disable semi -->

```js
['--harmony']
```

Y `process.argv`:

<!-- eslint-disable semi -->

```js
['/usr/local/bin/node', 'script.js', '--version']
```

## process.execPath

<!-- YAML
added: v0.1.100
-->

* {string}

La propiedad `process.execPath` devuelve el nombre de ruta absoluto del ejecutable que inició el proceso Node.js.

<!-- eslint-disable semi -->

```js
'/usr/local/bin/node'
```

## process.exit([code])

<!-- YAML
added: v0.1.13
-->

* `code` {integer} El código de salida. **Predeterminado:** `0`.

El método `process.exit()` indica a Node.js que finalice el proceso sincrónicamente con un estado de salida de `code`. Si se omite el `code`, la salida utiliza el código 'success' `0` o el valor de `process.exitCode` si se ha establecido. Node.js no terminará hasta que todos los oyentes del evento [`'exit'`] sean llamados.

Para salir con un código 'failure':

```js
process.exit(1);
```

La shell que ejecutó Node.js debería ver el código de salida como `1`.

Llamar a `process.exit()` forzará al proceso a cerrarse tan rápido como sea posible incluso si todavía hay operaciones asincrónicas pendientes que no se han completado en su totalidad, incluyendo operaciones I/O para `process.stdout` y `process.stderr`.

En la mayoría de las situaciones, no es realmente necesario llamar a `process.exit()` explícitamente. El proceso Node.js se saldrá por su propia cuenta *si no hay trabajo adicional pendiente* en el bucle del evento. La propiedad `process.exitCode` puede establecerse para decirle al proceso cuál código de salida utilizar cuando el proceso se cierre con gracia.

El siguiente ejemplo ilustra un *mal uso* del método `process.exit()` que podría conducir a que los datos impresos en stdout se trunquen y pierdan:

```js
// Esto es un ejemplo de qué *no* hacer:
if (someConditionNotMet()) {
  printUsageToStdout();
  process.exit(1);
}
```

La razón por la que esto es problemático es que las escrituras a `process.stdout` en Node.js a veces son *asincrónicas* y pueden ocurrir sobre múltiples ticks del bucle del evento Node.js. El llamar a `process.exit()`, sin embargo, forza al proceso a cerrarse *antes* de que se realicen esas escrituras adicionales en `stdout`.

En lugar de llamar a `process.exit()` directamente, el código *debe* establecer el `process.exitCode` y permitirle al proceso que se cierre naturalmente al evitar programar cualquier trabajo adicional para el bucle del evento:

```js
// Cómo establecer correctamente el código de salida al dejar
// que el proceso se cierre con gracia.
if (someConditionNotMet()) {
  printUsageToStdout();
  process.exitCode = 1;
}
```

Si es necesario terminar el proceso Node.js debido a una condición de error, arrojar un error *sin capturar* y permitir que el proceso se termine como corresponde es más seguro que llamar a `process.exit()`.

In [`Worker`][] threads, this function stops the current thread rather than the current process.

## process.exitCode

<!-- YAML
added: v0.11.8
-->

* {integer}

Un número que será el código de salida del proceso, cuando el proceso se cierre con gracia o se cierre a través de [`process.exit()`][] sin especificar un código.

Especificar un código a [`process.exit(code)`][`process.exit()`] anulará cualquier configuración previa de `process.exitCode`.

## process.getegid()

<!-- YAML
added: v2.0.0
-->

El método `process.getegid()` devuelve la identidad del grupo efectivo numérico del proceso Node.js. (Vea getegid(2).)

```js
if (process.getegid) {
  console.log(`Current gid: ${process.getegid()}`);
}
```

Esta función sólo está disponible en plataformas POSIX (es decir, no en Windows o en Android).

## process.geteuid()

<!-- YAML
added: v2.0.0
-->

* Devuelve: {Object}

El método `process.geteuid()` devuelve la identidad numérica del usuario efectivo del proceso. (Vea geteuid(2).)

```js
if (process.geteuid) {
  console.log(`Current uid: ${process.geteuid()}`);
}
```

Esta función sólo está disponible en plataformas POSIX (es decir, no en Windows o en Android).

## process.getgid()

<!-- YAML
added: v0.1.31
-->

* Devuelve: {Object}

El método `process.getgid()` devuelve la identidad numérica del grupo del proceso. (Vea getgid(2).)

```js
if (process.getgid) {
  console.log(`Current gid: ${process.getgid()}`);
}
```

Esta función sólo está disponible en plataformas POSIX (es decir, no en Windows o en Android).

## process.getgroups()

<!-- YAML
added: v0.9.4
-->

* Devuelve: {integer[]}

El método `process.getgroups()` devuelve un array con los IDs del grupo suplementario. POSIX lo deja sin especificar si el ID del grupo efectivo está incluida, pero Node.js asegura que siempre lo está.

Esta función sólo está disponible en plataformas POSIX (es decir, no en Windows o en Android).

## process.getuid()

<!-- YAML
added: v0.1.28
-->

* Devuelve: {integer}

El método `process.getuid()` devuelve la identidad numérica del usuario del proceso. (Vea getuid(2).)

```js
if (process.getuid) {
  console.log(`Current uid: ${process.getuid()}`);
}
```

Esta función sólo está disponible en plataformas POSIX (es decir, no en Windows o en Android).

## process.hasUncaughtExceptionCaptureCallback()

<!-- YAML
added: v9.3.0
-->

* Devuelve: {boolean}

Indica si un callback ha sido establecido utilizando [`process.setUncaughtExceptionCaptureCallback()`][].

## process.hrtime([time])

<!-- YAML
added: v0.7.6
-->

* `time` {integer[]} El resultado de una llamada previa a `process.hrtime()`
* Devuelve: {integer[]}

This is the legacy version of [`process.hrtime.bigint()`][] before `bigint` was introduced in JavaScript.

The `process.hrtime()` method returns the current high-resolution real time in a `[seconds, nanoseconds]` tuple `Array`, where `nanoseconds` is the remaining part of the real time that can't be represented in second precision.

`time` es un parámetro opcional que debe ser el resultado de una llamada `process.hrtime()</0 previa para diff con el tiempo actual. Si el parámetro
pasado no es una <code>Array` dupla, se arrojará un `TypeError`. El pasar una array de usuario definido en lugar del resultado de una llamada previa a `process.hrtime()` conducirá a un comportamiento indefinido.

Estos tiempos son relativos a un tiempo arbitrario en el pasado, y no están relacionados a las horas del día y, por lo tanto, no están sujetos a la deriva del reloj. El uso principal es para medir el rendimiento entre los intervalos:

```js
const NS_PER_SEC = 1e9;
const time = process.hrtime();
// [ 1800216, 25 ]

setTimeout(() => {
  const diff = process.hrtime(time);
  // [ 1, 552 ]

  console.log(`Benchmark took ${diff[0] * NS_PER_SEC + diff[1]} nanoseconds`);
  // el punto de referencia tomó 1000000552 nanosegundos
}, 1000);
```

## process.hrtime.bigint()

<!-- YAML
added: v10.7.0
-->

* Devuelve: {bigint}

The `bigint` version of the [`process.hrtime()`][] method returning the current high-resolution real time in a `bigint`.

Unlike [`process.hrtime()`][], it does not support an additional `time` argument since the difference can just be computed directly by subtraction of the two `bigint`s.

```js
const start = process.hrtime.bigint();
// 191051479007711n

setTimeout(() => {
  const end = process.hrtime.bigint();
  // 191052633396993n

  console.log(`Benchmark took ${end - start} nanoseconds`);
  // Benchmark took 1154389282 nanoseconds
}, 1000);
```

## process.initgroups(user, extraGroup)

<!-- YAML
added: v0.9.4
-->

* `user` {string|number} El nombre de usuario o su identificador numérico.
* `extraGroup` {string|number} Un nombre de grupo o identificador numérico.

El método `process.initgroups()` lee el archivo `/etc/group` e inicializa la lista de acceso de grupo, utilizando todos los grupos en los cuales el usuario es miembro. Esto es una operación privilegiada que requiere que el proceso Nothe.js tenga acceso a `root` o la capacidad de `CAP_SETGID`.

Note que se debe tener cuidado al eliminar privilegios:

```js
console.log(process.getgroups());         // [ 0 ]
process.initgroups('bnoordhuis', 1000);   // cambia el usuario
console.log(process.getgroups());         // [ 27, 30, 46, 1000, 0 ]
process.setgid(1000);                     // elimina el gid del root
console.log(process.getgroups());         // [ 27, 30, 46, 1000 ]
```

Esta función sólo está disponible en plataformas POSIX (es decir, no en Windows o en Android). This feature is not available in [`Worker`][] threads.

## process.kill(pid[, signal])

<!-- YAML
added: v0.0.6
-->

* `pid` {number} Un ID del proceso
* `signal` {string|number} La señal a enviar, ya sea una string o un número. **Predeterminado:** `'SIGTERM'`.

El método `process.kill()` envía la `signal` al proceso identificado por `pid`.

Los nombres de las señales son strings como `'SIGINT'` o `'SIGHUP'`. Vea los [Eventos de Señal](#process_signal_events) y kill(2) para más información.

Este método arrojará un error si el `pid` objetivo no existe. En un caso especial, una señal de `0` puede ser usada para probar la existencia de un proceso. Las plataformas Windows arrojarán un error si el `pid` es usado para aniquilar un grupo de proceso.

A pesar de que el nombre de esta función sea `process.kill()`, realmente sólo es un transmisor de señal, como la llamada de sistema `kill`. La señal enviada puede hacer algo distinto a aniquilar el proceso objetivo.

```js
process.on('SIGHUP', () => {
  console.log('Got SIGHUP signal.');
});

setTimeout(() => {
  console.log('Exiting.');
  process.exit(0);
}, 100);

process.kill(process.pid, 'SIGHUP');
```

When `SIGUSR1` is received by a Node.js process, Node.js will start the debugger. See [Signal Events](#process_signal_events).

## process.mainModule

<!-- YAML
added: v0.1.17
-->

* {Object}

La propiedad `process.mainModule` proporciona una manera alternativa de recuperar [`require.main`][]. La diferencia es que si el módulo principal cambia en tiempo de ejecución, [`require.main`][] puede todavía referirse al módulo principal original en módulos que se requerían antes de que ocurriera el cambio. Generalmente, es seguro asumir que ambos se refieran al mismo módulo.

Al igual que con [`require.main`][], `process.mainModule` será `undefined` si no hay un script de entrada.

## process.memoryUsage()

<!-- YAML
added: v0.1.16
changes:

  - version: v7.2.0
    pr-url: https://github.com/nodejs/node/pull/9587
    description: Added `external` to the returned object.
-->

* Devuelve: {Object} * `rss` {integer} * `heapTotal` {integer} * `heapUsed` {integer} * `external` {integer}

El método `process.memoryUsage()` devuelve un objeto que describe el uso de memoria del proceso Node.js medido en bytes.

Por ejemplo, el código:

```js
console.log(process.memoryUsage());
```

Generará:

<!-- eslint-skip -->

```js
{
  rss: 4935680,
  heapTotal: 1826816,
  heapUsed: 650472,
  external: 49879
}
```

`heapTotal` y `heapUsed` se refieren al uso de memoria de V8. `external` se refiere al uso de memoria de objetos C++ vinculados a objetos JavaScript administrados por V8. `rss`, que por sus siglas en inglés se refiere a Resident Set Size, es la cantidad de espacio ocupado en el dispositivo de memoria principal (que es un subconjunto de memoria total asignada) para el proceso, la cual incluye el *montículo*, el *segmento de código* y el *stack*.

El *montículo* es donde los objetos, strings y cierres son almacenados. Las variables son almacenadas en el *stack* y el código JavaScript actual reside en el *segmento de código*.

When using [`Worker`][] threads, `rss` will be a value that is valid for the entire process, while the other fields will only refer to the current thread.

## process.nextTick(callback[, ...args])

<!-- YAML
added: v0.1.26
changes:

  - version: v1.8.1
    pr-url: https://github.com/nodejs/node/pull/1077
    description: Additional arguments after `callback` are now supported.
-->

* `callback` {Function}
* `...args` {any} Argumentos adicionales a pasar al invocar el `callback`

El método `process.nextTick()` añade el `callback` a la "cola de siguiente tramo". Una vez que el turno actual del bucle de evento se complete, se llamarán todas las callbacks actualmente en la cola de siguiente tramo.

Esto *no* es un alias simple de [`setTimeout(fn, 0)`][]. Es mucho más eficiente. Se ejecuta antes de que los eventos adicionales de I/O (incluyendo temporizadores) se activen en tramos subsecuentes del bucle de evento.

```js
console.log('start');
process.nextTick(() => {
  console.log('nextTick callback');
});
console.log('scheduled');
// Salida:
// inicio
// programado
// callback nextTick
```

Esto es importando al desarrollar APIs para ofrecerle a los usuarios la oportunidad de asignar manejadores de evento *después* de que un objeto se haya construido, pero antes de que I/O haya ocurrido:

```js
function MyThing(options) {
  this.setupOptions(options);

  process.nextTick(() => {
    this.startDoingStuff();
  });
}

const thing = new MyThing();
thing.getReadyForStuff();

// thing.startDoingStuff() se llama ahora, no antes.
```

Es muy importante para las APIs ser 100% sincrónicas o 100% asincrónicas. Considere este ejemplo:

```js
// ¡ADVERTENCIA!  ¡NO LO UTILICE!  ¡MALO, INSEGURO, PELIGROSO!
function maybeSync(arg, cb) {
  if (arg) {
    cb();
    return;
  }

  fs.stat('file', cb);
}
```

Esta API es peligrosa porque en el siguiente caso:

```js
const maybeTrue = Math.random() > 0.5;

maybeSync(maybeTrue, () => {
  foo();
});

bar();
```

No se está claro si primero se llamará a `foo()` o a `bar()`.

El siguiente enfoque es mucho mejor:

```js
function definitelyAsync(arg, cb) {
  if (arg) {
    process.nextTick(cb);
    return;
  }

  fs.stat('file', cb);
}
```

The next tick queue is completely drained on each pass of the event loop **before** additional I/O is processed. As a result, recursively setting `nextTick()` callbacks will block any I/O from happening, just like a `while(true);` loop.

## process.noDeprecation

<!-- YAML
added: v0.8.0
-->

* {boolean}

La propiedad `process.noDeprecation` indica que la bandera `--no-deprecation` está establecida en el proceso Node.js actual. See the documentation for the [`'warning'` event](#process_event_warning) and the [`emitWarning()` method](#process_process_emitwarning_warning_type_code_ctor) for more information about this flag's behavior.

## process.pid

<!-- YAML
added: v0.1.15
-->

* {integer}

La propiedad `process.pid` devuelve el PID del proceso.

```js
console.log(`This process is pid ${process.pid}`);
```

## process.platform

<!-- YAML
added: v0.1.16
-->

* {string}

La propiedad `process.platform` devuelve una string que identifica la plataforma del sistema operativo en el cual se está ejecutando el proceso Node.js.

Los posibles valores actuales son:

* `'aix'`
* `'darwin'`
* `'freebsd'`
* `'linux'`
* `'openbsd'`
* `'sunos'`
* `'win32'`

```js
console.log(`This platform is ${process.platform}`);
```

El valor `'android'` también puede ser devuelto si el Node.js está construido en el sistema operativo Android. Sin embargo, el soporte de Android en Node.js [es experimental](https://github.com/nodejs/node/blob/master/BUILDING.md#androidandroid-based-devices-eg-firefox-os).

## process.ppid

<!-- YAML
added: v9.2.0
-->

* {integer}

La propiedad `process.ppid` devuelve el PID del proceso primario actual.

```js
console.log(`The parent process is pid ${process.ppid}`);
```

## process.release

<!-- YAML
added: v3.0.0
changes:

  - version: v4.2.0
    pr-url: https://github.com/nodejs/node/pull/3212
    description: The `lts` property is now supported.
-->

* {Object}

La propiedad `process.release` devuelve un `Object` que contiene los metadatos relacionados con la versión actual, incluyendo los URLs para el tarball fuente y los tarballs de sólo encabezado.

`process.release` contiene las siguientes propiedades:

* `name` {string} un valor que siempre será `'node'` para Node.js. Para versiones legacy io.js, este será `'io.js'`.
* `sourceUrl` {string} una URL absoluta que apunta al archivo *`.tar.gz`* que contiene el código fuente de la versión actual.
* `headersUrl`{string} una URL absoluta que apunta al archivo *`.tar.gz`* que contiene sólo los archivos fuente de encabezado para la versión actual. Este archivo es significativamente más pequeño que el archivo fuente completo y puede ser utilizado para compilar add-ons nativos de Node.js.
* `libUrl` {string} una URL absoluta que apunta a un archivo *`node.lib`* que coincide con la arquitectura y versión del lanzamiento actual. El archivo es utilizado para compilar add-ons nativos de Node.js. *Esta propiedad sólo está presente en construcciones Windows de Node.js y no aparecerá en ninguna otra plataforma.*
* `lts` {string} una etiqueta de string que identifica la etiqueta [LTS](https://github.com/nodejs/Release) para esta versión. Esta propiedad sólo existe en versiones LTS y es `undefined` para todos los otros tipos de versiones, incluyendo las versions *Actuales*. Actualmente, lo valores válidos son: 
  * `'Argon'` para la línea LTS 4.x que comienza con 4.2.0.
  * `'Boron'` para la línea LTS 6.x que comienza con 6.9.0.
  * `'Carbon'` para la línea LTS 8.x que comienza con 8.9.1.

<!-- eslint-skip -->

```js
{
  name: 'node',
  lts: 'Argon',
  sourceUrl: 'https://nodejs.org/download/release/v4.4.5/node-v4.4.5.tar.gz',
  headersUrl: 'https://nodejs.org/download/release/v4.4.5/node-v4.4.5-headers.tar.gz',
  libUrl: 'https://nodejs.org/download/release/v4.4.5/win-x64/node.lib'
}
```

En builds personalizados de versiones no lanzadas del árbol de origen, solo puede estar presente la propiedad `name`. No se debe confiar en que las propiedades adicionales existan.

## process.send(message\[, sendHandle[, options]\]\[, callback\])

<!-- YAML
added: v0.5.9
-->

* `message` {Object}
* `sendHandle` {net.Server|net.Socket}
* `opciones` {Object}
* `callback` {Function}
* Devuelve: {boolean}

Si se genera Node.js con un canal IPC, el método `process.send()` puede ser utilizado para enviar mensajes al proceso primario. Los mensajes serán recibidos como un evento [`'message'`][] en el objeto [`ChildProcess`][] del proceso primario.

Si Node.js no fue generado con un canal IPC, `process.send()` será `undefined`.

El mensaje pasa a través de la serialización y análisis. El mensaje resultante podría no ser el mismo enviado originalmente.

## process.setegid(id)

<!-- YAML
added: v2.0.0
-->

* `id` {string|number} Un nombre o ID de grupo

El método `process.setegid()` establece la identidad del grupo efectivo del proceso. (Vea setgid(2).) El `id` puede ser pasado como un ID numérico o como un string de nombre de grupo. Si un nombre de grupo es especificado, este método se bloquea mientras se resuelve el ID numérico asociado.

```js
if (process.getegid && process.setegid) {
  console.log(`Current gid: ${process.getegid()}`);
  try {
    process.setegid(501);
    console.log(`New gid: ${process.getegid()}`);
  } catch (err) {
    console.log(`Failed to set gid: ${err}`);
  }
}
```

Esta función sólo está disponible en plataformas POSIX (es decir, no en Windows o en Android). This feature is not available in [`Worker`][] threads.

## process.seteuid(id)

<!-- YAML
added: v2.0.0
-->

* `id` {string|number} Un nombre o ID de usuario

El método `process.seteuid()` establece la identidad efectiva del usuario del proceso. (Vea seteuid(2).) El `id` puede ser pasado como un ID numérico o como una string de nombre de usuario. Si un nombre de usuario es especificado, el método se bloquea mientras resuelve el ID numérico asociado.

```js
if (process.geteuid && process.seteuid) {
  console.log(`Current uid: ${process.geteuid()}`);
  try {
    process.seteuid(501);
    console.log(`New uid: ${process.geteuid()}`);
  } catch (err) {
    console.log(`Failed to set uid: ${err}`);
  }
}
```

Esta función sólo está disponible en plataformas POSIX (es decir, no en Windows o en Android). This feature is not available in [`Worker`][] threads.

## process.setgid(id)

<!-- YAML
added: v0.1.31
-->

* `id` {string|number} El nombre o ID del grupo

El método `process.setgid()` establece la identidad del grupo del proceso. (Vea setgid(2).) El `id` puede ser pasado como un ID numérico o como un string de nombre de grupo. Si un nombre de grupo es especificado, este método se bloquea mientras se resuelve el ID numérico asociado.

```js
if (process.getgid && process.setgid) {
  console.log(`Current gid: ${process.getgid()}`);
  try {
    process.setgid(501);
    console.log(`New gid: ${process.getgid()}`);
  } catch (err) {
    console.log(`Failed to set gid: ${err}`);
  }
}
```

Esta función sólo está disponible en plataformas POSIX (es decir, no en Windows o en Android). This feature is not available in [`Worker`][] threads.

## process.setgroups(groups)

<!-- YAML
added: v0.9.4
-->

* `groups` {integer[]}

El método `process.setgroups()` establece los IDs de grupo suplementario para el proceso Node.js. Esta es una operación privilegiada que requiere que el proceso Node.js tenga `root` o la capacidad `CAP_SETGID`.

El array `groups` puede contener IDs de grupo numéricos, nombres de grupo o ambos.

Esta función sólo está disponible en plataformas POSIX (es decir, no en Windows o en Android). This feature is not available in [`Worker`][] threads.

## process.setuid(id)

<!-- YAML
added: v0.1.28
-->

* `id` {integer | string}

El método `process.setuid(id)` establece la identidad del usuario del proceso. (Vea setuid(2).) El `id` puede ser pasado como un ID numérico o como una string de nombre de usuario. Si un nombre de usuario es especificado, el método se bloquea mientras se resuelve el ID numérico asociado.

```js
if (process.getuid && process.setuid) {
  console.log(`Current uid: ${process.getuid()}`);
  try {
    process.setuid(501);
    console.log(`New uid: ${process.getuid()}`);
  } catch (err) {
    console.log(`Failed to set uid: ${err}`);
  }
}
```

Esta función sólo está disponible en plataformas POSIX (es decir, no en Windows o en Android). This feature is not available in [`Worker`][] threads.

## process.setUncaughtExceptionCaptureCallback(fn)

<!-- YAML
added: v9.3.0
-->

* `fn` {Function|null}

The `process.setUncaughtExceptionCaptureCallback()` function sets a function that will be invoked when an uncaught exception occurs, which will receive the exception value itself as its first argument.

Si se establece dicha función, el evento [`'uncaughtException'`][] no será emitido. Si `--abort-on-uncaught-exception` fue pasado desde la línea de comando o establecido a través de [`v8.setFlagsFromString()`][], el proceso no abortará.

To unset the capture function, `process.setUncaughtExceptionCaptureCallback(null)` may be used. Calling this method with a non-`null` argument while another capture function is set will throw an error.

Utilizar esta función es mutuamente exclusivo con el uso del módulo incorporado [`domain`][] desaprobado.

## process.stderr

* {Stream}

La propiedad `process.stderr` devuelve un stream conectado a `stderr` (fd `2`). Es un [`net.Socket`][] (el cual es un stream [Dúplex](stream.html#stream_duplex_and_transform_streams)) a menos que fd `2` se refiera a un archivo, en cuyo caso es un stream [Escribible](stream.html#stream_writable_streams).

`process.stderr` differs from other Node.js streams in important ways. See [note on process I/O](process.html#process_a_note_on_process_i_o) for more information.

## process.stdin

* {Stream}

La propiedad `process.stdin` devuelve un stream conectado a `stdin` (fd `0`). Es un [`net.Socket`][] (el cual es un stream [Dúplex](stream.html#stream_duplex_and_transform_streams)) a menos que fd `0` se refiera a un archivo, en cuyo caso es un stream [Legible](stream.html#stream_readable_streams).

```js
process.stdin.setEncoding('utf8');

process.stdin.on('readable', () => {
  let chunk;
  // Use a loop to make sure we read all available data.
  while ((chunk = process.stdin.read()) !== null) {
    process.stdout.write(`data: ${chunk}`);
  }
});

process.stdin.on('end', () => {
  process.stdout.write('end');
});
```

Como un stream [Dúplex](stream.html#stream_duplex_and_transform_streams), `process.stdin` también puede ser usado en el modo "viejo" que es compatible con scripts escritos para Node.js anterior a v0.10. Para mayor información, vea la [compatibilidad de Stream](stream.html#stream_compatibility_with_older_node_js_versions).

In "old" streams mode the `stdin` stream is paused by default, so one must call `process.stdin.resume()` to read from it. Note que también el llamar a `process.stdin.resume()` cambiaría el stream a modo "viejo".

## process.stdout

* {Stream}

La propiedad `process.stdout` devuelve un stream conectado a `stdout` (fd `1`). Es un [`net.Socket`][] (el cual es un stream [Dúplex](stream.html#stream_duplex_and_transform_streams)) a menos que fd `1` se refiera a un archivo, en cuyo caso es un stream [Escribible](stream.html#stream_writable_streams).

Por ejemplo, para copiar `process.stdin` en `process.stdout`:

```js
process.stdin.pipe(process.stdout);
```

`process.stdout` differs from other Node.js streams in important ways. See [note on process I/O](process.html#process_a_note_on_process_i_o) for more information.

### Una nota sobre I/O de procesos

`process.stdout` y `process.stderr` difieren de otros streams de Node.js d maneras importantes:

1. Son usados internamente por [`console.log()`][] y [`console.error()`][], respectivamente.
2. Las escrituras pueden ser sincrónicas, dependiendo a qué esté conectado el stream y si el sistema es Windows o POSIX: 
  * Archivos: *sincrónicos* en Windows y POSIX
  * TTYs (Terminales): *asíncronos* en Windows, *sincrónicos* en POSIX
  * Pipes (y sockets): *sincrónicos* en Windows, *asíncronos* en POSIX

Estos comportamientos se deben parcialmente a razones históricas, y cambiarlos crearía una incompatibilidad hacia atrás, pero también son esperados por algunos usuarios.

Las escrituras sincrónicas evitan problemas tales como el output escrito con `console.log()` o `console.error()` sea intercalado inesperadamente, o no se escriba en absoluto si `process.exit()` es llamado antes de que se complete la escritura asíncrona. Vea [`process.exit()`][] para mayor información.

***Advertencia***: Las escrituras sincrónicas bloquean el bucle del evento hasta que se complete la escritura. Esto puede ser casi instantáneo en el caso del output a un archivo, pero con una carga alta del sistema, los pipes que no están siendo leídos en el extremo receptor, o con terminales o sistemas de archivos lentos, es posible que el bucle de eventos sea bloqueado con la suficiente frecuencia o el tiempo suficiente para tener graves impactos negativos en el rendimiento. Esto puede no ser un problema al escribirle a una sesión terminal interactiva, pero considérelo de una forma particularmente cuidadosa al hacer el registro de producción para los streams de salida del proceso.

Para verificar si un stream está conectado a un contexto [TTY](tty.html#tty_tty), verifique la propiedad `isTTY`.

Por ejemplo:

```console
$ node -p "Boolean(process.stdin.isTTY)"
true
$ echo "foo" | node -p "Boolean(process.stdin.isTTY)"
false
$ node -p "Boolean(process.stdout.isTTY)"
true
$ node -p "Boolean(process.stdout.isTTY)" | cat
false
```

Vea la documentación de [TTY](tty.html#tty_tty) para mayor información.

## process.throwDeprecation

<!-- YAML
added: v0.9.12
-->

* {boolean}

La propiedad `process.throwDeprecation` indica si la bandera`--throw-deprecation` está establecida en el proceso Node.js actual. Vea la documentación para el [evento `'warning'`](#process_event_warning) y el [método `emitWarning()`](#process_process_emitwarning_warning_type_code_ctor) para más información acerca de este comportamiento de la bandera.

## process.title

<!-- YAML
added: v0.1.104
-->

* {string}

La propiedad `process.title` devuelve el título del proceso actual (es decir, devuelve el valor actual de `ps`). Asignar un nuevo valor para `process.title` modifica el valor actual de `ps`.

Cuando se asigna un nuevo valor, las diferentes plataformas impondrán restricciones de longitud máxima diferentes en el título. Usualmente, dichas restricciones son muy limitadas. Por ejemplo, en Linux y macOS, `process.title` es limitado al tamaño del nombre binario más la longitud de los argumentos de línea de comando porque el configurar el `process.title` sobrescribe la memoria `argv` del proceso. Node.js v0.8 allowed for longer process title strings by also overwriting the `environ` memory but that was potentially insecure and confusing in some (rather obscure) cases.

## process.traceDeprecation

<!-- YAML
added: v0.8.0
-->

* {boolean}

La propiedad `process.traceDeprecation` indica si la bandera `--trace-deprecation` está establecida en el proceso Node.js actual. Vea la documentación para el [evento `'warning'`](#process_event_warning) y el [método `emitWarning()`](#process_process_emitwarning_warning_type_code_ctor) para más información acerca de este comportamiento de la bandera.

## process.umask([mask])

<!-- YAML
added: v0.1.19
-->

* `mask` {number}

El método `process.umask()` establece o devuelve una máscara de creación de modo de archivo del proceso Node.js. Los procesos secundarios heredan la máscara del proceso primario. Invocado sin un argumento, se devuelve la máscara actual; de lo contrario, la umask se establece en el valor del argumento y se devuelve la máscara anterior.

```js
const newmask = 0o022;
const oldmask = process.umask(newmask);
console.log(
  `Changed umask from ${oldmask.toString(8)} to ${newmask.toString(8)}`
);
```

This feature is not available in [`Worker`][] threads.

## process.uptime()

<!-- YAML
added: v0.5.0
-->

* Devuelve: {number}

El método `process.uptime()` devuelve el número de segundos que el proceso Node.js actual ha estado ejecutando.

El valor de retorno incluye fracciones de un segundo. Utilice `Math.floor()` para obtener segundos completos.

## process.version

<!-- YAML
added: v0.1.3
-->

* {string}

La propiedad `process.version` devuelve la string de la versión Node.js.

```js
console.log(`Version: ${process.version}`);
```

## process.versions

<!-- YAML
added: v0.2.0
changes:

  - version: v4.2.0
    pr-url: https://github.com/nodejs/node/pull/3102
    description: The `icu` property is now supported.
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/15785
    description: The `v8` property now includes a Node.js specific suffix.
-->

* {Object}

La propiedad `process.versions` devuelve un objeto que enumera la strings de versión de Node.js y sus dependencias. `process.versions.modules` indica la versión ABI actual, la cual es incrementada cada vez que un API de C++ cambia. Node.js rechazará cargar módulos que fueron compilados en contra de una versión ABI de módulo diferente.

```js
console.log(process.versions);
```

Generará un objeto similar a:

<!-- eslint-skip -->

```js
{ http_parser: '2.7.0',
  node: '8.9.0',
  v8: '6.3.292.48-node.6',
  uv: '1.18.0',
  zlib: '1.2.11',
  ares: '1.13.0',
  modules: '60',
  nghttp2: '1.29.0',
  napi: '2',
  openssl: '1.0.2n',
  icu: '60.1',
  unicode: '10.0',
  cldr: '32.0',
  tz: '2016b' }
```

## Código de Salida

Node.js normalmente se cerrará con un código de estado `0` cuando no hayan más operaciones asincrónicas pendientes. Los siguientes códigos de estado son usados en otros casos:

* `1` **Excepción Fatal No Capturada** - Hubo una excepción no capturada y no fue manejada por un dominio o por un manejador de evento [`'uncaughtException'`][].
* `2` - Sin utilizar (reservado por Bash para mal uso incorporado)
* `3` **Error Interno de Análisis de JavaScript** - El código fuente interno de JavaScript en el proceso bootstrapping de Node.js causó un error de análisis. Esto es extremadamente raro y generalmente sólo puede ocurrir durante el desarrollo del mismo Node.js.
* `4` **Fallo Interno de Evaluación de JavaScript** - El código fuente interno de JavaScript en el proceso bootstrapping de Node.js falló al devolver un valor de una función el evaluarse. Esto es extremadamente raro y generalmente sólo puede ocurrir durante el desarrollo del mismo Node.js.
* `5` **Error Fatal** - Hubo un error fatal irrecuperable en V8. Usualmente, se imprimirá un mensaje en stderr con el prefijo `FATAL
ERROR`.
* `6` **Manejador de Excepción Interno Sin Función** - Hubo una excepción sin capturar, pero la función manejadora de excepción interna fatal de alguna manera fue establecida a una no-función, y no pudo ser llamada.
* `7` **Fallo de Tiempo de Ejecución del Manejador Interno de Excepción ** - Hubo una excepción sin capturar, y la misma función manejadora interna de excepción fatal arrojó un error al intentar manejarlo. Esto puede ocurrir, por ejemplo, si un manejador [`'uncaughtException'`][] o `domain.on('error')` arroja un error.
* `8` - No utilizado. En versiones anteriores de Node.js, el código de salida 8 en ocasiones indicaba una excepción no capturada.
* `9` - **Argumento Inválido** - Una opción desconocida fue especificada o una opción que requiere un valor fue proporcionada sin un valor.
* `10` **Fallo Interno de Tiempo de Ejecución de JavaScript** - El código fuente interno de JavaScript en el proceso bootstrapping de Node.js arrojó un error cuando se llamó a la función bootstrapping. Esto es extremadamente raro, y generalmente sólo puede ocurrir durante el desarrollo del mismo Node.js.
* `12` **Argumento de Depuración Inválido** - Las opciones `--inspect` y/o `--inspect-brk` fueron establecidas, pero el número de puerto escogido era inválido o no estaba disponible.
* `>128` **Salidas de Señales** - Si Node.js recibe una señal fatal como `SIGKILL` or `SIGHUP`, entonces su código de salida será `128` más el valor del código de señal. Esta es una práctica estándar de POSIX, como los códigos de salida están definidos para ser enteros de 7 bits, y las salidas por señal establecen el bit de orden mayor, y entonces contienen el valor del código de señal. Por ejemplo, la señal `SIGABRT` tiene el valor `6`, entonces el código de salida esperado sería `128` + `6`, o `134`.