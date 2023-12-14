# Python M2M Library

`m2m-auth` is a comprehensive library used to authenticate calls between internal services and API’s.

> [!IMPORTANT]
> If you need to use M2M authentication with an existing Python2 service, please use this library instead *(link)*

## Installation

To install `m2m-auth`, include the following at the very bottom of your project’s `requirements.txt` file:

```
--extra-index-url https://nexus.revup.com/repo/m2m-auth/packages
https://nexus.revup.com/repo/m2m-auth/packages/m2m-auth/0.0.16.tar.gz
```

Then run:

```
pip install -r requirements.txt
```

## Using the module

Begin by importing the m2m requests module into your project.

```python
from m2m_client.request import M2MRequest
```

### Outgoing Requests

The request types available through m2m-auth are the traditional RESTful HTTP request types.

> [!NOTE]
> When using any of the request methods, you must supply your `client_id`, `client_secret`, and `audience` as either arguments or as environment variables. These are the values that you obtained when you created an Auth0 Client for your service *(link to external doc)*.

`get(url, params=None, client_id=None, client_secret=None, audience=None, **kwargs)`

`post(url, data=None, json=None, client_id=None, client_secret=None, audience=None, **kwargs)`

`put(url, data=None, client_id=None, client_secret=None, audience=None, **kwargs)`

`patch(url, data=None, client_id=None, client_secret=None, audience=None, **kwargs)`

`delete(url, client_id=None, client_secret=None, audience=None, **kwargs)`

> [!WARNING]
> All `client_secret` values must be stored securely (NOT checked into version control). We encourage storing them in a solution such as AWS Secrets Manager via Terraform or as Gitlab CI/CD variables.


| Arguments | Description | Example
| --- | --- | --- |
| `url` | The url of the service you want to communicate with. | url1.revup.com |
| `params` | Additional data you want to send in the URL's query string. | `{'key':'value'}` |
| `data` | Additonal data you want to send (as a dictionary or list of tuples). | `{'key':'value', 'key2':'value2'}` |
| `json` | Use this parameter when specifically sending json data. Requests will serialize your data and add the correct `Content-Type` header automatically (`application/json`). The `json` parameter is ignored if `data` is passed. | `{'key':'value'}` |
| `client_id` | Your application's client id. | abc123 |
| `client_secret` | Your application's client secret. | ****** |
| `audience` | The unique identifier for the target API you want to access. | newaudience.revup.com |

### Incoming Requests

Use the `authenticate` decorator on a given endpoint to authenticate **incoming** m2m requests.

The decorator takes in two parameters:
* audience: the unique identifier for the API that you want to access.
* auth0 domain: the Auth0 tenant that hosts your Auth0 client.

Here's an example implementation:

```python
from m2m_client.flask.decorators import authenticate

@app.route(‘/api/hello/’)
@authenticate(‘testapi’, ‘revup-user-test.us.auth0.com’)
def hello():
	return ‘Hello, World!’
```

### Errors and Exceptions

* `NullClientIdError` is raised if the client_id is not supplied.
* `NullClientSecretError` is raised if the client_secret is not supplied.
* `NullAudienceError` is raised if the audience is not supplied.

* `Auth0RateLimitReachedException` is raised if the Auth0 rate limit is reached.
* `AuthenticationError` is raised for any other authentication error.
