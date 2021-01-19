+++
author = "Vladimir Lukiyanov"
title = "Paginating APIs with generators for data scientists"
date = "2021-01-14"
lastmod = "2020-01-17"
description = ""
tags = [
"python", "tutorial", "advanced", "api", "rest", "pydantic", "yarl", "pandas"
]
+++

Software engineers learn to deal with many data structures - different kinds of lists, arrays, sets, streams - but besides tabular data in Pandas or SQL data scientists are likely to actively interact with just two in Python: lists like `[1,2,1]`, and dictionaries like `{"a": 1, "b": 2}`; sets like `{1,2,3}` are a possible third. In this post we present the argument for learning about and using iterators in the context of gathering data from APIs; we will also discuss 3rd party libraries which reduce code duplication when working with iterators, in particular `cytoolz`. As a prerequisite you should have some idea about iterators and generator functions, as found in the [generators article](https://wiki.python.org/moin/Generators) on the Python wiki or Chapter 14 of Luciano Ramalho's [Fluent Python](https://learning.oreilly.com/library/view/fluent-python/9781491946237/).

# Setting up

As a running example we're going to use the [Guardian API](https://open-platform.theguardian.com/documentation/) which allows search through the archives of articles using boolean queries. It's free to register for an API key.

For calling the API, following a similar scheme as in [my previous post about robust API data gathering](https://vlukiyanov.github.io/robust-api-data-gathering-in-python-for-data-scientists/), we define a generic function to call the [Guardian API](https://open-platform.theguardian.com/documentation/):

```python
def call_get(url: URL) -> Dict[str, Any]:
    """
    Given a URL to call the Guardian API, inject the API key, make the call
    and return the resulting parsed JSON; this function handles per second
    rate limits and retries.

    :param url: URL to make a GET request on for the Guardian API
    :return: parsed JSON result, as a dictionary
    """
    return requests.get(url.update_query({"api-key": GUARDIAN_API_KEY})).json()
```

# Pagination and iterators

