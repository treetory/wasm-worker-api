# :rocket: WASM Worker API [WIP]

A tiny, promise-based two-file helper that lets you call compiled with [Emscripten](https://kripken.github.io/emscripten-site/docs/introducing_emscripten/about_emscripten.html) C/C++ functions from a worker.

### :warning: Warning: *highly* experimental! :warning:

## What?

When working with WebAssembly and Emscripten, sometimes you have to import the Emscripten's 'glue' code that contains some information that is needed for the compiled code to run. This is a *pretty big* file (the one I use for testing is around 64Kb, compiled with [-Oz](https://kripken.github.io/emscripten-site/docs/tools_reference/emcc.html#emcc-oz) option). If your C/C++ code performs only pure computational work without accessing the DOM there is no reason for the glue code to be run on the main thread.

The `wasm-worker-api` offers a way to communicate with compiled C/C++ code running on a separate thread (worker) from the main thread, with some additional conveniences (e.g. allocating/freeing the memory when using arrays).

## Install

The easiest way to get started is to use a CDN.

Include this in your `index.html`...
```html
  <script src="https://unpkg.com/wasm-worker-api@0.0.4/umd/client.min.js"></script>
```
and this in your worker (note that the wasm-worker should be included **after** wasm's 'glue' code, otherwise it will throw an error):
```js
  // worker
  importScripts(
    'wasm-glue-code.js',
    'https://unpkg.com/wasm-worker-api@0.0.4/umd/worker.min.js'
  )
```
Alternatively, you can install it via `yarn` or `npm`:
```
yarn add wasm-worker-api -S
```
Then import the client part in your js file:
```js
import wasmWorkerAPI from 'wasm-worker-api'
```

## Example

```C++
  // add.cc
  #include <emscripten.h>

  extern "C" {
    EMSCRIPTEN_KEEPALIVE
    int add (int a, int b) {
      return a + b;
    }
  }
```
Compile the above C++ file with Emscripten, like this `emcc add.cc -o add.js -s WASM=1`. The compilation results in the two files: `add.js` (the 'glue' code) and `add.wasm`.
```js
  // worker.js
  importScripts(
    'add.js', // the aforementioned 'glue'
    'https://unpkg.com/wasm-worker-api@0.0.4/umd/worker.min.js'
  )
```

```html
<!-- index.html -->
<script src="https://unpkg.com/wasm-worker-api@0.0.4/umd/client.min.js"></script>
<script>
  const worker = new Worker('worker.js')

  // an array of objects that describe the exported C++ functions
  const functions = [
    {
      name: 'add',
      ret: 'number',
      args: ['number', 'number']
    }
  ]

  async function init () {
    // register the functions
    const api = await wasmWorkerAPI(worker, functions)

    // now ready to call the functions
    console.log('5 + 2 = %d', await api.add(5, 2))
  }

  init()
</script>
```

## API

**`wasmWorkerAPI (worker: Worker, functions: Array<object>) => Promise<api: object>`**

**Parameters**:
* **`worker`** - an instance of the [Worker](https://developer.mozilla.org/en-US/docs/Web/API/Worker).
* **`functions`** - an array of objects that describe exported C++ functions. The objects have the following shape:
  ```ts
  {
    name: string,
    args?: Array<string>,
    ret?: string
  }
  ```
  * *`name`* - the name of a C++ function.
  * *`args`* - an array of strings that represent types of the function's arguments. Possible values are: `string`, `number`, `8`, `U8`, `16`, `U16`, `32`, `U32`, `F32`, `F64`.
  * *`ret`* - the function's return value. Possible values are the same as with *`args`*.

  The types `8`, `U8`, `16`, `U16`, `32`, `U32`, `F32`, `F64` represent [typed arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray).

  You can omit the `args` and `ret` properties if you define functions in the `EMSCRIPTEN_BINDINGS` block and the functions don't accept or return arrays.

**Returns**: a promise that resolves to an object containing compiled C/C++ functions. Every function in the object returns a promise that resovles to it's result.

## Caveats

 * Classes aren't supported (yet).

 * Currently, the `Module` object in a worker should contain the following methods:
    * `addOnPostRun` - in all cases.
    * `cwrap` - in all cases (except when you define your functions in the `EMSCRIPTEN_BINDINGS` block).
    * `_malloc` and `_free` - if arrays are either passed to or returned from a function.

  * If a C++ function returns an array (pointer), when you call it from JS you have to pass the array's length as the very last parameter because otherwise there is no way to know how many elements it contains. You **don't** need to mention this parameter in the `args` array when you call the `wasmWorkerAPI` function.

>Note: probably, you should avoid returning arrays from wasm functions at all because it requires some extra work on the JS side, which can easily become a performance bottleneck.