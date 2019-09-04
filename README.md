# simple-graphql-client
...for now, this is just a set of best practices so your team can continue to love you.

## Why this repo?

I have yet to see a Python library that makes interacting with a GraphQL server easier, I'm hoping to write something to help the world. In the meantime, I will list common pitfalls and how to avoid them.

# API Abstraction

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

### TL;DR

 1. **DON'T** create parameterized GraphQL queries by concatenating strings. 

 2. **DO** use Python's built-in `.format()` method on string type. 

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

   return graphql_get(query)
```

Concatenating strings gets very hard to read very quickly, meaning that very soon, you won't be able to tell for sure if the query is malformed or not. This is especially so when you have to quote tokens in the query, so you're mixing single, double, and triple quotes: `'''name: "''' + some_var . + '''"'''`.

Here's what you *should do* instead:

```python
def get_users(limit=10):
   query = '''
    query ListUsers {{
        #            ^
        #            | double braces
        # Note: +----+
        #       |
        #       +--------+
        #                | single braces
        #                V
        listUsers(limit: {limit}) {{
           userId
           email
        }}
    }}
   '''.format(limit=limit)

   return graphql_get(query)
```

This way you can write the query as a single string, and variables can be interpolated in a way that adds minimal noise to it. The main thing to remember here is that when you need a literal opening or closing curly brace, you'll need two of them. A single `{}` pair indicates that a variable will be added there using the `.format()` method. You can add a token within single braces as a placeholder for the `.format()` method. Example: `'hello, {name}'.format(name="Dave")`.

# Query abstraction

When making a single to the API, it is helpful to also abstract out the fact that the data is being fetch from an API. That way the client truly only knows that calling some function will return some _generic_ object (typically, probably a dict). For example, the sample function `get_users()` above does just that.

The pitfall here is that, as it stands, the abstracted function `get_users()` leaks a bit of information about where the data is coming from. That is, the object returned by that function contains the raw GraphQL output:

```python
get_users(query)
# {
#   'data': {
#     'listUsers': [
#        {'userId': 123, 'email': 'user@email.com'},
#        {'userId': 456, 'email': 'person@email.com'},
#        ...
#     ]
#   },
#   'errors': [
#      {'message': 'Oops, something broke', 'errorType': 'MalformedHTTPRequest'},
#      ...
#   ]
# }
```

Notice that the response is nested inside an attribute `data`, inside which is the  name of the actual GraphQL query (or mutation). In order for the client to get to the data, it'll need to know the name of the query, etc. Also, if there are errors, instead of returning a non 200 HTTP code, the API will return a 200 with a payload within the `errors` attribute.

A better abstraction for that call would look for errors in the response and raise actual exceptions, and it would also "unroll" that response so all the client gets is the actual data that's by default nested in the response:

```python
def get_users(limit=10):
   query = '''
    query ListUsers {{
        #            ^
        #            | double braces
        # Note: +----+
        #       |
        #       +--------+
        #                | single braces
        #                V
        listUsers(limit: {limit}) {{
           userId
           email
        }}
    }}
   '''.format(limit=limit)

   resp = graphql_get(query)

   if resp.get('errors'):
      raise Exception(resp['errors'][0]['message'])

   return resp['data']['listUsers']
```

Now the client can anticipate a runtime exception, but also gets the data at the top level response object: 

```python
try:
    get_users(query)
    # [
    #   {'userId': 123, 'email': 'user@email.com'},
    #   {'userId': 456, 'email': 'person@email.com'},
    #   ...
    # ]
except Exception error:
    print('Error  :( \n{}'.format(str(error)))
```
