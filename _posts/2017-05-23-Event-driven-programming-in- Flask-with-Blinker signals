---
layout: post
title:  "Event-driven programming in Flask with Blinker signals"
date:   2017-05-23 18:34:56 +0530
categories: gsoc
---
![Signals]({{site.baseurl}}/images/signal.jpg)
## Setting up blinker:

The Open Event Project offers event managers a platform to organize all kinds of events including concerts, conferences, summits and regular meetups. In the server part of the project, the issue at hand was to perform multiple tasks in background(we use celery for this) whenever some changes occured wither in the event, or the speakers/sessions associated with the event.

The usual approach to this would be applying a function call after any relevant changes are made. But the statements making these changes were distributed all over the project at multiple places. It would be cumbersome to add 3-4 function calls(which are irrelevant to the function they are being executed) in so may places. Moreover, the code would get unstructured with this and it would be really hard to maintain this code over time.

That's when signals came to our rescue. From Flask 0.6, there is integrated support for signalling in Flask, refer http://flask.pocoo.org/docs/latest/signals/ . The [Blinker](https://github.com/jek/blinker) library is used here to implement signals. If you're coming from some other language, signals are analogous to events.

Given below is the code to create named signals in a custom namespace:

```python
from blinker import Namespace

event_signals = Namespace()
speakers_modified = event_signals.signal('event_json_modified')
```

If you want to emit a signal, you can do so by calling the send() method:
```python
speakers_modified.send(current_app._get_current_object(), event_id=event.id, speaker_id=speaker.id)
```

From the user guide itself:
Try to always pick a good sender. If you have a class that is emitting a signal, pass self as sender. If you are emitting a signal from a random function, you can pass current_app._get_current_object() as sender.

To subscribe to a signal, [blinker](https://github.com/jek/blinker) provides neat decorator based signal subscriptions.
```python
@speakers_modified.connect
def name_of_signal_handler(app, **kwargs):
```

## Some Design Decisions:

When sending the signal, the signal may be sending lots of information, which your signal may or may not want. e.g when you have multiple subscribers listening to the same signal. Some of the information sent by the signal may not be of use to your specific function. Thus we decided to enforce the pattern below to ensure flexibility throughout the project.
```python
@speakers_modified.connect
def new_handler(app, **kwargs):
# do whatever you want to do with kwargs['event_id']
```

In this case, the function `new_handler` needs to perform some task solely based on the event_id. If the function was of the form `def new_handler(app, event_id)`, an error would be raised by the app. A big plus of this approach, if you want to send some more info with the signal, for the sake of example, if you also want to send speaker_name along with the signal, this pattern ensures that no error is raised by any of the subscribers defined before this change was made.

## When to use signals and when not ?

The call to send a signal will of course be lying in another function itself. The signal and the function should be independent of each other. If the task done by any of the signal subscribers, even remotely affects your current function, a signal shouldn't be used, use a fucntion call instead.

That's all for now. Have some fun signaling ;) . 
