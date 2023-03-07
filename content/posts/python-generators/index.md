---
title: "Learn Python Generators"
date: 2023-03-05T22:09:50+09:00
draft: true
categories:
- python
tags:
- python
- decorators
- programming
- tutorial
- prime numbers
- sql
---

*Originally posted on 2020-04-02*

#### __*There are already many very helpful tutorials on the subject out there. In this piece I wanted to share my own take and experience with python generators that I have gathered while working on production-level Python code.*__

## Introduction

Whether you are familiar with Python generators or you have only heard a thing or two about them, allow me to summarise the 2 main properties of generator functions: 
- generators are **stateful** functions. 
- generators **delay computations until necessary**. 

As you are about to see, these properties make generators incredibly useful. 


## Generator functions vs generator instances

Let's take a look at a simple example: 

```python
def generator_func():
    a = 0
    yield a
    a = 10
    yield a
```

```python
>>> generator_inst = generator_func()
```

Notice how we're using the special `yield` statement instead of the typical `return`. This is done on purpose: `generator_func` is a **generator function** while `generator_inst` is a **generator instance**, the "true" generator if you prefer. 

The purpose of a generator function is to build the generator instance, while the generator instance is the actual object that is going to run the specified code. This is because, as we will see more in detail later, **generators are stateful**. Thus we need to initialize an instance of this generator in its initial state, that will change everytime we make call it. 

We can trace an analogy with Python classes: everything starts with the class definition, which describes how different parts of it work together, and when called returns an instance of itself that holds all the inner methods and variables. In short, class definitions are meant to define and build, while class instances are meant to be built and executed. 


As you might have inferred based on this analogy, a generator instance, just like a class instance, is not a value. A generator instance is a **mapped computation** object that sits in a certain memory address (unlike functions). We can check it like so: 

```python
>>> generator_func
<function __main__.generator_func()>
>>> generator_instance
<generator object generator at 0x1111401a8>
```

Notice how the generator instance is an object located in the `0x1111401a8` memory address. 

From now on we'll refer to the generator instance simply as a *generator*. Albeit, don't be suprised if the term 'generator' is used to refer to the generator function as well. 


## Retrieve values from generators using `yield`/`next`

At this point, you might be thinking:

### _"Wait... but if inspecting the generator returns this address that stores the computation, how do I access the value that it's supposed to return?"_

This is where the `yield`/`next` pair truly kicks in. 

`yield` is a statment, i.e. a syntactic expression, that specifies the value that a generator returns, or, more precisely, the value that the generator __yields__, while `next` is a special Python function that summons the generator's inner code to execute until it encounters a `yield` statment, at which point it simply returns (yields) the specified value. 
    
The reason why the generator *yields* instead of *returning*, is because a function return implies that the function has finished executing, that there's nothing left for that function to do. A generator can *yield* as many values as a programmer desires. In fact, as we'll see later, you can make a generator *yield* values indefinetely. When the `return` expression is used in a generator, the generator exits and computation is finished. 

Following our class analogy, you can think of `next` as a class method. 

Let's look at our example again:

```python
def generator_func():
    a = 0
    yield a
    a = 10
    yield a
    
generator_inst = generator_func()
```

Let's pass the generator instance to the special `next` function.

``` python
>>> next(generator_inst)
0
```

Awesome! We got a value out of our generator! So what just happened? 

The generator started on line 1, and assigned `0` to the variable `a`. Then it executed the `yield a` statement and yielded the value of `a`. 

### _"So... is this it then?"_

Well, the generator is now **paused**: it stopped at the yield statement, and is waiting for us to call it again to resume its computation. So let's do just that:

``` python
>>> next(generator_inst)
10
```

We got the new value of `a` out! Let's try one more time:

``` python
>>> next(generator_inst)
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-5-50ee8ff2a4f6> in <module>
----> 1 next(generator_inst)

StopIteration:
```

Oh oh, something wrong must have happened there... But this is actually the expected behaviour: once the Python interpreter understand that there's not more code to be executed inside of a generator, it will promptly raise a special exception called `StopIteration`. In fact, the exact same thing will happen if we add a `return` statement at the end:

```python
def generator_func():
    a = 0
    yield a
    a = 10
    yield a
    return
    
generator_inst = generator_func()
next(generator_inst) # yields 0
next(generator_inst) # yields 10
next(generator_inst) # StopIteration is raised
```

```shell
0
10
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-5-50ee8ff2a4f6> in <module>
----> 1 next(generator_inst)

StopIteration:
```

You might be shaking your head and wondering why has it been designed this way. Can't the generator signal that it finished in some other way? Well, not really, not in Python at least. 

But this exception is actually very useful, it can be easily captured and will thus prevent unnecessary computation. 

You can of course watch out for the exception explcitely:

```python
generator_inst = generator_func()
while true:
    try:
        value = next(generator_inst)
        print(value)
    except StopIteration:
        break
print('done')
```

```shell
0
10
done
```

But the Python way would be do it in a for loop. Why? Because in additionto calling `next` on the generator for you, Python `for` loops automatically check for the `StopIteration` exception for you and will exit the loop context as soon as this exception is raised. 


```python
for value in generator_func(): # you can call the function immediately
    print(value)
print('done')
```

```shell
0
10
done
```

Much, much, much neater. 


## Generators are stateful

This means that in between calls, a generator is capable of storing state. Think of state as a variable or a collection of variables that holds values. A generator is thus capable of returning values that depend on that state. 

Let us illustrate it with another generator function where we have a stateful variable `a`:

``` python
def generator_func():
    a = 0
    while a < 5:
        a += 1
        yield a
        
g = generator_func()
for value in g:
    print(value)
```

```shell
1
2
3
4
5
```

Everytime the generator is called, it executes all the lines of code until it reaches a `yield` statement. During this execution, the value of `a` changes and is then yielded back to us. 

This functionality can be very useful if we need the output of the generator to depend on a previous state. 


## Pass values back to the generator

We have seen how to get values form a generator, but we are yet to see how we can **pass values back to the generator**. 

Yes, you heard that right. In addition to yielding a continuous stream of output values, generators can also accept a continuous stream of input values, which they can then use to compute the next output (they are stateful after all). 

To pass a value back to the generator, use the `.send(value)` method on the generator instance. 

Let's see how this works with an example:


```python
def my_generator(): 
    a = yield
    print('a:', a)
    a = yield
    print('a:', a)
    yield
    
g = my_generator()
next(g)
g.send(1)
g.send(2)
```

```shell
a: 1
a: 2
```

You might be wondering why we are calling the `next` function in the beggining. It's due to the order in which the `yield` statement executes its actions:

- it first returns the value that it's yielding and **pauses**
- it then assigns the value that it's passed with `.send(value)` and the generator continues running until it encounters the next `yield` statement

So can you retrieve and send values at the same time? Of course! In the following example we will print the stateful variables inside of the generator to better understand what's going on:

```python
def my_generator():
    a = 2
    print('1) state ->  a:', a)
    b = yield a
    
    print('2) state ->  a:', a, ', b:', b)
    b = yield a * b
    
    print('3) state ->  a:', a, ', b:', b)
    yield a * b
    
    print('4) state ->  a:', a, ', b:', b)

g = my_generator()
print('yield:', next(), '\n')
print('yield:', g.send(10), '\n')
print('yield:', g.send(200), '\n')
print('yield:', next(g), '\n')
```

```shell
1) state ->  a: 2
yield: 2 

2) state ->  a: 2 , b: 10
yield: 20 

3) state ->  a: 2 , b: 200
yield: 400 

4) state ->  a: 2 , b: 1000

---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-59-96abd9b795ea> in <module>
     16 print('yield:', gen.send(10), '\n')
     17 print('yield:', gen.send(100), '\n')
---> 18 print('yield:', gen.send(1000), '\n')
     19 print('yield:', next(gen), '\n')

StopIteration:
```

Note how the last call will raise a `StopIteration` exception. Remember: The exception is raised when the generator reaches the end of its code and there are no more `yield` statements left to execute. 


## Generators delay computations until necessary 

### A prime numbers generator

A more eloquent way of putting it would be to say that that generators **map an input value to a computation** as opposed to what classic functions do, which is mapping an input value to an output value. The computation is delayed until the value of that computation is explicitely requested. This paradigm is commonly referred to as **lazy evaluation**. 

Suppose you want to calculate all prime numbers up to a certain max integer. Normally you would have to calculate them all at once, wait for it to finish, and then proceed with the next thing you want to do with them. 

With the help of generators, you can retrieve the values as they are being computed. You could say you are *subscribing* to the stream the stream of prime numbers. This is a very powerful idea, since it allows us to do something with the values we already have without having to wait for the entire computation to finish. 

Let's start with a function that calculates whether a number is prime:

```python
def is_prime(num):
    for i in range(2, int(num**0.5) + 1):
        if num % i == 0:
            return False
    return True
```

Now let's write up the generator:

```python
def generate_primes(max_num):
    for num in range(2, max_num):
        if is_prime(num):
            yield num
```

And iterate over its values:

```python
for value in generate_primes(50):
    print(value)
```

```shell
2
3
5
7
11
13
17
19
23
29
31
37
41
43
47
```

### Animate a plot as prime numbers are being generated

Let's try something a bit more fancy. As the prime numbers are being computed, we plot them in polar coordinates on a graph. The prime number's value **p** will be both the distance from the origin **r** and the angle **θ** (in radians). The cartesian coordinates are thus $(p \ cos(p), p \ sin(p))$, ie $e^{i\pi}=-1$.

```python
import math
import time

import matplotlib.animation as animation
import matplotlib.pyplot as plt
import numpy as np
from matplotlib import style

def is_prime(num):
    for i in range(2, int(num**0.5) + 1):
        if num % i == 0:
            return False
    return True

def generate_primes(max_num):
    for num in range(2, max_num):
        if is_prime(num):
            yield num

def polar_to_cartesian(r, theta):
    x = r * math.cos(theta)
    y = r * math.sin(theta)
    return x, y

style.use('fivethirtyeight')
fig = plt.figure()
ax1 = fig.add_subplot(1,1,1)

N_MAX = 2500
g = generate_primes(N_MAX)
xs = []
ys = []

def animate(i):
    try:
        value = next(g)
    except StopIteration:
        return
    x, y = polar_to_cartesian(value, value)
    xs.append(x)
    ys.append(y)
    ax1.clear()
    ax1.scatter(xs, ys, s=5)
    ax1.set_xlim([-N_MAX, N_MAX])
    ax1.set_ylim([-N_MAX, N_MAX])
    
    fig.suptitle('Prime numbers in polar coordinates', fontsize=12)
    plt.xticks(fontsize=12)
    plt.yticks(fontsize=12)

anim = animation.FuncAnimation(fig, animate, interval=5)
plt.show()
```

Which should generate the following plot:

<!--![Prime Numbers Spiral](images/prime_numbers_spiral_2.gif) -->
{{< figure src="images/prime_numbers_spiral_2.gif" title="Prime Numbers Spiral"  width="100%" >}}


Do you notice how the prime number form these spirals from the origin? This plot illustrates, as Grant from [3blue1brown](https://www.youtube.com/channel/UCYO_jab_esuFRV4b17AJtAw/featured) has put it: 

> How pretty but pointless patterns in polar plots of primes prompt pretty important ponderings on properties of those primes. 

Definitely checkout his [video](https://www.youtube.com/watch?v=EK32jo7i5LQ&t=1s) on this to learn more (100% worth it). 

<br/>

## Infinite streams

One of the most useful things you can do with generators is creating inifite streams of values. One real example that I have encountered was when I was writing a SQL query with a lot of conditional `WHERE` clauses. The first condition needs to start with a `WHERE`, while all the following ones need to be preceded with `AND`. 

```sql
SELECT * FROM table
WHERE condition1
AND condition2
```

The problem we had was that we didn't know if any condition was going to be applied at all. Luckily, a simple generator came to the rescue:

```python
def where_generator():
    yield 'WHERE'
    while True:
        yield 'AND'

where = where_generator()
for _ in range(4):
    print(next(where))
```

```shell{promptUser: alice}{promptHost: dev.localhost}
WHERE
AND
AND
AND
```

And that's it!

If you want to go to the next level of Pythonista, try using the `itertools` library provided out of the box by Python. 

```python
from itertools import chain, repeat

where = chain(['WHERE'], repeat('AND'))
for _ in range(4):
    print(next(where))
```

```shell{promptUser: alice}{promptHost: dev.localhost}
WHERE
AND
AND
AND
```

Thank you for reading!
