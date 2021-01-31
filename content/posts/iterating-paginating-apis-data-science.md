+++
author = "Vladimir Lukiyanov"
title = "Paginating APIs with generators for data scientists"
date = "2021-01-14"
lastmod = "2020-01-31"
description = ""
tags = [
"python", "tutorial", "advanced", "api", "rest", "pydantic", "yarl", "pandas"
]
+++

Software engineers deal with many data structures - different kinds of lists, arrays, sets, streams - but besides tabular data in Pandas or SQL many data scientists are likely to actively interact with just two in Python: lists like `[1,2,1]`, and dictionaries like `{"a": 1, "b": 2}`; sets like `{1,2,3}` are a possible third. In this post we present the argument for learning about and using iterators in the context of gathering data from APIs, though there are many other applications in data science. We will also discuss 3rd party libraries which reduce code duplication when working with iterators, in particular `cytoolz`. As a prerequisite you should have some idea about iterators and generator functions, as found in the [generators article](https://wiki.python.org/moin/Generators) on the Python wiki or Chapter 14 of Luciano Ramalho's [Fluent Python](https://learning.oreilly.com/library/view/fluent-python/9781491946237/).

[also good] https://treyhunner.com/2016/12/python-iterator-protocol-how-for-loops-work/

# Setting up

As a running example we're going to use the [Guardian API](https://open-platform.theguardian.com/documentation/), and endpoint of which allows searching through the archives of articles using boolean queries. It's free to register for an API key.

For calling the API, following a similar scheme as in [the previous post about robust API data gathering](https://vlukiyanov.github.io/robust-api-data-gathering-in-python-for-data-scientists/), we define a generic function to call the [Guardian API](https://open-platform.theguardian.com/documentation/):

```python
from typing import Any, Dict

import requests
from yarl import URL

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

For completeness, we could decorate this function with rate limits and retries - these are details which we omit when they are not important to the discussion.

# Pagination and iterators

When making a call to a 3rd party API endpoint which can return an arbitrary number of results, pagination is quite common. There are two broad types of pagination[^pagination]:

* _Next page cursor_. You make an initial call to the API, and if there is a next page for your request you get a cryptic token like `b9643810f4eb4011b39d98e2c71907c8` which you inject into your next call; if there is no token you know there are no more pages of results.
* _Offset, sometimes with page size_. You specify a page number like `2` and either the response includes the number of pages, or you are expected to increment the page until you get an unsuccessful response.

In either case, after you make a call to the API, the next call is determined by some state and the previous result. For the [Guardian API](https://open-platform.theguardian.com/documentation/) [search endpoint](https://open-platform.theguardian.com/documentation/search) you could implement something like the following to accumulate a list of data for a paginated API call:

```python
acc = []
for page in range(1, 20 + 1):
    response = call_get(url.update_query({"page_size": page_size, "page": page}))
    acc.extend(response["response"]["results"])
```

If you were to wrap this up into a function, you would have to decide on what parameters to take, number of pages, perhaps number of final results. Is there a better way? An observation - we are using a Python list to accumulate the values, a Python list allows us to seek to a particular value, to mutate a particular value, to append, amongst other properties. Most of these properties are not useful to the problem at hand, we are unwrapping an unknown number of values - and here is where iterators come in. For the [Guardian API](https://open-platform.theguardian.com/documentation/) [search endpoint](https://open-platform.theguardian.com/documentation/search) the following is a possible implementation of the same code using a generator expression:

```python
from typing import Iterable

from yarl import URL

def iter_call_get(url: URL, page_size: int = 50) -> Iterable[Dict[str, Any]]:
    """
    Given a URL to call the Guardian API, paginate in page size of page_size,
    yielding the results of the search one by one until no more data is
    available; the API documents the results ordered by the publication date.

    :param url: URL to paginate, like URL("https://content.guardianapis.com/search?q=debates")
    :param page_size: page size for the API, default 50, maximum 50
    :return: dictionary results of the nested list "response" -> "results"
    """
    for page in itertools.count(1):
        response = call_get(url.update_query({"page_size": page_size, "page": page}))
        results = response["response"].get("results", [])
        if not len(results):
            return
        yield from results
        if page == response["response"]["pages"]:
            return
```

To deconstruct this we proceed line by line. First note that `itertools.count(1)` which is the infinite range `1, 2, 3, 4, 5, ...`. Until we hit one of the stop statements `return` for the generator expression, when there is demand we will call the API and yield the results one by one, incrementing the page when more results are needed. Adding a `print(url)` statement within the `for` loop we can observe this iterator in action:

```python
In [1]: i = iter_call_get(URL("https://content.guardianapis.com/search?q=debates"))

In [2]: next(i)
Out[2]: 
{'id': 'us-news/2020/sep/30/presidential-debates-format-overhauled-trump-biden',
 'type': 'article',
 'sectionId': 'us-news',
 'sectionName': 'US news',
 'webPublicationDate': '2020-09-30T18:22:56Z',
 'webTitle': 'Presidential debates format to be overhauled after calamity in Cleveland',
 'webUrl': 'https://www.theguardian.com/us-news/2020/sep/30/presidential-debates-format-overhauled-trump-biden',
 'apiUrl': 'https://content.guardianapis.com/us-news/2020/sep/30/presidential-debates-format-overhauled-trump-biden',
 'isHosted': False,
 'pillarId': 'pillar/news',
 'pillarName': 'News'}
 
In [3]: j = iter_call_get(URL("https://content.guardianapis.com/search?q=abcdefghijk"))

In [4]: next(j)
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-4-36a6314a8ca5> in <module>
----> 1 next(j)

StopIteration:  
```

Calling `next` on an iterator either yields a single element or raises `StopIteration` (though it is rare to use this method directly). In the above example, `i` contains results so calling `next` yields a single result after making a single API call, while `j` contains no results so a `StopIteration` error is raised after making a single API call. These are all the operations on an iterator, so the interface is quite simple, and results are evaluated lazily - no more data than needed is fetched. In the case where you ever need to evaluate an iterator non-lazily, just call `list(iterator)`, which will consume the whole iterator:

```python
In [1]: x = iter(range(3))

In [2]: list(x)
Out[2]: [0, 1, 2]
```

Neither using `next` for consuming an iterator using `list` is the usual way of interacting with an iterator, the for loop is the most common way of consuming data.

# Operations on iterators - `itertools`

While the iterators interface is simple, there are a number of operations available in the standard library off which [`itertools`](https://docs.python.org/3/library/itertools.html) is part of. Here are a handful of examples:

* Dropping elements from iterator using [`itertools.islice`](https://docs.python.org/3/library/itertools.html#itertools.islice): 
  ```python
  In [1]: import itertools

  In [2]: x = iter(range(3))
    
  In [3]: list(itertools.islice(x, 1, 2))
  Out[3]: [1]
  ```
  This operation works from the start of the iterator. Unlike list slicing you cannot slice from the end.
* Taking elements from the iterator until a condition is no longer satisfied using [`itertools.takewhile`](https://docs.python.org/3/library/itertools.html#itertools.takewhile): 
  ```python
  In [4]: z = iter(range(10))

  In [5]:  list(itertools.takewhile(lambda y: y < 3, z))
  Out[5]: [0, 1, 2]
  ```
  In essence this operation completes the resulting iterator once the condition is no longer satisfied.
* Filtering an iterator to only elements which satisfy a certain condition using `filter`:
  ```python
  In [6]: list(filter(lambda x: x % 2 == 0, iter(range(10))))
  Out[6]: [0, 2, 4, 6, 8]
  ```
* Map and for comprehension to transform an iterable, probably the most common operation:
  ```python
  In [7]: list(map(lambda y: 2 * y, iter(range(3))))
  Out[7]: [0, 2, 4]
  
  In [8]: list((2 * y for y in iter(range(3))))
  Out[8]: [0, 2, 4]

  In [9]: list((2 * y for y in iter(range(3)) if y < 3))
  Out[9]: [0, 2, 4]
  ```

In the context of the [Guardian API](https://open-platform.theguardian.com/documentation/) example, we can combine the above to take the first 10 posts about Boris Johnson in the Politics section. First we create a wrapper method:

```python
from pydantic import BaseModel

class SearchResult(BaseModel):
    id: str
    type: str
    sectionId: Optional[str]
    sectionName: Optional[str]
    webPublicationDate: datetime
    webTitle: str
    webUrl: Optional[pydantic.AnyUrl]
    apiUrl: Optional[pydantic.AnyUrl]
    isHosted: Optional[bool]
    pillarId: Optional[str]
    pillarName: Optional[str]

def iter_search(q: str) -> Iterable[SearchResult]:
    """
    Given a search query like "boris OR johnson", as documented in the Guardian
    API, return an iterable of SearchResult objects; the API documents the
    results ordered by the publication date.

    :param q: query, for example "boris OR johnson"
    :return: iterable of SearchResult objects
    """
    url = (URL("https://content.guardianapis.com") / "search").with_query({"q": q})
    return map(SearchResult.parse_obj, iter_call_get(url))
```

We can then compose the above using `itertools.islice` and a comprehension:

```python
# Example query, first 10 results in the "Politics" section
posts = itertools.islice(
    (
        item
        for item in iter_search("boris OR johnson")
        if item.sectionName == "Politics"
    ),
    0,
    10
)
```

This is not dissimilar to operations with Pandas - we apply a filter and then take the head.

# Operations on iterators - `cytoolz`

There are a number of 3rd party Python libraries which provide additional operations on iterators, of which `cytoolz` is a good example (also check out [`funcy`](https://funcy.readthedocs.io/en/stable/)). Mostly these libraries imitate APIs found in Clojure, Scala and other functional languages. Here are a handful of examples with `cytoolz`:

* Simplified slicing with [`cytoolz.take`](https://toolz.readthedocs.io/en/latest/api.html#toolz.itertoolz.take), [`cytoolz.first`](https://toolz.readthedocs.io/en/latest/api.html#toolz.itertoolz.first), [`cytoolz.last`](https://toolz.readthedocs.io/en/latest/api.html#toolz.itertoolz.last) and [`cytoolz.drop`](https://toolz.readthedocs.io/en/latest/api.html#toolz.itertoolz.drop):
  ```python
  In [1]: import cytoolz

  In [2]: cytoolz.itertoolz.last(iter(range(3)))
  Out[2]: 2

  In [3]: cytoolz.itertoolz.first(iter(range(3)))
  Out[3]: 0

  In [4]: cytoolz.itertoolz.take(2, iter(range(3)))
  Out[4]: <itertools.islice at 0x7fab844eda10>

  In [5]: list(cytoolz.itertoolz.take(2, iter(range(3))))
  Out[5]: [0, 1]

  In [6]: list(cytoolz.itertoolz.drop(2, iter(range(3))))
  Out[6]: [2]
  ```
  While these are shorthands for operations that can be implemented relatively easily without 3rd party libraries, they can aid in readability.

* Generating sequences of repeated function applications using [`cytoolz.itertoolz.iterate`](https://toolz.readthedocs.io/en/latest/api.html#toolz.itertoolz.iterate), where given a function `f` and a starting value `x`, we generate an infinite iterator `x, f(x), f(f(x)), ...`. A simple example is the following:
  ```python
  In [7]: list(cytoolz.itertoolz.take(5, cytoolz.itertoolz.iterate(lambda x: x + 1, 0)))
  Out[7]: [0, 1, 2, 3, 4]
  ```
  
* Grouping into a dictionary with a key function using [`cytoolz.itertoolz.groupby`](https://toolz.readthedocs.io/en/latest/api.html#toolz.itertoolz.groupby), which should not be confused with the [`itertools.groupby`](https://docs.python.org/3/library/itertools.html#itertools.groupby) [^beware]:
  ```python
  In [8]: cytoolz.itertoolz.groupby(lambda x: x % 2, iter(range(10)))
  Out[8]: {0: [0, 2, 4, 6, 8], 1: [1, 3, 5, 7, 9]}
  ```

* Sliding partitions, akin to n-grams, using [`cytoolz.itertoolz.sliding_window`](https://toolz.readthedocs.io/en/latest/api.html#toolz.itertoolz.sliding_window):
  ```python
  In [9]: list(cytoolz.itertoolz.sliding_window(3, iter(range(5))))
  Out[9]: [(0, 1, 2), (1, 2, 3), (2, 3, 4)]
  ```

* Splitting an input into equally sized chunks using [`cytoolz.itertoolz.partition`](https://toolz.readthedocs.io/en/latest/api.html#toolz.itertoolz.partition) and [`cytoolz.itertoolz.partition_all`](https://toolz.readthedocs.io/en/latest/api.html#toolz.itertoolz.partition_all):
  ```python
  In [10]: list(cytoolz.itertoolz.partition_all(3, iter(range(10))))
  Out[10]: [(0, 1, 2), (3, 4, 5), (6, 7, 8), (9,)]
  
  In [11]: list(cytoolz.itertoolz.partition(3, iter(range(10))))  
  Out[11]: [(0, 1, 2), (3, 4, 5), (6, 7, 8)]
  ```
  
There are many other small gems hidden in `cytoolz`, and an additional curried interface which can create quite functional code, but which may often make code look harder to read. Here is an example of the curried interface, if you like this style of programming see [].

[^beware]: [`itertools.groupby`](https://docs.python.org/3/library/itertools.html#itertools.groupby) behaves differently to other standard libraries, e.g. Scala, as well as the SQL `GROUP BY` - it can still be useful but this is something to be aware of. The [`cytoolz.itertoolz.groupby`](https://toolz.readthedocs.io/en/latest/api.html#toolz.itertoolz.groupby) is more akin to SQL `GROUP BY` and other standard libraries like Scala.

[^pagination]: For the API developer the choice is affected by the implementation of the backend or database; as a user you don't usually get a choice.