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

Software engineers learn to deal with many data structures - different kinds of lists, arrays, sets, streams - but besides tabular data in Pandas or SQL many data scientists are likely to actively interact with just two in Python: lists like `[1,2,1]`, and dictionaries like `{"a": 1, "b": 2}`; sets like `{1,2,3}` are a possible third. In this post we present the argument for learning about and using iterators in the context of gathering data from APIs, though there are many other applications in data science; we will also discuss 3rd party libraries which reduce code duplication when working with iterators, in particular `cytoolz`. As a prerequisite you should have some idea about iterators and generator functions, as found in the [generators article](https://wiki.python.org/moin/Generators) on the Python wiki or Chapter 14 of Luciano Ramalho's [Fluent Python](https://learning.oreilly.com/library/view/fluent-python/9781491946237/).

# Setting up

As a running example we're going to use the [Guardian API](https://open-platform.theguardian.com/documentation/), and endpoint of which allows searching through the archives of articles using boolean queries. It's free to register for an API key.

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

We can decorate this function with rate limits and retries, details which are not directly important.

# Pagination and iterators

When making a call to a 3rd party API endpoint which can return an arbitrary number of results, pagination is quite common. There are two broad types of pagination[^pagination]:

* Next page cursor. You make a call to the API, and if there is a next page for your request you get a cryptic token `b9643810f4eb4011b39d98e2c71907c8` which you somehow inject into your next call; if there is no token you know there are no more pages of results.
* Offset, sometimes with page size. You specify a page number like `2` and either the response includes the number of pages, or you are expected to increment the page until you get an unsuccessful response.

In either case, after you make a call to the API, the next call is determined by some state and the previous result. For the [Guardian API](https://open-platform.theguardian.com/documentation/) [search endpoint](https://open-platform.theguardian.com/documentation/search) you could implement something like to accumulate a list of data for an API call that can be paginated:

```python
acc = []
for page in range(1, 20 + 1):
    response = call_get(url.update_query({"page_size": page_size, "page": page}))
    acc.extend(response["response"]["results"])
```

If you wrap this up into a function, you have to decide on what parameters to take, number of pages, perhaps number of final results. But is there a better way? We are using a list to accumulate the values, a list allows us to seek to a particular value, to mutate a particular value, to append, amongst others. Most of these properties are not very useful to the problem at hand, we are unwrapping an unknown number of values - here is where iterators come in. For the [Guardian API](https://open-platform.theguardian.com/documentation/) [search endpoint](https://open-platform.theguardian.com/documentation/search) the following is a possible implementation using a generator expression:

```python
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

To deconstruct this, we can start with `itertools.count(1)`; this is an infinite range `1, 2, 3, 4, 5, ...`. Until we hit a stop statement `return` for the generator expression, on demand we will call the API and yield the results one by one. Adding a `print(url)` statement within the `for` comprehension we can explore this iterator in more detail:

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

Calling `next` on an iterator either yields a single element or raises `StopIteration`, though it is rare to use this method directly. In the above example `i` contains results, so calling `next` yields a single result, while `j` contains no results so a `StopIteration` error is raised. These are all the operations on an iterator, the interface is quite simple, and results are evaluated lazily - no more data than needed is fetched. In the case where you ever need to evaluate an iterator non-lazily, just call `list(iterator)`, which will consume the whole iterator:

```python
In [1]: x = range(3)  # range implements the Iterator interface

In [2]: list(x)
Out[2]: [0, 1, 2]
```

# Operations on iterators - `itertools`

While the iterators interface is simple, there are a surprising number of fundamental operations available in [`itertools`](https://docs.python.org/3/library/itertools.html), which is part of the Python standard library; here are a handful of examples:

* Dropping elements from iterator using [`itertools.islice`](https://docs.python.org/3/library/itertools.html#itertools.islice): 
  ```python
  In [1]: import itertools

  In [2]: x = range(3)
    
  In [3]: list(itertools.islice(x, 1, 2))
  Out[3]: [1]
  ```
  This operation works from the start of the operator, unlike list slicing you cannot slide from the end.
* Taking elements from the iterator while a condition is satisfied using [`itertools.takewhile`](https://docs.python.org/3/library/itertools.html#itertools.takewhile): 
  ```python
  In [4]: z = range(10)

  In [5]:  list(itertools.takewhile(lambda y: y < 3, z))
  Out[5]: [0, 1, 2]
  ```
  This operation completes the resulting iterator once the condition is satisfied.
* Filtering elements which satisfy a certain condition using `filter`:
  ```python
  In [6]: list(filter(lambda x: x % 2 == 0, range(10)))
  Out[6]: [0, 2, 4, 6, 8]
  ```
* Map and comprehension, which can be used to transform an iterable, probably the most common operation:
  ```python
  In [13]: list(map(lambda y: 2 * y, range(3)))
  Out[13]: [0, 2, 4]
  
  In [14]: list((2 * y for y in range(3)))
  Out[14]: [0, 2, 4]

  In [15]: list((2 * y for y in range(3) if y < 3))
  Out[15]: [0, 2, 4]
  ```

In the context of the [Guardian API](https://open-platform.theguardian.com/documentation/) example, we can use the above to take the first 10 posts about Boris Johnson in the Politics section, first we create a wrapper method:

```python
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
    Given a search query like "boris OR johnson", as documented on the Guardian
    API, return an iterable of SearchResult objects; the API documents the
    results ordered by the publication date.

    :param q: query, for example "boris OR johnson"
    :return: iterable of SearchResult objects
    """
    url = (URL("https://content.guardianapis.com") / "search").with_query({"q": q})
    return map(SearchResult.parse_obj, iter_call_get(url))
```

Which we can then compose using `itertools.islice` and a comprehension:

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

# Operations on iterators - `cytoolz`


[^pagination]: For the API developer the choice is affected by the implementation of the backend or database; as a user you don't usually get a choice.