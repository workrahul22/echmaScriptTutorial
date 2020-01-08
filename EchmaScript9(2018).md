This blog post explains the ECMAScript proposal “Asynchronous Iteration” by Domenic Denicola and Kevin Smith.

# Asynchronous iteration  

With ECMAScript 6, JavaScript got built-in support for synchronously iterating over data. But what about data that is delivered asynchronously? For example, lines of text, read asynchronously from a file or an HTTP connection.

This proposal brings support for that kind of data. Before we go into it, let’s first recap synchronous iteration.
Synchronous iteration  

Synchronous iteration was introduced with ES6 and works as follows:

- Iterable: an object that signals that it can be iterated over, via a method whose key is Symbol.iterator.
- Iterator: an object returned by invoking [Symbol.iterator]() on an iterable. It wraps each iterated element in an object and returns it via its method next() – one at a time.
- IteratorResult: an object returned by next(). Property value contains an iterated element, property done is true after the last element (value can usually be ignored then; it’s almost always undefined).

I’ll demonstrate via an Array:

```JavaScript
> const iterable = ['a', 'b'];
> const iterator = iterable[Symbol.iterator]();
> iterator.next()
{ value: 'a', done: false }
> iterator.next()
{ value: 'b', done: false }
> iterator.next()
{ value: undefined, done: true }
```

## Asynchronous iteration  

The problem is that the previously explained way of iterating is synchronous, it doesn’t work for asynchronous sources of data. For example, in the following code, readLinesFromFile() cannot deliver its asynchronous data via synchronous iteration:

```Javascript
for (const line of readLinesFromFile(fileName)) {
    console.log(line);
}
```

The proposal specifies a new protocol for iteration that works asynchronously:

    Async iterables are marked via Symbol.asyncIterator.
    Method next() of an async iterator returns Promises for IteratorResults (vs. IteratorResults directly).

You may wonder whether it would be possible to instead use a synchronous iterator that returns one Promise for each iterated element. But that is not enough – whether or not iteration is done is generally determined asynchronously.

Using an asynchronous iterable looks as follows. Function createAsyncIterable() is explained later. It converts its synchronously iterable parameter into an async iterable.

```Javascript
const asyncIterable = createAsyncIterable(['a', 'b']);
const asyncIterator = asyncIterable[Symbol.asyncIterator]();
asyncIterator.next()
.then(iterResult1 => {
    console.log(iterResult1); // { value: 'a', done: false }
    return asyncIterator.next();
})
.then(iterResult2 => {
    console.log(iterResult2); // { value: 'b', done: false }
    return asyncIterator.next();
})
.then(iterResult3 => {
    console.log(iterResult3); // { value: undefined, done: true }
});
```

Within an asynchronous function, you can process the results of the Promises via await and the code becomes simpler:

```Javascript
async function f() {
    const asyncIterable = createAsyncIterable(['a', 'b']);
    const asyncIterator = asyncIterable[Symbol.asyncIterator]();
    console.log(await asyncIterator.next());
        // { value: 'a', done: false }
    console.log(await asyncIterator.next());
        // { value: 'b', done: false }
    console.log(await asyncIterator.next());
        // { value: undefined, done: true }
}
```

The interfaces for async iteration  

In TypeScript notation, the interfaces look as follows.

```Javascript
interface AsyncIterable {
    [Symbol.asyncIterator]() : AsyncIterator;
}

interface AsyncIterator {
    next() : Promise<IteratorResult>;
}
interface IteratorResult {
    value: any;
    done: boolean;
}
```

# for-await-of  

The proposal also specifies an asynchronous version of the for-of loop: for-await-of:

```Javascript
async function f() {
    for await (const x of createAsyncIterable(['a', 'b'])) {
        console.log(x);
    }
}

// Output:
// a
// b
```

## for-await-of and rejections  

Similarly to how await works in async functions, the loop throws an exception if next() returns a rejection:

```Javascript
function createRejectingIterable() {
    return {
        [Symbol.asyncIterator]() {
            return this;
        },
        next() {
            return Promise.reject(new Error('Problem!'));
        },
    };
}
(async function () { // (A)
    try {
        for await (const x of createRejectingIterable()) {
            console.log(x);
        }
    } catch (e) {
        console.error(e);
            // Error: Problem!
    }
})(); // (B)
```

Note that we have just used an Immediately Invoked Async Function Expression (IIAFE, pronounced “yaffee”). It starts in line (A) and ends in line (B). We need to do that because for-of-await doesn’t work at the top level of modules and scripts. It does work everywhere where await can be used. Namely, in async functions and async generators (which are explained later).

### for-await-of and sync iterables  

for-await-of can also be used to iterate over sync iterables:

```Javascript
(async function () {
    for await (const x of ['a', 'b']) {
        console.log(x);
    }
})();
// Output:
// a
// b
```

for-await-of converts each iterated value via Promise.resolve() to a Promise, which it then awaits. That means that it works for both Promises and normal values.
Asynchronous generators  

Normal (synchronous) generators help with implementing synchronous iterables. Asynchronous generators do the same for asynchronous iterables.

For example, we have previously used the function createAsyncIterable(syncIterable) which converts a syncIterable into an asynchronous iterable. This is how you would implement this function via an async generator:

```Javascript
async function* createAsyncIterable(syncIterable) {
    for (const elem of syncIterable) {
        yield elem;
    }
}
```

Note the asterisk after function:

    A normal function is turned into a normal generator by putting an asterisk after function.
    An async function is turned into an async generator by doing the same.

How do async generators work?

    A normal generator returns a generator object genObj. Each invocation genObj.next() returns an object {value,done} that wraps a yielded value.
    An async generator returns a generator object genObj. Each invocation genObj.next() returns a Promise for an object {value,done} that wraps a yielded value.

Queuing next() invocations  

The JavaScript engine internally queues invocations of next() and feeds them to an async generator once it is ready. That is, after calling next(), you can call again, right away; you don’t have to wait for the Promise it returns to be settled. In most cases, though, you do want to wait for the settlement, because you need the value of done in order to decide whether to call next() again or not. That’s how the for-await-of loop works.

Use cases for calling next() several times without waiting for settlements include:

Use case: Retrieving Promises to be processed via Promise.all(). If you know how many elements there are in an async iterable, you don’t need to check done.

```Javascript
const asyncGenObj = createAsyncIterable(['a', 'b']);
const [{value:v1},{value:v2}] = await Promise.all([
    asyncGenObj.next(), asyncGenObj.next()
]);
console.log(v1, v2); // a b
```

Use case: Async generators as sinks for data, where you don’t always need to know when they are done.

```Javascript
const writer = openFile('someFile.txt');
writer.next('hello'); // don’t wait
writer.next('world'); // don’t wait
await writer.return(); // wait for file to close
```

You can use await and for-await-of inside async generators. For example:

```Javascript
async function* prefixLines(asyncIterable) {
    for await (const line of asyncIterable) {
        yield '> ' + line;
    }
}
```

One interesting aspect of combining await and yield is that await can’t stop yield from returning a Promise, but it can stop that Promise from being settled:

```Javascript
async function* asyncGenerator() {
    console.log('Start');
    const result = await doSomethingAsync(); // (A)
    yield 'Result: '+result; // (B)
    console.log('Done');
}
```

Let’s take a closer look at line (A) and (B):

    The yield in line (B) fulfills a Promise. That Promise is returned by next() immediately.
    Before that Promise is fulfilled, the operand of await (the Promise returned by doSomethingAsync() in line (A)) must be fulfilled.

That means that these two lines correspond (roughly) to this code:

```Javascript
return new Promise((resolve, reject) => {
    doSomethingAsync()
    .then(result => {
        resolve({
            value: 'Result: '+result,
            done: false,
        });
    });
});
```

If you want to dig deeper – this is a rough approximation of how async generators work:

```Javascript
const BUSY = Symbol('BUSY');
const COMPLETED = Symbol('COMPLETED');
function asyncGenerator() {
    const settlers = [];
    let step = 0;
    return {
        [Symbol.asyncIterator]() {
            return this;
        },
        next() {
            return new Promise((resolve, reject) => {
                settlers.push({resolve, reject});
                this._run();
            });
        }
        _run() {
            setTimeout(() => {
                if (step === BUSY || settlers.length === 0) {
                    return;
                }
                const currentSettler = settlers.shift();
                try {
                    switch (step) {
                        case 0:
                            step = BUSY;
                            console.log('Start');
                            doSomethingAsync()
                            .then(result => {
                                currentSettler.resolve({
                                    value: 'Result: '+result,
                                    done: false,
                                });
                                // We are not busy, anymore
                                step = 1;
                                this._run();
                            })
                            .catch(e => currentSettler.reject(e));
                            break;
                        case 1:
                            console.log('Done');
                            currentSettler.resolve({
                                value: undefined,
                                done: true,
                            });
                            step = COMPLETED;
                            this._run();
                            break;
                        case COMPLETED:
                            currentSettler.resolve({
                                value: undefined,
                                done: true,
                            });
                            this._run();
                            break;
                    }
                }
                catch (e) {
                    currentSettler.reject(e);
                }
            }, 0);
        }
    }
}
```

This code assumes that next() is always called without arguments. A complete implementation would have to queue arguments, too.
yield* in async generators  

yield* in async generators works analogously to how it works in normal generators – like a recursive invocation:

```Javascript
async function* gen1() {
    yield 'a';
    yield 'b';
    return 2;
}
async function* gen2() {
    const result = yield* gen1(); // (A)
        // result === 2
}
```

In line (A), gen2() calls gen1(), which means that all elements yielded by gen1() are yielded by gen2():

```Javascript
(async function () {
    for await (const x of gen2()) {
        console.log(x);
    }
})();
// Output:
// a
// b
```

The operand of yield* can be any async iterable. Sync iterables are automatically converted to async iterables, just like for for-await-of.
Errors  

In normal generators, next() can throw exceptions. In async generators, next() can reject the Promise it returns:

```Javascript
async function* asyncGenerator() {
    // The following exception is converted to a rejection
    throw new Error('Problem!');
}
asyncGenerator().next()
.catch(err => console.log(err)); // Error: Problem!
```

Converting exceptions to rejections is similar to how async functions work.
Async function vs. async generator function  

### Async function:

    Returns immediately with a Promise.
    That Promise is fulfilled via return and rejected via throw.

```Javascript
(async function () {
    return 'hello';
})()
.then(x => console.log(x)); // hello

(async function () {
    throw new Error('Problem!');
})()
.catch(x => console.error(x)); // Error: Problem!
```

### Async generator function:

    Returns immediately with an async iterable.
    Every invocation of next() returns a Promise. yield x fulfills the “current” Promise with {value: x, done: false}. throw err rejects the “current” Promise with err.

```Javascript
async function* gen() {
    yield 'hello';
}
const genObj = gen();
genObj.next().then(x => console.log(x));
    // { value: 'hello', done: false }
```

Examples  

The source code for the examples is available via the repository async-iter-demo on GitHub.
Using asynchronous iteration via Babel  

The example repo uses babel-node to run its code. This is how it configures Babel in its package.json:

```Javascript
{
  "dependencies": {
    "babel-preset-es2015-node": "···",
    "babel-preset-es2016": "···",
    "babel-preset-es2017": "···",
    "babel-plugin-transform-async-generator-functions": "···"
    ···
  },
  "babel": {
    "presets": [
      "es2015-node",
      "es2016",
      "es2017"
    ],
    "plugins": [
      "transform-async-generator-functions"
    ]
  },
  ···
}
```

Example: turning an async iterable into an Array  

Function takeAsync() collects all elements of asyncIterable in an Array. I don’t use for-await-of in this case, I invoke the async iteration protocol manually. I also don’t close asyncIterable if I’m finished before the iterable is done.

```Javascript
/**
 * @returns a Promise for an Array with the elements
 * in `asyncIterable`
 */
async function takeAsync(asyncIterable, count=Infinity) {
    const result = [];
    const iterator = asyncIterable[Symbol.asyncIterator]();
    while (result.length < count) {
        const {value,done} = await iterator.next();
        if (done) break;
        result.push(value);
    }
    return result;
}
```

This is the test for takeAsync():

```Javascript
test('Collect values yielded by an async generator', async function() {
    async function* gen() {
        yield 'a';
        yield 'b';
        yield 'c';
    }

    assert.deepStrictEqual(await takeAsync(gen()), ['a', 'b', 'c']);
    assert.deepStrictEqual(await takeAsync(gen(), 3), ['a', 'b', 'c']);
    assert.deepStrictEqual(await takeAsync(gen(), 2), ['a', 'b']);
    assert.deepStrictEqual(await takeAsync(gen(), 1), ['a']);
    assert.deepStrictEqual(await takeAsync(gen(), 0), []);
});
```

Note how nicely async functions work together with the mocha test framework: for asynchronous tests, the second parameter of test() can return a Promise.
Example: a queue as an async iterable  

The example repo also has an implementation for an asynchronous queue, called AsyncQueue. It’s implementation is relatively complex, which is why I don’t show it here. This is the test for AsyncQueue:

```Javascript
test('Enqueue before dequeue', async function() {
    const queue = new AsyncQueue();
    queue.enqueue('a');
    queue.enqueue('b');
    queue.close();
    assert.deepStrictEqual(await takeAsync(queue), ['a', 'b']);
});
test('Dequeue before enqueue', async function() {
    const queue = new AsyncQueue();
    const promise = Promise.all([queue.next(), queue.next()]);
    queue.enqueue('a');
    queue.enqueue('b');
    return promise.then(arr => {
        const values = arr.map(x => x.value);
        assert.deepStrictEqual(values, ['a', 'b']);
    });
});
```

Example: reading text lines asynchronously  

Let’s implement code that reads text lines asynchronously. We’ll do it in three steps.

Step 1: read text data in chunks via the Node.js ReadStream API (which is based on callbacks) and push it into an AsyncQueue (which was introduced in the previous section).

```Javascript
/**
 * Creates an asynchronous ReadStream for the file whose name
 * is `fileName` and feeds it into an AsyncQueue that it returns.
 *
 * @returns an async iterable
 */
function readFile(fileName) {
    const queue = new AsyncQueue();
    const readStream = createReadStream(fileName,
        { encoding: 'utf8', bufferSize: 1024 });
    readStream.on('data', buffer => {
        const str = buffer.toString('utf8');
        queue.enqueue(str);
    });
    readStream.on('end', () => {
        // Signal end of output sequence
        queue.close();
    });
    return queue;
}
```

Step 2: Use for-await-of to iterate over the chunks of text and yield lines of text.

```Javascript
/**
 * Turns a sequence of text chunks into a sequence of lines
 * (where lines are separated by newlines)
 *
 * @returns an async iterable
 */
async function* splitLines(chunksAsync) {
    let previous = '';
    for await (const chunk of chunksAsync) {
        previous += chunk;
        let eolIndex;
        while ((eolIndex = previous.indexOf('\n')) >= 0) {
            const line = previous.slice(0, eolIndex);
            yield line;
            previous = previous.slice(eolIndex+1);
        }
    }
    if (previous.length > 0) {
        yield previous;
    }
}
```

Step 3: combine the two previous functions. We first feed chunks of text into a queue via readFile() and then convert that queue into an async iterable over lines of text via splitLines().

```Javascript
/**
 * @returns an async iterable
 */
function readLines(fileName) {
    // `queue` is an async iterable
    const queue = readFile(fileName);
    return splitLines(queue);
}
```

Lastly, this is how you’d use readLines() from within a Node.js script:

```Javascript
(async function () {
    const fileName = process.argv[2];
    for await (const line of readLines(fileName)) {
        console.log('>', line);
    }
})();
```

WHATWG Streams are async iterables  

WHATWG streams are async iterables, meaning that you can use for-await-of to process them:

```Javascript
const rs = openReadableStream();
for await (const chunk of rs) {
    ···
}
```

### The specification of asynchronous iteration  

The spec introduces several new concepts and entities:

    Two new interfaces, AsyncIterable and AsyncIterator
    New well-known intrinsic objects: %AsyncGenerator%, %AsyncFromSyncIteratorPrototype%, %AsyncGeneratorFunction%, %AsyncGeneratorPrototype%, %AsyncIteratorPrototype%.
    One new well-known symbol: Symbol.asyncIterator

No new global variables are introduced by this feature.
Async generators  

If you want to understand how async generators work, it’s best to start with Sect. “AsyncGenerator Abstract Operations”. They key to understanding async generators is to understand how queuing works.

Two internal properties of async generator objects play important roles w.r.t. queuing:

    [[AsyncGeneratorState]] contains the state the generator is currently in: "suspendedStart", "suspendedYield", "executing", "completed" (it is undefined before it is fully initialized)
    [[AsyncGeneratorQueue]] holds pending invocations of next/throw/return. Each queue entry contains two fields:
        [[Completion]]: the parameter of next(), throw() or return() that lead to the entry being enqueued. The type of the completion (normal, throw, return) indicates which method call created the entry and determines what happens after dequeuing.
        [[Capability]]: the PromiseCapability of the pending Promise.

The queue is managed mainly via two operations:

    Enqueuing happens via AsyncGeneratorEnqueue(). This is the operation that is called by next(), return() and throw(). It adds an entry to the AsyncGeneratorQueue. Then AsyncGeneratorResumeNext() is called, but only if the generator’s state isn’t "executing":
        Therefore, if a generator calls next(), return() or throw() from inside itself then the effects of that call will be delayed.
        await leads to a suspension of the generator, but its state remains "executing". Hence, it will not be resumed by AsyncGeneratorEnqueue().

    Dequeuing happens via AsyncGeneratorResumeNext(). AsyncGeneratorResumeNext() is invoked after enqueuing, but also after settling a queued Promise (e.g. via yield), because there may now be new queued pending Promises, allowing execution to continue. If the queue is empty, return immediately. Otherwise, the current Promise is the first element of the queue:
        If the async generator was suspended by yield, it is resumed and continues to run. The current Promise is later settled via AsyncGeneratorResolve() or AsyncGeneratorReject().
        If the generator is already completed, this operation calls AsyncGeneratorResolve() and AsyncGeneratorReject() itself, meaning that all queued pending Promises will eventually be settled.

### Async-from-Sync Iterator Objects  

To get an async iterator from an object iterable, you call GetIterator(iterable, async) (async is a symbol). If iterable doesn’t have a method [Symbol.asyncIterator](), GetIterator() retrieves a sync iterator via method iterable[Symbol.iterator]() and converts it to an async iterator via CreateAsyncFromSyncIterator().
The for-await-of loop  

for-await-of works almost exactly like for-of, but there is an await whenever the contents of an IteratorResult are accessed. You can see that by looking at Sect. “Runtime Semantics: ForIn/OfBodyEvaluation”. Notably, iterators are closed similarly, via IteratorClose(), towards the end of this section.
Alternatives to async iteration  

Let’s look at two alternatives to async iteration for processing async data.
Alternative 1: Communicating Sequential Processes (CSP)  

The following code demonstrates the CSP library js-csp:

```Javascript
var csp = require('js-csp');

function* player(name, table) {
  while (true) {
    var ball = yield csp.take(table); // dequeue
    if (ball === csp.CLOSED) {
      console.log(name + ": table's gone");
      return;
    }
    ball.hits += 1;
    console.log(name + " " + ball.hits);
    yield csp.timeout(100); // wait
    yield csp.put(table, ball); // enqueue
  }
}

csp.go(function* () {
  var table = csp.chan(); // (A)

  csp.go(player, ["ping", table]); // (B)
  csp.go(player, ["pong", table]); // (C)

  yield csp.put(table, {hits: 0}); // enqueue
  yield csp.timeout(1000); // wait
  table.close();
});
```

player defines a “process” that is instantiated twice (in line (B) and in line (C), via csp.go()). The processes are connected via the “channel” table, which is created in line (A) and passed to player via its second parameter. A channel is basically a queue.

How does CSP compare to async iteration?

    The coding style is also synchronous.
    Channels feels like a good abstraction for producing and consuming async data.
    Making the connections between processes explicit, as channels, means that you can configure how they work (how much is buffered, when to block, etc.).
    The abstraction “channel” works for many use cases: communication with and between web workers, distributed programming, etc.

### Alternative 2: Reactive Programming  

The following code demonstrates Reactive Programming via the JavaScript library RxJS:

```Javascript
const button = document.querySelector('button');
Rx.Observable.fromEvent(button, 'click') // (A)
  .throttle(1000) // at most one event per second
  .scan(count => count + 1, 0)
  .subscribe(count => console.log(`Clicked ${count} times`));
```

In line (A), we create a stream of click events via fromEvent(). These events are then filtered so that there is at most one event per second. Every time there is an event, scan() counts how many events there have been, so far. In the last line, we log all counts.

How does Reactive Programming compare to async iteration?

    The coding style is not as familiar, but there are similarities to Promises.
    On the other hand, chaining operations (such as throttle()) works well for many push-based data sources (DOM events, server-sent events, etc.).
    Async iteration is for pull streams and single consumers. Reactive programming is for push streams and potentially multiple consumers. The former is better suited for I/O and can handle backpressure.

There is an ECMAScript proposal for Reactive Programming, called “Observable” (by Jafar Husain).
Is async iteration worth it?  

Now that I’ve used asynchronous iteration a little, I’ve come to a few conclusions. These conclusions are evolving, so let me know if you disagree with anything.

    Promises have become the primitive building block for everything async in JavaScript. And it’s a joy to see how everything is constantly improving: more and more APIs are using Promises, stack traces are getting better, performance is increasing, using them via async functions is great, etc.

    We need some kind of support for asynchronous sequences of data. It wouldn’t necessarily have to be a language mechanism; it could just be part of the standard library. At the moment, the situation is similar to one-time async data before Promises: various patterns exist, but there is no clear standard and interoperability.

    Async iteration brings with it considerable additional cognitive load:
        There are now normal functions, generator functions, async functions and async generator functions. And each kind of function exists as declaration, expression and method. Knowing what to use when is becoming more complicated.
        Async iteration combines iteration with Promises. Individually, each of the two patterns takes a while to fully figure out (especially if you include generators under iteration). Combining them significantly increases the learning curve.

    Operations are missing: Sync iterables can be converted to Arrays via the spread operator and accessed via Array destructuring. There are no equivalent operations for async iterables.

    Converting legacy APIs to async iteration isn’t easy. A queue that is asynchronously iterable helps, but the pieces don’t fit together as neatly as I would like. In comparison, going from callbacks to Promises is quite elegant.

Update 2018-01-25: This proposal has reached stage 4 and will be part of ECMAScript 2018.

The ECMAScript proposal “Rest/Spread Properties” by Sebastian Markbåge enables:

    The rest operator (...) in object destructuring. At the moment, this operator only works for Array destructuring and in parameter definitions.

    The spread operator (...) in object literals. At the moment, this operator only works in Array literals and in function and method calls.

The rest operator (...) in object destructuring  

Inside object destructuring patterns, the rest operator (...) copies all enumerable own properties of the destructuring source into its operand, except those that were already mentioned in the object literal.

```Javascript
const obj = {foo: 1, bar: 2, baz: 3};
const {foo, ...rest} = obj;
    // Same as:
    // const foo = 1;
    // const rest = {bar: 2, baz: 3};
```

If you are using object destructuring to handle named parameters, the rest operator enables you to collect all remaining parameters:

```Javascript
function func({param1, param2, ...rest}) { // rest operator
    console.log('All parameters: ',
        {param1, param2, ...rest}); // spread operator
    return param1 + param2;
}
```

Syntactic restrictions  

Per top level of each object literal, you can use the rest operator at most once and it must appear at the end:

```Javascript
const {...rest, foo} = obj; // SyntaxError
const {foo, ...rest1, ...rest2} = obj; // SyntaxError
```

You can, however, use the rest operator several times if you nest it:

```Javascript
const obj = {
    foo: {
        a: 1,
        b: 2,
        c: 3,
    },
    bar: 4,
    baz: 5,
};
const {foo: {a, ...rest1}, ...rest2} = obj;
// Same as:
// const a = 1;
// const rest1 = {b: 2, c: 3};
// const rest2 = {bar: 4, baz: 5};
```

The spread operator (...) in object literals  

Inside object literals, the spread operator (...) inserts all enumerable own properties of its operand into the object created via the literal:

```Javascript
> const obj = {foo: 1, bar: 2, baz: 3};
> {...obj, qux: 4}
{ foo: 1, bar: 2, baz: 3, qux: 4 }
```

Note that order matters even if property keys don’t clash, because objects record insertion order:

```Javascript
> {qux: 4, ...obj}
{ qux: 4, foo: 1, bar: 2, baz: 3 }
```

If keys clash, order determines which entry “wins”:

```Javascript
> const obj = {foo: 1, bar: 2, baz: 3};
> {...obj, foo: true}
{ foo: true, bar: 2, baz: 3 }
> {foo: true, ...obj}
{ foo: 1, bar: 2, baz: 3 }
```

Common use cases for the object spread operator  

In this section, we’ll look at things that you can use the spread operator for. I’ll also show how to do these things via Object.assign(), which is very similar to the spread operator (we’ll compare them in more detail later).
Cloning objects  

Cloning the enumerable own properties of an object obj:

```Javascript
const clone1 = {...obj};
const clone2 = Object.assign({}, obj);
```

The prototypes of the clones are always Object.prototype, which is the default for objects created via object literals:

```Javascript
> Object.getPrototypeOf(clone1) === Object.prototype
true
> Object.getPrototypeOf(clone2) === Object.prototype
true
> Object.getPrototypeOf({}) === Object.prototype
true
```

Cloning an object obj, including its prototype:

```Javascript
const clone1 = {__proto__: Object.getPrototypeOf(obj), ...obj};
const clone2 = Object.assign(
    Object.create(Object.getPrototypeOf(obj)), obj);
```

Note that __proto__ inside object literals is only a mandatory feature in web browsers, not in JavaScript engines in general.
True clones of objects  

Sometimes you need to faithfully copy all own properties of an object obj and their attributes (writable, enumerable, ...), including getters and setters. Then Object.assign() and the spread operator don’t work. You need to use property descriptors:

```Javascript
const clone1 = Object.defineProperties({},
    Object.getOwnPropertyDescriptors(obj));
```

If you additionally want to preserve the prototype of obj, you can use Object.create():

```Javascript
const clone2 = Object.create(
    Object.getPrototypeOf(obj),
    Object.getOwnPropertyDescriptors(obj));
```

Object.getOwnPropertyDescriptors() is explained in “Exploring ES2016 and ES2017”.
Pitfall: cloning is always shallow  

Keep in mind that with all the ways of cloning that we have looked at, you only get shallow copies: If one of the original property values is an object, the clone will refer to the same object, it will not be (recursively, deeply) cloned itself:

```Javascript
const original = { prop: {} };
const clone = Object.assign({}, original);

console.log(original.prop === clone.prop); // true
original.prop.foo = 'abc';
console.log(clone.prop.foo); // abc
```

Various other use cases  

Merging two objects obj1 and obj2:

```Javascript
const merged = {...obj1, ...obj2};
const merged = Object.assign({}, obj1, obj2);
```

Filling in defaults for user data:

```Javascript
const DEFAULTS = {foo: 'a', bar: 'b'};
const userData = {foo: 1};

const data = {...DEFAULTS, ...userData};
const data = Object.assign({}, DEFAULTS, userData);
    // {foo: 1, bar: 'b'}
```

Non-destructively updating property foo:

```Javascript
const obj = {foo: 'a', bar: 'b'};
const obj2 = {...obj, foo: 1};
const obj2 = Object.assign({}, obj, {foo: 1});
    // {foo: 1, bar: 'b'}
```

Specifying the default values for properties foo and bar inline:

```Javascript
const userData = {foo: 1};
const data = {foo: 'a', bar: 'b', ...userData};
const data = Object.assign({}, {foo:'a', bar:'b'}, userData);
    // {foo: 1, bar: 'b'}
```

Spreading objects versus Object.assign()  

The spread operator and Object.assign() are very similar. The main difference is that spreading defines new properties, while Object.assign() sets them. What exactly that means is explained later.
The two ways of using Object.assign()  

There are two ways of using Object.assign():

First, destructively (an existing object is changed):

Object.assign(target, source1, source2);

Here, target is modified; source1 and source2 are copied into it.

Second, non-destructively (no existing object is changed):

const result = Object.assign({}, source1, source2);

Here, a new object is created via an empty object literal and source1 and source2 are copied into it. At the end, this new object is returned and assigned to result.

The spread operator is very similar to the second way of using Object.assign(). Next, we’ll look at where the two are similar and where they differ.
Both spread and Object.assign() read values via a “get” operation  

Both operations use normal “get” operations to read property values from the source, before writing them to the target. As a result, getters are turned into normal data properties during this process.

Let’s look at an example:

```Javascript
const original = {
    get foo() {
        return 123;
    }
};
```

original has the getter foo (its property descriptor has the properties get and set):

```Javascript
> Object.getOwnPropertyDescriptor(original, 'foo')
{ get: [Function: foo],
  set: undefined,
  enumerable: true,
  configurable: true }
```

But it its clones clone1 and clone2, foo is a normal data property (its property descriptor has the properties value and writable):

```Javascript
> const clone1 = {...original};
> Object.getOwnPropertyDescriptor(clone1, 'foo')
{ value: 123,
  writable: true,
  enumerable: true,
  configurable: true }

> const clone2 = Object.assign({}, original);
> Object.getOwnPropertyDescriptor(clone2, 'foo')
{ value: 123,
  writable: true,
  enumerable: true,
  configurable: true }
```

Spread defines properties, Object.assign() sets them  

The spread operator defines new properties in the target, Object.assign() uses a normal “set” operation to create them. That has two consequences.
Targets with setters  

First, Object.assign() triggers setters, spread doesn’t:

```Javascript
Object.defineProperty(Object.prototype, 'foo', {
    set(value) {
        console.log('SET', value);
    },
});
const obj = {foo: 123};
```

The previous piece of code installs a setter foo that is inherited by all normal objects.

If we clone obj via Object.assign(), the inherited setter is triggered:

```Javascript
> Object.assign({}, obj)
SET 123
{}
```

With spread, it isn’t:

> { ...obj }
{ foo: 123 }

Object.assign() also triggers own setters during copying, it does not overwrite them.
Targets with read-only properties  

Second, you can stop Object.assign() from creating own properties via inherited read-only properties, but not the spread operator:

```Javascript
Object.defineProperty(Object.prototype, 'bar', {
    writable: false,
    value: 'abc',
});
```

The previous piece of code installs the read-only property bar that is inherited by all normal objects.

As a consequence, you can’t use assignment to create the own property bar, anymore (you only get an exception in strict mode; in sloppy mode, setting fails silently):

> const tmp = {};
> tmp.bar = 123;
TypeError: Cannot assign to read only property 'bar'

In the following code, we successfully create the property bar via an object literal. This works, because object literals don’t set properties, they define them:

```Javascript
const obj = {bar: 123};
```

However, Object.assign() uses assignment for creating properties, which is why we can’t clone obj:

> Object.assign({}, obj)
TypeError: Cannot assign to read only property 'bar'

Cloning via the spread operator works:

> { ...obj }
{ bar: 123 }

Both spread and Object.assign() only consider own enumerable properties  

Both operations ignore all inherited properties and all non-enumerable own properties.

The following object obj inherits one (enumerable!) property from proto and has two own properties:

```Javascript
const proto = {
    inheritedEnumerable: 1,
};
const obj = Object.create(proto, {
    ownEnumerable: {
        value: 2,
        enumerable: true,
    },
    ownNonEnumerable: {
        value: 3,
        enumerable: false,
    },
});
```

If you clone obj, the result only has the property ownEnumerable. The properties inheritedEnumerable and ownNonEnumerable are not copied:

> {...obj}
{ ownEnumerable: 2 }
> Object.assign({}, obj)
{ ownEnumerable: 2 }

ES2018: RegExp named capture groups
[2017-05-15] dev, javascript, esnext, es2018, regexp
(Ad, please don’t block)

The proposal “RegExp Named Cainstpture Groups” by Gorkem Yakin, Daniel Ehrenberg is at stage 4. This blog post explains what it has to offer.

Before we get to named capture groups, let’s take a look at numbered capture groups; to introduce the idea of capture groups.
Numbered capture groups  

Numbered capture groups enable you to take apart a string with a regular expression.

Successfully matching a regular expression against a string returns a match object matchObj. Putting a fragment of the regular expression in parentheses turns that fragment into a capture group: the part of the string that it matches is stored in matchObj.

Prior to this proposal, all capture groups were accessed by number: the capture group starting with the first parenthesis via matchObj[1], the capture group starting with the second parenthesis via matchObj[2], etc.

For example, the following code shows how numbered capture groups are used to extract year, month and day from a date in ISO format:

```Javascript
const RE_DATE = /([0-9]{4})-([0-9]{2})-([0-9]{2})/;

const matchObj = RE_DATE.exec('1999-12-31');
const year = matchObj[1]; // 1999
const month = matchObj[2]; // 12
const day = matchObj[3]; // 31
```

Referring to capture groups via numbers has several disadvantages:

    Finding the number of a capture group is a hassle: you have to count parentheses.
    You need to see the regular expression if you want to understand what the groups are for.
    If you change the order of the capture groups, you also have to change the matching code.

All issues can be somewhat mitigated by defining constants for the numbers of the capture groups. However, capture groups are an all-around superior solution.
Named capture groups  

The proposed feature is about identifying capture groups via names:

(?<year>[0-9]{4})

Here we have tagged the previous capture group #1 with the name year. The name must be a legal JavaScript identifier (think variable name or property name). After matching, you can access the captured string via matchObj.groups.year.

The captured strings are not properties of matchObj, because you don’t want them to clash with current or future properties created by the regular expression API.

Let’s rewrite the previous code so that it uses named capture groups:

```Javascript
const RE_DATE = /(?<year>[0-9]{4})-(?<month>[0-9]{2})-(?<day>[0-9]{2})/;

const matchObj = RE_DATE.exec('1999-12-31');
const year = matchObj.groups.year; // 1999
const month = matchObj.groups.month; // 12
const day = matchObj.groups.day; // 31
```

Named capture groups also create indexed entries; as if they were numbered capture groups:

```Javascript
const year2 = matchObj[1]; // 1999
const month2 = matchObj[2]; // 12
const day2 = matchObj[3]; // 31
```

Destructuring can help with getting data out of the match object:

```Javascript
const {groups: {day, year}} = RE_DATE.exec('1999-12-31');
console.log(year); // 1999
console.log(day); // 31
```

Named capture groups have the following benefits:

    It’s easier to find the “ID” of a capture group.
    The matching code becomes self-descriptive, as the ID of a capture group describes what is being captured.
    You don’t have to change the matching code if you change the order of the capture groups.
    The names of the capture groups also make the regular expression easier to understand, as you can see directly what each group is for.

You can freely mix numbered and named capture groups.
Backreferences  

\k<name> in a regular expression means: match the string that was previously matched by the named capture group name. For example:

```Javascript
const RE_TWICE = /^(?<word>[a-z]+)!\k<word>$/;
RE_TWICE.test('abc!abc'); // true
RE_TWICE.test('abc!ab'); // false
```

The backreference syntax for numbered capture groups works for named capture groups, too:

```Javascript
const RE_TWICE = /^(?<word>[a-z]+)!\1$/;
RE_TWICE.test('abc!abc'); // true
RE_TWICE.test('abc!ab'); // false
```

You can freely mix both syntaxes:

```Javascript
const RE_TWICE = /^(?<word>[a-z]+)!\k<word>!\1$/;
RE_TWICE.test('abc!abc!abc'); // true
RE_TWICE.test('abc!abc!ab'); // false
```

replace() and named capture groups  

The string method replace() supports named capture groups in two ways.

First, you can mention their names in the replacement string:

```Javascript
const RE_DATE = /(?<year>[0-9]{4})-(?<month>[0-9]{2})-(?<day>[0-9]{2})/;
console.log('1999-12-31'.replace(RE_DATE,
    '$<month>/$<day>/$<year>'));
    // 12/31/1999
```

Second, each replacement function receives an additional parameter that holds an object with data captured via named groups. For example (line A):

```JavaScript
const RE_DATE = /(?<year>[0-9]{4})-(?<month>[0-9]{2})-(?<day>[0-9]{2})/;
console.log('1999-12-31'.replace(
    RE_DATE,
    (g0,y,m,d,offset,input, {year, month, day}) => // (A)
        month+'/'+day+'/'+year));
    // 12/31/1999
```

These are the parameters of the callback in line A:

    g0 contains the whole matched substring, '1999-12-31'
    y, m, d are matches for the numbered groups 1–3 (which are created via the named groups year, month, day).
    offset specifies where the match was found.
    input contains the complete input string.
    The last parameter is new and contains one property for each of the three named capture groups year, month and day. We use destructuring to access those properties.

The following code shows another way of accessing the last argument:

```JavaScript
console.log('1999-12-31'.replace(RE_DATE,
    (...args) => {
        const {year, month, day} = args[args.length-1];
        return month+'/'+day+'/'+year;
    }));
    // 12/31/1999
```

We receive all arguments via the rest parameter args. The last element of the Array args is the object with the data from the named groups. We access it via the index args.length-1.
Named groups that don’t match  

If an optional named group does not match, its property is set to undefined (but still exists):

```JavaScript
const RE_OPT_A = /^(?<as>a+)?$/;
const matchObj = RE_OPT_A.exec('');

// We have a match:
console.log(matchObj[0] === ''); // true

// Group <as> didn’t match anything:
console.log(matchObj.groups.as === undefined); // true

// But property `as` exists:
console.log('as' in matchObj.groups); // true
```

Implementations  

    The Babel plugin transform-modern-regexp by Dmitry Soshnikov supports named capture groups.
    V8 6.0+ has support behind a flag.

The relevant V8 is not yet in Node.js (7.10.0). You can check via:

node -p process.versions.v8

In Chrome Canary (60.0+), you can enable named capture groups as follows. First, look up the path of the Chrome Canary binary via the about: URL. Then start Canary like this (you only need the double quotes if the path contains a space):

$ alias canary='"/tmp/Google Chrome Canary.app/Contents/MacOS/Google Chrome Canary"'
$ canary --js-flags='--harmony-regexp-named-captures'

Further reading  

    Chapter “Regular Expressions” in “Speaking JavaScript”
    Chapter “New regular expression features” in “Exploring ES6”
    Chapter “Destructuring” in “Exploring ES6”

ES2018: RegExp Unicode property escapes
[2017-07-19] dev, javascript, esnext, es2018, regexp
(Ad, please don’t block)
ads via Carbon
Adobe Creative Cloud for Teams starting at $29.99 per month.
ads via Carbon

The proposal “RegExp Unicode Property Escapes” by Mathias Bynens is at stage 4. This blog post explains how it works.
Overview  

JavaScript lets you match characters by mentioning the “names” of sets of characters. For example, \s stands for “whitespace”:

> /^\s+$/u.test('\t \n\r')
true

The proposal lets you additionally match characters by mentioning their Unicode character properties (what those are is explained next) inside the curly braces of \p{}. Two examples:

> /^\p{White_Space}+$/u.test('\t \n\r')
true
> /^\p{Script=Greek}+$/u.test('μετά')
true

As you can see, one of the benefits of property escapes is is that they make regular expressions more self-descriptive. Additional benefits will become clear later.

Before we delve into how property escapes work, let’s examine what Unicode character properties are.
Unicode character properties  

In the Unicode standard, each character has properties – metadata describing it. Properties play an important role in defining the nature of a character. Quoting the Unicode Standard, Sect. 3.3, D3:

    The semantics of a character are determined by its identity, normative properties, and behavior.

Examples of properties  

These are a few examples of properties:

    Name: a unique name, composed of uppercase letters, digits, hyphens and spaces. For example:
        A: Name = LATIN CAPITAL LETTER A
        😀: Name = GRINNING FACE
    General_Category: categorizes characters. For example:
        x: General_Category = Lowercase_Letter
        $: General_Category = Currency_Symbol
    White_Space: used for marking invisible spacing characters, such as spaces, tabs and newlines. For example:
        \t: White_Space = True
        π: White_Space = False
    Age: version of the Unicode Standard in which a character was introduced. For example: The Euro sign € was added in version 2.1 of the Unicode standard.
        €: Age = 2.1
    Block: a contiguous range of code points. Blocks don’t overlap and their names are unique. For example:
        S: Block = Basic_Latin (range U+0000..U+007F)
        Д: Block = Cyrillic (range U+0400..U+04FF)
    Script: is a collection of characters used by one or more writing systems.
        Some scripts support several writing systems. For example, the Latin script supports the writing systems English, French, German, Latin, etc.
        Some languages can be written in multiple alternate writing systems that are supported by multiple scripts. For example, Turkish used the Arabic script before it transitioned to the Latin script in the early 20th century.
        Examples:
            α: Script = Greek
            א: Script = Hebrew

Types of properties  

The following types of properties exist:

    Enumerated property: a property whose values are few and named. General_Category is an enumerated property.
    Closed enumerated property: an enumerated property whose set of values is fixed and will not be changed in future versions of the Unicode Standard.
    Boolean property: a closed enumerated property whose values are True and False. Boolean properties are also called binary, because they are like markers that characters either have or not. White_Space is a binary property.
    Numeric property: has values that are integers or real numbers.
    String-valued property: a property whose values are strings.
    Catalog property: an enumerated property that may be extended as the Unicode Standard evolves. Age and Script are catalog properties.
    Miscellaneous property: a property whose values are not Boolean, enumerated, numeric, string or catalog values. Name is a miscellaneous property.

Matching properties and property values  

Properties and property values are matched as follows:

    Loose matching: case, whitespace, underscores and hyphens are ignored when comparing properties and property values. For example, "General_Category", "general category", "-general-category-", "GeneralCategory" are all considered to be the same property.
    Aliases: the data files PropertyAliases.txt and PropertyValueAliases.txt define alternative ways of referring to properties and property values.
        Most aliases have long forms and short forms. For example:
            Long form: General_Category
            Short form: gc
        Examples of property value aliases (per line, all values are considered equal):
            Lowercase_Letter, Ll
            Currency_Symbol, Sc
            True, T, Yes, Y
            False, F, No, N

Unicode property escapes for regular expressions  

Unicode property escapes look like this:

    Match all characters whose property prop has the value value:

    \p{prop=value}

    Match all characters that do not have a property prop whose value is value:

    \P{prop=value}

    Match all characters whose binary property bin_prop is True:

    \p{bin_prop}

    Match all characters whose binary property bin_prop is False:

    \P{bin_prop}

Forms (3) and (4) can also be used as an abbreviation for General_Category. For example: \p{Lowercase_Letter} is an abbreviation for \p{General_Category=Lowercase_Letter}

Important: In order to use property escapes, regular expressions must have the flag /u. Prior to /u, \p is the same as p.
Details  

Things to note:

    Property escapes do not support loose matching. You must use aliases exactly as they are mentioned in PropertyAliases.txt and PropertyValueAliases.txt
    Implementations must support at least the following Unicode properties and their aliases:
        General_Category
        Script
        Script_Extensions
        The binary properties listed in the specification (and no others, to guarantee interoperability). These include, among others: Alphabetic, Uppercase, Lowercase, White_Space, Noncharacter_Code_Point, Default_Ignorable_Code_Point, Any, ASCII, Assigned, ID_Start, ID_Continue, Join_Control, Emoji_Presentation, Emoji_Modifier, Emoji_Modifier_Base.

Examples  

Matching whitespace:

> /^\p{White_Space}+$/u.test('\t \n\r')
true

Matching letters:

> /^\p{Letter}+$/u.test('πüé')
true

Matching Greek letters:

> /^\p{Script=Greek}+$/u.test('μετά')
true

Matching Latin letters:

> /^\p{Script=Latin}+$/u.test('Grüße')
true
> /^\p{Script=Latin}+$/u.test('façon')
true
> /^\p{Script=Latin}+$/u.test('mañana')
true

Matching lone surrogate characters:

> /^\p{Surrogate}+$/u.test('\u{D83D}')
true
> /^\p{Surrogate}+$/u.test('\u{DE00}')
true

Note that Unicode code points in astral planes (such as emojis) are composed of two JavaScript characters (a leading surrogate and a trailing surrogate). Therefore, you’d expect the previous regular expression to match the emoji 😀, which is all surrogates:

> '😀'.length
2
> '😀'.charCodeAt(0).toString(16)
'd83d'
> '😀'.charCodeAt(1).toString(16)
'de00'

However, with the /u flag, property escapes match code points, not JavaScript characters:

> /^\p{Surrogate}+$/u.test('😀')
false

In other words, 😀 is considered to be a single character:

> /^.$/u.test('😀')
true

Trying it out  

V8 5.8+ implement this proposal, it is switched on via --harmony_regexp_property:

    Node.js: node --harmony_regexp_property
        Check Node’s version of V8 via npm version
    Chrome:
        Go to chrome://version/
        Check the version of V8.
        Find the “Executable Path”. For example: /Applications/Google Chrome.app/Contents/MacOS/Google Chrome
        Start Chrome: '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome' --js-flags="--harmony_regexp_property"

Further reading  

JavaScript:

    “Unicode and JavaScript” (in “Speaking JavaScript”)
    Regular expressions: “New flag /u (unicode)” (in “Exploring ES6”)

The Unicode standard:

    Unicode Technical Report #23: The Unicode Character Property Model (Editors: Ken Whistler, Asmus Freytag)
    Unicode® Standard Annex #44: Unicode Character Database (Editors: Mark Davis, Laurențiu Iancu, Ken Whistler)
    Unicode Character Database: PropList.txt, PropertyAliases.txt, PropertyValueAliases.txt
    “Unicode character property” (Wikipedia)

ES2018: RegExp lookbehind assertions
[2017-05-16] dev, javascript, esnext, es2018, regexp
(Ad, please don’t block)
ads via Carbon
Limited time offer: Get 10 free Adobe Stock images.
ads via Carbon

The proposal “RegExp Lookbehind Assertions” by Gorkem Yakin, Nozomu Katō, Daniel Ehrenberg is part of ES2018. This blog post explains it.

A lookaround assertion is a construct inside a regular expression that specifies what the surroundings of the current location must look like, but has no other effect. It is also called a zero-width assertion.

The only lookaround assertion currently supported by JavaScript is the lookahead assertion, which matches what follows the current location. This blog post describes a proposal for a lookbehind assertion, which matches what precedes the current location.
Lookahead assertions  

A lookahead assertion inside a regular expression means: whatever comes next must match the assertion, but nothing else happens. That is, nothing is captured and the assertion doesn’t contribute to the overall matched string.

Take, for example, the following regular expression

const RE_AS_BS = /aa(?=bb)/;

It matches the string 'aabb', but the overall matched string does not include the b’s:

const match1 = RE_AS_BS.exec('aabb');
console.log(match1[0]); // 'aa'

Furthermore, it does not match a string that doesn’t have two b’s:

const match2 = RE_AS_BS.exec('aab');
console.log(match2); // null

A negative lookahead assertion means that what comes next must not match the assertion. For example:

> const RE_AS_NO_BS = /aa(?!bb)/;
> RE_AS_NO_BS.test('aabb')
false
> RE_AS_NO_BS.test('aab')
true
> RE_AS_NO_BS.test('aac')
true

Lookbehind assertions  

Lookbehind assertions work like lookahead assertions, but in the opposite direction.
Positive lookbehind assertions  

For a positive lookbehind assertion, the text preceding the current location must match the assertion (but nothing else happens).

const RE_DOLLAR_PREFIX = /(?<=\$)foo/g;
'$foo %foo foo'.replace(RE_DOLLAR_PREFIX, 'bar');
    // '$bar %foo foo'

As you can see, 'foo' is only replaced if it is preceded by a dollar sign. You can also see that the dollar sign is not part of the total match, because the latter is completely replaced by 'bar'.

Achieving the same result without a lookbehind assertion is less elegant:

const RE_DOLLAR_PREFIX = /(\$)foo/g;
'$foo %foo foo'.replace(RE_DOLLAR_PREFIX, '$1bar');
    // '$bar %foo foo'

And this approach doesn’t work if the prefix should be part of the previous match:

> 'a1ba2ba3b'.match(/(?<=b)a.b/g)
[ 'a2b', 'a3b' ]

Negative lookbehind assertions  

A negative lookbehind assertion only matches if the current location is not preceded by the assertion, but has no other effect. For example:

const RE_NO_DOLLAR_PREFIX = /(?<!\$)foo/g;
'$foo %foo foo'.replace(RE_NO_DOLLAR_PREFIX, 'bar');
    // '$foo %bar bar'

There is no simple (general) way to achieve the same result without a lookbehind assertion.
Conclusions  

Lookahead assertions make most sense at the end of regular expressions. Lookbehind assertions make most sense at the beginning of regular expressions.

The use cases for lookaround assertions are:

    replace()
    match() (especially if the regular expression has the flag /g)
    split() (note the space at the beginning of ' b,c'):

    > 'a, b,c'.split(/,(?= )/)
    [ 'a', ' b,c' ]

Other than those use cases, you can just as well make the assertion a real part of the regular expression.
Further reading  

    V8 JavaScript Engine: RegExp lookbehind assertions
    Section “Manually Implementing Lookbehind” in “Speaking JavaScript”

ES2018: s (dotAll) flag for regular expressions
[2017-07-20] dev, javascript, esnext, es2018, regexp
(Ad, please don’t block)
ads via Carbon
Students and Teachers, save up to 60% on Adobe Creative Cloud.
ads via Carbon

The proposal “s (dotAll) flag for regular expressions” by Mathias Bynens is at stage 4. This blog post explains how it works.
Overview  

Currently, the dot (.) in regular expressions doesn’t match line terminator characters:

> /^[^]$/.test('\n')
true

The proposal specifies the regular expression flag /s that changes that:

> /^.$/s.test('\n')
true

Limitations of the dot (.) in regular expressions  

The dot (.) in regular expressions has two limitations.

First, it doesn’t match astral (non-BMP) characters such as emoji:

> /^.$/.test('😀')
false

This can be fixed via the /u (unicode) flag:

> /^.$/u.test('😀')
true

Second, the dot does not match line terminator characters:

> /^.$/.test('\n')
false

That can currently only be fixed by replacing the dot with work-arounds such as [^] (“all characters except no character”) or [\s\S] (“either whitespace nor not whitespace”).

> /^[^]$/.test('\n')
true
> /^[\s\S]$/.test('\n')
true

Line terminators recognized by ECMAScript  

Line termators in ECMAScript affect:

    The dot, in all regular expressions that don’t have the flag /s.
    The anchors ^ and $ if the flag /m (multiline) is used.

The following for characters are considered line terminators by ECMAScript:

    U+000A LINE FEED (LF) (\n)
    U+000D CARRIAGE RETURN (CR) (\r)
    U+2028 LINE SEPARATOR
    U+2029 PARAGRAPH SEPARATOR

There are additionally some newline-ish characters that are not considered line terminators by ECMAScript:

    U+000B VERTICAL TAB (\v)
    U+000C FORM FEED (\f)
    U+0085 NEXT LINE

Those three characters are matched by the dot without a flag:

> /^...$/.test('\v\f\u{0085}')
true

The proposal  

The proposal introduces the regular expression flag /s (short for “singleline”), which leads to the dot matching line terminators:

> /^.$/s.test('\n')
true

The long name of /s is dotAll:

> /./s.dotAll
true
> /./s.flags
's'
> new RegExp('.', 's').dotAll
true
> /./.dotAll
false

dotAll vs. multiline  

    dotAll only affects the dot.
    multiline only affects ^ and $.

FAQ  
Why is the flag named /s?  

dotAll is a good description of what the flag does, so, arguably, /a or /d would have been better names. However, /s is already an established name (Perl, Python, Java, C#, ...).

ES2018: Promise.prototype.finally()
[2017-07-26] dev, javascript, esnext, es2018, promises
(Ad, please don’t block)
ads via Carbon
Limited time offer: Get 10 free Adobe Stock images.
ads via Carbon

The proposal “Promise.prototype.finally” by Jordan Harband is at stage 4. This blog post explains it.
How does it work?  

.finally() works as follows:

```JavaScript
promise
.then(result => {···})
.catch(error => {···})
.finally(() => {···});
```

finally’s callback is always executed. Compare:

    then’s callback is only executed if promise is fulfilled.
    catch’s callback is only executed if promise is rejected. Or if then’s callback throws an exception or returns a rejected Promise.

In other words: Take the following piece of code.

```JavaScript
promise
.finally(() => {
    «statements»
});
```

This piece of code is equivalent to:

```JavaScript
promise
.then(
    result => {
        «statements»
        return result;
    },
    error => {
        «statements»
        throw error;
    }
);
```

Use case  

The most common use case is similar to the most common use case of the synchronous finally clause: cleaning up after you are done with a resource. That should always happen, regardless of whether everything went smoothly or there was an error.

For example:

```JavaScript
let connection;
db.open()
.then(conn => {
    connection = conn;
    return connection.select({ name: 'Jane' });
})
.then(result => {
    // Process result
    // Use `connection` to make more queries
})
···
.catch(error => {
    // handle errors
})
.finally(() => {
    connection.close();
});
```

.finally() is similar to finally {} in synchronous code  

In synchronous code, the try statement has three parts: The try clause, the catch clause and the finally clause.

In Promises:

    The try clause very loosely corresponds to invoking a Promise-based function or calling .then().
    The catch clause corresponds to the .catch() method of Promises.
    The finally clause corresponds to the new Promise method .finally() introduced by the proposal.

However, where finally {} can return and throw, returning has no effect inside the callback .finally(), only throwing. That’s because the method can’t distinguish between the callback returning explicitly and it finishing without doing so.
Availability  

    The npm package promise.prototype.finally is a polyfill for .finally().
    V8 5.8+ (e.g. in Node.js 8.1.4+): available behind the flag --harmony-promise-finally (details).

Further reading  

    “Promises for asynchronous programming” in “Exploring ES6”

ES2018: Template Literal Revision
[2016-09-01] dev, javascript, esnext, es2018, template literals
(Ad, please don’t block)
ads via Carbon
Adobe Creative Cloud for Teams starting at $29.99 per month.
ads via Carbon

The ECMAScript proposal “Template Literal Revision” by Tim Disney reached stage 4 and will be part of ECMAScript 2018. It proposes to give the innards of tagged template literals more syntactic freedom.
Tag functions and escape sequences  

With tagged template literals, you can make a function call by mentioning a function before a template literal:

> String.raw`\u{4B}`
'\\u{4B}'

String.raw is a so-called tag function. Tag functions receive two versions of the fixed string pieces (template strings) in a template literal:

    Cooked: escape sequences are interpreted. `\u{4B}` becomes 'K'.
    Raw: escape sequences are normal text. `\u{4B}` becomes '\\u{4B}'.

The following tag function illustrates how that works:

```JavaScript
function tagFunc(tmplObj, substs) {
    return {
        Cooked: tmplObj,
        Raw: tmplObj.raw,
    };
}
```

Using the tag function:

> tagFunc`\u{4B}`;
{ Cooked: [ 'K' ], Raw: [ '\\u{4B}' ] }

For more information on tag functions, consult Sect. “Implementing tag functions” in “Exploring ES6”.
Problem: some text is illegal after backslashes  

The problem is that even with the raw version, you don’t have total freedom within template literals in ES2016. After a backslash, some sequences of characters are not legal anymore:

    \u starts a Unicode escape, which must look like \u{1F4A4} or \u004B.
    \x starts a hex escape, which must look like \x4B.
    \ plus digit starts an octal escape (such as \141). Octal escapes are forbidden in template literals and strict mode string literals.

That prevents tagged template literals such as:

latex`\unicode`
windowsPath`C:\uuu\xxx\111`

Solution  

The solution is drop all syntactic restrictions related to escape sequences. Then illegal escape sequences simply show up verbatim in the raw representation. But what about the cooked representation? Every template string with an illegal escape sequence is an undefined element in the cooked Array:

> tagFunc`\uu ${1} \xx`
{ Cooked: [ undefined, undefined ], Raw: [ '\\uu ', ' \\xx' ] }

