Welcome to my blog. This is a collection of notes, projects, etc. that I want to share with the programming multiverse. Right now I'm interested in the following topics in no particular order: NoSQL, Cassandra, HBase, Machine Learning, Data Science, Statistics, Mathematics, Python, Javascript, Crystal, Django, AIOHTTP, Asynchronous Programming, React.js, PySpark, Sentiment Analysis, Deep Learning, Twitter data. Hopefully you can find something interesting/fun on my site :)

You can contact me at boulderwebdev94 (@t) g m a i 1 .com with comments, questions, etc.

# Decorators v.s. Middleware in AIOHTTP for Handling JSON Requests
**5/26/2018**

### Problem
While playing around with [AIOHTTP](https://docs.aiohttp.org/en/stable/index.html) and learning how to use its server interface, one of the problems I faced was figuring out how to protect JSON-only routes from non-json requests. If you write something like
``` python
@routes.post("/my-route")
async def my_route(req):
    data = await req.json()
    ...
```
you are given a `json.decoder.JSONDecodeError` whenever someone tries to send a non-json request. There are two potential solutions to this problem but both introduce some rigidity to how you write your programs. The first of which is to construct a middleware which takes in a request, checks if the handler is a JSON-only route, and then sends the request with the data on its way. The other solution is to implement a decorator which performs the same type of actions. Since both of these methods seem valid it looks like the only metric which remains is to look at how fast both perform. Please note that I have a complete copy of the code at the bottom of this post, so it may be helpful to play around with it while reading.

### Middleware approach
Setting up [middleware in AIOHTTP](https://docs.aiohttp.org/en/stable/web_advanced.html#middlewares) is a relatively painless process: it only involves calling a decorator on a coroutine which eats a request object and a handler coroutine. This decorator [applies](https://github.com/aio-libs/aiohttp/blob/07105705781776aea2c7a4fe10d8e2c6641284c3/aiohttp/web_middlewares.py#L25) a flag on our coroutine so that when the [application runs the middleware](https://github.com/aio-libs/aiohttp/blob/master/aiohttp/web_app.py#L337) it can run a partial function application. We can setup a middleware to verify that routes protected by json are given json data, but this will require a flag which we setup in a decorator.
``` python
def json_request(f):
    f.JSON_REQUEST = True
    return f
```

This flag allows our middleware to differentiate between requests which are JSON-only and standard HTTP requests. Also, note that the request object requires an application header in order for the json to work

``` python
@web.middleware
async def json_request_middleware(req, handler):
    '''Verifies that a route only handling json is protected.'''
    json_request = handler.__dict__.get('JSON_REQUEST', False)
    if json_request:
        if req.content_type = 'application/json':
            try:
                data = await req.json()
                return await handler(req, data)
            except:
              pass
        return web.json_response({"msg": "invalid request"}, status=400)
    return await handler(req)

...

# setup the middleware through the application definition
app = web.Application(
    middlewares = (json_request_middleware,)
)
```

Once we've setup the middleware we can protect our routes with

``` python
@routes.post('/my-route')
@json_protected
async def my_route(req, data):
    ...
```

### Decorator approach

Another approach to this problem is to create a decorator which performs the same operations

``` python
from functools import wraps

def json_request_decorator(handler):
    @wraps(handler)
    async def wrapper(*args, **kwargs):
        req = args[0]
        invalid_response = web.json_response({
            "msg": "This endpoint only accepts valid JSON"
        }, status=400)
        if req.content_type != 'application/json':
            return invalid_response
        try:
            data = await req.json()
            return await handler(req, data)
        except:
            return invalid_response
    return wrapper
```

The code here looks almost exactly the same, and we can protect a route by writing

``` python
@routes.post('/my-route')
@json_request_decorator
async def my_route(req, data):
    ...
```

### Analysis

If we compare the two strategies they both have a similar flavor, but with the decorator approach it is slightly less complex since we are not adding metadata to a route coroutine. This is the preferred approach out of the two for applications which have content-types differing between the routes since we are selecting the routes to check. (If we wanted to accept multiple content-types, such as json, xml, etc. on the same routes there is a [suggested approach](https://aiohttp.readthedocs.io/en/stable/web_advanced.html#custom-routing-criteria) given in the documentation). In the case that we are building a JSON-only API/(micro)service then the middleware approach is the better option from an architectural standpoint since it saves a line of code for every route.

The only remaining piece to look at is if there is much of a difference in performance overhead. For the decorator approach we are just building a new coroutine when the application first starts up while the middleware is just a chunk of code that's called on every request. Regardless of the solution chosen, the same 10 or so lines of code are called, so it's not really worth profiling the code to try and see if there is a significant performance difference. You can look at the lines referenced

####  Pages Mentioned

* [https://docs.aiohttp.org/en/stable/index.html](https://docs.aiohttp.org/en/stable/index.html)
* [https://docs.aiohttp.org/en/stable/web_advanced.html#middlewares](https://docs.aiohttp.org/en/stable/web_advanced.html#middlewares)
* [https://github.com/aio-libs/aiohttp/blob/07105705781776aea2c7a4fe10d8e2c6641284c3/aiohttp/web_middlewares.py#L25](https://github.com/aio-libs/aiohttp/blob/07105705781776aea2c7a4fe10d8e2c6641284c3/aiohttp/web_middlewares.py#L25)
* [https://github.com/aio-libs/aiohttp/blob/master/aiohttp/web_app.py#L337](https://github.com/aio-libs/aiohttp/blob/master/aiohttp/web_app.py#L337)
* [https://aiohttp.readthedocs.io/en/stable/web_advanced.html#custom-routing-criteria](https://aiohttp.readthedocs.io/en/stable/web_advanced.html#custom-routing-criteria)

### Complete Code

``` python
# json_middleware.py
from aiohttp import web

def json_request(f):
    f.JSON_REQUEST = True
    return f

@web.middleware
async def json_request_middleware(req, handler):
    '''Verifies that a route only handling json is protected.'''
    json_request = handler.__dict__.get('JSON_REQUEST', False)
    if json_request:
        if req.content_type = 'application/json':
            try:
                data = await req.json()
                return await handler(req, data)
            except:
              pass
        return web.json_response({"msg": "invalid request"}, status=400)
    return await handler(req)

@json_request
async def index(req, data):
    print(data)
    return web.json_response({"msg": "success!"})

app = web.Application(
    middlewares = (json_request_middleware,)
)

app.add_routes([
    web.post('/', index),
])

if __name__ == "__main__":
    web.run_app(app)
```

``` python
# json_decorator.py
from aiohttp import web

def json_request(handler):
    @wraps(handler)
    async def wrapper(*args, **kwargs):
        req = args[0]
        invalid_response = web.json_response({
            "msg": "This endpoint only accepts valid JSON"
        }, status=400)
        if req.content_type != 'application/json':
            return invalid_response
        try:
            data = await req.json()
            return await handler(req, data)
        except Exception as e:
            return invalid_response
    return wrapper

@json_request
async def index(req, data):
    print(data)
    return web.json_response({"msg": "success!"})

app = web.Application()
app.add_routes([
    web.post('/', index),
])

if __name__ == "__main__":
    web.run_app(app)
```

