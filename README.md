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

# Parameterizing queries

Often you'll need to write queries that include variable interpolation. The first rule of writing useful code is to write readable code.

> The first rule of writing useful code is to write readable code. ~Me

Here's what you *should not* do:

```python
def get_users(limit=10):
   query = '''
    query ListUsers {
        # DO NOT concatenate string -----+
        #                                |
        #                   +------------+
        #                   |
        #                   V
        listUsers(limit:''' + limit + ''') {
           userId
           email
        }
    }
   '''
```

In other words:

 1. DON'T create parameterized GraphQL queries by concatenating strings. This gets very hard to read very quickly, meaning that very soon, you won't be able to tell for sure if the query is malformed or not.
 
Here's what you *should do* instead:

```python
def get_users(limit=10):
   query = '''
    query ListUsers {{
        # Note: ---------+--------------------+
        #                |  single brackets   |
        #                | /                  |
        #                |         +----------+
        #                |         |  double brackets
        #                |         | /
        #                V         V
        listUsers(limit: {limit}) {{
           userId
           email
        }}
    }}
   '''.format(limit=limit)
```

In other words:

 2. DO use Python's built-in `.format()` method on string type. This way you can write the query as a single string, and variables can be interpolated in a way that adds minimal noise to it. The main thing to remember here is that when you need a literal opening or closing bracket, you'll need two of them. A single `{}` pair indicates that a variable will be added there using the `.format()` method. You can add a token within single brackets as a placeholder for the `.format()` method. Example: `'hello, {name}'.format(name="Dave")`.
