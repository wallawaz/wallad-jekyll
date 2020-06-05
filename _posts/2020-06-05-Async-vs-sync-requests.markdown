---
layout: post
title: Async network requests with aiohttp
date: 2020-06-04
---

A current project I am working on requires obtaining data from similar companies within the same market for analysis.
Since most of these entities do not provide an API, the only means I had to fetch data was to build scrapers.

While building these scrapers, I first went to the ever popular [Requests](https://2.python-requests.org/en/master/) library out of habbit.
However, since the data being accessed is not sequential in nature, I thought this would be a great opportunity to start exploring Python's concurrent capabilities with [asyncio](https://docs.python.org/3/library/asyncio.html).

<br />
Concurrency in **asyncio** is implemented with [Coroutines](https://www.python.org/dev/peps/pep-0492/).
From the [docs](https://docs.python.org/3/glossary.html):
> Coroutines are a more generalized form of subroutines. Subroutines are entered at one point and exited at another point.
Coroutines can be entered, exited, and resumed at many different points.

The main benefit of coroutines is their capability to release control of program execution so further actions can be performed.
An **event loop** is used to manage coroutines and switch program execution amoung them.

Although coroutines were added to Python in 3.3 with the `yield from` statement; 3.5 added the `async` and `await` keywords to the standard library which
enabled clearer definition. Coroutines are declared with `async def` and are suspended with `await`. 

```python
async def groot():
    return "groot"

async def i_am():
    who = await groot()
    return f"I am {who}"

# Since coroutines need to be run an an event loop, plainly calling one will
# only return a coroutine object.
who_am_i = groot()
print(who_am_i)
<coroutine object groot at 0x7f6970ca4a40>
```
Scheduling on `asyncio.event_loop`
```python
# Pre Python 3.7:
loop = asyncio.get_event_loop()
future = asyncio.ensure_future(i_am())
loop.run_until_complete(future)
'I am groot'

# 3.7+ :)
asyncio.run(i_am())
'I am groot'
```
Getting back to scrapers... after some Google searching, I learned that Requests is not capable of running within a coroutine.
This lead me to [aiohttp](https://docs.aiohttp.org/en/stable/index.html) which seems to be the defacto library for performing network requests within a coroutine in Python 3.

**aiohttp** interfaces with **asyncio** by adding [Tasks](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task) to the event_loop. Each **Task** runs a coroutine within the event loop until suspension `(await)`.
Once supspended, the event loop will switch execution to another Task. Effectively **Tasks** take care of switching between coroutines.
<br>
The task framework works well when performing many network requests where the majority of runtime is spent waiting on network responses.

---
<br>
To compare the runtime of concurrent vs sequential network requests, I built a simple [url-fetch](https://github.com/wallawaz/url-fetch-example) program which fetches five urls and calculates total runtime.
The program is invoked with the command `url-fetch-example` and accepts `aiohttp|requests` as the `fetch-type` argument.

```
url-fetch-example --help
Usage: url-fetch-example [OPTIONS] [requests|aiohttp]

Options:
  --version  Show the version and exit.
  --help     Show this message and exit.
```

`fetch_type` determines if UrlFetcher will perform its **GET** requests sequentially or asynchronously. When running asynchronously `asyncio.run()` is invoked to create the event loop and run the main coroutine.
```python
# cli.py
def main(fetch_type):
    fetcher = UrlFetcher()
    if fetch_type == "requests":
        fetcher.seq_main()
    else:
        asyncio.run(fetcher.async_main())
```

The sequential example is fairly straightforward:
```python
    # url_fetcher.py
    def seq_main(self):
        responses = []
        start = default_timer()
        for url in self.URLS:
            headers = self._get_random_user_agent()

            url_start = default_timer()
            # using requests content manager to match aiohttp logic
            with requests.Session() as session:
                responses.append(session.get(url))
                self.url_times[url] = default_timer() - url_start
```



The async example creates a task for each url by creating a `fetch` coroutine. Each fetch task is appended to `async_tasks`.
```python
    # url_fetcher.py
    async def async_main(self):
        start = default_timer()
        for url in self.URLS:
            task = asyncio.create_task(self.fetch(url))
            self.async_tasks.append(task)
```

The `fetch` coroutine will be suspended with `await session.get(url)`.
<br>
Once *resp* is resolved, the event loop will switch back and return *resp*.
```python
    # url_fetcher.py
    async def fetch(self, url):
        """Create a task to fetch a url asynchronously.
        Each task creates its own ClientSession to fetch with different
        headers."""
        headers = self._get_random_user_agent()
        self.url_times[url] = default_timer()
        async with aiohttp.ClientSession(headers=headers) as session:
            resp = await session.get(url) 
            self.url_times[url] = default_timer() - self.url_times[url]
            return resp
```
Tasks are coordinated and run with `asyncio.gather()`. Their results are saved to *responses*.
```python
    # url_fetcher.py
    responses = await asyncio.gather(*self.async_tasks)
```
---
### Runtime comparisons
## requests
```
ben@LAPTOP-8MSR27L7:~/repos/url-fetch-example$ poetry run url-fetch-example requests
'https://apple.com':  0.64 sec.
'https://bing.com':  0.38 sec.
'https://github.com':  0.15 sec.
'https://google.com':  0.25 sec.
'https://yahoo.com':  0.89 sec.
----------
total time: 2.3150834999978542
```
## aiohttp
```
ben@LAPTOP-8MSR27L7:~/repos/url-fetch-example$ poetry run url-fetch-example aiohttp
'https://apple.com':  0.26 sec.
'https://bing.com':  0.22 sec.
'https://github.com':  0.12 sec.
'https://google.com':  0.26 sec.
'https://yahoo.com':  0.49 sec.
----------
total time: 0.49253609997686
```
Comparing the fetch times to each url, on average aiohttp was 0.192 seconds faster in fetching a url.
<br>
However, when comparing the **overall** runtime, aiohttp fetched all urls **4.7x** faster than requests!
<br>
Moving forward, aiohttp will provide a huge performance boost when fetching multiple webpages simultaneously.
