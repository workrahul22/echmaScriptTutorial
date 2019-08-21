ES proposal: Shared memory and atomics
[2017-01-26] dev, javascript, esnext, es proposal, concurrency
(Ad, please don’t block)
ads via Carbon
Limited time offer: Get 10 free Adobe Stock images.
ads via Carbon

The ECMAScript proposal “Shared memory and atomics” by Lars T. Hansen has reached stage 4 this week and will be part of ECMAScript 2017. It introduces a new constructor SharedArrayBuffer and a namespace object Atomics with helper functions. This blog post explains the details.

    Update 2017-02-24: Complete rewrite of Sect. 4, “Atomics: safely accessing shared data”.

Parallelism vs. concurrency  

Before we begin, let’s clarify two terms that are similar, yet distinct: “parallelism” and “concurrency”. Many definitions for them exist; I’m using them as follows:

    Parallelism (parallel vs. serial): execute multiple tasks simultaneously
    Concurrency (concurrent vs. sequential): execute several tasks during overlapping periods of time (and not one after another).

Both are closely related, but not the same:

    Parallelism without concurrency: single instruction, multiple data (SIMD). Multiple computations happen in parallel, but only a single task (instruction) is executed at any given moment.
    Concurrency without parallelism: multitasking via time-sharing on a single-core CPU.

However, it is difficult to use these terms precisely, which is why interchanging them is usually not a problem.
Models of parallelism  

Two models of parallelism are:

    Data parallelism: The same piece of code is executed several times in parallel. The instances operate on different elements of the same dataset. For example: MapReduce is a data-parallel programming model.

    Task parallelism: Different pieces of code are executed in parallel. Examples: web workers and the Unix model of spawning processes.

A history of JS parallelism  

    JavaScript started as being executed in a single thread. Some tasks could be performed asynchronously: browsers usually ran those tasks in separate threads and later fed their results back into the single thread, via callbacks.

    Web workers brought task parallelism to JavaScript: They are relatively heavweight processes. Each worker has its own global environment. By default, nothing is shared. Communication between workers (or between workers and the main thread) evolved:
        At first, you could only send and receive strings.
        Then, structured cloning was introduced: copies of data could be sent and received. Structured cloning works for most data (JSON data, Typed Arrays, regular expressions, Blob objects, ImageData objects, etc.). It can even handle cyclic references between objects correctly. However, error objects, function objects and DOM nodes cannot be cloned.
        Transferables move data between workers: the sending party loses access as the receiving party gains access to data.

    Computing on GPUs (which tend to do data parallelism well) via WebGL: It’s a bit of a hack and works as follows.
        Input: your data, converted into an image (pixel by pixel).
        Processing: OpenGL pixel shaders can perform arbitrary computations on GPUs. Your pixel shader transforms the input image.
        Output: again an image that you can convert back to your kind of data.

    SIMD (low-level data parallelism): is supported via the ECMAScript proposal SIMD.js. It allows you to perform operations (such as addition and square root) on several integers or floats at the same time.

    PJS (codenamed River Trail): the plan of this ultimately abandoned project was to bring high-level data parallelism (think map-reduce via pure functions) to JavaScript. However, there was not enough interest from developers and engine implementers. Without implementations, one could not experiment with this API, because it can’t be polyfilled. On 2015-01-05, Lars T. Hansen announced that an experimental implementation was going to be removed from Firefox.

The next step: SharedArrayBuffer  

What’s next? For low-level parallelism, the direction is quite clear: support SIMD and GPUs as well as possible. However, for high-level parallelism, things are much less clear, especially after the failure of PJS.

What is needed is a way to try out many approaches, to find out how to best bring high-level parallelism to JavaScript. Following the principles of the extensible web manifesto, the proposal “shared memory and atomics” (a.k.a. “Shared Array Buffers”) does so by providing low-level primitives that can be used to implement higher-level constructs.
Shared Array Buffers  

Shared Array Buffers are a primitive building block for higher-level concurrency abstractions. They allow you to share the bytes of a SharedArrayBuffer object between multiple workers and the main thread (the buffer is shared, to access the bytes, wrap it in a Typed Array). This kind of sharing has two benefits:

    You can share data between workers more quickly.
    Coordination between workers becomes simpler and faster (compared to postMessage()).

Creating and sending a Shared Array Buffer  

// main.js

const worker = new Worker('worker.js');

// To be shared
const sharedBuffer = new SharedArrayBuffer( // (A)
    10 * Int32Array.BYTES_PER_ELEMENT); // 10 elements

// Share sharedBuffer with the worker
worker.postMessage({sharedBuffer}); // clone

// Local only
const sharedArray = new Int32Array(sharedBuffer); // (B)

You create a Shared Array Buffer the same way you create a normal Array Buffer: by invoking the constructor and specifying the size of the buffer in bytes (line A). What you share with workers is the buffer. For your own, local, use, you normally wrap Shared Array Buffers in Typed Arrays (line B).

Warning: Cloning a Shared Array Buffer is the correct way of sharing it, but some engines still implement an older version of the API and require you to transfer it:

worker.postMessage({sharedBuffer}, [sharedBuffer]); // transfer (deprecated)

In the final version of the API, transferring a Shared Array Buffer means that you lose access to it.
Receiving a Shared Array Buffer  

The implementation of the worker looks as follows.

// worker.js

self.addEventListener('message', function (event) {
    const {sharedBuffer} = event.data;
    const sharedArray = new Int32Array(sharedBuffer); // (A)

    // ···
});

We first extract the Shared Array Buffer that was sent to us and then wrap it in a Typed Array (line A), so that we can use it locally.
Atomics: safely accessing shared data  
Problem: Optimizations make code unpredictable across workers  

In single threads, compilers can make optimizations that break multi-threaded code.

Take, for example the following code:

while (sharedArray[0] === 123) ;

In a single thread, the value of sharedArray[0] never changes while the loop runs (if sharedArray is an Array or Typed Array that wasn’t patched in some manner). Therefore, the code can be optimized as follows:

const tmp = sharedArray[0];
while (tmp === 123) ;

However, in a multi-threaded setting, this optimization prevents us from using this pattern to wait for changes made in another thread.

Another example is the following code:

// main.js
sharedArray[1] = 11;
sharedArray[2] = 22;

In a single thread, you can rearrange these write operations, because nothing is read in-between. For multiple threads, you get into trouble whenever you expect the writes to be done in a specific order:

// worker.js
while (sharedArray[2] !== 22) ;
console.log(sharedArray[1]); // 0 or 11

These kinds of optimizations make it virtually impossible to synchronize the activity of multiple workers operating on the same Shared Array Buffer.
Solution: atomics  

The proposal provides the global variable Atomics whose methods have three main use cases.
Use case: synchronization  

Atomics methods can be used to synchronize with other workers. For example, the following two operations let you read and write data and are never rearranged by compilers:

    Atomics.load(ta : TypedArray<T>, index) : T
    Atomics.store(ta : TypedArray<T>, index, value : T) : T

The idea is to use normal operations to read and write most data, while Atomics operations (load, store and others) ensure that the reading and writing is done safely. Often, you’ll use custom synchronization mechanisms, such as locks, whose implementations are based on Atomics.

This is a very simple example that always works, thanks to Atomics (I’ve omitted setting up sharedArray):

// main.js
console.log('notifying...');
Atomics.store(sharedArray, 0, 123);

// worker.js
while (Atomics.load(sharedArray, 0) !== 123) ;
console.log('notified');

Use case: waiting to be notified  

Using a while loop to wait for a notification is not very efficient, which is why Atomics has operations that help:

    Atomics.wait(ta: Int32Array, index, value, timeout)
    waits for a notification at ta[index], but only if ta[index] is value.

    Atomics.wake(ta : Int32Array, index, count)
    wakes up count workers that are waiting at ta[index].

Use case: atomic operations  

Several Atomics operations perform arithmetic and can’t be interrupted while doing so, which helps with synchronization. For example:

    Atomics.add(ta : TypedArray<T>, index, value) : T

Roughly, this operation performs:

ta[index] += value;

Problem: torn values  

Another problematic effect with shared memory is torn values (garbage): when reading, you may see an intermediate value – neither the value before a new value was written to memory nor the new value.

Sect “Tear-Free Reads” in the spec states that there is no tear if and only if:

    Both reading and writing happens via Typed Arrays (not DataViews).

    Both Typed Arrays are aligned with their Shared Array Buffers:

    sharedArray.byteOffset % sharedArray.BYTES_PER_ELEMENT === 0

    Both Typed Arrays have the same number of bytes per element.

In other words, torn values are an issue whenever the same Shared Array Buffer is accessed via:

    One or more DateViews
    One or more unaligned Typed Arrays
    Typed Arrays with different element sizes

To avoid torn values in these cases, use Atomics or synchronize.
Shared Array Buffers in use  
Shared Array Buffers and the run-to-completion semantics of JavaScript  

JavaScript has so-called run-to-completion semantics: every function can rely on not being interrupted by another thread until it is finished. Functions become transactions and can perform complete algorithms without anyone seeing the data they operate on in an intermediate state.

Shared Array Buffers break run to completion (RTC): data a function is working on can be changed by another thread during the runtime of the function. However, code has complete control over whether or not this violation of RTC happens: if it doesn’t use Shared Array Buffers, it is safe.

This is loosely similar to how async functions violate RTC. There, you opt into a blocking operation via the keyword await.
Shared Array Buffers and asm.js and WebAssembly  

Shared Array Buffers enable emscripten to compile pthreads to asm.js. Quoting an emscripten documentation page:

    [Shared Array Buffers allow] Emscripten applications to share the main memory heap between web workers. This along with primitives for low level atomics and futex support enables Emscripten to implement support for the Pthreads (POSIX threads) API.

That is, you can compile multithreaded C and C++ code to asm.js.

Discussion on how to best bring multi-threading to WebAssembly is ongoing. Given that web workers are relatively heavyweight, it is possible that WebAssembly will introduce lightweight threads. You can also see that threads are on the roadmap for WebAssembly’s future.
Sharing data other than integers  

At the moment, only Arrays of integers (up to 32 bits long) can be shared. That means that the only way of sharing other kinds of data is by encoding them as integers. Tools that may help include:

    TextEncoder and TextDecoder: The former converts strings to instances of Uint8Array. The latter does the opposite.

    stringview.js: a library that handles strings as arrays of characters. Uses Array Buffers.

    FlatJS: enhances JavaScript with ways of storing complex data structures (structs, classes and arrays) in flat memory (ArrayBuffer and SharedArrayBuffer). JavaScript+FlatJS is compiled to plain JavaScript. JavaScript dialects (TypeScript etc.) are supported.

    TurboScript: is a JavaScript dialect for fast parallel programming. It compiles to asm.js and WebAssembly.

Eventually, there will probably be additional – higher-level – mechanisms for sharing data. And experiments will continue to figure out what these mechanisms should look like.
How much faster is code that uses Shared Array Buffers?  

Lars T. Hansen has written two implementations of the Mandelbrot algorithm (as documented in his article “A Taste of JavaScript’s New Parallel Primitives” where you can try them out online): A serial version and a parallel version that uses multiple web workers. For up to 4 web workers (and therefore processor cores), speed-up improves almost linearly, from 6.9 frames per seconds (1 web worker) to 25.4 frames per seconds (4 web workers). More web workers bring additional performance improvements, but more modest ones.

Hansen notes that the speed-ups are impressive, but going parallel comes at the cost of the code being more complex.
Example  

Let’s look at a more comprehensive example. Its code is available on GitHub, in the repository shared-array-buffer-demo. And you can run it online.
Using a shared lock  

In the main thread, we set up shared memory so that it encodes a closed lock and send it to a worker (line A). Once the user clicks, we open the lock (line B).

// main.js

// Set up the shared memory
const sharedBuffer = new SharedArrayBuffer(
    1 * Int32Array.BYTES_PER_ELEMENT);
const sharedArray = new Int32Array(sharedBuffer);

// Set up the lock
Lock.initialize(sharedArray, 0);
const lock = new Lock(sharedArray, 0);
lock.lock(); // writes to sharedBuffer

worker.postMessage({sharedBuffer}); // (A)

document.getElementById('unlock').addEventListener(
    'click', event => {
        event.preventDefault();
        lock.unlock(); // (B)
    });

In the worker, we set up a local version of the lock (whose state is shared with the main thread via a Shared Array Buffer). In line B, we wait until the lock is unlocked. In lines A and C, we send text to the main thread, which displays it on the page for us (how it does that is not shown in the previous code fragment). That is, we are using self.postMessage() much like console.log() in these two lines.

// worker.js

self.addEventListener('message', function (event) {
    const {sharedBuffer} = event.data;
    const lock = new Lock(new Int32Array(sharedBuffer), 0);

    self.postMessage('Waiting for lock...'); // (A)
    lock.lock(); // (B) blocks!
    self.postMessage('Unlocked'); // (C)
});

It is noteworthy that waiting for the lock in line B stops the complete worker. That is real blocking, which hasn’t existed in JavaScript until now (await in async functions is an approximation).
Implementing a shared lock  

Next, we’ll look at an ES6-ified version of a Lock implementation by Lars T. Hansen that is based on SharedArrayBuffer.

In this section, we’ll need (among others) the following Atomics function:

    Atomics.compareExchange(ta : TypedArray<T>, index, expectedValue, replacementValue) : T
    If the current element of ta at index is expectedValue, replace it with replacementValue. Return the previous (or unchanged) element at index.

The implementation starts with a few constants and the constructor:

const UNLOCKED = 0;
const LOCKED_NO_WAITERS = 1;
const LOCKED_POSSIBLE_WAITERS = 2;

// Number of shared Int32 locations needed by the lock.
const NUMINTS = 1;

class Lock {

    /**
     * @param iab an Int32Array wrapping a SharedArrayBuffer
     * @param ibase an index inside iab, leaving enough room for NUMINTS
     */
    constructor(iab, ibase) {
        // OMITTED: check parameters
        this.iab = iab;
        this.ibase = ibase;
    }

The constructor mainly stores its parameters in instance properties.

The method for locking looks as follows.

/**
 * Acquire the lock, or block until we can. Locking is not recursive:
 * you must not hold the lock when calling this.
 */
lock() {
    const iab = this.iab;
    const stateIdx = this.ibase;
    var c;
    if ((c = Atomics.compareExchange(iab, stateIdx, // (A)
    UNLOCKED, LOCKED_NO_WAITERS)) !== UNLOCKED) {
        do {
            if (c === LOCKED_POSSIBLE_WAITERS // (B)
            || Atomics.compareExchange(iab, stateIdx,
            LOCKED_NO_WAITERS, LOCKED_POSSIBLE_WAITERS) !== UNLOCKED) {
                Atomics.wait(iab, stateIdx, // (C)
                    LOCKED_POSSIBLE_WAITERS, Number.POSITIVE_INFINITY);
            }
        } while ((c = Atomics.compareExchange(iab, stateIdx,
        UNLOCKED, LOCKED_POSSIBLE_WAITERS)) !== UNLOCKED);
    }
}

In line A, we change the lock to LOCKED_NO_WAITERS if its current value is UNLOCKED. We only enter the then-block if the lock is already locked (in which case compareExchange() did not change anything).

In line B (inside a do-while loop), we check if the lock is locked with waiters or not unlocked. Given that we are about to wait, the compareExchange() also switches to LOCKED_POSSIBLE_WAITERS if the current value is LOCKED_NO_WAITERS.

In line C, we wait if the lock value is LOCKED_POSSIBLE_WAITERS. The last parameter, Number.POSITIVE_INFINITY, means that waiting never times out.

After waking up, we continue the loop if we are not unlocked. compareExchange() also switches to LOCKED_POSSIBLE_WAITERS if the lock is UNLOCKED. We use LOCKED_POSSIBLE_WAITERS and not LOCKED_NO_WAITERS, because we need to restore this value after unlock() temporarily set it to UNLOCKED and woke us up.

The method for unlocking looks as follows.


    /**
     * Unlock a lock that is held.  Anyone can unlock a lock that
     * is held; nobody can unlock a lock that is not held.
     */
    unlock() {
        const iab = this.iab;
        const stateIdx = this.ibase;
        var v0 = Atomics.sub(iab, stateIdx, 1); // A

        // Wake up a waiter if there are any
        if (v0 !== LOCKED_NO_WAITERS) {
            Atomics.store(iab, stateIdx, UNLOCKED);
            Atomics.wake(iab, stateIdx, 1);
        }
    }

    // ···
}

In line A, v0 gets the value that iab[stateIdx] had before 1 was subtracted from it. The subtraction means that we go (e.g.) from LOCKED_NO_WAITERS to UNLOCKED and from LOCKED_POSSIBLE_WAITERS to LOCKED.

If the value was previously LOCKED_NO_WAITERS then it is now UNLOCKED and everything is fine (there is no one to wake up).

Otherwise, the value was either LOCKED_POSSIBLE_WAITERS or UNLOCKED. In the former case, we are now unlocked and must wake up someone (who will usually lock again). In the latter case, we must fix the illegal value created by subtraction and the wake() simply does nothing.
Conclusion for the example  

This gives you a rough idea how locks based on SharedArrayBuffer work. Keep in mind that multithreaded code is notoriously difficult to write, because things can change at any time. Case in point: lock.js is based on a paper documenting a futex implementation for the Linux kernel. And the title of that paper is “Futexes are tricky” (PDF).

If you want to go deeper into parallel programming with Shared Array Buffers, take a look at synchronic.js and the document it is based on (PDF).
The API for shared memory and atomics  
SharedArrayBuffer  

Constructor:

    new SharedArrayBuffer(length)
    Create a buffer for length bytes.

Static property:

    get SharedArrayBuffer[Symbol.species]
    Returns this by default. Override to control what slice() returns.

Instance properties:

    get SharedArrayBuffer.prototype.byteLength()
    Returns the length of the buffer in bytes.

    SharedArrayBuffer.prototype.slice(start, end)
    Create a new instance of this.constructor[Symbol.species] and fill it with the bytes at the indices from (including) start to (excluding) end.

Atomics  

The main operand of Atomics functions must be an instance of Int8Array, Uint8Array, Int16Array, Uint16Array, Int32Array or Uint32Array. It must wrap a SharedArrayBuffer.

All functions perform their operations atomically. The ordering of store operations is fixed and can’t be reordered by compilers or CPUs.
Loading and storing  

    Atomics.load(ta : TypedArray<T>, index) : T
    Read and return the element of ta at index.

    Atomics.store(ta : TypedArray<T>, index, value : T) : T
    Write value to ta at index and return value.

    Atomics.exchange(ta : TypedArray<T>, index, value : T) : T
    Set the element of ta at index to value and return the previous value at that index.

    Atomics.compareExchange(ta : TypedArray<T>, index, expectedValue, replacementValue) : T
    If the current element of ta at index is expectedValue, replace it with replacementValue. Return the previous (or unchanged) element at index.

Simple modification of Typed Array elements  

Each of the following functions changes a Typed Array element at a given index: It applies an operator to the element and a parameter and writes the result back to the element. It returns the original value of the element.

    Atomics.add(ta : TypedArray<T>, index, value) : T
    Perform ta[index] += value and return the original value of ta[index].

    Atomics.sub(ta : TypedArray<T>, index, value) : T
    Perform ta[index] -= value and return the original value of ta[index].

    Atomics.and(ta : TypedArray<T>, index, value) : T
    Perform ta[index] &= value and return the original value of ta[index].

    Atomics.or(ta : TypedArray<T>, index, value) : T
    Perform ta[index] |= value and return the original value of ta[index].

    Atomics.xor(ta : TypedArray<T>, index, value) : T
    Perform ta[index] ^= value and return the original value of ta[index].

Waiting and waking  

Waiting and waking requires the parameter ta to be an instance of Int32Array.

    Atomics.wait(ta: Int32Array, index, value, timeout=Number.POSITIVE_INFINITY) : ('not-equal' | 'ok' | 'timed-out')
    If the current value at ta[index] is not value, return 'not-equal'. Otherwise go to sleep until we are woken up via Atomics.wake() or until sleeping times out. In the former case, return 'ok'. In the latter case, return 'timed-out'. timeout is specified in milliseconds. Mnemonic for what this function does: “wait if ta[index] is value”.

    Atomics.wake(ta : Int32Array, index, count)
    Wake up count workers that are waiting at ta[index].

Miscellaneous  

    Atomics.isLockFree(size)
    This function lets you ask the JavaScript engine if operands with the given size (in bytes) can be manipulated without locking. That can inform algorithms whether they want to rely on built-in primitives (compareExchange() etc.) or use their own locking. Atomics.isLockFree(4) always returns true, because that’s what all currently relevant supports.

FAQ  
What browsers support Shared Array Buffers?  

At the moment, I’m aware of:

    Firefox (50.1.0+): go to about:config and set javascript.options.shared_memory to true
    Safari Technology Preview (Release 21+): enabled by default.
    Chrome Canary (58.0+): There are two ways to switch it on.
        Via chrome://flags/ (“Experimental enabled SharedArrayBuffer support in JavaScript”)
        --js-flags=--harmony-sharedarraybuffer --enable-blink-feature=SharedArrayBuffer

Further reading  

More information on Shared Array Buffers and supporting technologies:

    “Shared memory – a brief tutorial” by Lars T. Hansen

    “A Taste of JavaScript’s New Parallel Primitives” by Lars T. Hansen [a good intro to Shared Array Buffers]

    “SharedArrayBuffer and Atomics Stage 2.95 to Stage 3” (PDF), slides by Shu-yu Guo and Lars T. Hansen (2016-11-30) [slides accompanying the ES proposal]

    “The Basics of Web Workers” by Eric Bidelman [an introduction to web workers]

Other JavaScript technologies related to parallelism:

    “The Path to Parallel JavaScript” by Dave Herman [a general overview of where JavaScript is heading after the abandonment of PJS]
    “Write massively-parallel GPU code for the browser with WebGL” by Steve Sanderson [fascinating talk that explains how to get WebGL to do computations for you on the GPU]

Background on parallelism:

    “Concurrency is not parallelism” by Rob Pike [Pike uses the terms “concurrency” and “parallelism” slightly differently than I do in this blog post, providing an interesting complementary view]

Acknowledgement: I’m very grateful to Lars T. Hansen for reviewing this blog post and for answering my SharedArrayBuffer-related questions.


ES proposal: Object.entries() and Object.values()
[2015-11-20] dev, javascript, esnext, es proposal
(Ad, please don’t block)
ads via Carbon
Limited time offer: Get 10 free Adobe Stock images.
ads via Carbon

The following ECMAScript proposal is at stage 4: “Object.values/Object.entries” by Jordan Harband. This blog post explains it.
Object.entries()  

This method has the following signature:

Object.entries(value : any) : Array<[string,any]>

If a JavaScript data structure has keys and values then an entry is a key-value pair, encoded as a 2-element Array. Object.entries(x) coerces x to an Object and returns the entries of its enumerable own string-keyed properties, in an Array:

> Object.entries({ one: 1, two: 2 })
[ [ 'one', 1 ], [ 'two', 2 ] ]

Properties, whose keys are symbols, are ignored:

> Object.entries({ [Symbol()]: 123, foo: 'abc' });
[ [ 'foo', 'abc' ] ]

Object.entries() finally gives us a way to iterate over the properties of an object (read here why objects aren’t iterable by default):

let obj = { one: 1, two: 2 };
for (let [k,v] of Object.entries(obj)) {
    console.log(`${JSON.stringify(k)}: ${JSON.stringify(v)}`);
}
// Output:
// "one": 1
// "two": 2

Setting up Maps via Object.entries()  

Object.entries() also lets you set up a Map via an object. This is more concise than using an Array of 2-element Arrays, but keys can only be strings.

let map = new Map(Object.entries({
    one: 1,
    two: 2,
}));
console.log(JSON.stringify([...map]));
    // [["one",1],["two",2]]

FAQ: Object.entries()  

    Why is the return value of Object.entries() an Array and not an iterator?
    The relevant precedent in this case is Object.keys(), not, e.g., Map.prototype.entries().

    Why does Object.entries() only return the enumerable own string-keyed properties?
    Again, this is done to be consistent with Object.keys(). That method also ignores properties whose keys are symbols. Eventually, there may be a method Reflect.ownEntries() that returns all own properties.

Object.values()  

Object.values() has the following signature:

Object.values(value : any) : Array<any>

It works much like Object.entries(), but, as its name suggests, it only returns the values of the own enumerable string-keyed properties:

> Object.values({ one: 1, two: 2 })
[ 1, 2 ]

ES proposal: string padding
[2015-11-26] dev, javascript, esnext, es proposal
(Ad, please don’t block)
ads via Carbon
Limited time offer: Get 10 free Adobe Stock images.
ads via Carbon

The ECMAScript proposal “String padding” by Jordan Harband & Rick Waldron is part of ECMAScript 2017. This blog post explains it.
Padding strings  

Use cases for padding strings include:

    Displaying tabular data in a monospaced font.
    Adding a count or an ID to a file name or a URL: 'file 001.txt'
    Aligning console output: 'Test 001: ✓'
    Printing hexadecimal or binary numbers that have a fixed number of digits: '0x00FF'

String.prototype.padStart(maxLength, fillString=' ')  

This method (possibly repeatedly) prefixes the receiver with fillString, until its length is maxLength:

> 'x'.padStart(5, 'ab')
'ababx'

If necessary, a fragment of fillString is used so that the result’s length is exactly maxLength:

> 'x'.padStart(4, 'ab')
'abax'

If the receiver is as long as, or longer than, maxLength, it is returned unchanged:

> 'abcd'.padStart(2, '#')
'abcd'

If maxLength and fillString.length are the same, fillString becomes a mask into which the receiver is inserted, at the end:

> 'abc'.padStart(10, '0123456789')
'0123456abc'

If you omit fillString, a string with a single space in it is used (' '):

> 'x'.padStart(3)
'  x'

A simple implementation of padStart()  

The following implementation gives you a rough idea of how padStart() works, but isn’t completely spec-compliant (for a few edge cases).

String.prototype.padStart =
function (maxLength, fillString=' ') {
    let str = String(this);
    if (str.length >= maxLength) {
        return str;
    }

    fillString = String(fillString);
    if (fillString.length === 0) {
        fillString = ' ';
    }

    let fillLen = maxLength - str.length;
    let timesToRepeat = Math.ceil(fillLen / fillString.length);
    let truncatedStringFiller = fillString
        .repeat(timesToRepeat)
        .slice(0, fillLen);
    return truncatedStringFiller + str;
};

String.prototype.padEnd(maxLength, fillString=' ')  

padEnd() works similarly to padStart(), but instead of inserting the repeated fillString at the start, it inserts it at the end:

> 'x'.padEnd(5, 'ab')
'xabab'
> 'x'.padEnd(4, 'ab')
'xaba'
> 'abcd'.padEnd(2, '#')
'abcd'
> 'abc'.padEnd(10, '0123456789')
'abc0123456'
> 'x'.padEnd(3)
'x  '

Only the last line of an implementation of padEnd() is different, compared to the implementation of padStart():

return str + truncatedStringFiller;

FAQ: string padding  
Why aren’t the padding methods called padLeft and padRight?  

For bidirectional or right-to-left languages, the terms left and right don’t work well. Therefore, the naming of padStart and padEnd follows the existing names startsWith and endsWith.

ES proposal: Object.getOwnPropertyDescriptors()
[2016-02-04] dev, javascript, esnext, es proposal
(Ad, please don’t block)
ads via Carbon
Learn faster with Educative.io’s rich, text-based programming courses.
ads via Carbon

The ECMAScript proposal “Object.getOwnPropertyDescriptors()” by Jordan Harband and Andrea Giammarchi is part of ECMAScript 2017. This blog post explains it.
Overview  

Object.getOwnPropertyDescriptors(obj) accepts an object obj and returns an object result:

    For each own (non-inherited) property of obj, it adds a property to result whose key is the same and whose value is the former property’s descriptor.

Property descriptors describe the attributes of a property (its value, whether it is writable, etc.). For more information, consult Sect. “Property Attributes and Property Descriptors” in “Speaking JavaScript”.

This is an example of using Object.getOwnPropertyDescriptors():

const obj = {
    [Symbol('foo')]: 123,
    get bar() { return 'abc' },
};
console.log(Object.getOwnPropertyDescriptors(obj));

// Output:
// { [Symbol('foo')]:
//    { value: 123,
//      writable: true,
//      enumerable: true,
//      configurable: true },
//   bar:
//    { get: [Function: bar],
//      set: undefined,
//      enumerable: true,
//      configurable: true } }

This is how you would implement Object.getOwnPropertyDescriptors():

function getOwnPropertyDescriptors(obj) {
    const result = {};
    for (let key of Reflect.ownKeys(obj)) {
        result[key] = Object.getOwnPropertyDescriptor(obj, key);
    }
    return result;
}

Use cases for Object.getOwnPropertyDescriptors()  
Use case: copying properties into an object  

Since ES6, JavaScript already has a tool method for copying properties: Object.assign(). However, this method uses simple get and set operations to copy a property whose key is key:

const value = source[key]; // get
target[key] = value; // set

That means that it doesn’t properly copy properties with non-default attributes (getters, setters, non-writable properties, etc.). The following example illustrates this limitation. The object source has a getter whose key is foo:

const source = {
    set foo(value) {
        console.log(value);
    }
};
console.log(Object.getOwnPropertyDescriptor(source, 'foo'));
// { get: undefined,
//   set: [Function: foo],
//   enumerable: true,
//   configurable: true }

Using Object.assign() to copy property foo to object target fails:

const target1 = {};
Object.assign(target1, source);
console.log(Object.getOwnPropertyDescriptor(target1, 'foo'));
// { value: undefined,
//   writable: true,
//   enumerable: true,
//   configurable: true }

Fortunately, using Object.getOwnPropertyDescriptors() together with Object.defineProperties() works:

const target2 = {};
Object.defineProperties(target2, Object.getOwnPropertyDescriptors(source));
console.log(Object.getOwnPropertyDescriptor(target2, 'foo'));
// { get: undefined,
//   set: [Function: foo],
//   enumerable: true,
//   configurable: true }

Use case: cloning objects  

Shallow cloning is similar to copying properties, which is why Object.getOwnPropertyDescriptors() is a good choice here, too.

This time, we use Object.create() that has two parameters:

    The first parameter specifies the prototype of the object it returns.
    The optional second parameter is a property descriptor collection like the ones returned by Object.getOwnPropertyDescriptors().

const clone = Object.create(Object.getPrototypeOf(obj),
    Object.getOwnPropertyDescriptors(obj));

Use case: cross-platform object literals with arbitrary prototypes  

The syntactically nicest way of using an object literal to create an object with an arbitrary prototype prot is to use the special property __proto__:

const obj = {
    __proto__: prot,
    foo: 123,
};

Alas, that feature is only guaranteed to be there in browsers. The common work-around is Object.create() and assignment:

const obj = Object.create(prot);
obj.foo = 123;

But you can also use Object.getOwnPropertyDescriptors():

const obj = Object.create(
    prot,
    Object.getOwnPropertyDescriptors({
        foo: 123,
    })
);

Another alternative is Object.assign():

const obj = Object.assign(
    Object.create(prot),
    {
        foo: 123,
    }
);

Pitfall: copying methods that use super  

A method that uses super is firmly connected with its home object (the object it is stored in). There is currently no way to copy or move such a method to a different object.
Further reading  

JavaScript design process:

    An explanation of the TC39 process and its stages. This process governs the design and evolution of JavaScript.
    List of proposals that are currently at stage 3 or 4.

Object.getOwnPropertyDescriptors() and property descriptors:

    ECMAScript proposal “Object.getOwnPropertyDescriptors” by Jordan Harband and Andrea Giammarchi
    Sect. “Property Attributes and Property Descriptors” in “Speaking JavaScript”.

ES proposal: Trailing commas in function parameter lists and calls
[2015-11-27] dev, javascript, esnext, es proposal
(Ad, please don’t block)
ads via Carbon
Power-up the lifetime value of your app users with rewarded video ads
ads via Carbon

The following ECMAScript proposal is at stage 3: “Trailing commas in function parameter lists and calls” by Jeff Morrison. This blog post explains it.
Trailing commas in object literals and Array literals  

Trailing commas are ignored in object literals:

let obj = {
    first: 'Jane',
    last: 'Doe',
};

And they are also ignored in Array literals:

let arr = [
    'red',
    'green',
    'blue',
];
console.log(arr.length); // 3

Why is that useful? There are two benefits.

First, rearranging items is simpler, because you don’t have to add and remove commas if the last item changes its position.

Second, it helps version control systems with tracking what actually changed. For example, going from:

[
    'foo'
]

to:

[
    'foo',
    'bar'
]

leads to both the line with 'foo' and the line with 'bar' being marked as changed, even though the only real change is the latter line being added.
Proposal: allow trailing commas in parameter definitions and function calls  

Given the benefits of optional and ignored trailing commas, the proposed feature is to bring them to parameter definitions and function calls.

For example, the following function declaration causes a SyntaxError in ECMAScript 6, but would be legal with the proposal:

function foo(
    param1,
    param2,
) {}

Similarly, the proposal would make this invocation of foo() syntactically legal:

foo(
    'abc',
    'def',
);

ES proposal: Shared memory and atomics
[2017-01-26] dev, javascript, esnext, es proposal, concurrency
(Ad, please don’t block)

The ECMAScript proposal “Shared memory and atomics” by Lars T. Hansen has reached stage 4 this week and will be part of ECMAScript 2017. It introduces a new constructor SharedArrayBuffer and a namespace object Atomics with helper functions. This blog post explains the details.

    Update 2017-02-24: Complete rewrite of Sect. 4, “Atomics: safely accessing shared data”.

Parallelism vs. concurrency  

Before we begin, let’s clarify two terms that are similar, yet distinct: “parallelism” and “concurrency”. Many definitions for them exist; I’m using them as follows:

    Parallelism (parallel vs. serial): execute multiple tasks simultaneously
    Concurrency (concurrent vs. sequential): execute several tasks during overlapping periods of time (and not one after another).

Both are closely related, but not the same:

    Parallelism without concurrency: single instruction, multiple data (SIMD). Multiple computations happen in parallel, but only a single task (instruction) is executed at any given moment.
    Concurrency without parallelism: multitasking via time-sharing on a single-core CPU.

However, it is difficult to use these terms precisely, which is why interchanging them is usually not a problem.
Models of parallelism  

Two models of parallelism are:

    Data parallelism: The same piece of code is executed several times in parallel. The instances operate on different elements of the same dataset. For example: MapReduce is a data-parallel programming model.

    Task parallelism: Different pieces of code are executed in parallel. Examples: web workers and the Unix model of spawning processes.

A history of JS parallelism  

    JavaScript started as being executed in a single thread. Some tasks could be performed asynchronously: browsers usually ran those tasks in separate threads and later fed their results back into the single thread, via callbacks.

    Web workers brought task parallelism to JavaScript: They are relatively heavweight processes. Each worker has its own global environment. By default, nothing is shared. Communication between workers (or between workers and the main thread) evolved:
        At first, you could only send and receive strings.
        Then, structured cloning was introduced: copies of data could be sent and received. Structured cloning works for most data (JSON data, Typed Arrays, regular expressions, Blob objects, ImageData objects, etc.). It can even handle cyclic references between objects correctly. However, error objects, function objects and DOM nodes cannot be cloned.
        Transferables move data between workers: the sending party loses access as the receiving party gains access to data.

    Computing on GPUs (which tend to do data parallelism well) via WebGL: It’s a bit of a hack and works as follows.
        Input: your data, converted into an image (pixel by pixel).
        Processing: OpenGL pixel shaders can perform arbitrary computations on GPUs. Your pixel shader transforms the input image.
        Output: again an image that you can convert back to your kind of data.

    SIMD (low-level data parallelism): is supported via the ECMAScript proposal SIMD.js. It allows you to perform operations (such as addition and square root) on several integers or floats at the same time.

    PJS (codenamed River Trail): the plan of this ultimately abandoned project was to bring high-level data parallelism (think map-reduce via pure functions) to JavaScript. However, there was not enough interest from developers and engine implementers. Without implementations, one could not experiment with this API, because it can’t be polyfilled. On 2015-01-05, Lars T. Hansen announced that an experimental implementation was going to be removed from Firefox.

The next step: SharedArrayBuffer  

What’s next? For low-level parallelism, the direction is quite clear: support SIMD and GPUs as well as possible. However, for high-level parallelism, things are much less clear, especially after the failure of PJS.

What is needed is a way to try out many approaches, to find out how to best bring high-level parallelism to JavaScript. Following the principles of the extensible web manifesto, the proposal “shared memory and atomics” (a.k.a. “Shared Array Buffers”) does so by providing low-level primitives that can be used to implement higher-level constructs.
Shared Array Buffers  

Shared Array Buffers are a primitive building block for higher-level concurrency abstractions. They allow you to share the bytes of a SharedArrayBuffer object between multiple workers and the main thread (the buffer is shared, to access the bytes, wrap it in a Typed Array). This kind of sharing has two benefits:

    You can share data between workers more quickly.
    Coordination between workers becomes simpler and faster (compared to postMessage()).

Creating and sending a Shared Array Buffer  

// main.js

const worker = new Worker('worker.js');

// To be shared
const sharedBuffer = new SharedArrayBuffer( // (A)
    10 * Int32Array.BYTES_PER_ELEMENT); // 10 elements

// Share sharedBuffer with the worker
worker.postMessage({sharedBuffer}); // clone

// Local only
const sharedArray = new Int32Array(sharedBuffer); // (B)

You create a Shared Array Buffer the same way you create a normal Array Buffer: by invoking the constructor and specifying the size of the buffer in bytes (line A). What you share with workers is the buffer. For your own, local, use, you normally wrap Shared Array Buffers in Typed Arrays (line B).

Warning: Cloning a Shared Array Buffer is the correct way of sharing it, but some engines still implement an older version of the API and require you to transfer it:

worker.postMessage({sharedBuffer}, [sharedBuffer]); // transfer (deprecated)

In the final version of the API, transferring a Shared Array Buffer means that you lose access to it.
Receiving a Shared Array Buffer  

The implementation of the worker looks as follows.

// worker.js

self.addEventListener('message', function (event) {
    const {sharedBuffer} = event.data;
    const sharedArray = new Int32Array(sharedBuffer); // (A)

    // ···
});

We first extract the Shared Array Buffer that was sent to us and then wrap it in a Typed Array (line A), so that we can use it locally.
Atomics: safely accessing shared data  
Problem: Optimizations make code unpredictable across workers  

In single threads, compilers can make optimizations that break multi-threaded code.

Take, for example the following code:

while (sharedArray[0] === 123) ;

In a single thread, the value of sharedArray[0] never changes while the loop runs (if sharedArray is an Array or Typed Array that wasn’t patched in some manner). Therefore, the code can be optimized as follows:

const tmp = sharedArray[0];
while (tmp === 123) ;

However, in a multi-threaded setting, this optimization prevents us from using this pattern to wait for changes made in another thread.

Another example is the following code:

// main.js
sharedArray[1] = 11;
sharedArray[2] = 22;

In a single thread, you can rearrange these write operations, because nothing is read in-between. For multiple threads, you get into trouble whenever you expect the writes to be done in a specific order:

// worker.js
while (sharedArray[2] !== 22) ;
console.log(sharedArray[1]); // 0 or 11

These kinds of optimizations make it virtually impossible to synchronize the activity of multiple workers operating on the same Shared Array Buffer.
Solution: atomics  

The proposal provides the global variable Atomics whose methods have three main use cases.
Use case: synchronization  

Atomics methods can be used to synchronize with other workers. For example, the following two operations let you read and write data and are never rearranged by compilers:

    Atomics.load(ta : TypedArray<T>, index) : T
    Atomics.store(ta : TypedArray<T>, index, value : T) : T

The idea is to use normal operations to read and write most data, while Atomics operations (load, store and others) ensure that the reading and writing is done safely. Often, you’ll use custom synchronization mechanisms, such as locks, whose implementations are based on Atomics.

This is a very simple example that always works, thanks to Atomics (I’ve omitted setting up sharedArray):

// main.js
console.log('notifying...');
Atomics.store(sharedArray, 0, 123);

// worker.js
while (Atomics.load(sharedArray, 0) !== 123) ;
console.log('notified');

Use case: waiting to be notified  

Using a while loop to wait for a notification is not very efficient, which is why Atomics has operations that help:

    Atomics.wait(ta: Int32Array, index, value, timeout)
    waits for a notification at ta[index], but only if ta[index] is value.

    Atomics.wake(ta : Int32Array, index, count)
    wakes up count workers that are waiting at ta[index].

Use case: atomic operations  

Several Atomics operations perform arithmetic and can’t be interrupted while doing so, which helps with synchronization. For example:

    Atomics.add(ta : TypedArray<T>, index, value) : T

Roughly, this operation performs:

ta[index] += value;

Problem: torn values  

Another problematic effect with shared memory is torn values (garbage): when reading, you may see an intermediate value – neither the value before a new value was written to memory nor the new value.

Sect “Tear-Free Reads” in the spec states that there is no tear if and only if:

    Both reading and writing happens via Typed Arrays (not DataViews).

    Both Typed Arrays are aligned with their Shared Array Buffers:

    sharedArray.byteOffset % sharedArray.BYTES_PER_ELEMENT === 0

    Both Typed Arrays have the same number of bytes per element.

In other words, torn values are an issue whenever the same Shared Array Buffer is accessed via:

    One or more DateViews
    One or more unaligned Typed Arrays
    Typed Arrays with different element sizes

To avoid torn values in these cases, use Atomics or synchronize.
Shared Array Buffers in use  
Shared Array Buffers and the run-to-completion semantics of JavaScript  

JavaScript has so-called run-to-completion semantics: every function can rely on not being interrupted by another thread until it is finished. Functions become transactions and can perform complete algorithms without anyone seeing the data they operate on in an intermediate state.

Shared Array Buffers break run to completion (RTC): data a function is working on can be changed by another thread during the runtime of the function. However, code has complete control over whether or not this violation of RTC happens: if it doesn’t use Shared Array Buffers, it is safe.

This is loosely similar to how async functions violate RTC. There, you opt into a blocking operation via the keyword await.
Shared Array Buffers and asm.js and WebAssembly  

Shared Array Buffers enable emscripten to compile pthreads to asm.js. Quoting an emscripten documentation page:

    [Shared Array Buffers allow] Emscripten applications to share the main memory heap between web workers. This along with primitives for low level atomics and futex support enables Emscripten to implement support for the Pthreads (POSIX threads) API.

That is, you can compile multithreaded C and C++ code to asm.js.

Discussion on how to best bring multi-threading to WebAssembly is ongoing. Given that web workers are relatively heavyweight, it is possible that WebAssembly will introduce lightweight threads. You can also see that threads are on the roadmap for WebAssembly’s future.
Sharing data other than integers  

At the moment, only Arrays of integers (up to 32 bits long) can be shared. That means that the only way of sharing other kinds of data is by encoding them as integers. Tools that may help include:

    TextEncoder and TextDecoder: The former converts strings to instances of Uint8Array. The latter does the opposite.

    stringview.js: a library that handles strings as arrays of characters. Uses Array Buffers.

    FlatJS: enhances JavaScript with ways of storing complex data structures (structs, classes and arrays) in flat memory (ArrayBuffer and SharedArrayBuffer). JavaScript+FlatJS is compiled to plain JavaScript. JavaScript dialects (TypeScript etc.) are supported.

    TurboScript: is a JavaScript dialect for fast parallel programming. It compiles to asm.js and WebAssembly.

Eventually, there will probably be additional – higher-level – mechanisms for sharing data. And experiments will continue to figure out what these mechanisms should look like.
How much faster is code that uses Shared Array Buffers?  

Lars T. Hansen has written two implementations of the Mandelbrot algorithm (as documented in his article “A Taste of JavaScript’s New Parallel Primitives” where you can try them out online): A serial version and a parallel version that uses multiple web workers. For up to 4 web workers (and therefore processor cores), speed-up improves almost linearly, from 6.9 frames per seconds (1 web worker) to 25.4 frames per seconds (4 web workers). More web workers bring additional performance improvements, but more modest ones.

Hansen notes that the speed-ups are impressive, but going parallel comes at the cost of the code being more complex.
Example  

Let’s look at a more comprehensive example. Its code is available on GitHub, in the repository shared-array-buffer-demo. And you can run it online.
Using a shared lock  

In the main thread, we set up shared memory so that it encodes a closed lock and send it to a worker (line A). Once the user clicks, we open the lock (line B).

// main.js

// Set up the shared memory
const sharedBuffer = new SharedArrayBuffer(
    1 * Int32Array.BYTES_PER_ELEMENT);
const sharedArray = new Int32Array(sharedBuffer);

// Set up the lock
Lock.initialize(sharedArray, 0);
const lock = new Lock(sharedArray, 0);
lock.lock(); // writes to sharedBuffer

worker.postMessage({sharedBuffer}); // (A)

document.getElementById('unlock').addEventListener(
    'click', event => {
        event.preventDefault();
        lock.unlock(); // (B)
    });

In the worker, we set up a local version of the lock (whose state is shared with the main thread via a Shared Array Buffer). In line B, we wait until the lock is unlocked. In lines A and C, we send text to the main thread, which displays it on the page for us (how it does that is not shown in the previous code fragment). That is, we are using self.postMessage() much like console.log() in these two lines.

// worker.js

self.addEventListener('message', function (event) {
    const {sharedBuffer} = event.data;
    const lock = new Lock(new Int32Array(sharedBuffer), 0);

    self.postMessage('Waiting for lock...'); // (A)
    lock.lock(); // (B) blocks!
    self.postMessage('Unlocked'); // (C)
});

It is noteworthy that waiting for the lock in line B stops the complete worker. That is real blocking, which hasn’t existed in JavaScript until now (await in async functions is an approximation).
Implementing a shared lock  

Next, we’ll look at an ES6-ified version of a Lock implementation by Lars T. Hansen that is based on SharedArrayBuffer.

In this section, we’ll need (among others) the following Atomics function:

    Atomics.compareExchange(ta : TypedArray<T>, index, expectedValue, replacementValue) : T
    If the current element of ta at index is expectedValue, replace it with replacementValue. Return the previous (or unchanged) element at index.

The implementation starts with a few constants and the constructor:

const UNLOCKED = 0;
const LOCKED_NO_WAITERS = 1;
const LOCKED_POSSIBLE_WAITERS = 2;

// Number of shared Int32 locations needed by the lock.
const NUMINTS = 1;

class Lock {

    /**
     * @param iab an Int32Array wrapping a SharedArrayBuffer
     * @param ibase an index inside iab, leaving enough room for NUMINTS
     */
    constructor(iab, ibase) {
        // OMITTED: check parameters
        this.iab = iab;
        this.ibase = ibase;
    }

The constructor mainly stores its parameters in instance properties.

The method for locking looks as follows.

/**
 * Acquire the lock, or block until we can. Locking is not recursive:
 * you must not hold the lock when calling this.
 */
lock() {
    const iab = this.iab;
    const stateIdx = this.ibase;
    var c;
    if ((c = Atomics.compareExchange(iab, stateIdx, // (A)
    UNLOCKED, LOCKED_NO_WAITERS)) !== UNLOCKED) {
        do {
            if (c === LOCKED_POSSIBLE_WAITERS // (B)
            || Atomics.compareExchange(iab, stateIdx,
            LOCKED_NO_WAITERS, LOCKED_POSSIBLE_WAITERS) !== UNLOCKED) {
                Atomics.wait(iab, stateIdx, // (C)
                    LOCKED_POSSIBLE_WAITERS, Number.POSITIVE_INFINITY);
            }
        } while ((c = Atomics.compareExchange(iab, stateIdx,
        UNLOCKED, LOCKED_POSSIBLE_WAITERS)) !== UNLOCKED);
    }
}

In line A, we change the lock to LOCKED_NO_WAITERS if its current value is UNLOCKED. We only enter the then-block if the lock is already locked (in which case compareExchange() did not change anything).

In line B (inside a do-while loop), we check if the lock is locked with waiters or not unlocked. Given that we are about to wait, the compareExchange() also switches to LOCKED_POSSIBLE_WAITERS if the current value is LOCKED_NO_WAITERS.

In line C, we wait if the lock value is LOCKED_POSSIBLE_WAITERS. The last parameter, Number.POSITIVE_INFINITY, means that waiting never times out.

After waking up, we continue the loop if we are not unlocked. compareExchange() also switches to LOCKED_POSSIBLE_WAITERS if the lock is UNLOCKED. We use LOCKED_POSSIBLE_WAITERS and not LOCKED_NO_WAITERS, because we need to restore this value after unlock() temporarily set it to UNLOCKED and woke us up.

The method for unlocking looks as follows.


    /**
     * Unlock a lock that is held.  Anyone can unlock a lock that
     * is held; nobody can unlock a lock that is not held.
     */
    unlock() {
        const iab = this.iab;
        const stateIdx = this.ibase;
        var v0 = Atomics.sub(iab, stateIdx, 1); // A

        // Wake up a waiter if there are any
        if (v0 !== LOCKED_NO_WAITERS) {
            Atomics.store(iab, stateIdx, UNLOCKED);
            Atomics.wake(iab, stateIdx, 1);
        }
    }

    // ···
}

In line A, v0 gets the value that iab[stateIdx] had before 1 was subtracted from it. The subtraction means that we go (e.g.) from LOCKED_NO_WAITERS to UNLOCKED and from LOCKED_POSSIBLE_WAITERS to LOCKED.

If the value was previously LOCKED_NO_WAITERS then it is now UNLOCKED and everything is fine (there is no one to wake up).

Otherwise, the value was either LOCKED_POSSIBLE_WAITERS or UNLOCKED. In the former case, we are now unlocked and must wake up someone (who will usually lock again). In the latter case, we must fix the illegal value created by subtraction and the wake() simply does nothing.
Conclusion for the example  

This gives you a rough idea how locks based on SharedArrayBuffer work. Keep in mind that multithreaded code is notoriously difficult to write, because things can change at any time. Case in point: lock.js is based on a paper documenting a futex implementation for the Linux kernel. And the title of that paper is “Futexes are tricky” (PDF).

If you want to go deeper into parallel programming with Shared Array Buffers, take a look at synchronic.js and the document it is based on (PDF).
The API for shared memory and atomics  
SharedArrayBuffer  

Constructor:

    new SharedArrayBuffer(length)
    Create a buffer for length bytes.

Static property:

    get SharedArrayBuffer[Symbol.species]
    Returns this by default. Override to control what slice() returns.

Instance properties:

    get SharedArrayBuffer.prototype.byteLength()
    Returns the length of the buffer in bytes.

    SharedArrayBuffer.prototype.slice(start, end)
    Create a new instance of this.constructor[Symbol.species] and fill it with the bytes at the indices from (including) start to (excluding) end.

Atomics  

The main operand of Atomics functions must be an instance of Int8Array, Uint8Array, Int16Array, Uint16Array, Int32Array or Uint32Array. It must wrap a SharedArrayBuffer.

All functions perform their operations atomically. The ordering of store operations is fixed and can’t be reordered by compilers or CPUs.
Loading and storing  

    Atomics.load(ta : TypedArray<T>, index) : T
    Read and return the element of ta at index.

    Atomics.store(ta : TypedArray<T>, index, value : T) : T
    Write value to ta at index and return value.

    Atomics.exchange(ta : TypedArray<T>, index, value : T) : T
    Set the element of ta at index to value and return the previous value at that index.

    Atomics.compareExchange(ta : TypedArray<T>, index, expectedValue, replacementValue) : T
    If the current element of ta at index is expectedValue, replace it with replacementValue. Return the previous (or unchanged) element at index.

Simple modification of Typed Array elements  

Each of the following functions changes a Typed Array element at a given index: It applies an operator to the element and a parameter and writes the result back to the element. It returns the original value of the element.

    Atomics.add(ta : TypedArray<T>, index, value) : T
    Perform ta[index] += value and return the original value of ta[index].

    Atomics.sub(ta : TypedArray<T>, index, value) : T
    Perform ta[index] -= value and return the original value of ta[index].

    Atomics.and(ta : TypedArray<T>, index, value) : T
    Perform ta[index] &= value and return the original value of ta[index].

    Atomics.or(ta : TypedArray<T>, index, value) : T
    Perform ta[index] |= value and return the original value of ta[index].

    Atomics.xor(ta : TypedArray<T>, index, value) : T
    Perform ta[index] ^= value and return the original value of ta[index].

Waiting and waking  

Waiting and waking requires the parameter ta to be an instance of Int32Array.

    Atomics.wait(ta: Int32Array, index, value, timeout=Number.POSITIVE_INFINITY) : ('not-equal' | 'ok' | 'timed-out')
    If the current value at ta[index] is not value, return 'not-equal'. Otherwise go to sleep until we are woken up via Atomics.wake() or until sleeping times out. In the former case, return 'ok'. In the latter case, return 'timed-out'. timeout is specified in milliseconds. Mnemonic for what this function does: “wait if ta[index] is value”.

    Atomics.wake(ta : Int32Array, index, count)
    Wake up count workers that are waiting at ta[index].

Miscellaneous  

    Atomics.isLockFree(size)
    This function lets you ask the JavaScript engine if operands with the given size (in bytes) can be manipulated without locking. That can inform algorithms whether they want to rely on built-in primitives (compareExchange() etc.) or use their own locking. Atomics.isLockFree(4) always returns true, because that’s what all currently relevant supports.

FAQ  
What browsers support Shared Array Buffers?  

At the moment, I’m aware of:

    Firefox (50.1.0+): go to about:config and set javascript.options.shared_memory to true
    Safari Technology Preview (Release 21+): enabled by default.
    Chrome Canary (58.0+): There are two ways to switch it on.
        Via chrome://flags/ (“Experimental enabled SharedArrayBuffer support in JavaScript”)
        --js-flags=--harmony-sharedarraybuffer --enable-blink-feature=SharedArrayBuffer

Further reading  

More information on Shared Array Buffers and supporting technologies:

    “Shared memory – a brief tutorial” by Lars T. Hansen

    “A Taste of JavaScript’s New Parallel Primitives” by Lars T. Hansen [a good intro to Shared Array Buffers]

    “SharedArrayBuffer and Atomics Stage 2.95 to Stage 3” (PDF), slides by Shu-yu Guo and Lars T. Hansen (2016-11-30) [slides accompanying the ES proposal]

    “The Basics of Web Workers” by Eric Bidelman [an introduction to web workers]

Other JavaScript technologies related to parallelism:

    “The Path to Parallel JavaScript” by Dave Herman [a general overview of where JavaScript is heading after the abandonment of PJS]
    “Write massively-parallel GPU code for the browser with WebGL” by Steve Sanderson [fascinating talk that explains how to get WebGL to do computations for you on the GPU]

Background on parallelism:

    “Concurrency is not parallelism” by Rob Pike [Pike uses the terms “concurrency” and “parallelism” slightly differently than I do in this blog post, providing an interesting complementary view]

Acknowledgement: I’m very grateful to Lars T. Hansen for reviewing this blog post and for answering my SharedArrayBuffer-related questions.
