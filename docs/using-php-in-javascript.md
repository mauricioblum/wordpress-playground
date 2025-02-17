# Compiling PHP to WebAssembly and using it in JavaScript

The [`src/php-wasm`](https://github.com/WordPress/wordpress-playground/blob/trunk/src/php-wasm/) module brings PHP into JavaScript as a WebAssembly module:

```js
import { createPHP } from 'php-wasm';

const PHPLoaderModule = await import('/php.js');
const php = await createPHP(PHPLoaderModule);
console.log(php.run(`<?php echo "Hello from PHP!";`).stdout);
// Output: "Hello from PHP!"
```

It consists of two major building blocks:

-   [PHP to WebAssembly build pipeline](#php-to-webassembly-build-pipeline)
-   [JavaScript bindings for the WebAssembly PHP](#javascript-bindings-for-the-webassembly-php)

See also the [API Reference](api/php-wasm.md).

## PHP to WebAssembly build pipeline

The pipeline lives in [`wasm/Dockerfile`](https://github.com/WordPress/wordpress-playground/blob/trunk/src/php-wasm/wasm/Dockerfile). It was originally forked from [seanmorris/php-wasm](https://github.com/seanmorris/php-wasm)

In broad strokes, that `Dockerfile`:

-   Installs all the necessary linux packages (like `build-essential`)
-   Downloads PHP and the required libraries, e.g. `sqlite3`.
-   Applies a few patches.
-   Compiles everything using [Emscripten](https://emscripten.org/), a drop-in replacement for the C compiler.
-   Compiles `php_wasm.c` – a convenient API for JavaScript.
-   Outputs a `php.wasm` file and one or more JavaScript loaders, depending on the configuration.
-   Transforms the Emscripten's default `php.js` output into an ESM module with additional features.

To find out more about each step, refer directly to the [Dockerfile](https://github.com/WordPress/wordpress-playground/blob/trunk/src/php-wasm/wasm/Dockerfile).

### Building

To build all PHP versions, run `npm run build:php:web` in the repository root. You'll find the output files in `src/php-wasm/build-wasm`. To build a specific
version, run `npm run build:php:web:<VERSION>`, e.g. `npm run build:php:web:8.2`.

### PHP extensions

PHP is built with several extensions listed in the [`Dockerfile`](https://github.com/WordPress/wordpress-playground/blob/trunk/src/php-wasm/wasm/Dockerfile).

Some extensions, like `zip`, can be turned on or off during the build. Others, like `sqlite3`, are hardcoded.

If you need to turn off one of the hardcoded extensions, feel free to open an issue in this repo. Better yet, this project needs contributors. You are more than welcome to open a PR and author the change you need.

### C API exposed to JavaScript

The C API exposed to JavaScript lives in the [`wasm/build-assets/php_wasm.c`](https://github.com/WordPress/wordpress-playground/blob/trunk/src/php-wasm/wasm/build-assets/php_wasm.c) file. The most important functions are:

-   `void phpwasm_init()` – It creates a new PHP context and must be called before running any PHP code.
-   `int phpwasm_run(char *code)` – Runs a PHP script and writes the output to /tmp/stdout and /tmp/stderr. Returns the exit code.
-   `void phpwasm_refresh()` – Destroy the current PHP context and starts a new one. Call it after running one PHP script and before running another.

Refer to the inline documentation in [`php_wasm.c`](https://github.com/WordPress/wordpress-playground/blob/trunk/src/php-wasm/wasm/build-assets/php_wasm.c) to learn more.

### Build configuration

The build is configurable via the [Docker `--build-arg` feature](https://docs.docker.com/engine/reference/commandline/build/#set-build-time-variables---build-arg). Currently it's not possible to specify these options via `npm run build:php:web` so you'll need to run `docker build` manually. For example, to change the PHP version, specify the `PHP_VERSION` option during the build:

```sh
docker build . --build-arg PHP_VERSION=7.4.0
```

**Supported build options:**

-   `PHP_VERSION` – The PHP version to build, default: `8.0.24`. This value must point to an existing branch of the <https://github.com/php/php-src.git> repository when prefixed with `PHP-`. For example, `7.4.0` is valid because the branch `PHP-7.4.0` exists, but just `7` is invalid because there's no branch `PHP-7`. The PHP versions that are known to work are `7.4.*` and `8.0.*`. Others likely work as well but they haven't been tried.
-   `EMSCRIPTEN_ENVIRONMENT` – `web` or `node`, default: `web`. The platform to build for. When building for `web`, two JavaScript loaders will be created: `php-web.js` and `php-webworker.js`. When building for Node.js, only one loader called `php-node.js` will be created.
-   `WITH_LIBXML` – `yes` or `no`, default: `no`. Whether to build with `libxml2` and the `dom`, `xml`, and `simplexml` PHP extensions (`DOMDocument`, `SimpleXML`, ..).
-   `WITH_LIBZIP` – `yes` or `no`, default: `yes`. Whether to build with `zlib`, `libzip`, and the `zip` PHP extension (`ZipArchive`).
-   `WITH_VRZNO` – `yes` or `no`, default: `yes` when PHP_VERSION is 7.\*. Whether to build with [the `vrzno` PHP extension](https://github.com/seanmorris/vrzno/fork) that enables running JavaScript code from PHP.
-   `WITH_NODEFS` – `yes` or `no`, default: `no`. Whether to include [the Emscripten's NODEFS JavaScript library](https://emscripten.org/docs/api_reference/Filesystem-API.html#filesystem-api-nodefs). It's useful for loading files and mounting directories from the local filesystem when running php.wasm from Node.js.

### JavaScript module

The `php.js` file generated by this build process is **not** a vanilla Emscripten module. Instead, it's an ESM module that wraps the regular Emscripten output and adds some extra functionality.

Here's the API it exposes:

```js
// php.wasm size in bytes:
export const dependenciesTotalSize = 5644199;

// php.wasm filename:
export const dependencyFilename = 'php.wasm';

// Run Emscripten's generated module:
export default function (jsEnv, emscriptenModuleArgs) {}
```

## JavaScript bindings for the WebAssembly PHP

`php-wasm` provides a JavaScript API to interact with the WebAssembly
module. It lives in the `src` directory and consists of:

-   A `PHP` class to directly interface with the WebAssembly module.
-   A `PHPServer` class to use PHP for handling HTTP requests.
-   A `PHPBrowser` class to handle cookies and redirects emitted by `PHPServer`

### Building

To build the JavaScript API, run `npm run build:js` in the repository root. 

## API

Below you'll find a few especially relevant parts of the API. Consult the [php-wasm API reference page](api/php-wasm.md) to learn about the rest of it.

### startPHP

<!-- include /docs/api/php-wasm.startphp.md#startPHP() function -->

startPHP<!-- -->(\
&emsp;&emsp;&emsp;<!-- -->phpLoaderModule<!-- -->: [any](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#any)<!-- -->, \
&emsp;&emsp;&emsp;<!-- -->runtime<!-- -->: [JavascriptRuntime](api/php-wasm.startphp.md)<!-- -->, \
&emsp;&emsp;&emsp;<!-- -->phpModuleArgs?<!-- -->: [any](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#any)<!-- -->, \
&emsp;&emsp;&emsp;<!-- -->dataDependenciesModules?<!-- -->: [any](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#any)<!-- -->[]\
)<!-- -->: [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)<!-- -->&lt;[PHP](api/php-wasm.startphp.md)<!-- -->&gt;

* `phpLoaderModule` – The ESM-wrapped Emscripten module. Consult the Dockerfile for the build process.
* `runtime` – The current JavaScript environment. One of: NODE, WEB, or WEBWORKER.
* `phpModuleArgs` – Optional. The Emscripten module arguments, see https://emscripten.org/docs/api_reference/module.html#affecting-execution.
* `dataDependenciesModules` – Optional. A list of the ESM-wrapped Emscripten data dependency modules.
* Returns: PHP instance.


Initializes the PHP runtime with the given arguments and data dependencies.

This function handles the entire PHP initialization pipeline. In particular, it:

* Instantiates the Emscripten PHP module
* Wires it together with the data dependencies and loads them
* Ensures is all happens in a correct order
* Waits until the entire loading sequence is finished

Basic usage:

```js
 const phpLoaderModule = await import("/php.js");
 const php = await startPHP(phpLoaderModule, "web");
 console.log(php.run(`<?php echo "Hello, world!"; `));
 // { stdout: ArrayBuffer containing the string "Hello, world!", stderr: [''], exitCode: 0 }
```
**The `/php.js` module:**

In the basic usage example, `php.js` is **not** a vanilla Emscripten module. Instead,
it's an ESM module that wraps the regular Emscripten output and adds some
extra functionality. It's generated by the Dockerfile shipped with this repo.
Here's the API it provides:

```js
// php.wasm size in bytes:
export const dependenciesTotalSize = 5644199;

// php.wasm filename:
export const dependencyFilename = 'php.wasm';

// Run Emscripten's generated module:
export default function(jsEnv, emscriptenModuleArgs) {}
```
**PHP Filesystem:**

Once initialized, the PHP has its own filesystem separate from the project
files. It's provided by [Emscripten and uses its FS library](https://emscripten.org/docs/api_reference/Filesystem-API.html).

The API exposed to you via the PHP class is succinct and abstracts
await certain unintuitive parts of low-level FS interactions.

Here's how to use it:

```js
// Recursively create a /var/www directory
php.mkdirTree('/var/www');

console.log(php.fileExists('/var/www/file.txt'));
// false

php.writeFile('/var/www/file.txt', 'Hello from the filesystem!');

console.log(php.fileExists('/var/www/file.txt'));
// true

console.log(php.readFile('/var/www/file.txt'));
// "Hello from the filesystem!

// Delete the file:
php.unlink('/var/www/file.txt');
```
For more details consult the PHP class directly.

**Data dependencies:**

Using existing PHP packages by manually recreating them file-by-file would
be quite inconvenient. Fortunately, Emscripten provides a "data dependencies"
feature.

Data dependencies consist of a `dependency.data` file and a `dependency.js` loader and
can be packaged with the [file_packager.py tool](api/php-wasm.startphp.md).
This project requires wrapping the Emscripten-generated `dependency.js` file in an ES
module as follows:

1. Prepend `export default function(emscriptenPHPModule) {'; `
2. Prepend `export const dependencyFilename = '<DATA FILE NAME>'; `
3. Prepend `export const dependenciesTotalSize = <DATA FILE SIZE>;`
4. Append `}`

Be sure to use the `--export-name="emscriptenPHPModule"` file_packager.py option.

You want the final output to look as follows:

```js
export const dependenciesTotalSize = 5644199;
export const dependencyFilename = 'dependency.data';
export default function(emscriptenPHPModule) {
   // Emscripten-generated code:
   var Module = typeof emscriptenPHPModule !== 'undefined' ? emscriptenPHPModule : {};
   // ... the rest of it ...
}
```
Such a constructions enables loading the `dependency.js` as an ES Module using
`import("/dependency.js")`<!-- -->.

Once it's ready, you can load PHP and your data dependencies as follows:

```js
 const [phpLoaderModule, wordPressLoaderModule] = await Promise.all([
   import("/php.js"),
   import("/wp.js")
 ]);
 const php = await startPHP(phpLoaderModule, "web", {}, [wordPressLoaderModule]);
```

<!-- /include /docs/api/php-wasm.startphp.md#startPHP() function -->

### php-server.js

<!-- include /docs/api/php-wasm.phpserver.md#PHPServer class -->

<b>Signature:</b>

```typescript
class PHPServer 
```
A fake PHP server that handles HTTP requests but does not
bind to any port.

## Constructors

### PHPServer<!-- -->(<!-- -->php<!-- -->: [PHP](api/php-wasm.phpserver.md)<!-- -->, config<!-- -->: [PHPServerConfigation](api/php-wasm.phpserver.md)<!-- -->)

* `php` – The PHP instance.
* `config` – Server configuration.

Constructs a new instance of the `PHPServer` class

## Properties

* `absoluteUrl`   readonly [string](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#the-primitives-string-number-and-boolean) – The absolute URL of this PHPServer instance.
* `php`    [PHP](api/php-wasm.phpserver.md) – The PHP instance

## Methods

### request<!-- -->(<!-- -->request<!-- -->: [PHPServerRequest](api/php-wasm.phpserver.md)<!-- -->)<!-- -->: [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)<!-- -->&lt;[PHPResponse](api/php-wasm.phpserver.md)<!-- -->&gt;

* `request` – The request.
* Returns: The response.


Serves the request – either by serving a static file, or by
dispatching it to the PHP runtime.



## Example

```js
import { createPHP, PHPServer } from 'php-wasm';

const PHPLoaderModule = await import('/php.js');
const php = await createPHP(PHPLoaderModule);

// Create a file to serve:
php.mkdirTree('/www');
php.writeFile('/www/index.php', '<?php echo "Hi from PHP!"; ');

// Create a server instance:
const server = new PHPServer(php, {
    // PHP FS path to serve the files from:
    documentRoot: '/www',

    // Used to populate $_SERVER['SERVER_NAME'] etc.:
    absoluteUrl: 'http://127.0.0.1'
});

console.log(
   server.request({ path: '/index.php' }).body
);
// Output: "Hi from PHP!"
```

<!-- /include /docs/api/php-wasm.phpserver.md#PHPServer class -->
