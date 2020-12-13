+++
author = "Vladimir Lukiyanov"
title = "Data from APIs in Python"
date = "2020-12-12"
lastmod = "2020-12-12"
description = ""
tags = [
"python", "tutorial", "advanced", "api", "rest"
]
+++

There are a plenty of tutorials and guides on gathering data from APIs in Python, especially for data scientists. Instead of repeating the basics, this guide presents some ideas, components, and recipes which tackle issues a data scientist may encounter extending their code to run in a more robust setting.

Using APIs usually involves navigating incoherent documentation, a small myriad of standards (most of which aren't followed), and often struggling with existing SDKs - the advantage of HTTP based APIs is it is easy to write your own code to pull in the data, and with the right Python practices easy to solidify it.

# Setting up

We are going to be using the [TfL Unified API](https://api-portal.tfl.gov.uk/) as a running example. To gain access to this API you will need to register for an application key, which consists of an `app_id` and `app_key`.

# URL structures with `yarl`

Manipulating URLs is a common operation when dealing with HTTP APIs, the two most common operations are adding paths to an endpoint and updating query parameters. Whilst you can do this using pure strings, [urllib](https://docs.python.org/3/library/urllib.request.html#urllib-examples) or [requests itself](https://requests.readthedocs.io/en/master/user/quickstart/#passing-parameters-in-urls), this can become unwieldy and prone to error, especially when you are building up the URL iteratively. Using [yarl](https://github.com/aio-libs/yarl) can often improve this, as the example below demonstrates.

```python3
>>> from yarl import URL
>>> API_ENDPOINT = URL("https://api.tfl.gov.uk/")
>>> API_ENDPOINT / "Line" / "victoria" / "StopPoints" 
URL('https://api.tfl.gov.uk/Line/victoria/StopPoints')
>>> (API_ENDPOINT / "Line" / "victoria" / "StopPoints").update_query({"app_id": "...", "app_key": "..."})
URL('https://api.tfl.gov.uk/Line/victoria/StopPoints?app_id=...&app_key=...')
```

The [`yarl` documentation](https://yarl.readthedocs.io/en/latest/) is easy to read, and goes through most common operations on URLs. We can use yarl to create a generic function to call the [TfL Unified API](https://api-portal.tfl.gov.uk/) with the query parameters `app_id` and `app_key` we noted when registering, this can be done at the very end of a request:

```python3
import requests
from yarl import URL

def get_id_key():
    return {"app_id": APP_ID, "app_key": APP_KEY}

def call_get(url: URL) -> str:
    return requests.get(url.update_query(get_id_key())).text
```

This could then be used to create specialised function to gather data from particular endpoints:

```python3
API_ENDPOINT = URL("https://api.tfl.gov.uk/")

def line_stop_points(line_id: str) -> str:
    return call_get(API_ENDPOINT / "Line" / line_id / "StopPoints")
```

At the moment these just return the unparsed strings, and we'll build these up with sundries such as parsing.

# Managing API quotas with `ratelimit`

Most APIs have rate limits or quotas to protect their backends from excessive load. These limits are often phrased as a combination of the number of requests in a given time period (e.g. 500 per minute), and the concurrency of your requests (e.g. no more than two requests at any time). When gathering data without concurrency, as is the case with any standard Python code, only the "500 per minute" type of limit is important, and this is where [`ratelimit`](https://pypi.org/project/ratelimit/) can help.

The `ratelimit` functionality is imported as a [decorator](https://realpython.com/primer-on-python-decorators/), an advanced language feature which allows modifying the behaviour of a function. For example, to limit our `call_get` function to a maximum of 500 calls every minute, we write:

```python3
import requests
from ratelimit import limits
from yarl import URL

def get_id_key():
    return {"app_id": APP_ID, "app_key": APP_KEY}

@limits(calls=500, period=60)
def call_get(url: URL) -> str:
    return requests.get(url.update_query(get_id_key())).text
```

This means that when our code calls `call_get` with some input, this call is routed through the `@limits(calls=500, period=60)` decorator which applies the rate limit logic, keeping a counter of calls; if we call `call_get` more than 500 times in a one-minute interval then an exception `ratelimit.RateLimitException` will be raised.

# Handling failures with `tenacity`

Calling an external API over the internet can result is a number of errors. A lot of these errors aren't handled by the underlying operating system and are passed to you from requests, and quite a lot of them are retryable. For `call_get` some examples of errors to retry include, if the server is overloaded and responds with a 500 response, or if your internet connection is interrupted; in addition to these errors, we also have to handle the `ratelimit.RateLimitException` which will be raised if we make requests too quickly.

There are a number of Python libraries to choose from, [`tenacity`](https://github.com/jd/tenacity) is a maintained and modern library; as with `ratelimit` it works using a decorator. The [`tenacity` documentation](https://tenacity.readthedocs.io/en/latest/) is easy to read, though the style of the library means you have to do an `from tenacity import *` import to be able to copy and paste from the examples. To handle the `RateLimitException` and general exceptions we can write something like this.

```python3
import requests
from ratelimit import limits, RateLimitException
from tenacity import *
from yarl import URL

def get_id_key():
    return {"app_id": APP_ID, "app_key": APP_KEY}

@retry(
    wait=wait_exponential(max=60, min=1),
    stop=stop_after_attempt(3),
    before=before_log(logger, logging.DEBUG),
)
@retry(
    retry=retry_if_exception_type(RateLimitException),
    wait=wait_fixed(1),
    stop=stop_after_delay(60),
    before=before_log(logger, logging.DEBUG),
)
@limits(calls=500, period=60)
def call_get(url: URL) -> str:
    return requests.get(url.update_query(get_id_key())).text
```

The first `@retry` above the `@limits` will be passed through first, and this will catch the `RateLimitException` and retry every second for a maximum of 60 seconds, the second decorator will soak up any other errors. One side effect is the `RateLimitException` will itself be retries up to 3 times.

# Parsing with `pydantic`

It is rare for an API to return data in the format we need, there is always an initial transformation stage. 

# Loading with `pandas`

# Handling pagination with iterators

# Testing with `requests-mock`

# Scaling out using `gevent`

# Threading it all together
