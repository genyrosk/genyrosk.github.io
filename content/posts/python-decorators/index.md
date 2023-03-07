---
title: "Learn Python Decorators"
date: 2023-03-05T22:09:57+09:00
draft: false
categories:
- python
tags:
- python
- decorators
- programming
- tutorial
- functional programming
- composed functions
---

*Originally posted on 2020-04-03*

Whether you are a beginner or an advanced programmer, there are many reasons why you should choose Python for your next coding project. Python is a scripted language with an almost _stupidly simple_ syntax, it is easy to learn and powerful to use. One of the main reasons that ~~I love~~ people love Python is that it offers quite a few specific metaphors (a.k.a. paradigms) for doing certain things. One of these things involves **wrapping functions with some repetitively used code**. And so, in your quest to becoming a true Pythonista lies a special kind of function that will help you improve and streamline your code: **decorators**.

## Wrapping a function

Imagine we have some API endpoint. We will simulate the behaviour of that endpoint with the `api_endpoint` function, which will return a response after a 1 second delay.


```python
import time
def api_endpoint(seconds=1):
    time.sleep(seconds)
    return 'API response'
```

Let's say that in our application we have a function that performs a request to this api endpoint and subsequently returns the result.


```python
def call_api():
    result = api_endpoint()
    return result

result = call_api()
print('result: ', result)
```

    result:  API response


Now let's imagine that we would like to time the execution of our api call. We can easily do this by using the `time.time` function and calculating the difference between the *before* and *after* times.


```python
def call_api_timed():
    # [1] >>> start timer
    start = time.time()

    # [2] >>> call the api
    result = api_endpoint()

    # [3] >>> end timer, print the time difference
    #         and return the result
    end = time.time()
    print('seconds:', end - start)
    return result

result = call_api_timed()
print('result: ', result)
```

    seconds: 1.0051519870758057
    result:  API response


Really simple and works as expected!


## The Python way

In Python we strive to do the minimum that gets the job done. That's the essence of a true Pythonista: no upfront design required, no complicated code necessary. Simply write the most bare bones, quick and dumb program you can that gets the job done and move on (go home to your kids what are you doing so late at your job?).

However, there are times when keeping things primitive only makes the overall picture increasingly complicated. This is especially true when our application performs lots of repetitive tasks. This is where the magic of Python lies: Python allows you to extend your code in a simple way, without introducing much overhead. For that purpose it offers a number of coding paradigms, one of which we're going to be using today: **decorators**.

> *_A **decorator** is a repeatable bit of code that you want to run **before** and **after** some function execution._*

In our example we wrote 3 lines of code within our `call_api` function in order to keep track of how long the api takes to respond. Imagine now that we have multiple different functions that call different API endpoints and we would also like to time all of them. If we keep doing it the same way as we did earlier, a lot of code would start to get duplicated.


```python
#
# We would have to repeat the same lines of code
# for all the following functions
# ...
def api_endpoint_2():
    pass

def api_endpoint_3():
    pass

def api_endpoint_4():
    pass

# ...
# ...
# ...
```

The problem with duplication is that it affects code **readability** and **maintainability**: our code becomes harder to read and more difficult and time consuming to maintain.

To solve this issue, let us define a function that automatically does the timing for us.


```python
def time_me(func):
    def wrapper():
        ### BEFORE CODE
        # [1] >>> start timer
        start = time.time()

        ### EXECUTE FUNCTION
        # [2] >>> call the function we're wrapping
        result = func()

        ### AFTER CODE
        # [3] >>> end timer, print the time difference
        #         and return the result
        end = time.time()
        print('seconds:', end - start)
        return result
    return wrapper

timed_api_call = time_me(call_api)
timed_api_call()
```

    seconds: 1.0002648830413818
    'API response'



In the above example we wrote the function `time_me`, which accepts another function (our API endpoint calls) as argument and then effectively returns this function *wrapped* with some extra code: in our case the same lines we wrote before.

Notice that `time_me` indeed returns a function:


```python
timed_api_call
```




    <function __main__.time_me.<locals>.wrapper()>



Thus, whenever we call `timed_api_call`, we're actually calling the `wrapper` code block within the `time_me` function. This paradigm is often referred to as **function composition**.

Ins't this just plain awesome?! We can now wrap this timer around any function to print out the time it took to execute and also get the result from the function execution.

Nevertheless, this introduces another, albeit minor, problem: long and convoluted lines of code. If we start to chain functions one after another, our code will become increasingly hard to read.




```python
first_function(second_function(third_function(...)))
```

In addition, we still need to explicitly perform the function composition by writing `timed_api_call = time_me(call_api)`. There's got to be a better way...


## Decorator syntax

Enter the decorator syntactic sugar. With this special syntax, we can wrap/decorate a function with another function (that we call a decorator) using `@` followed by the decorator symbol above the function definition. Like so:


```python
@decorator_func
def function_being_wrapped():
    pass
```

You can now perhaps see why the name "decorator" was chosen. We are decorating our function with it.  

Every time we call `function_being_wrapped`, `decorator_func` is executed beforehand. A decorator is just that: an ergonomic wrapper function.

There is no more extra code we need to write, we simply **decorate** our `call_api` function with the `time_me` decorator:


```python
@time_me
def call_api():
    res = api_endpoint()
    return res

result = call_api()
print('result: ', result)
```

    seconds: 1.004122257232666
    result:  API response


It just works!


### How to deal with arguments

We can also extend the functionality of our decorator by having it pass incoming arguments to the wrapped function.

We want the function that is returned by `time_me`, i.e. `wrapper`, to accept and pass on the arguments that the wrapped function accepts. Luckily Python has some additional syntactic sugar to allow for an arbitrary number of arguments and keyed arguments to be passed around.


```python
def time_me(func):
    def wrapper(*args, **kwargs): ## accept arguments        
        ### BEFORE CODE
        start = time.time() # start time

        ### EXECUTE FUNCTION
        # >>> pass the arguments to the function we're wrapping
        print('arguments passed:', args, kwargs)
        result = func(*args, **kwargs)

        ### AFTER CODE
        end = time.time()
        print('seconds:', end - start)
        return result
    return wrapper

@time_me
def call_api(seconds):
    res = api_endpoint(seconds=seconds)
    return res

result = call_api(1.5)
print('result: ', result)
```

    arguments passed: (1.5,) {}
    seconds: 1.5012753009796143
    result:  API response


### Decorators that accept arguments

"Can we pass arguments to the decorator itself?" you might ask. The answer is yes! But we would have to do some additional modifications to allow our decorator to accept arguments. Let's say you would want this function to sometimes print seconds, while other times you would rather have it print milliseconds.


```python
from functools import wraps

def time_me(milliseconds=False):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            ### BEFORE CODE
            start = time.time()

            ### EXECUTE FUNCTION
            result = func(*args, **kwargs)

            ### AFTER CODE
            # >>> print in seconds or milliseconds
            end = time.time()
            if milliseconds:
                print('milliseconds:', (end - start)*1000)
            else:
                print('seconds:', end - start)
            return result
        return wrapper
    # >>> time_me will return the "actual" decorator
    return decorator

@time_me(milliseconds=True)
def call_api():
    res = api_endpoint()
    return res
```


```python
call_api()
```

    milliseconds: 1000.8730888366699
    'API response'


Success!

Don't worry too much about the `@wraps` decorator: it's a special Python function that renames the wrapper name into the function that it's wrapping.


```python
## without @wraps
print(call_api)
```

    <function time_me.<locals>.decorator.<locals>.wrapper at 0x11160a2f0>



```python
## with @wraps
print(call_api)
```

    <function call_api at 0x11160a268>


It helps with debugging. Keep this in mind when you're writing open source code or if you're expecting other people to work with your functions.  

### How to write a decorator that works with or without arguments

With a little bit of care, you can also define decorators that can be used both with and without arguments. Most likely, you don’t need this, but sometimes it is nice to have the flexibility.


```python{19-24}
from functools import wraps

def time_me(_func=None, *, milliseconds=False):

    ### same as before
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            start = time.time()
            result = func(*args, **kwargs)
            end = time.time()
            if milliseconds:
                print('milliseconds:', (end - start)*1000)
            else:
                print('seconds:', end - start)
            return result
        return wrapper

    ### >>> call the decorator if the first argument is a function
    ### >>> otherwise return the decorator function (without calling it)
    if _func is None:
        return decorator
    else:
        return decorator(_func)


```

Then all the following can work:


```python
@time_me
def call_api_1():
    res = api_endpoint()
    return res

@time_me()
def call_api_2():
    res = api_endpoint()
    return res

@time_me(milliseconds=True)
def call_api_3():
    res = api_endpoint()
    return res
```


```python
print(call_api_1())
print(call_api_2())
print(call_api_3())
```

    seconds: 1.0042738914489746
    API response
    seconds: 1.0040597915649414
    API response
    milliseconds: 1002.3050308227539
    API response


## Bonus: Class implementation

Python treats everything as an object. **Everything**. That includes variables, functions, classes, lists, etc.

Python includes special class syntax that allows a decorator to be expressed in a class form. Whenever a class instance is called, the Python interpreter calls the `__call__` method in the background for us. We can intercept this calling by overriding this class method in our class definition.



```python
class TimeMe:

    def __init__(self, milliseconds=False):
        self.milliseconds = milliseconds

    def __call__(self, func):
        @wraps(func)
        def decorator(*args, **kwargs):
            start = time.time()
            result = func(*args, **kwargs)
            end = time.time()
            if self.milliseconds:
                print('milliseconds:', (end - start)*1000)
            else:
                print('seconds:', end - start)
            return result
        return decorator     
```


```python
timer = TimeMe(milliseconds=True)

@timer
def some_func_1(x):
    return 2*x

@TimeMe(milliseconds=True)
def some_func_2(x):
    return 3*x
```


```python
print('result: ', some_func_1(100))
print('result: ', some_func_2(100))
```

```shell
milliseconds: 0.00095367431640625
result:  200
milliseconds: 0.0021457672119140625
result:  300
```


I hope this helped. Thank you for reading! 

