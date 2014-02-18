Sample program: [server.asyncawait.js](server.asyncawait.js) using [sweet.js](http://sweetjs.org/) approximation of the proposed grammar and desugaring.

To run the example:
```Shell
npm install 
sjs server.asyncawait.js | node --harmony 
```

# Async Functions for  ECMAScript

The introduction of Promises and Generators in ECMAScript presents an opportunity to dramatically improve the language-level model for writing asynchronous code in ECMAScript.  

A similar proposal was made with [Defered Functions](http://wiki.ecmascript.org/doku.php?id=strawman:deferred_functions) during ES6 discussions.  The proposal here supports the same use cases, using similar or the same syntax, but directly building upon generators and promises instead of defining custom mechanisms.

## Example

Take the following example, first written using Promises.  This code chains a set of animations on an element, stopping when there is an exception in an animation, and returning the value produced by the final succesfully executed animation.

```JavaScript
function chainAnimationsPromise(elem, animations) {
    var ret = null;
    var p = currentPromise;
    for(var anim in animations) {
        p = p.then(function(val) {
            ret = val;
            return anim(elem);
        })
    }
    return p.catch(function(e) {
        /* ignore and keep going */
    }).then(function() {
        return ret;
    });
}
```

Already with promises, the code is much improved from a straight callback style, where this sort of looping and exception handling is challenging.

[Task.js](http://taskjs.org/) and similar libraries offer a way to use generators to further simplify the code maintaining the same meaning:

```JavaScript
function chainAnimationsGenerator(elem, animations) {
    return spawn(function*() {
        var ret = null;
        try {
            for(var anim in animations) {
                ret = yield anim(elem);
            }
        } catch(e) { /* ignore and keep going */ }
        return ret;
    });
}
```

This is a marked improvement.  All of the promise boilerplate above and beyond the semantic content of the code is removed, and the body of the inner function represents user intent.  However, there is an outer layer of boilerplate to wrap the code in an additional generator function and pass it to a library to convert to a promise.  This layer needs to be repeated in every function that uses this mechanism to produce a promise.  This is so common in typical async Javascript code, that there is value in removing the need for the remaining boilerplate.

With async functions, all the remaining boiler plate is removed, leaving only the semantically meaningful code in the program text:

```JavaScript
async function chainAnimationsAsync(elem, animations) {
    var ret = null;
    try {
        for(var anim in animations) {
            ret = await anim(elem);
        }
    } catch(e) { /* ignore and keep going */ }
    return ret;
}
```

This is morally similar to generators, which are a function form that produces a Generator object.  This new async function form produces a Promise object.

## Details

Async functions are a thin sugar over generators and a `spawn` function which converts generators into promise objects.  The internal generator object is never exposed directly to user code, so the rewrite below can be optimized significantly.

### Rewrite

```
async function <name>?<argumentlist><body>

=>

function <name>?<argumentlist>{ return spawn(function*() <body>); }
```

### Spawning

The `spawn` used in the above desugaring is a call to the following algorithm.  This algorithm does not need to be exposed directly as an API to user code, it is part of the semantics of async functions.

```JavaScript
function spawn(genF) {
    return new Promise(function(resovle,reject) {
        var gen = genF();
        function step(nextF) {
            var next;
            try {
                next = nextF();
            } catch(e) {
                // finished with failure, reject the promise
                reject(next); 
                return;
            }
            if(next.done) {
                // finished with success, resolve the promise
                resolve(next.value);
                return;
            } 
            // not finished, chain off the yielded promise and `step` again
            Promise.cast(next.value).then(function(v) {
                step(function() { return gen.next(v); });      
            }, function(e) {
                step(function() { return gen.throw(e); });
            });
        }
        step(function() { return gen.next(undefined) });
    })
}
```

### Syntax

The set of syntax forms are the same as for generators.

```JavaScript
AsyncMethod :
    async PropertyName (StrictFormalParameters)  { FunctionBody } 
AsyncDeclaration :
    function async BindingIdentifier ( FormalParameters ) { FunctionBody }
AsyncExpression :
    function async BindingIdentifier? ( FormalParameters ) { FunctionBody }

// If needed - see syntax options below
AwaitExpression :
    await [Lexical goal InputElementRegExp]   AssignmentExpression 
```

### Async Generators

Applying the ```async``` keyword to a function expression causes it to return a Promise. This begs the question: what does an async generator function return?

A typical ES6 function sends its result to the client via a return statement. If the async modifier is added to the function, the data is sent to the client as an argument to the Promise's then() method.  Swapping a function's result from the return position to the argument position is the Continuation-Passing Style transformation.

An iterator is similar to a typical JavaScript function, in that the position of the function's result is the return value.  

```JavaScript
Array.prototype[@@iterator] = function() {
    var self = this;
    let index = -1;
    return {
        next: () => {
            // result of function sent as return value
            return { done: index >= self.length, value: self[++index] }; 
        },
        throw: () => { index = self.length; }
    };
}
```

If we swap the arguments and return type of the @@iterator function we get @@iteratee, a function that accepts an iterator and returns no arguments.

```JavaScript
Array.prototype[@@iteratee] = function(iterator) {
    let done;
    for(var count = 0, len = this.length; !done && count < len; count++) {
        // result of function sent as argument
        {done} = iterator.next(this[count]);
    }
    iterator.close();
}
```

Objects that support the iteration protocol can be iterated using the ```for of```  special form.

```Javascript
for(let value of [1,2,3]) {
    console.log(value);
}
```

The code above the desugars into...

```Javascript
let iterator = [1,2,3][@@iterator](),
    done = false,
    value;

    do {
        {done, value} = iterator.next();
        if (!done) {
            console.log(value);
        }
    } while(!done);
```

Objects that support the observation protocol can be observed using the ```for await``` special form,  which is only  available inside of a async generator function.

```Javascript
async function*() {
    // retrieve an array of numbers from a promise
    var collection = await getNumbers();
    // iterate an Iterable
    for(let value of [1,2,3]) {
        yield value;
    }
    
    // observe an Observable
    for(let value await websocket) {
        yield value;
    }
}
```

Iterable objects return an iterator that yields values. Observable objects accept an iteratee that receives values. Adding an Observation protocol to ES6 is straightforward because ES6 Iterators are capable of both sending and receiving notifications. Let's take a look at how can add support to Array for the observation using the proposed @@iteratee symbol.

```JavaScript
WebSocket.prototype[@@iteratee] = function(iterator) {
    this.onmessage = function(e) {
        let {done} = iterator.next(e);
        if (done) {
            this.close();
        }
    };
    
    this.onerror = function(e) {
        if (iterator.throw !== undefined) {
            iterator.throw(e);
        }
        else {
            throw e;
        }
    }
    
    this.onclose = function() {
        if (iterator.close) {
            iterator.close();
        }
    }
};

async function numbersPlusOne*(addressOfNumberStream){
    var connection = new WebSocket('ws://netflix.com/firsttenthousandnaturalnumbers', ['soap', 'xmpp']);
    
    try {
        for(let x await connection) {
            log x + 1;
        }
    }
    catch(e) {
        alert("There was an error: " + e);
    }
}
```


This expression...

```JavaScript
async function*(){
    var connection = new WebSocket('ws://netflix.com/firsttenthousandnaturalnumbers', ['soap', 'xmpp']);
    
    try {
        for(let x await connection) {
            yield x + 1;
        }
    }
    catch(e) {
        alert("There was an error: " + e);
    }
    
    yield 9;
    yield 1000;
}
```

...desugars to this...

```JavaScript
function() {
    return {
        @@iteratee: function(iteratee) {
            var connection = new WebSocket('ws://netflix.com/firsttenthousandnaturalnumbers', ['soap', 'xmpp']);
            var done = () => {
                let {done, value} = iteratee.next(9);
                if (done) {
                    break;
                }
                iteratee.next(1000);
            };
            connection[@@iteratee]({
                next: function(value) {
                    return iteratee.next(value + 1);
                },
                throw: function(e) {
                    alert("There was an error: " + e);
                    
                    done();
                },
                close: function() {
                    done();
                }
            });
            
        }
    }
}

async function*(){
    var connection = new WebSocket('ws://html5rocks.websocket.org/echo', ['soap', 'xmpp']);
    
    for(await x of connection) {
        yield x;
    }
}
```

...is syntactic sugar for 
In generators, both `yield` and `yield*` can be used.  In async functions, only `await` is allowed.  The direct analogoue of `yeild*` does not make sense in async functions because it would need to repeatedly await the inner operation, but does not know what value to pass into each await (for `yield*`, it just passes in undefined because iterators do not accept incoming values).

It has been suggested that the syntax could be reused for different semantics - sugar for Promise.all.  This would accept a value that is an array of Promises, and would (asynchronously) return an array of values returned by the promises.  This is expected to be one of the most common Promise-related oprerations that would not yet have syntax sugar after the core of this proposal is available. 

### Awaiting Non-Promise

When the value passed to `await` is a Promise, the completion of the async function is scheduled on completion of the Promise.  For non-promises, behaviour aligns with Promise conversation rules according to the proposed semantic polyfill.

### Surface syntax
Instead of `async function`/`await`, the following are options:
- `function^`/`await`
- `function!`/`yield`
- `function!`/`await`
- `function^`/`yield`

### Arrows
The same approach can apply to arrow functions.  For example, assuming the `async function` syntax:   `async () => yield fetch('www.bing.com')` or `async (z) => yield z*z` or `async () => { yield 1; return 1; }`.

### Notes on Types
For generators, an `Iterable<T>` is always returned, and the type of each yield argument must be `T`.  Return should not be passed any argument.

For async functions, a `Promise<T>` is returned, and the type of return expressions must be `T`.  Yield's arguments are `any`.
