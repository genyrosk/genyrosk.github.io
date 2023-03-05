---
title: "Python Context Managers"
date: 2023-03-05T22:10:14+09:00
draft: true
tags:
- python
- context manager
- decorator
- generator
- rate limiter
- tutorial
---

**Originally posted on ... 2020-04-04**

## A real problem I encountered

<!-- [[snippet]]
| While learning to use Python Context Managers, I encountered a neat use case for them. At the time I was writing a data pipeline that was heavily querying an API and I would often quickly reach the **API quota limits** and the script would abruptly quit. My initial solution was to catch the exception and implement a loop that would try again after a time delay. It was a quick and dirty solution with a lot of boilerplate code, but it worked as a first approach. Later I found myself in the same problem when querying a different API and I realised that a more structured approach was necessary. 

Some time later I found out the existence of the [ratelimiter](https://github.com/RazerM/ratelimiter) package, and to my surprise: it also used context managers and the code looked very similar ! In addition, the package features a very neat usage of the decorator syntax and implements both synchronous and asynchronous versions.  

[[snippet]]
| In this article we are going to go through the **basics of the context manager functionality**, and then see it in action when implemented into a rate limiter class.  -->


## The purpose of context managers 

The purpose of a context manager is to encapsulate a sequence of "setup" and "teardown" steps into a neat block of code. Any code that runs within that block runs within the context of a particular resource that is created in the process. In the background, a context manager implements this process for you. This process usually involves some sort of creation followed by deletion, an opening followed by closing (of a connection, for example), a building step followed by a breaking step (more metaphors are welcome).  

Sounds confusing ? Don't worry, let's clear it up with some examples. 


### Opening a file 

To open a file in Python you would usually write the follwing: 

```python
f = open('data.txt')
text = f.read()
print(text)
f.close()
```

```shell
Hello World
```

Notice the `open` and `close` logic. 

This code is equivalent to the following context manager: 

```python
with open('data.txt') as f:
    ## inside the context manager block 
    text = f.read()
    print(text)
    
## outside the context manager block 
```

```shell
Hello World
```

With a context manmager, the user no longer has to explicitely specify the closing step: the context manager does it for you as soon as you start a new line outside the block. Neat. 


## Building a context manager ... 

## ... as a class

A context manager can be implemented with special class underscore methods, namely `__enter__` and `__exit__`. 

Let's implement the file reader context manager: 

```python
class readfile:
    
    def __init__(self, file_name, method):
        self.f = open(file_name, method)
    
    def __enter__(self):
        # Code to acquire resource
        print('>> __enter__')
        return self.f
        
    def __exit__(self, *args): # __exit__ takes 3 extra arguments that we don't need to worry about
        # Code to release resource
        print('>> __exit__')
        self.f.close()
```

```python
with readfile('data.txt', 'r') as f:
    text = f.read()
    print(text)
```

```shell
>> __enter__
Hello World

>> __exit__
```

Note how `self.f`, the resource object, is passed around the class and is ultimately surfaced by the `__enter__` method which allows the used to interact with the object inside the block of the context manager. 


### ...as a generator 

If you're familiar with Python generators and the logic you're writing is simple enough, you can implement a context manager with the generator syntax. 

```python
from contextlib import contextmanager

@contextmanager
def readfile(file_name, method):
    # Code to acquire resource
    f = open(file_name, method)
    try:
        print('>> yield')
        yield f
    finally:
        # Code to release resource
        print('>> finally')
        f.close()
```

```python
with readfile('data.txt', 'r') as f:
    text = f.read()
    print(text)
```

```shell
>> yield
Hello World

>> finally
```

This syntax looks much simpler by using the `contextmanager` function to decorate the generator and convert it into a context manager. In the  background this decorator implements the `__enter__` and `__exit__` methods for us so we don't have to worry about them. All we need is a `try ... finally` logic. 


## The request limiter

### A simulated API

Let's simulate an API behaviour with the following generator, which normally returns a `200 OK` message when everything is ok, but will return an `403 Quota Exceeded` error message if a pre-defined quota limit has been exceeded. We will interface with the API generator using an intermediary `call_api` function to make things easier on the user side code. 

```python
import time 
        
def API(ping=0.15):
    limit = 5    # allow up to 5 calls ...
    period = 1.0 # ... per second
    calls = []
    while True:
        time.sleep(ping) # simulate response delay 
        now = time.time()
        call_window = [t for t in calls if now - t < period]
        if len(call_window) < limit:
            calls = call_window + [now]
            yield '200 OK'
        else:
            yield '403 Quota Exceeded'

api = API()
def call_api():
    return next(api)
```

Now if we simulate this API object with very rapid calls, we should get an error message back once we exceed the quota of 5 calls per second. 

```python
start = time.time()
for i in range(10):
    result = call_api()
    print(f'second [{time.time() - start:.2f}] call [{i}]: {result}')
```

```shell
second [0.15] call [0]: 200 OK
second [0.31] call [1]: 200 OK
second [0.46] call [2]: 200 OK
second [0.62] call [3]: 200 OK
second [0.77] call [4]: 200 OK
second [0.92] call [5]: 403 Quota Exceeded
second [1.07] call [6]: 403 Quota Exceeded
second [1.22] call [7]: 200 OK
second [1.37] call [8]: 200 OK
second [1.52] call [9]: 200 OK
```

But if we introduce a long enough delay between the calls, the API will no longer respond with a `403` error message. 

```python
start = time.time()
for i in range(10):
    time.sleep(0.2)
    result = call_api()
    print(f'second [{time.time() - start:.2f}] call [{i}]: {result}')
```

```shell
second [0.35] call [0]: 200 OK
second [0.71] call [1]: 200 OK
second [1.07] call [2]: 200 OK
second [1.43] call [3]: 200 OK
second [1.79] call [4]: 200 OK
second [2.15] call [5]: 200 OK
second [2.50] call [6]: 200 OK
second [2.86] call [7]: 200 OK
second [3.21] call [8]: 200 OK
second [3.57] call [9]: 200 OK
```

However, this is far from ideal, since we're stuck introducing delays into our code. What we actually would like to do is to be able to call the API as soon as the quota window has passed. We need to keep track of the request times and throttle our own calls. 

### First attempt: function wrapper 

Let's start with a function wrapper. Say we need to call the API an arbitrary number of times. 

```python
def request_limiter(n, max_calls, period=1.0):
    results = []
    calls = []
    
    # call the api n times
    for _ in range(n):

        # Sleep cycle logic:
        # if we're exceeding the limit ...
        if len(calls) >= max_calls:
            # calculate the minimum sleeping time
            sleeptime = period - calls[-1] + calls[0] 
            if sleeptime > 0:
                # and wait
                time.sleep(sleeptime)

        # API call 
        result = call_api()
        results += [result]

        # keep track of calls within the time window
        now = time.time()
        calls += [now]
        calls = [t for t in calls if now - t < period] 
    
    return results
```

```python
results = request_limiter(10, 5)
for i, result in enumerate(results):
    print(f'call [{i}]: {result}')
```

```shell
call [0]: 200 OK
call [1]: 200 OK
call [2]: 200 OK
call [3]: 200 OK
call [4]: 200 OK
call [5]: 200 OK
call [6]: 200 OK
call [7]: 200 OK
call [8]: 200 OK
call [9]: 200 OK
```


### Improved version: as a decorator 

The main limitation of the simple functional implementation above is that we have to provide to it the number of times that we want to query the API in order to be able to then keep a memory of the state inside the function. What if we don't know how many times we need to call the API ? What if these calls come from different places ? 

We need to keep track of the calls that we're making dynamically, i.e. the state needs to live outside of the function making the calls: a stateful decorator. 

Although stateful decorators can be implemented using the function itself as the [object state](https://realpython.com/primer-on-python-decorators/#stateful-decorators), it's much more intelligible to implement it in a classic class structure.  

```python
import time
from functools import wraps

class RequestLimiter:
    
    def __init__(self, max_calls, period=1.0):
        self.max_calls = max_calls # number of calls limit
        self.period = period       # time window 
        self.calls = []
    
    def __call__(self, f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            
            # Sleep cycle 
            if len(self.calls) >= self.max_calls:
                sleeptime = self.period - self._timespan
                if sleeptime > 0:
                    time.sleep(sleeptime)
            
            # function call 
            rv = f(*args, **kwargs) # `rv` = return value
            
            # keep track of calls within the time window
            now = time.time()
            self.calls += [now]
            self.calls = [t for t in self.calls if now - t < self.period]
            
            return rv
        return wrapper

    @property
    def _timespan(self):
        return self.calls[-1] - self.calls[0] 
```

```python
@RequestLimiter(5)
def call_api_throttled():
    return call_api()

start = time.time()
for i in range(10):
    result = call_api_throttled()
    print(f'second [{time.time() - start:.2f}] call [{i}]: {result}')
```

```shell
second [0.15] call [0]: 200 OK
second [0.31] call [1]: 200 OK
second [0.46] call [2]: 200 OK
second [0.61] call [3]: 200 OK
second [0.77] call [4]: 200 OK
second [1.31] call [5]: 200 OK
second [1.46] call [6]: 200 OK
second [1.62] call [7]: 200 OK
second [1.77] call [8]: 200 OK
second [1.92] call [9]: 200 OK
```

### The context manager version

In this particular context manager implementation, the context lives outside of the `with` block. However: every time we enter the `with` block we're back again inside the same context and the same state as when we left. Thus, the context manager keeps track of the state outside of the `with` block and enforces the required behaviour when we enter the `with` block. 

```python
import time
from functools import wraps

class RequestLimiter:
    
    def __init__(self, max_calls, period=1.0):
        self.max_calls = max_calls # number of calls limit
        self.period = period       # time window 
        self.calls = []
    
    def __enter__(self):
        # Sleep cycle 
        if len(self.calls) >= self.max_calls:
            sleeptime = self.period - self._timespan
            if sleeptime > 0:
                time.sleep(sleeptime)
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        # keep track of calls within the time window
        now = time.time()
        self.calls += [now]
        self.calls = [t for t in self.calls if now - t < self.period]
    
    @property
    def _timespan(self):
        return self.calls[-1] - self.calls[0] 
```

```python
limiter = RequestLimiter(5) # the state lives outside of the function calls 
start = time.time()
for i in range(10):
    with limiter: 
        # >>>>> we enter the context manager's state <<<<<
        # ------------------------------------------------
        # based on its state, the context manager decides
        # when to to execute the code inside the block 
        result = call_api()
        print(f'second [{time.time() - start:.2f}] call [{i}]: {result}')
```

```shell
second [0.15] call [0]: 200 OK
second [0.31] call [1]: 200 OK
second [0.46] call [2]: 200 OK
second [0.61] call [3]: 200 OK
second [0.76] call [4]: 200 OK
second [1.31] call [5]: 200 OK
second [1.46] call [6]: 200 OK
second [1.61] call [7]: 200 OK
second [1.76] call [8]: 200 OK
second [1.92] call [9]: 200 OK
```

### Adding decorator syntax support

We can add decorator syntax support with a simple modification of the `__call__` method, just as we did before. However, this time we don 't need to implement the entire logic again: simply make use of the context manager logic within the decorator. 

```python{11-16}
import time
from functools import wraps

class RequestLimiter:
    
    def __init__(self, max_calls, period=1.0):
        self.max_calls = max_calls # number of calls limit
        self.period = period       # time window 
        self.calls = []
    
    def __call__(self, f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            with self:
                return f(*args, **kwargs)
        return wrapper
    
    def __enter__(self):
        # Sleep cycle 
        if len(self.calls) >= self.max_calls:
            sleeptime = self.period - self._timespan
            if sleeptime > 0:
                time.sleep(sleeptime)
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        # keep track of calls within the time window
        now = time.time()
        self.calls += [now]
        self.calls = [t for t in self.calls if now - t < self.period]
    
    @property
    def _timespan(self):
        return self.calls[-1] - self.calls[0] 
```

This is as simple and elegant as it gets.

Let's take this decorator for a spin.

```python
@RequestLimiter(5)
def call_api_throttled():
    return call_api()

start = time.time()
for i in range(10):
    result = call_api_throttled()
    print(f'second [{time.time() - start:.2f}] call [{i}]: {result}')
```

```shell
second [0.15] call [0]: 200 OK
second [0.31] call [1]: 200 OK
second [0.46] call [2]: 200 OK
second [0.61] call [3]: 200 OK
second [0.77] call [4]: 200 OK
second [1.31] call [5]: 200 OK
second [1.47] call [6]: 200 OK
second [1.62] call [7]: 200 OK
second [1.77] call [8]: 200 OK
second [1.93] call [9]: 200 OK
```

Same as before.

Thank you for reading ! 
