
## eventlet monkey_patch

```commandline
>>> import threading
>>> threading.current_thread.__module__
'threading'
>>> import eventlet
>>> eventlet.monkey_patch()
>>> threading.current_thread.__module__
'eventlet.green.threading'
>>>
```