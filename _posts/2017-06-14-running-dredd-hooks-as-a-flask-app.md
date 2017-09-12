---
layout: post
title:  "Running Dredd Hooks As A Flask App"
date:   2017-06-07 18:34:56 +0530
categories: gsoc
---
![Hook]({{site.baseurl}}/images/hook.jpg)


The Open Event Server is based on the microframework Flask from its initial phases. After implementing API documentation, we decided to implement Dredd testing in the open event API.

As seen in the previous blog post, we learnt how to [Isolate each request in Dredd testing](http://blog.fossasia.org/isolating-each-request-in-dredd-testing-for-open-event-server/). Flask, being a microframework itself, came to our rescue as we could easily bind the database engine to the Flask app. The Flask app can be initialize in the `before_all` hook easily as shown below:
```python
def before_all(transaction):
    app = Flask(__name__)
    app.config.from_object('config.TestingConfig')
```

The database can be binded to the app as follows:
```python
def before_all(transaction):
    app = Flask(__name__)
    app.config.from_object('config.TestingConfig')
    db.init_app(app)
    Migrate(app, db)
```

The real challenge was now to bind the application context when applying the database fixtures. In a normal Flask application this can be done as following:
```python
with app.app_context():
  #perform your operation
```
While for unit tests in python:
```python
with app.test_request_context():
  #perform tests
```

But as all the hooks are seperate from each other, Dredd-hooks-python supports idea of a single stash list where you can store all the desired variables(a list or the name `stash` is not necessary). 

So, the app and db can be added to [stash](http://dredd.readthedocs.io/en/latest/hooks-python/#sharing-data-between-steps-in-request-stash) as shown below:
```python
@hooks.before_all
def before_all(transaction):
    app = Flask(__name__)
    app.config.from_object('config.TestingConfig')
    db.init_app(app)
    Migrate(app, db)
    stash['app'] = app
    stash['db'] = db
```

These variables stored in the stash can be used efficiently as below:
```python
@hooks.before_each
def before_each(transaction):
    with stash['app'].app_context():
        db.engine.execute("drop schema if exists public cascade")
        db.engine.execute("create schema public")
        db.create_all()
``` 
and many other such examples.