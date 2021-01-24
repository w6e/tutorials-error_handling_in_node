# Error Handling in NodeJS

Error handling in NodeJS is a topic many new developers struggle with,
especially if they are used to writing code in synchronous languages such as C,
Ruby, Python...
I wrote this summary to explain error handling in NodeJS to a friend of mine. It helped him. In hopes it may help you or someone you know, I share it here. 

NodeJS is asynchronous by design. It executes in a single thread, yet can
manage a large number of tasks concurrently. Execution on a single thread
implies there is only **one call stack**. So if we put a blocking operation on
the call stack, our thread is blocked entirely until that operation
finishes. That's why NodeJS makes heavy use of asynchronous language constructs
such as **callbacks, promises and async/ await**.

*This is they key in building up an understanding of understand error handling
in NodeJS.*

## Error handling in the synchronous paradigm

Error handling in synchronous code is easy to implement and reason about. Most
higher-level synchronous languages offer some variation of `try{... } catch(exception) {... }` syntax
which can be used to handle errors.

### Let's do a basic example

#### Ruby
We will use Ruby to illustrate error handling in a **synchronous language**.

We define a function `inverse` which expects a numeric input. If we call
`inverse` with a non-numeric argument input, the function *raises an exception* using `raise`.

```
def inverse(x)
  raise ArgumentError, 'Argument is not numeric' unless x.is_a? Numeric
  1.0 / x
end
```

Now we call inverse without proper exception handling

```
def callInverseUnsafe
  puts inverse 5 # => 0.2
  puts inverse 'a string' # => ArgumentError crashes the program
end

callInverseUnsafe
```

To pevent our program from crashing, we must explicitly handle that exception: 

```
def call InverseSafe
  begin
    puts inverse 5 # => 0.2
    puts inverse 'a string' # => ArgumentError, program keeps running :) 
  rescue ArgumentError => e
    puts e.message
  end
end

callInverseSafe
```

`callInverseSafe` handles `ArgumentError` exceptions using the `rescue` syntax. If we call `inverse` with a non-numeric argument, the program executes the code inside our `rescue` block and keeps running. It does not crash since we told it how to act in case this type of error occurs.

#### JavaScript
Javascript offers `try/ catch` syntax to deal with errors in synchronous code.
Here is what that ruby code above might look like in JS.

```
function inverse(x) {
  if (typeof x !== 'number') throw(new TypeError('Argument is not numeric')) 
  return 1.0 / x
}

function callInverseUnsafe() {
  console.log( 'callInverseUnsafe()...' )
  console.log( inverse(5) ) // => 0.2
  console.log( inverse('a string') ) // => // TypeError crashes the program
}

function callInverseSafe() {
  console.log ( 'callInverseSafe()... ' )
  try {
    console.log( inverse(5) ) // => 0.2
    console.log( inverse('a string') ) // => TypeError, program keeps running :)
  } catch(ex) {
    console.log(ex.name)
  }
}

callInverseSafe()
callInverseUnsafe()
```

Similar to ruby, Javascript uses `throw` to raise exceptions and `try {... } catch(ex) {... }` to handle them. `throw` is equivalent to `raise` and `try/catch` is equivalent to `begin/rescue`.


## Error handling in the *asynchronous paradigm*

Alright, so far we know that synchronous languages use some type of `try/catch` syntax construct to manage exceptions which are `thrown`. We also saw that Javascript can do this too. So what's the issue? Why can't we use what we learnt so far to handle errors in NodeJS?


### Understanding the Call Stack

I suppose you know what a call stack is. If you don't, no worries, it's simple - **in synchronous code!**

Let's say you execute the following code

```
function a() {
    b()
}

function b() {
    c()
}

function c() {
    console.log('C');
}

a()
```

So function `a` gets executed, before `a` gets executed it is pushed on top of the call stack.

```
call stack
----------
a
```

a calls b, so b is pushed on top of the call stack

```
call stack
----------
b
a
```

b calls c, so c is pushed on top of the call stack

```
call stack
----------
c
b
a
```

c reaches its last statement (`console.log`) executes it, then c is popped off the call stack

```
call stack
----------
b
a
```

b reaches its last statement, then b is popped off the call stack

```
call stack
----------
a
```

a reaches its last statement, then a is popped of the call stack

```
call stack
----------
---
```

the call stack is empty, the program exits.


####Error Handling and the Call Stack

Remember above, we looked at synchronous code and implemented exception handling with `try/catch`. How does the interpreter know what to do when an exception is thrown? 

It looks up the closest `try/catch` block in the call stack! In the JavaScript example above, the call stack at the moment the exception is thrown looks like this

```
call stack
----------
inverse
callInverseSafe
```

Inverse throws the `Argument is not numeric exception`. `inverse` is popped off the stack. The interpreter finds the closest `try/catch` block on the call stack by checking each function on the stack from top to bottom. It finds the `try/catch` block in `callInverseSafe` and immediately executes the code inside the `catch` block to handle the exception. Then it pops `callInverseSafe` off the call stack and continues executing the code that follows.

Let's look at the same example without exception handling. Here we call `callToInverse*Unsafe*` instead of `callToInverse*Safe*`

```
call stack
----------
inverse
callInverseUnsafe
```

As before `inverse` throws the exception, the interpreter looks for a `try/catch` block in the call stack. It does not find one. It reaches the bottom of the call stack - the error has **bubbled up the callstack**! Since the program does not know what else to do with the error, it shuts down the process in which it runs, the program has crashed.



### The Call Stack with Asynchronous Code

Alright, now that we understand the call stack and exception handling in synchronous code, we can easily see why `try/catch` will not work with asynchronous code.

Suppose we try to handle an error in asynchronous nodeJS code using `try/catch`. Here we use the `fs` module which contains functions to interact with the file system. `fs.readFile` asynchronously reads the entire contents of a file. 

Lets try to read a file which does not exist

```
function readSomeFile() {
    fs.readFile('./this-file-does-not-exist.txt', 'utf-8', (err, data) => {
        if (err) throw err // throw the error if readFile passes one to the callback
        console.log('File read: ', data) // log the file content
    }
}

function readFileAndCatchErrors() {
    try {
        readSomeFile()
    } catch(ex) {
        console.log('Exception caught: ', ex.message
    }
}

readFileAndCatchErrors()

```

So, we have a function `readSomeFile` which uses `fs.readFile(filename, format, callback)` to read the contents of a file asynchronously. As in the synchronous examples above, we use `try/catch` to catch the exception thrown inside the callback of `fs.readFile`.

If we run this code, the program crashes and tells us why: `Error: ENOENT: no such file or directory, open './this-file-does-not-exist.txt'`

It crashes even though we handled the exception with `try/catch`. Why? 

Remember how the interpreter looks for a `try/catch` block on the call stack when an exception is thrown? The function which contains the `try/catch` block we wrote in `readSomeFileAndCatchErrors` **is not on the call stack** by the time the `fs.readFile` callback throws its error! 

When we run our program, the call stack befofre `fs.readFile` is called looks like this

```
call stack
----------
readSomeFile
readFileAndCatchErrors
```

Then `fs.readFile` is called by `readSomeFile` and it schedules the callback we passed `(err, data) => ...` to be executed when the file contents have been read successfully or when an error occurs in the process.

Right after the callback is scheduled (google nodeJS event loop, event demultiplexer or reactor pattern if you want to know what that means exactly), our function `readSomeFile` finishes executing and is **popped off the call stack**. 

```
call stack
----------
readFileAndCatchErrors
```


Since there is nothing else to do in our `readFileAndCatchErrors` function, it also gets popped off the call stack: 


```
call stack
----------
---
```

In the meantime, the nodeJS runtime is trying to read our file from the filesystem. It does so by issuing *system calls* to the host OS. Since the file does not exist, the OS raises an exception `ENOENT` thereby informing nodeJS runtime that the file does not exist on disk.

Now, node executes the callback we passed to `fs.readFile`, passing the error as the first parameter to the callback. The callback throws the error: `if (err) throw (err)`

The interpreter looks for a `try/catch` block on the call stack to handle the exception. But oh no! The call stack is empty. There is no `try/catch` block on the call stack because the function which contains our `try/catch` has already finished executing and therefore was popped off the call stack.


**That's why `try/catch`** does not work with asynchronous code. Whenever we attempt to use `try/catch` to handle errors in asynchronous operations, the code which contains our `try/catch` has already left the call stack by the time the error is thrown and the interpreter begins looking for a `try/catch`


### Proper Error Handling in Asynchronous Code


Now that we know how **not** to do it, let's learn how to do it **right**.

There are 4 error handling mechanisms in asynchronous nodeJS 

* `(err, data) => ...` passing errors to callbacks 
* `.then(res).catch(err => ...)` `catch` using promises 
* `try/catch` using `async/await` syntax
* `.on('error')` event driven error handling with `EventEmitter`

#### Passing errors to callbacks

tbd...

#### Handling erros with promises

tbd...

#### `try/catch` with `async/await` (based on promises

tbd...

#### Event driven error handling with `EventEmitter`

tbd...



## Summary

If you've read this far, great job! I hope things are a littler clearer now. In summary, you have to remember that asynchronous code does not build call stacks the way synchronous code does. Since `try/catch` works by locating the closest `catch` on the call stack, we can't use this construct with asynchronous code. 

That's why nodeJS offers its own error handling mechanisms for async code. There is the *errors first* callback convention for callback based async APIs. Then there are promises, which use `.catch(err => ...)` to handle errors and `try/catch` for `async/await`. Finally we learnt about `EventEmitter` and event driven error handling, which is particularly useful for long-running pieces of code that we want to be resilient; think web servers for example.

Nobody wants to use software that fails unexepectedly and it is your job to write yours in such a way that it can handle operational errors without failing. Never trust anything outside of the process which runs your code. APIs can 500, databases connections can fail, users can provide invalid inputs. If you assume that everything your app uses will fail at some point, you have the right frame of mind to write resilient apps.
Think about all possible scenarios a piece of code could encounter and plan for failure!
