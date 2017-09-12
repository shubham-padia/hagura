---
layout: post
title:  "Modifying flask-rest-jsonapi Exception Handling to Enable Support for Sentry"
date:   2017-06-07 18:34:56 +0530
categories: gsoc
---
![Sentry]({{site.baseurl}}/images/sentry.jpg)

In [open-event-server](https://github.com/fossasia/open-event-orga-server) Project, we had enabled [sentry](https://sentry.io/welcome/) support in the project. So first of all,

### What is Sentry ? 

Sentry provides open source error tracking that shows you every crash in your stack as it happens, with the details needed to prioritize, identify, reproduce, and fix each issue. 

We enabled the basic error tracking with the following two simple lines,
```python
from raven.contrib.flask import Sentry
sentry = Sentry(app, dsn='https://<key>:<secret>@sentry.io/<project>')
```

But after sometime, we noticed that we were not catching any app-related error in Sentry, while migration related errors were being caught. This meant that sentry was functioning properly in the app, but it was having some trouble in identifying uncaught exceptions.

After a lot of digging, it came to our knowledge that our api framework, [flask-rest-jsonapi](https://github.com/miLibris/flask-rest-jsonapi) caught all unknown exceptions while dispatching the request. After catching the exceptions, it gave a jsonapi error with status 500 in return. Following is the code responsible for that:

```python
except Exception as e:
            if current_app.config['DEBUG'] is True:
                raise e
            exc = JsonApiException('', 'Unknown error')
            return make_response(json.dumps(jsonapi_errors([exc.to_dict()])),
                                 exc.status,
                                 headers)
```

We now had to let these exceptions go uncaught and that required us to modify the api framework. So we modified the api-framework's exception handling as shown below

```python
if 'API_PROPOGATE_UNCAUGHT_EXCEPTIONS' in current_app.config:
                if current_app.config['API_PROPOGATE_UNCAUGHT_EXCEPTIONS'] is True:
                    raise
            if current_app.config['DEBUG'] is True:
                raise e
            exc = JsonApiException({'pointer': ''}, 'Unknown error')
            return make_response(json.dumps(jsonapi_errors([exc.to_dict()])),
                                 exc.status,
                                 headers)
```

A config parameter named `API_PROPOGATE_UNCAUGHT_EXCEPTIONS` was added to the config to turn this feature on or off.

But with all this done, we were able to catch errors with sentry, but the json-api spec compliant error messages on an unknown error were not being returned due to this new change. So it was decided to handle these uncaught exceptions in the app itself. We used flask's default error handlers to tackle this situation

```python
@app.errorhandler(500)
def internal_server_error(error):
    exc = JsonApiException({'pointer': ''}, 'Unknown error')
    return make_response(json.dumps(jsonapi_errors([exc.to_dict()])), exc.status,
                         {'Content-Type': 'application/vnd.api+json'})
```

Thus, all uncaught exceptions were now returning a proper json-api spec compliant error response.

Related links:
- Flask error handlers:  http://flask.pocoo.org/docs/0.12/patterns/errorpages/#error-handlers
- Json-api spec: http://jsonapi.org/
- Json-api errors: http://jsonapi.org/examples/#error-objects
