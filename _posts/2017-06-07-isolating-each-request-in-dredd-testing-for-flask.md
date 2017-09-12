---
layout: post
title:  "Isolating each request in Dredd testing for Flask"
date:   2017-06-07 18:34:56 +0530
categories: gsoc
---
![Dredd]({{site.baseurl}}/images/dredd.png)


In the [Open Event Server](https://github.com/fossasia/open-event-orga-server), we are using [Api-blueprint](https://apiblueprint.org/) along with [aglio](https://github.com/danielgtaylor/aglio) for API documentation in the project. The foremost concern with any API documentation is making sure it remains updated to the actual implementation of the API backend.

It was only a matter of days that we realized we needed some sort of testing mechanism for the API documentation. [Dredd](https://github.com/apiaryio/dredd) came to our rescue in that moment. Quoting the official documentation:
> Dredd is a language-agnostic command-line tool for validating API description document against backend implementation of the API. Dredd reads your API description and step by step validates whether your API implementation replies with responses as they are described in the documentation.


For loading DB fixtures, handling authentication, sessions etc. Dredd Hooks are used. Quoting their official documention:
> Similar to any other testing framework, Dredd supports executing code around each test step. Hooks are code blocks executed in defined stage of execution lifecycle. In the hooks code you have an access to compiled HTTP transaction object which you can modify.

Now after setting up dredd testing successfully, the challenge was to make every request remains isolated with respect to the database i.e making sure that database changes by one request do not affect the database used by another request, similar to what is done in case of unit tests ;). 

In unit tests, you can just access the session of the tests after it is completed and gracefully rollback the changes made by the test. But as dredd is completely seperated from the API backend being tested, accessing that session is not possible.

So the only choice that remains is purging the database after each request and recreating it with the required data before each request.

To do the above, we initalize a vanilla flask app and attach a database to it:
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

Before each request, you purge the pre-existing database to remove the changes from the previous request. After that a new db is created with necessary repopulation.

```python
@hooks.before_each
def before_each(transaction):
    with stash['app'].app_context():
        db.engine.execute("drop schema if exists public cascade")
        db.engine.execute("create schema public")
        db.create_all()
        stamp()
        populate_db_with_required_static_values()
```

Database fixtures specific to the request can be loaded in the `before` hook in the following way:

```python
@hooks.before("Users > Users Collection > List All Users")
def user_get_list(transaction):
    """
    GET /users
    :param transaction:
    :return:
    """
    with stash['app'].app_context():
        user = UserFactory()
        db.session.add(user)
        db.session.commit()
```

Relevant links:
- API-Blueprint: https://apiblueprint.org/
- Dredd: https://github.com/apiaryio/dredd
- Dredd-hooks-python: https://github.com/apiaryio/dredd-hooks-python
