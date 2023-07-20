# asyncio and You

This guide will attempt to explain parts of `asyncio`, the built-in Python library for asynchronous code handling, and how they work with bot development. interactions.py uses `asyncio` for all of its asynchronous code handling, so it is important to understand how it works.

!!! warn "Note About This Guide"
    **This guide is not a full guide on `asyncio`** - it is only meant to explain the parts of `asyncio` that are relevant to bot development, and will simplify things in order to make it easier to understand.

    If you want to learn more about `asyncio`, you can read the [official documentation](https://docs.python.org/3/library/asyncio.html). There are also plenty of other guides out there that explain asynchronous programming in more detail - Googling is your friend.

## Basic Concept of `asyncio`

Programming languages has had an issue for a long time - it is very difficult to do multiple things at once. This is because most programming languages are made to be synchronous - they can only do one thing at a time. This is essentially like someone going down a list of chores they need to do and doing them one by one, never skipping any. This is fine for a large amount of programs, but for applications that need to be able to handle large amounts of requests or just need to do a lot of things, this is a problem.'

There are many ways of working around this. Perhaps the most common is creating "threads" or "processes" to split up the work (often called "parallelization") - this is like getting multiple people to go down the list of chores and letting them do a one each. This works, but it is very difficult to do correctly - you need to make sure none of the processes overlap, as otherwise it will cause issues. This is also very difficult to debug, as it is hard to tell what is happening in each process.

`asyncio` and asynchronous programming is another way of working around this. Instead of splitting up the work into multiple processes, it splits up the work into "tasks" - these tasks are then run one at a time, but if one task is waiting for something to happen (such as a request to Discord), it will move on to the next task before eventually switching back. This is like having one person go down the list of chores, but if they need to wait for something to happen (such as the dishwasher finishing), they will move on to the next chore and come back to the previous one when it is done. This has the benefit of being much easier to debug, as you can see exactly what is happening at each step, and being easier to use.

### The Event Loop

As mentioned earlier, tasks are ran one at a time, and can hop around from task to task as needed. This is all handled by the event loop - the event loop is what runs the tasks, and is what switch between tasks. It *is* a loop too - while it obviously gets much more complicated than this, the basic idea is that it runs through all the tasks, sees which tasks got deferred because it was waiting on something, and then runs through them again.

`asyncio` provides one such event loop for us to use. It is managed by said library itself, meaning interactions.py has no control of it, and so you don't need to worry about it too much. However, it is important to know that it exists, as it is what makes asynchronous programming possible.

### Coroutines

Coroutines are the building blocks of asynchronous programming. They are essentially functions that can be paused and resumed at any time - perfect for tasks for event loops.

In Python, they are marked by using `async def`:

```python
async def my_function():
    ...
```

...and ran by using `await`:

```python
await my_function()
```

As you can see, Python does a great deal to abstract what's really going on here - besides for these special keywords, using coroutines is almost exactly the same as using normal functions. You'll see them used a lot in interactions.py - all commands and events require asynchronous functions after all. Generally, if you need to `await` something, you probably want it in a coroutine declared by `async def`.

You can safely run a "normal" (synchronous/routine) function inside of a coroutine, but you cannot run a coroutine inside of a routine. We'll discuss how this can be worked around throughout this guide.

### `asyncio.run`

As mentioned earlier, you cannot run a coroutine inside of a routine. Also, we mentioned the event loop, but how do we start it anyways? We need a way to run the coroutine function to get everything else going...

Well, that's where [`asyncio.run`](https://docs.python.org/3/library/asyncio-runner.html#asyncio.run) comes in. `asyncio.run` is a function that will start the event loop and run a coroutine as the entry point. For example, take a look at this code:

```python
import asyncio

async def my_function():
    print("Well, that was fun!")

async def main():
    print("Hello, world!")
    await my_function()

asyncio.run(main())
```

Clearly, our code here needs a way into `main` in order to get the program running, and we can't do `main()` or `await main()`. However, we can do `asyncio.run(main())` - this will start the event loop and run `main` as the entry point instead. Once we've done that, we don't need to use `asyncio.run` again - we can just use `await` in our coroutine functions.

Note that `bot.start()` for interactions.py is essentially just a wrapper for `asyncio.run(bot.astart())` with a few bells and whistles - you can see that in its code. Thus, the event loop is only created when you properly start the bot if you use `bot.start()`.

## Blocking

You may have noticed that the event loop relies on functions telling it when to switch to another task through `await`. The event loop trusts that each function will not run long enough before running into an `await` to compromise on its functionality and speed. However, anything that's not marked with an `await`, from simple addition to opening a file, will not tell the event loop to switch tasks. This isn't an issue for simple things, but more complex synchronous functions can take a long time to run, and will "block" the event loop - prevent the event loop from being able to do anything, including running any tasks, until the function is done.

Naturally, this poses a problem for asynchronous programming, as your bot can completely hang thanks to these functions. These types of functions are found commonly in a couple of things:
- Many standard function/libraries that access a file (IE `open(...)` and image manipulation) are blocking. This is fine for small files, but due to how file reading works, this can take a long time for large files and become a major issue.
- Many traditional libraries that access the internet are blocking. Using the internet requires waiting for a response, which can be slow depending on the connection and the website. This can also become a major issue.
- Many traditional libraries that use a database are blocking. When making queries, the database may take a while to retrieve the data, which can become a major issue.

In general, if a synchronous function takes long enough for you to notice it is taking more than milliseconds of time, it is likely blocking. There are multiple solutions around this.

### Using Asynchronous Libraries

The best solution to blocking problems is to use asynchronous libraries. These libraries are designed to be used with asynchronous programming, and so they will not block the event loop. Common blocking libraries/functions and their replacements include:
- [`time.sleep`](https://docs.python.org/3/library/time.html#time.sleep) - use [`asyncio.sleep`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep) instead.
- [`requests`](https://requests.readthedocs.io/en/latest/) and [`urllib`](https://urllib3.readthedocs.io/en/stable/) - use [`aiohttp`](https://docs.aiohttp.org/en/stable/) (installed with interactions.py) or [`httpx`](https://www.python-httpx.org/) instead.
- [`open`](https://docs.python.org/3/library/functions.html#open) - use [`aiofiles`](https://pypi.org/project/aiofiles/) for large files, though smaller files are generally not an issue.
- Databases:
    - SQLite: [`sqlite3`](https://docs.python.org/3/library/sqlite3.html) - use [`aiosqlite`](https://pypi.org/project/aiosqlite/) instead.
    - MySQL: [`mysql-connector-python`](https://dev.mysql.com/doc/connector-python/en/) - use [`aiomysql`](https://aiomysql.readthedocs.io/en/latest/) instead.
    - Postgres: [`psycopg`](https://www.psycopg.org/psycopg3/docs/) - use [`asyncpg`](https://magicstack.github.io/asyncpg/current/) or the [asynchronous mode of `psycopg3`](https://www.psycopg.org/psycopg3/docs/advanced/async.html) instead.
    - MongoDB: [`pymongo`](https://pymongo.readthedocs.io/en/stable/) - use [`motor`](https://motor.readthedocs.io/en/stable/) instead.

### Executing Blocking Functions in a Thread

Some libraries or functions, like making and editing images through `Pillow`, do not have asynchronous alternatives. In these cases, you can use [`asyncio.to_thread`](https://docs.python.org/3/library/asyncio-task.html#running-in-threads) to run the function in a thread, which will prevent it from blocking the event loop. For example:

```python
import asyncio

def blocking_function(arg, *, arg2):
    # do something that takes a long time
    pass

async def my_function():
    await asyncio.to_thread(blocking_function, 1, arg2=2)
```

Note that this is not an ideal solution. Opening a lot of threads can slow a program down, and you may run into issues with the Python's GIL. However, it is a solution that works, and is better than nothing.
