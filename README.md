# ECMAScript Async Explicit Resource Management

**ECMAScript Async Explicit Resource Management has been subsumed by the [ECMAScript Explicit Resource Management Proposal](https://github.com/tc39/proposal-explicit-resource-management). Further discussion should occur in that repository.**

This proposal relates to async functionality deferred from the original
[Explicit Resource Management][using] proposal and shares the
same motivations:

> This proposal intends to address a common pattern in software development regarding
> the lifetime and management of various resources (memory, I/O, etc.). This pattern
> generally includes the allocation of a resource and the ability to explicitly
> release critical resources.

> Please note: Specification text for this proposal is yet to be updated since this split occurred.

For example, ECMAScript Async Generator Functions expose this pattern through the `return` method, as a means to
explicitly evaluate `finally` blocks to ensure user-defined cleanup logic is preserved:

```js
async function * g() {
  const stream = acquireStream(); // critical resource
  try {
    ...
  }
  finally {
    await stream.close(); // cleanup
  }
}

const obj = g();
try {
  const r = await obj.next();
  ...
}
finally {
  await obj.return(); // calls finally blocks in `g`
}
```

As such, we propose the adoption of a novel syntax to simplify this common pattern:

```js
// in an async function:
async function * g() {
  await using handle = acquireFileHandle(); // async-block-scoped critical resource
} // async cleanup

// in a block in an async context:
{
  await using obj = g(); // block-scoped declaration
  const r = await obj.next();
} // calls finally blocks in `g` and awaits result
```

In addition, we propose the addition of a disposable container object to assist
with managing multiple resources:

- `AsyncDisposableStack` &mdash; A stack-based container of asynchronously disposable resources.

## Status

**Stage:** 3  \
**Champion:** Ron Buckton (@rbuckton)  \
**Last Presented:** March, 2023 ([slides](https://1drv.ms/p/s!AjgWTO11Fk-Tkodu1RydtKh2ZVafxA?e=yasS3Y),
[notes #1](https://github.com/tc39/notes/blob/main/meetings/2023-03/mar-21.md#async-explicit-resource-management),
[notes #2](https://github.com/tc39/notes/blob/main/meetings/2023-03/mar-23.md#async-explicit-resource-management-again))

_For more information see the [TC39 proposal process](https://tc39.es/process-document/)._

## Authors

- Ron Buckton (@rbuckton)

# Motivations

This proposal is motivated by a number of cases:

- Inconsistent patterns for resource management:
  - ECMAScript Iterators: `iterator.return()`
  - WHATWG Stream Readers: `reader.releaseLock()`
  - NodeJS FileHandles: `handle.close()`
  - Emscripten C++ objects handles: `Module._free(ptr) obj.delete() Module.destroy(obj)`
- Avoiding common footguns when managing resources:
  ```js
  const writer = getWritableStream();
  ...
  await writer.close(); // Oops, should have been in a try/finally
  ```
- Scoping resources:
  ```js
  const writer = ...;
  try {
    ... // ok to use `writer`
  }
  finally {
    await writer.close();
  }
  // not ok to use `writer`, but still in scope
  ```
- Ensuring cleanup is properly `await`-ed:
  ```js
  const writer = ...;
  try {
    ...
  }
  finally {
    writer.close(); // oops, forgot `await`
  }
  ```
- Avoiding common footguns when managing multiple resources:
  ```js
  const a = ...;
  const b = ...;
  try {
    ...
  }
  finally {
    await a.close(); // Oops, issue if `b.close()` depends on `a`.
    await b.close(); // Oops, `b` never reached if `a.close()` throws.
  }
  ```
- Avoiding lengthy code when managing multiple resources correctly:
  ```js
  { // block avoids leaking `a` or `b` to outer scope
    const a = ...;
    try {
      const b = ...;
      try {
        ...
      }
      finally {
        await b.close(); // ensure `b` is closed before `a` in case `b`
                         // depends on `a`
      }
    }
    finally {
      await a.close(); // ensure `a` is closed even if `b.close()` throws
    }
  }
  // both `a` and `b` are out of scope
  ```
  Compared to:
  ```js
  // avoids leaking `a` or `b` to outer scope
  // ensures `b` is disposed before `a` in case `b` depends on `a`
  // ensures `a` is disposed even if disposing `b` throws
  await using a = ..., b = ...;
  ...
  ```
- Non-blocking memory/IO applications:
  ```js
  import { AsyncReaderWriterLock } from "...";
  const lock = new AsyncReaderWriterLock();

  export async function readData() {
    // wait for outstanding writer and take a read lock
    await using lockHandle = await lock.read();
    ... // any number of readers
    await ...;
    ... // still in read lock after `await`
  } // release the read lock, awaiting the result

  export async function writeData(data) {
    // wait for all readers and take a write lock
    await using lockHandle = await lock.write();
    ... // only one writer
    await ...;
    ... // still in write lock after `await`
  } // release the write lock, awaiting the result
  ```

# Prior Art

<!-- Links to similar concepts in existing languages, prior proposals, etc. -->

- ECMAScript:
  - [Explicit Resource Management][using]
- C#:
  - [`using` statement](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/using-statement)
  - [`using` declaration](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-8.0/using#using-declaration)
- Java: [`try`-with-resources statement](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)
- Python: [`with` statement](https://docs.python.org/3/reference/compound_stmts.html#the-with-statement)

# Definitions

- _Resource_ &mdash; An object with a specific lifetime, at the end of which either a lifetime-sensitive operation
  should be performed or a non-gargbage-collected reference (such as a file handle, socket, etc.) should be closed or
  freed.
- _Resource Management_ &mdash; A process whereby "resources" are released, triggering any lifetime-sensitive operations
  or freeing any related non-garbage-collected references.
- _Implicit Resource Management_ &mdash; Indicates a system whereby the lifetime of a "resource" is managed implicitly
  by the runtime as part of garbage collection, such as:
  - `WeakMap` keys
  - `WeakSet` values
  - `WeakRef` values
  - `FinalizationRegistry` entries
- _Explicit Resource Management_ &mdash; Indicates a system whereby the lifetime of a "resource" is managed explicitly
  by the user either **imperatively** (by directly calling a method like `Symbol.dispose` or `Symbol.asyncDispose`) or **declaratively** (through a block-scoped declaration like `using` or `await using`).

# Syntax

## `await using` Declarations

```js
// an asynchronously-disposed, block-scoped resource
await using x = expr1;            // resource w/ local binding
await using y = expr2, z = expr4; // multiple resources
```

An `await using` declaration can appear in the following contexts:
- The top level of a _Module_ anywhere _VariableStatement_ is allowed, as long as it is not immediately nested inside
  of a _CaseClause_ or _DefaultClause_.
- In the body of an async function or async generator anywhere a _VariableStatement_ is allowed, as long as it is not
  immediately nested inside of a _CaseClause_ or _DefaultClause_.
- In the head of a `for-of` or `for-await-of` statement.

## `await using` in `for-of` and `for-await-of` Statements

```js
for (await using x of y) ...

for await (await using x of y) ...
```

You can use a `await using` declaration in a `for-of` or `for-await-of` statement inside of an async context to
explicitly bind each iterated value as an async disposable resource. `for-await-of` does not implicitly make a non-async
`using` declaration into an async `await using` declaration, as the `await` markers in  `for-await-of` and `await using`
are explicit indicators for distinct cases: `for await` *only* indicates async iteration, while `await using` *only*
indicates async disposal. For example:

```js

// sync iteration, sync disposal
for (using x of y) ; // no implicit `await` at end of each iteration

// sync iteration, async disposal
for (await using x of y) ; // implicit `await` at end of each iteration

// async iteration, sync disposal
for await (using x of y) ; // implicit `await` at end of each iteration

// async iteration, async disposal
for await (await using x of y) ; // implicit `await` at end of each iteration
```

While there is some overlap in that the last three cases introduce some form of implicit `await` during execution, it
is intended that the presence or absence of the `await` modifier in a `using` declaration is an explicit indicator as to
whether we are expecting the iterated value to have an `@@asyncDispose` method. This distinction is in line with the
behavior of `for-of` and `for-await-of`:

```js
const iter = { [Symbol.iterator]() { return [].values(); } };
const asyncIter = { [Symbol.asyncIterator]() { return [].values(); } };

for (const x of iter) ; // ok: `iter` has @@iterator
for (const x of asyncIter) ; // throws: `asyncIter` does not have @@iterator

for await (const x of iter) ; // ok: `iter` has @@iterator (fallback)
for await (const x of asyncIter) ; // ok: `asyncIter` has @@asyncIterator

```

`using` and `await using` have the same distinction:

```js
const res = { [Symbol.dispose]() {} };
const asyncRes = { [Symbol.asyncDispose]() {} };

using x = res; // ok: `res` has @@dispose
using x = asyncRes; // throws: `asyncRes` does not have @@dispose

await using x = res; // ok: `res` has @@dispose (fallback)
await using x = asyncres; // ok: `asyncRes` has @@asyncDispose
```

This results in a matrix of behaviors based on the presence of each `await` marker:

```js
const res = { [Symbol.dispose]() {} };
const asyncRes = { [Symbol.asyncDispose]() {} };
const iter = { [Symbol.iterator]() { return [res, asyncRes].values(); } };
const asyncIter = { [Symbol.asyncIterator]() { return [res, asyncRes].values(); } };

for (using x of iter) ;
// sync iteration, sync disposal
// - `iter` has @@iterator: ok
// - `res` has @@dispose: ok
// - `asyncRes` does not have @@dispose: *error*

for (using x of asyncIter) ;
// sync iteration, sync disposal
// - `asyncIter` does not have @@iterator: *error*

for (await using x of iter) ;
// sync iteration, async disposal
// - `iter` has @@iterator: ok
// - `res` has @@dispose (fallback): ok
// - `asyncRes` has @@asyncDispose: ok

for (await using x of asyncIter) ;
// sync iteration, async disposal
// - `asyncIter` does not have @@iterator: error

for await (using x of iter) ;
// async iteration, sync disposal
// - `iter` has @@iterator (fallback): ok
// - `res` has @@dispose: ok
// - `asyncRes` does not have @@dispose: error

for await (using x of asyncIter) ;
// async iteration, sync disposal
// - `asyncIter` has @@asyncIterator: ok
// - `res` has @@dispose: ok
// - `asyncRes` does not have @@dispose: error

for await (await using x of iter) ;
// async iteration, async disposal
// - `iter` has @@iterator (fallback): ok
// - `res` has @@dispose (fallback): ok
// - `asyncRes` does has @@asyncDispose: ok

for await (await using x of asyncIter) ;
// async iteration, async disposal
// - `asyncIter` has @@asyncIterator: ok
// - `res` has @@dispose (fallback): ok
// - `asyncRes` does has @@asyncDispose: ok
```

Or, in table form:

| Syntax                           | Iteration                      | Disposal                     |
|:---------------------------------|:------------------------------:|:----------------------------:|
| `for (using x of y)`             | `@@iterator`                   | `@@dispose`                  |
| `for (await using x of y)`       | `@@iterator`                   | `@@asyncDispose`/`@@dispose` |
| `for await (using x of y)`       | `@@asyncIterator`/`@@iterator` | `@@dispose`                  |
| `for await (await using x of y)` | `@@asyncIterator`/`@@iterator` | `@@asyncDispose`/`@@dispose` |

# Grammar

Please refer to the [specification text][Specification] for the most recent version of the grammar.

# Semantics

## `await using` Declarations

### `await using` Declarations with Explicit Local Bindings

```grammarkdown
UsingDeclaration :
  `using` `await` BindingList `;`

LexicalBinding :
    BindingIdentifier Initializer
```

When an `await using` declaration is parsed with _BindingIdentifier_ _Initializer_, the bindings created in the
declaration are tracked for disposal at the end of the containing async function body, _Block_, or _Module_:

```js
{
  ... // (1)
  await using x = expr1;
  ... // (2)
}
```

The above example has similar runtime semantics as the following transposed representation:

```js
{
  const $$try = { stack: [], error: undefined, hasError: false };
  try {
    ... // (1)

    const x = expr1;
    if (x !== null && x !== undefined) {
      let $$dispose = x[Symbol.asyncDispose];
      if (typeof $$dispose !== "function") {
        $$dispose = x[Symbol.dispose];
      }
      if (typeof $$dispose !== "function") {
        throw new TypeError();
      }
      $$try.stack.push({ value: x, dispose: $$dispose });
    }

    ... // (2)
  }
  catch ($$error) {
    $$try.error = $$error;
    $$try.hasError = true;
  }
  finally {
    while ($$try.stack.length) {
      const { value: $$expr, dispose: $$dispose } = $$try.stack.pop();
      try {
        await $$dispose.call($$expr);
      }
      catch ($$error) {
        $$try.error = $$try.hasError ? new SuppressedError($$error, $$try.error) : $$error;
        $$try.hasError = true;
      }
    }
    if ($$try.hasError) {
      throw $$try.error;
    }
  }
}
```

If exceptions are thrown both in the statements following the `await using` declaration and in the call to
`[Symbol.asyncDispose]()`, all exceptions are reported.

### `await using` Declarations with Multiple Resources

An `await using` declaration can mix multiple explicit bindings in the same declaration:

```js
{
  ...
  await using x = expr1, y = expr2;
  ...
}
```

These bindings are again used to perform resource disposal when the _Block_ or _Module_ exits, however in this case each
resource's `[Symbol.asyncDispose]()` is invoked in the reverse order of their declaration. This is _approximately_
equivalent to the following:

```js
{
  ... // (1)
  await using x = expr1;
  await using y = expr2;
  ... // (2)
}
```

Both of the above cases would have similar runtime semantics as the following transposed representation:

```js
{
  const $$try = { stack: [], error: undefined, hasError: false };
  try {
    ... // (1)

    const x = expr1;
    if (x !== null && x !== undefined) {
      let $$dispose = x[Symbol.asyncDispose];
      if (typeof $$dispose !== "function") {
        $$dispose = x[Symbol.dispose];
      }
      if (typeof $$dispose !== "function") {
        throw new TypeError();
      }
      $$try.stack.push({ value: x, dispose: $$dispose });
    }

    const y = expr2;
    if (y !== null && y !== undefined) {
      let $$dispose = y[Symbol.asyncDispose];
      if (typeof $$dispose !== "function") {
        $$dispose = y[Symbol.dispose];
      }
      if (typeof $$dispose !== "function") {
        throw new TypeError();
      }
      $$try.stack.push({ value: y, dispose: $$dispose });
    }

    ... // (2)
  }
  catch ($$error) {
    $$try.error = $$error;
    $$try.hasError = true;
  }
  finally {
    while ($$try.stack.length) {
      const { value: $$expr, dispose: $$dispose } = $$try.stack.pop();
      try {
        await $$dispose.call($$expr);
      }
      catch ($$error) {
        $$try.error = $$try.hasError ? new SuppressedError($$error, $$try.error) : $$error;
        $$try.hasError = true;
      }
    }
    if ($$try.hasError) {
      throw $$try.error;
    }
  }
}
```

Since we must always ensure that we properly release resources, we must ensure that any abrupt completion that might
occur during binding initialization results in evaluation of the cleanup step. When there are multiple declarations in
the list, we track each resource in the order they are declared. As a result, we must release these resources in reverse
order.

### `await using` Declarations and `null` or `undefined` Values

This proposal has opted to ignore `null` and `undefined` values provided to `await using` declarations. This is similar
to the behavior of `using` in the original [Explicit Resource Management][using] proposal, which also
allows `null` and `undefined`, as well as C#, which also allows `null`,. One primary reason for this behavior is to
simplify a common case where a resource might be optional, without requiring duplication of work or needless
allocations:

```js
if (isResourceAvailable()) {
  await using resource = getResource();
  ... // (1)
  resource.doSomething()
  ... // (2)
}
else {
  // duplicate code path above
  ... // (1) above
  ... // (2) above
}
```

Compared to:

```js
await using resource = isResourceAvailable() ? getResource() : undefined;
... // (1) do some work with or without resource
resource?.doSomething();
... // (2) do some other work with or without resource
```

### `await using` Declarations and Values Without `[Symbol.asyncDispose]` or `[Symbol.dispose]`

If a resource does not have a callable `[Symbol.asyncDispose]` or `[Symbol.asyncDispose]` member, a `TypeError` would be thrown **immediately** when the resource is tracked.

### `await using` Declarations in `for-of` and `for-await-of` Loops

A `await using` declaration _may_ occur in the _ForDeclaration_ of a `for-await-of` loop:

```js
for await (await using x of iterateResources()) {
  // use x
}
```

In this case, the value bound to `x` in each iteration will be _asynchronously_ disposed at the end of each iteration.
This will not dispose resources that are not iterated, such as if iteration is terminated early due to `return`,
`break`, or `throw`.

`await using` declarations _may not_ be used in in the head of a `for-of` or `for-in` loop.

### Implicit Async Interleaving Points ("implicit `await`")

The `await using` syntax introduces an implicit async interleaving point (i.e., an implicit `await`) whenever control
flow exits an async function body, _Block_, or _Module_ containing a `await using` declaration. This means that two
statements that currently execute in the same microtask, such as:

```js
async function f() {
  {
    a();
  } // exit block
  b(); // same microtask as call to `a()`
}
```

will instead execute in different microtasks if a `await using` declaration is introduced:

```js
async function f() {
  {
    await using x = ...;
    a();
  } // exit block, implicit `await`
  b(); // different microtask from call to `a()`.
}
```

It is important that such an implicit interleaving point be adequately indicated within the syntax. We believe that
the presence of `await using` within such a block is an adequate indicator, since it should be fairly easy to recognize
a _Block_ containing a `await using` statement in well-formated code.

It is also feasible for editors to use features such as syntax highlighting, editor decorations, and inlay hints to
further highlight such transitions, without needing to specify additional syntax.

Further discussion around the `await using` syntax and how it pertains to implicit async interleaving points can be
found in [#1](https://github.com/tc39/proposal-async-explicit-resource-management/issues/1).

# Examples

The following show examples of using this proposal with various APIs, assuming those APIs adopted this proposal.

### WHATWG Streams API
```js
{
  await using stream = new ReadableStream(...);
  ...
} // 'stream' is canceled and result is awaited
```

### NodeJS Streams
```js
{
  await using writable = ...;
  writable.write(...);
} // 'writable.end()' is called and its result is awaited
```

### Three-Phase Commit Transactions
```js
// roll back transaction if either action fails
async function transfer(account1, account2) {
  await using tx = transactionManager.startTransaction(account1, account2);
  await account1.debit(amount);
  await account2.credit(amount);

  // mark transaction success if we reach this point
  tx.succeeded = true;
} // await transaction commit or rollback
```

# API

## Additions to `Symbol`

This proposal adds the `asyncDispose` property to the `Symbol` constructor whose value is the `@@asyncDispose` internal symbol:

**Well-known Symbols**
| Specification Name | \[\[Description]] | Value and Purpose |
|:-|:-|:-|
| _@@asyncDispose_ | *"Symbol.asyncDispose"* | A method that asynchronosly explicitly disposes of resources held by the object. Called by the semantics of `await using` declarations and by `AsyncDisposableStack` objects. |

**TypeScript Definition**
```ts
interface SymbolConstructor {
  readonly asyncDispose: unique symbol;
}
```

## Built-in Disposables

### `%AsyncIteratorPrototype%.@@asyncDispose()`

We propose to add `Symbol.asyncDispose` to the built-in `%AsyncIteratorPrototype%` as if it had the following behavior:

```js
%AsyncIteratorPrototype%[Symbol.asyncDispose] = async function () {
  await this.return();
}
```

## The Common `AsyncDisposable` Interface

### The `AsyncDisposable` Interface

An object is _async disposable_ if it conforms to the following interface:

| Property | Value | Requirements |
|:-|:-|:-|
| `@@asyncDispose` | An async function that performs explicit cleanup. | The function should return a `Promise`. |

**TypeScript Definition**
```ts
interface AsyncDisposable {
  /**
   * Disposes of resources within this object.
   */
  [Symbol.asyncDispose](): Promise<void>;
}
```

## The `AsyncDisposableStack` container object

`AsyncDisposableStack` is the async version of `DisposableStack`, introduced in the
[Explicit Resource Management][using] proposal and is a
container used to aggregate async disposables, guaranteeing that every disposable resource in the container is disposed
when the respective disposal method is called. If any disposable in the container throws an error during dispose, or
results in a rejected `Promise`, it would be thrown at the end (possibly wrapped in a `SuppressedError` if multiple
errors were thrown):

```js
class AsyncDisposableStack {
  constructor();

  /**
   * Gets a value indicating whether the stack has been disposed.
   * @returns {boolean}
   */
  get disposed();

  /**
   * Alias for `[Symbol.asyncDispose]()`.
   * @returns {Promise<void>}.
   */
  disposeAsync();

  /**
   * Adds a resource to the top of the stack. Has no effect if provided `null` or `undefined`.
   * @template {AsyncDisposable | Disposable | null | undefined} T
   * @param {T} value - An `AsyncDisposable` or `Disposable` object, `null`, or `undefined`.
   * @returns {T} The provided value.
   */
  use(value);

  /**
   * Adds a non-disposable resource and a disposal callback to the top of the stack.
   * @template T
   * @param {T} value - A resource to be disposed.
   * @param {(value: T) => void | Promise<void>} onDisposeAsync - A callback invoked to dispose the provided value.
   * @returns {T} The provided value.
   */
  adopt(value, onDisposeAsync);

  /**
   * Adds a disposal callback to the top of the stack.
   * @param {() => void | Promise<void>} onDisposeAsync - A callback to evaluate when this object is disposed.
   * @returns {void}
   */
  defer(onDisposeAsync);

  /**
   * Moves all resources currently in this stack into a new `AsyncDisposableStack`.
   * @returns {AsyncDisposableStack} The new `AsyncDisposableStack`.
   */
  move();

  /**
   * Asynchronously disposes of resources within this object.
   * @returns {Promise<void>}
   */
  [Symbol.asyncDispose]();

  [Symbol.toStringTag];
}
```

This class provids the following capabilities:
- Aggregation
- Interoperation and customization
- Assist in complex construction

> **NOTE:** `AsyncDisposableStack` is inspired by Python's
[`AsyncExitStack`](https://docs.python.org/3/library/contextlib.html#contextlib.AsyncExitStack).

### Aggregation

The `AsyncDisposableStack` classe provide the ability to aggregate multiple disposable resources into a single
container. When the `AsyncDisposableStack` container is disposed, each object in the container is also guaranteed to be
disposed (barring early termination of the program). If any resource throws an error during dispose, or results in a
rejected `Promise`, that error will be collected and rethrown after all resources are disposed. If there were multiple
errors, they will be wrapped in nested `SuppressedError` objects.

For example:

```js
const stack = new AsyncDisposableStack();
const resource1 = stack.use(getResource1());
const resource2 = stack.use(getResource2());
const resource3 = stack.use(getResource3());
await stack[Symbol.asyncDispose](); // dispose and await disposal result of resource3, then resource2, then resource1
```

If all of `resource1`, `resource2` and `resource3` were to throw during disposal, this would produce an exception
similar to the following:

```js
new SuppressedError(
  /*error*/ exception_from_resource3_disposal,
  /*suppressed*/ new SuppressedError(
    /*error*/ exception_from_resource2_disposal,
    /*suppressed*/ exception_from_resource1_disposal
  )
)
```

### Interoperation and Customization

The `AsyncDisposableStack` class also provides the ability to create a disposable resource from a simple callback. This
callback will be executed when the stack's disposal method is executed.

The ability to create a disposable resource from a callback has several benefits:

- It allows developers to leverage `await using` while working with existing resources that do not conform to the
  `Symbol.asyncDispose` mechanic:
  ```js
  {
    await using stack = new AsyncDisposableStack();
    const stream = stack.adopt(createReadableStream(), async reader => await reader.close());
    ...
  }
  ```
- It grants user the ability to schedule other cleanup work to evaluate at the end of the block similar to Go's
  `defer` statement:
  ```js
  async function f() {
    await using stack = new AsyncDisposableStack();
    stack.defer(async () => await someAsyncCleanupOperaiton());
    ...
  }
  ```

### Assist in Complex Construction

A user-defined disposable class might need to allocate and track multiple nested resources that should be asynchronously
disposed when the class instance is disposed. However, properly managing the lifetime of these nested resources during
construction can sometimes be difficult. The `move` method of `AsyncDisposableStack` helps to more easily manage
lifetime in these scenarios:

```js
const privateConstructorSentinel = {};
class PluginHost {
  #disposed = false;
  #disposables;
  #channel;
  #socket;

  /** @private */
  constructor(arg) {
    if (arg !== privateConstructorSentinel) throw new TypeError("Use PluginHost.create() instead");
  }
  
  // NOTE: there's no such thing as an async constructor
  static async create() {
    const host = new PluginHost(privateConstructorSentinel);

    // Create an AsyncDisposableStack that is disposed when the constructor exits.
    // If construction succeeds, we move everything out of `stack` and into
    // `#disposables` to be disposed later.
    await using stack = new AsyncDisposableStack();


    // Create an IPC adapter around process.send/process.on("message").
    // When disposed, it unsubscribes from process.on("message").
    host.#channel = stack.use(new NodeProcessIpcChannelAdapter(process));

    // Create a pseudo-websocket that sends and receives messages over
    // a NodeJS IPC channel.
    host.#socket = stack.use(new NodePluginHostIpcSocket(host.#channel));

    // If we made it here, then there were no errors during construction and
    // we can safely move the disposables out of `stack` and into `#disposables`.
    host.#disposables = stack.move();

    // If construction failed, then `stack` would be asynchronously disposed before reaching
    // the line above. Event handlers would be removed, allowing `#channel` and
    // `#socket` to be GC'd.
    return host;
  }

  loadPlugin(file) {
    // A disposable should try to ensure access is consistent with its "disposed" state, though this isn't strictly
    // necessary since some disposables could be reusable (i.e., a Connection with an `open()` method, etc.).
    if (this.#disposed) throw new ReferenceError("Object is disposed.");
    // ...
  }

  async [Symbol.asyncDispose]() {
    if (!this.#disposed) {
      this.#disposed = true;
      const disposables = this.#disposables;

      // NOTE: we can free `#socket` and `#channel` here since they will be disposed by the call to
      // `disposables[Symbol.asyncDispose]()`, below. This isn't strictly a requirement for every disposable, but is
      // good housekeeping since these objects will no longer be useable.
      this.#socket = undefined;
      this.#channel = undefined;
      this.#disposables = undefined;

      // Dispose all resources in `disposables`
      await disposables[Symbol.asyncDispose]();
    }
  }
}
```

# Relation to DOM APIs

This proposal does not necessarily require immediate support in the HTML DOM specification, as existing APIs can still
be adapted by using `DisposableStack` or `AsyncDisposableStack`. However, there are a number of APIs that could benefit
from this proposal and should be considered by the relevant standards bodies. The following is by no means a complete
list, and primarily offers suggestions for consideration. The actual implementation is at the discretion of the relevant
standards bodies.

> **NOTE:** This builds on the [list](https://github.com/tc39/proposal-explicit-resource-management#relation-to-dom-apis)
> defined in the [Explicit Resource Management][using] proposal.

- `AudioContext` &mdash; `@@asyncDispose()` as an alias or [wrapper][] for `close()`.
  - NOTE: `close()` here is asynchronous, but uses the same name as similar synchronous methods on other objects.
- `MediaKeySession` &mdash; `@@asyncDispose()` as an alias or [wrapper][] for `close()`.
  - NOTE: `close()` here is asynchronous, but uses the same name as similar synchronous methods on other objects.
- `PaymentRequest` &mdash; `@@asyncDispose()` could invoke `abort()` if the payment is still in the active state.
  - NOTE: `abort()` here is asynchronous, but uses the same name as similar synchronous methods on other objects.
- `PushSubscription` &mdash; `@@asyncDispose()` as an alias or [wrapper][] for `unsubscribe()`.
- `ReadableStream` &mdash; `@@asyncDispose()` as an alias or [wrapper][] for `cancel()`.
- `ReadableStreamDefaultReader` &mdash; Either `@@dispose()` as an alias or [wrapper][] for `releaseLock()`, or
  `@@asyncDispose()` as a [wrapper][] for `cancel()` (but probably not both).
- `ServiceWorkerRegistration` &mdash; `@@asyncDispose()` as a [wrapper][] for `unregister()`.
- `WritableStream` &mdash; `@@asyncDispose()` as an alias or [wrapper][] for `close()`.
  - NOTE: `close()` here is asynchronous, but uses the same name as similar synchronous methods on other objects.
- `WritableStreamDefaultWriter` &mdash; Either `@@dispose()` as an alias or [wrapper][] for `releaseLock()`, or
  `@@asyncDispose()` as a [wrapper][] for `close()` (but probably not both).

### Definitions

A _<dfn><a name="wrapper"></a>wrapper</dfn> for `x()`_ is a method that invokes `x()`, but only if the object is in a state
such that calling `x()` will not throw as a result of repeated evaluation.

A _<dfn><a name="adapter"></a>callback-adapting wrapper</dfn>_ is a _wrapper_ that adapts a continuation passing-style method
that accepts a callback into a `Promise`-producing method.

A _<dfn><a name="disposer"></a>single-use disposer</dfn> for `x()` and `y()`_ indicates a newly constructed disposable object
that invokes `x()` when constructed and `y()` when disposed the first time (and does nothing if the object is disposed
more than once).

# Relation to NodeJS APIs

This proposal does not necessarily require immediate support in NodeJS, as existing APIs can still be adapted by using
`DisposableStack`. However, there are a number of APIs that could benefit from this proposal and should be considered by
the NodeJS maintainers. The following is by no means a complete list, and primarily offers suggestions for
consideration. The actual implementation is at the discretion of the NodeJS maintainers.

> **NOTE:** This builds on the [list](https://github.com/tc39/proposal-explicit-resource-management#relation-to-nodejs-apis)
> defined in the [Explicit Resource Management][using] proposal.


- `fs.promises.FileHandle` &mdash; `@@asyncDispose()` as an alias or [wrapper][] for `close()`.
- `fs.Dir` &mdash; `@@asyncDispose()` as an alias or [wrapper][] for `close()`, `@@dispose()` as an alias or [wrapper][]
  for `closeSync()`.
- `http.ClientRequest` &mdash; Either `@@dispose()` or `@@asyncDispose()` as an alias or [wrapper][] for `destroy()`.
- `http.Server` &mdash; `@@asyncDispose()` as a [callback-adapting wrapper][] for `close()`.
- `http.ServerResponse` &mdash; `@@asyncDispose()` as a [callback-adapting wrapper][] for `end()`.
- `http.IncomingMessage` &mdash; Either `@@dispose()` or `@@asyncDispose()` as an alias or [wrapper][] for `destroy()`.
- `http.OutgoingMessage` &mdash; Either `@@dispose()` or `@@asyncDispose()` as an alias or [wrapper][] for `destroy()`.
- `http2.Http2Session` &mdash; `@@asyncDispose()` as a [callback-adapting wrapper][] for `close()`.
- `http2.Http2Stream` &mdash; `@@asyncDispose()` as a [callback-adapting wrapper][] for `close()`.
- `http2.Http2Server` &mdash; `@@asyncDispose()` as a [callback-adapting wrapper][] for `close()`.
- `http2.Http2SecureServer` &mdash; `@@asyncDispose()` as a [callback-adapting wrapper][] for `close()`.
- `http2.Http2ServerRequest` &mdash; Either `@@dispose()` or `@@asyncDispose()` as an alias or [wrapper][] for
  `destroy()`.
- `http2.Http2ServerResponse` &mdash; `@@asyncDispose()` as a [callback-adapting wrapper][] for `end()`.
- `https.Server` &mdash; `@@asyncDispose()` as a [callback-adapting wrapper][] for `close()`.
- `stream.Writable` &mdash; Either `@@dispose()` or `@@asyncDispose()` as an alias or [wrapper][] for `destroy()` or
  `@@asyncDispose` only as a [callback-adapting wrapper][] for `end()` (depending on whether the disposal behavior
  should be to drop immediately or to flush any pending writes).
- `stream.Readable` &mdash; Either `@@dispose()` or `@@asyncDispose()` as an alias or [wrapper][] for `destroy()`.
- ... and many others in `net`, `readline`, `tls`, `udp`, and `worker_threads`.

# Meeting Notes

* [TC39 July 24th, 2018](https://github.com/tc39/notes/blob/main/meetings/2018-07/july-24.md#explicit-resource-management)
  - [Conclusion](https://github.com/tc39/notes/blob/main/meetings/2018-07/july-24.md#conclusionresolution-7)
    - Stage 1 acceptance
* [TC39 July 23rd, 2019](https://github.com/tc39/notes/blob/main/meetings/2019-07/july-23.md#explicit-resource-management)
  - [Conclusion](https://github.com/tc39/notes/blob/main/meetings/2019-07/july-23.md#conclusionresolution-7)
    - Table until Thursday, inconclusive.
* [TC39 July 25th, 2019](https://github.com/tc39/notes/blob/main/meetings/2019-07/july-25.md#explicit-resource-management-for-stage-2-continuation-from-tuesday)
  - [Conclusion](https://github.com/tc39/notes/blob/main/meetings/2019-07/july-25.md#conclusionresolution-7):
    - Investigate Syntax
    - Approved for Stage 2
    - YK (@wycatz) & WH (@waldemarhorwat) will be stage 3 reviewers
* [TC39 October 10th, 2021](https://github.com/tc39/notes/blob/main/meetings/2021-10/oct-27.md#explicit-resource-management-update)
  - [Conclusion](https://github.com/tc39/notes/blob/main/meetings/2021-10/oct-27.md#conclusionresolution-1)
      - Status Update only
      - WH Continuing to review
      - SYG (@syg) added as reviewer
* TC39 December 1st, 2022 (notes TBA)
  - Conclusion
    - `await using` declarations, `Symbol.asyncDispose`, and `AsyncDisposableStack` remain at Stage 2 as an independent
      proposal.

# TODO

The following is a high-level list of tasks to progress through each stage of the [TC39 proposal process](https://tc39.github.io/process-document/):

### Stage 1 Entrance Criteria

* [x] Identified a "[champion][Champion]" who will advance the addition.
* [x] [Prose][Prose] outlining the problem or need and the general shape of a solution.
* [x] Illustrative [examples][Examples] of usage.
* [x] High-level [API][API].

### Stage 2 Entrance Criteria

* [x] [Initial specification text][Specification].
* [ ] [Transpiler support][Transpiler] (_Optional_).

### Stage 3 Entrance Criteria

* [x] [Complete specification text][Specification].
* [x] Designated reviewers have signed off on the current spec text:
  * [x] [Waldemar Horwat][Stage3Reviewer1] has [signed off][Stage3Reviewer1SignOff]
  * [x] Shu-yu Guo has signed off (NOTE: sign-off occured over Matrix)
* [x] The [ECMAScript editor][Stage3Editor] has [signed off][Stage3EditorSignOff] on the current spec text.

### Stage 4 Entrance Criteria

* [ ] [Test262](https://github.com/tc39/test262) acceptance tests have been written for mainline usage scenarios and [merged][Test262PullRequest].
* [ ] Two compatible implementations which pass the acceptance tests: [\[1\]][Implementation1], [\[2\]][Implementation2].
* [ ] A [pull request][Ecma262PullRequest] has been sent to tc39/ecma262 with the integrated spec text.
* [ ] The ECMAScript editor has signed off on the [pull request][Ecma262PullRequest].

## Implementations

- Built-ins from this proposal are available in [`core-js`](https://github.com/zloirock/core-js#async-explicit-resource-management)


<!-- # References -->

<!-- Links to other specifications, etc. -->


<!-- * [Title](url) -->


<!-- # Prior Discussion -->

<!-- Links to prior discussion topics on https://esdiscuss.org -->


<!-- * [Subject](https://esdiscuss.org) -->


<!-- The following are shared links used throughout the README: -->

[Champion]: #status
[Prose]: #motivations
[Examples]: #examples
[API]: #api
[Specification]: https://tc39.es/proposal-async-explicit-resource-management
[Transpiler]: #todo
[Stage3Reviewer1]: https://github.com/tc39/proposal-async-explicit-resource-management/pull/15
[Stage3Reviewer1SignOff]: https://github.com/tc39/proposal-async-explicit-resource-management/pull/15#pullrequestreview-1378277626
[Stage3Reviewer2]: #todo
[Stage3Reviewer2SignOff]: #todo
[Stage3Editor]: https://github.com/tc39/proposal-async-explicit-resource-management/issues/9
[Stage3EditorSignOff]: https://github.com/tc39/proposal-async-explicit-resource-management/pull/10#pullrequestreview-1274348423
[Test262PullRequest]: #todo
[Implementation1]: #todo
[Implementation2]: #todo
[Ecma262PullRequest]: #todo
[wrapper]: #wrapper
[callback-adapting wrapper]: #adapter
[single-use disposer]: #disposer
[using]: https://github.com/tc39/proposal-explicit-resource-management
