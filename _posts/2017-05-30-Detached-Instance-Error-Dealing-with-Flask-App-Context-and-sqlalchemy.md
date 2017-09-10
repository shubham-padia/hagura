---
layout: post
title:  "DetachedInstanceError: dealing with Celery, Flask’s app context and SQLAlchemy"
date:   2017-05-30 18:34:56 +0530
categories: gsoc
---
![Celery]({{site.baseurl}}/images/celery.jpg)

In the open event server project, we had chosen to go with celery for async background tasks. From the official website,
#### What is celery?
Celery is an asynchronous task queue/job queue based on distributed message passing.
#### What are tasks?
The execution units, called tasks, are executed concurrently on a single or more worker servers using multiprocessing.

After the tasks had been setup, an error constantly came up whenever a task was called:
` DetachedInstanceError: Instance <User at 0x7f358a4e9550> is not bound to a Session; attribute refresh operation cannot proceed `

The above error usually occurs when you try to access the session object after it has been closed. It may have been closed by an explicit `session.close()` call or after commiting the session with `session.commit()`.
The celery tasks in question were performing some database operations. So the first thought was that maybe these operations might be causing the error. To test this theory, the celery task was changed to :
```python
@celery.task(name='lorem.ipsum')
def lorem_ipsum():
    pass
```

But sadly, the error still remained. This proves that the celery task was just fine and the session was being closed whenever the celery task was called. The method in which the celery task was being called was of the following form:
```python
def restore_session(session_id):
    session = DataGetter.get_session(session_id)
    session.deleted_at = None
    lorem_ipsum.delay()
    save_to_db(session, "Session restored from Trash")
    update_version(session.event_id, False, 'sessions_ver')
```

In our app, the app_context was not being passed whenever a celery task was initiated. Thus, the celery task, whenever called, closed the previous app_context eventually closing the session along with it. The solution to this error would be to follow the pattern as suggested on http://flask.pocoo.org/docs/0.12/patterns/celery/. 

```python
def make_celery(app):
    celery = Celery(app.import_name, broker=app.config['CELERY_BROKER_URL'])
    celery.conf.update(app.config)
    task_base = celery.Task

    class ContextTask(task_base):
        abstract = True

        def __call__(self, *args, **kwargs):
            if current_app.config['TESTING']:
                with app.test_request_context():
                    return task_base.__call__(self, *args, **kwargs)
            with app.app_context():
                return task_base.__call__(self, *args, **kwargs)

    celery.Task = ContextTask
    return celery


celery = make_celery(current_app)
```

The `__call__` method ensures that celery task is provided with proper app context to work with.
