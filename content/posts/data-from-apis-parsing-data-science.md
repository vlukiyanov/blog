+++
author = "Vladimir Lukiyanov"
title = "Robust API data gathering for data scientists"
date = "2020-12-12"
lastmod = "2020-12-16"
description = ""
tags = [
"python", "tutorial", "advanced", "api", "rest"
]
+++

There are a plenty of tutorials and guides on gathering data from REST APIs using Python, especially for data scientists; the aim of this tutorial is to present some ideas, components, and recipes which tackle issues a data scientist may encounter extending their code to run in a more robust setting. Using these techniques, data science code can be a little closer to data engineering.

Techniques aside, using APIs usually involves navigating incoherent documentation, a small myriad of standards (most of which aren't followed), and often struggling with existing SDKs - the advantage of HTTP based APIs is it is easy to write your own code to pull in the data, and with the right Python practices easy to solidify it.

All the code in this article can be found in the [python-api-examples](https://github.com/vlukiyanov/blog-examples/tree/main/python-api-examples) folder of [vlukiyanov/blog-examples](https://github.com/vlukiyanov/blog-examples) both as a [Jupyter notebook](https://github.com/vlukiyanov/blog-examples/blob/main/python-api-examples/python_api_examples/request.ipynb) and an equivalent [Python file](https://github.com/vlukiyanov/blog-examples/blob/main/python-api-examples/python_api_examples/request.py).

# Setting up

As a running example we are going to answer an example data science question using the [TfL API](https://api-portal.tfl.gov.uk/) - we'll analyse the number of connections between tube lines by their distance from the centre of London. 

To gain access to the [TfL API](https://api-portal.tfl.gov.uk/) you will need to register for an application key, which consists of an `app_id` and `app_key`. It is worth noting that for the [TfL API](https://api-portal.tfl.gov.uk/), response types 
are documented using OpenAPI 3 specifications - a relatively rare but helpful observation which in we ignore[^openapi] - the aim here is to describe a set of general methods, rather than focusing on the [TfL API](https://api-portal.tfl.gov.uk/) or writing API SDKs.

# URL structures with `yarl`

Manipulating URLs is a common operation when dealing with HTTP APIs, the two most common operations are adding paths to an endpoint and updating query parameters. Whilst you can do this using pure strings, [urllib](https://docs.python.org/3/library/urllib.request.html#urllib-examples) or [requests itself](https://requests.readthedocs.io/en/master/user/quickstart/#passing-parameters-in-urls), this can become unwieldy and prone to error. This is especially true when you are building up the URL iteratively. Using [yarl](https://github.com/aio-libs/yarl) can often improve this, as the example below demonstrates.

```python3
>>> from yarl import URL
>>> API_ENDPOINT = URL("https://api.tfl.gov.uk/")
>>> API_ENDPOINT / "Line" / "victoria" / "StopPoints" 
URL('https://api.tfl.gov.uk/Line/victoria/StopPoints')
>>> (API_ENDPOINT / "Line" / "victoria" / "StopPoints").update_query({"app_id": "...", "app_key": "..."})
URL('https://api.tfl.gov.uk/Line/victoria/StopPoints?app_id=...&app_key=...')
```

The [`yarl` documentation](https://yarl.readthedocs.io/en/latest/) is easy to read, and goes through most common operations on URLs. We can use `yarl` to create a generic function to authenticate our calls to the [TfL API](https://api-portal.tfl.gov.uk/) with the query parameters `app_id` and `app_key` we noted when registering, this can be done at the very end of a request:

```python3
import requests
from yarl import URL

def get_id_key():
    return {"app_id": APP_ID, "app_key": APP_KEY}

def call_get(url: URL) -> str:
    return requests.get(url.update_query(get_id_key())).text
```

This could then be used to create specialised function sto gather data from particular endpoints:

```python3
API_ENDPOINT = URL("https://api.tfl.gov.uk/")

def tube_line_ids() -> List[str]:
    return [
            item["id"]
            for item in json.loads(
                call_get(API_ENDPOINT / "Line" / "Mode" / "tube" / "Route")
            )
        ]
```

This function will return the ids of all TfL tube lines which we will use later[^cache].

# Managing API quotas with `ratelimit`

The next stage in improving `call_get` is to make sure it can operate without rate limit issues. Most APIs have rate limits or quotas to protect their backends from excessive load. These limits are often phrased as a combination of the number of requests in a given time period (e.g. 500 per minute), and the concurrency of your requests (e.g. no more than two requests at any time). When gathering data without concurrency, as is the case with any standard Python code, only the "500 per minute" type of limit is important, and this is where [`ratelimit`](https://pypi.org/project/ratelimit/) can help.

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

Calling an external API can result is a number of errors. A lot of these errors aren't handled by the underlying operating system and are passed to you from `requests`, and quite a lot of them are retryable. For `call_get` some examples of errors to retry include, if the server is overloaded and responds with a 500 response, or if your internet connection is interrupted; in addition to these errors, we also have to handle the `ratelimit.RateLimitException` which will be raised if we make requests too quickly.

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
)
@retry(
    retry=retry_if_exception_type(RateLimitException),
    wait=wait_fixed(1),
    stop=stop_after_delay(60),
)
@limits(calls=500, period=60)
def call_get(url: URL) -> str:
    return requests.get(url.update_query(get_id_key())).text
```

The first `@retry` above the `@limits` will be passed through first, and this will catch the `RateLimitException` and retry every second for a maximum of 60 seconds, the second decorator will soak up any other errors. One side effect is the `RateLimitException` will itself be retries up to 3 times. With limits and retries implemented, the extract part of our code should be able to run unsupervised, so we are able to gather the data we need for our analysis.
https://pydantic-docs.helpmanual.io/

# Parsing with `pydantic` and loading with `pandas`

It is rare for an API to return data in the format we need to perform analysis[^openapi], so there is usually an initial transformation stage where we turn the JSON (or if you're unlucky XML) data into a cleaned up row in a table. [Pydantic](https://pydantic-docs.helpmanual.io/) is a good library for initial data processing to a fixed self-documenting schema, a more powerful alternative to [dataclasses](https://docs.python.org/3/library/dataclasses.html). 

The [`pydantic` documentation](https://pydantic-docs.helpmanual.io/) is easy to read, and goes through most of the ideas.If we go back to `line_stop_points` defined as 

```python3
def line_stop_points(line_id: str):
    return json.loads(call_get(API_ENDPOINT / "Line" / line_id / "StopPoints"))
```

then calling this as `line_stop_points("victoria")` we get a list of dictionaries of deeply nested data, for example the first dictionary looks like (with some nesting elided):

```python3
{
    "$type": "Tfl.Api.Presentation.Entities.StopPoint, Tfl.Api.Presentation.Entities",
    "naptanId": "940GZZLUBLR",
    "modes": ["tube"],
    "icsCode": "1000024",
    "stopType": "NaptanMetroStation",
    "stationNaptan": "940GZZLUBLR",
    "hubNaptanCode": "HUBBHO",
    "lines": [
        {
            "$type": "Tfl.Api.Presentation.Entities.Identifier, Tfl.Api.Presentation.Entities",
            "id": "victoria",
            "name": "Victoria",
            "uri": "/Line/victoria",
            "type": "Line",
            "crowding": {
                "$type": "Tfl.Api.Presentation.Entities.Crowding, Tfl.Api.Presentation.Entities"
            },
            "routeType": "Unknown",
            "status": "Unknown",
        }
    ],
    "lineGroup": [...],
    "lineModeGroups": [...],
    "status": True,
    "id": "940GZZLUBLR",
    "commonName": "Blackhorse Road Underground Station",
    "placeType": "StopPoint",
    "additionalProperties": [...],
    "children": [...],
    "lat": 51.586919,
    "lon": -0.04115,
}
```

For our analysis the data we want to collect is:

* Name of the station 
* Distance from centre of London
* Number of connecting lines

In `pydantic` we can represent this as the following class:

```python3
class LineStopPointInfo(BaseModel):
    name: str
    distance: confloat(ge=0)  # from the centre of London, (51.509865, -0.118092)
    connections: conint(ge=0)
```

The fields like `pydantic.conint` and `pydantic.confloat` both put tighter bounds on the data that this class can store, `ge=0` means greater or equal to 0; `pydantic` supports [many of these fields](https://pydantic-docs.helpmanual.io/usage/types/).

Writing parsing to take the complex JSON into this format is then easy and self-documenting:

```python3
def parse_result(result) -> LineStopPointInfo:
    return LineStopPointInfo(
        name=result["commonName"],
        distance=haversine((51.509865, -0.118092), (result["lat"], result["lon"])),
        connections=len(
            [
                item["id"]
                for item in result.get("lines", [])
                if item["id"] in tube_line_ids()
            ]
        ) - 1,  # one result is always for the given line
    )

def line_stop_points(line_id: str) -> List[LineStopPointInfo]:
    return [
        parse_result(result)
        for result in json.loads(
            call_get(API_ENDPOINT / "Line" / line_id / "StopPoints")
        )
    ]
```

Running `line_stop_points("victoria")` we get the following:

```python3
[
    LineStopPointInfo(
        name='Blackhorse Road Underground Station',
        distance=10.085472401947493,
        connections=0
     ),
     LineStopPointInfo(
        name='Brixton Underground Station',
        distance=5.2583159839292755,
        connections=0
     ),
     LineStopPointInfo(
        name='Euston Underground Station',
        distance=2.245332897017276,
        connections=1
     ),
    ...
]
```

The advantage of this approach is not immediately obvious, especially as our initial parsing transformation is not terribly complicated; however if we try to load data into `LineStopPointInfo` which doesn't satisfy the schema definition, we will get an error. For example running `LineStopPointInfo(name="test", distance=0, connections=-1)` or `LineStopPointInfo(name="test", distance="whatever", connections=0)` will result in an error (note that `LineStopPointInfo(name=0, distance=0, connections=0)` will *not*, but this is likely what you would expect when working in Python as `0` can be coerced to a string)[^pydantic].

Finally we can load our clean dataset in `pandas`, the following function will convert the `pydantic` objects

```python3
def line_stop_df(line_id: str) -> pd.DataFrame:
    return pd.DataFrame([item.dict() for item in line_stop_points(line_id)])
```

Putting this all together as `pd.concat([line_stop_df(line) for line in tube_line_ids()])).drop_duplicates("name")` we have the following output. This box plot shows the distribution of tube stations distances by their number of connections - there is mostly a clear (and boring) correlation, stations with many tube line connections are closer to the centre.

{{% center %}}
![Graph showing distribution of distance by number of connections](/data-from-apis-parsing-data-science-1.png#center)
{{% /center %}}

The two outliers are Baker Street (5 lines) and King's Cross St Pancras (6 lines).

Here is the final code for our pipeline.

```python3
import json
from functools import lru_cache
from typing import List

import pandas as pd
import requests
import seaborn as sns
from haversine import haversine
from pydantic import BaseModel, confloat, conint
from ratelimit import RateLimitException, limits
from tenacity import *
from yarl import URL

sns.set_theme(style="whitegrid")
sns.set(rc={"figure.figsize": (10, 6)})

from python_api_examples.secrets import APP_ID, APP_KEY


def get_id_key():
    return {"app_id": APP_ID, "app_key": APP_KEY}


@retry(
    wait=wait_exponential(max=60, min=1),
    stop=stop_after_attempt(3),
)
@retry(
    retry=retry_if_exception_type(RateLimitException),
    wait=wait_fixed(1),
    stop=stop_after_delay(60),
)
@limits(calls=500, period=60)
def call_get(url: URL) -> str:
    return requests.get(url.update_query(get_id_key())).text


API_ENDPOINT = URL("https://api.tfl.gov.uk/")


@lru_cache()
def tube_line_ids() -> List[str]:
    return [
        item["id"]
        for item in json.loads(
            call_get(API_ENDPOINT / "Line" / "Mode" / "tube" / "Route")
        )
    ]


class LineStopPointInfo(BaseModel):
    name: str
    distance: confloat(ge=0)  # from the centre of London, (51.509865, -0.118092)
    connections: conint(ge=0)


def parse_result(result) -> LineStopPointInfo:
    return LineStopPointInfo(
        name=result["commonName"],
        distance=haversine((51.509865, -0.118092), (result["lat"], result["lon"])),
        connections=len(
            [
                item["id"]
                for item in result.get("lines", [])
                if item["id"] in tube_line_ids()
            ]
        )
        - 1,  # one result is always for the given line
    )


def line_stop_points(line_id: str) -> List[LineStopPointInfo]:
    return [
        parse_result(result)
        for result in json.loads(
            call_get(API_ENDPOINT / "Line" / line_id / "StopPoints")
        )
    ]


def line_stop_df(line_id: str) -> pd.DataFrame:
    return pd.DataFrame([item.dict() for item in line_stop_points(line_id)])


df = pd.concat([line_stop_df(line) for line in tube_line_ids()]).drop_duplicates("name")

plot = sns.boxenplot(
    x="connections",
    y="distance",
    color="b",
    scale="linear",
    data=df,
)

plot.figure.savefig("figure.png")
```

Alternatively you can view the code as a [Jupyter notebook](https://github.com/vlukiyanov/blog-examples/blob/main/python-api-examples/python_api_examples/request.ipynb).

[^cache]: Given this data is static, we could use the `functools.lru_cache` (or `functools.cache` in Python 3.9) to prevent refetching - further details in the [functools documentation](https://docs.python.org/3/library/functools.html).

[^openapi]: Not that if we were building an SDK for the TfL API or creating a software integration, then the OpenAPI 3 specifications mentioned above can be used to auto-generate classes in our code; you can read about this for for Pydantic, see the [`datamodel-codegen` library](https://pydantic-docs.helpmanual.io/datamodel_code_generator/).

[^pydantic]: Note that [`pydantic` documentation](https://pydantic-docs.helpmanual.io/) does have both a memory and processing footprint, this is only important if you are trying to optimise extremely large datasets - for most data readability and correctness will outweigh these considerations. It is worth noting that there are other ways to achieve this as part of your data quality pipeline, for example using [greatexpectations](https://greatexpectations.io/).