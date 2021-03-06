# Error handling

Sometimes, things go wrong. The resource the client requested does not exist, we didn't receive the data we expected, or something unexpected happens. In short: an exception is raised.

Bocadillo makes it easy to catch specific exceptions to return appropriate HTTP responses.

## What is an error handler?

An **error handler** is a function (asynchronous or synchronous) that is called when an exception occurs.

Error handlers are given the `Request` and `Response` objects in the state they were just when the exception was raised, and the exception object itself. They can mutate the response (e.g. set the status code) in order to achieve their desired behavior.

For example, this is an error handler:

```python
from bocadillo import Request, Response

async def mute(req: Request, res: Response, exc: Exception):
    pass  # do nothing
```

## Registering error handlers

An error handler is associated to an exception class. This allows to know which error handler to call when a particular exception is raised (see next section).

To register an error handler, use the `@api.error_handler()` decorator:

```python
@api.error_handler(AttributeError)
def on_attribute_error(req, res, exc: AttributeError):
    res.status = 500
    res.media = {'error': {'attribute_not_found': exc.args[0]}}
```

For convenience, a non-decorator syntax is also available:

```python
def on_attribute_error(req, res, exc: AttributeError):
    res.status = 500
    res.media = {'error': {'attribute_not_found': exc.args[0]}}

api.add_error_handler(AttributeError, on_attribute_error)
```

## What happens when an error is raised?

When an exception is raised within an HTTP view or middleware, the following algorithm is used:

1. We iterate over the registered exception classes until we find one that the raised exception is a subclass of.
2. The latest registered error handler for that exception class is then called, and the (perhaps mutated) response is returned.
3. If no error handler was found:
    - A special error handler is called to convert the response to an `500 Internal Server Error` response. If [debug mode] is active, the response body is an HTML page containing the exception traceback. If debug mode is not active, the body is just plain text.
    - The response is sent to the client and the exception is re-raised to allow server-side logging.

## How `HTTPError` is handled

The `HTTPError` exception is used to return HTTP error responses within views.

Every Bocadillo application comes with an `HTTPError` error handler. The default handler returns an error response with the provided `status_code` and returns the error `detail` as plain text.

There's nothing special to `HTTPError`; it's just an exception. This means that you can customize this behavior by registering your own error handler for `HTTPError`.

For convenience though, common HTTP error handlers are available in the `bocadillo.error_handlers` module:

- `error_to_text()`: converts an exception to plain text (this is the default).
- `error_to_html()`: converts an exception to an HTML response.
- `error_to_media()`: converts an exception to a media response.

## Example

Consider the following application that simulates a game of chance:

```python
import os
from random import random
from bocadillo import API, HTTPError

class GameException(Exception):
    pass

class Lose(GameException):
    pass

class Win(GameException):
    pass

api = API()

@api.error_handler(GameException)
def on_game_exception(req, res, exc):
    res.text = "Something went wrong…"
    res.status_code = 500

@api.error_handler(Win)
def on_success(req, res, exc):
    res.text = "You win!"
    res.status_code = 200

@api.route("/play")
async def play(req, res):
    coin = random()
    if coin < 0.1:
        raise Win()
    elif coin < 0.999:
        raise Lose()
    else:
        raise RuntimeError("We did not expect that.")

if __name__ == "__main__":
    api.run(debug=os.getenv("DEBUG"))
```

- 10% of the time, a `Win` exception is raised. We do have registered an error handler for it, so it will be called and, since it does not re-raise it, the exception won't propagate.
- 89.9% of the time, a `Lose` exception is raised. We do not have any error handler registered for it, but we do have one for its parent class `GameException`, so it will be used.
- 0.1% of the time, a `RuntimeError` is raised. There is no error handler registered for this exception, so a standard 500 error response will be returned and the exception will be raised for server-side logging.

[debug mode]: ../api.md#debug-mode
