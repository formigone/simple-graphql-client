# simple-graphql-client
...for now, this is just a set of best practices so your team can continue to love you.

## Why this repo?

I have yet to see a Python library that makes interacting with a GraphQL server easier, I'm hoping to write something to help the world. In the meantime, I will list common pitfalls and how to avoid them.

# Abstraction

In its simplest form, a GraphQL client will look something like this:

```python
import requests

def graphql_get(query, host=GRAPHQL_ENDPOINT_URL, key=GRAPHQL_ENDPOINT_KEY):
    """Perform  GraphQL API query and returns the output as a dict."""

    data = {'query': query}
    headers = {'x-api-key': key}

    req = requests.post(url=host, headers=headers, json=data)
    return req.json()
```

And a simple client would interact with it as follows

```python
import os

# Optionally set creds as module globals via environment variables (12 factor app?)
GRAPHQL_ENDPOINT_URL = os.getenv('GRAPHQL_ENDPOINT_URL')
GRAPHQL_ENDPOINT_KEY = os.getenv('GRAPHQL_ENDPOINT_KEY')

query = '''
  query ListUsers {
    listUsers(limit: 10) {
      userId
      email
    }
  }
'''

users = graphq_get(query)
```

# TODO

 + Write about parameterized, call-specific abstraction functions
