# Django REST Live

`django-rest-live` adds real-time subscriptions over websockets to [Django REST Framework](https://github.com/encode/django-rest-framework)
by leveraging websocket support provided by [Django Channels](https://github.com/django/channels).

## Inspiration and Goals
The goal of this project is to enable realtime subscriptions without requiring any boilerplate or changing
any existing REST Framework views or serializers.
`django-rest-live` took initial inspiration from [this article by Kit La Touche](https://www.oddbird.net/2018/12/12/channels-and-drf/).

## Dependencies
- [Django](https://github.com/django/django/) (3.0 and up)
- [Django Channels](https://github.com/django/channels) (2.0 and up) 
- [Django REST Framework](https://github.com/encode/django-rest-framework/)
- [`channels_redis`](https://github.com/django/channels_redis) for
  [channel layer](https://channels.readthedocs.io/en/latest/topics/channel_layers.html) support in production.

## Installation

If your project already uses REST framework, but this is the first realtime component,
then make sure to install and properly configure Django Channels before continuing.

You can find details in [the Channels documentation](https://channels.readthedocs.io/en/latest/installation.html).

1. Add `rest_live` to your `INSTALLED_APPS`
```python
INSTALLED_APPS = [
    # Any other django apps
    "rest_framework",
    "channels",
    "rest_live",
]
```
    
2. Add `rest_live.consumers.SubscriptionConsumer` to your websocket routing. Feel'
free to choose any URL path, here we've chosen `/ws/subscribe/`. 
```python
from rest_live.consumers import SubscriptionConsumer

websockets = URLRouter(
    [path("ws/subscribe/", SubscriptionConsumer, name="subscriptions")]
)
application = ProtocolTypeRouter({
    "websocket": websockets
})
```

That's it! You're now ready to configure and use `django-rest-live`.

## Usage

These docs will use an example to-do app called `todolist` with the following models and serializers:
```python
# todolist/models.py
from django.db import models

class List(models.Model):
    name = models.CharField(max_length=64)

class Task(models.Model):
    text = models.CharField(max_length=140)
    done = models.BooleanField(default=False)
    list = models.ForeignKey("List", on_delete=models.CASCADE)

# todolist/serializers.py
from rest_framework import serializers

class TaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = Task
        fields = ["id", "text", "done"]

class TodoListSerializer(serializers.ModelSerializer):
    tasks = TaskSerializer(many=True, read_only=True)
    class Meta:
        model = List
        fields = ["id", "name", "tasks"]
```

### Basic Usage

An important thing to remember about the REST live package is that its sole purpose is sending updates to clients
over websocket connections. Clients should still use normal REST framework endpoints generated by ViewSets and views
to get initial data to populate a page, as well as any write-driven behavior (`POST`, `PATCH`, `PUT`, `DELETE`).
REST live gets rid of the need for periodic GET requests for updated data.

#### Server-Side
In order to tell REST live that you'd like to allow clients to subscribe to updates for a specific model, the package
provides the `@subscribable` class decorator. This decorator is meant to be applied to `ModelSerializer` subclasses,
so that the package can register both which models to allow subscriptions to as well as how those models should be
serialized when being sent to the client. To enable clients to subscribe to updates to individual to-dos, all you need
to do is apply the decorator to the `TodoSerializer`:

```python
# todolist/serializers.py
from rest_live.decorators import subscribable
...
@subscribable()
class TaskSerializer(serializers.ModelSerializer):
    ...
```

#### Client-Side
Subscribing to model updates from a client requires opening a [WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
connection to the URL you specified during setup. In our example case, that URL is `/ws/subscribe/`. After the connection
is established, send a JSON message (using `JSON.stringify()`) in this format:

```json5
{
  "model": "todolist.Task",
  "pk": 1 // where PK is the primary key of the model, generally `id` unless otherwise specified.
}
```

The model label should be in Django's standard `app.modelname` format, and the `pk` property is the [primary key for the
model](https://docs.djangoproject.com/en/3.1/topics/db/queries/#the-pk-lookup-shortcut), generally the `id` field.

The example message above would subscribe to updates for the todo task with an ID of 1. As mentioned above, the client
should make a GET request to get the entire list, with all its tasks and their associated IDs, to figure out which IDs
to subscribe to.

When the Task with primary key `1` updates, a message in this format will be sent over the websocket:

```json
{
    "model": "test_app.Todo",
    "payload": {"id": 1, "text": "test", "done": true},
    "action": "UPDATED"
}
```

Valid `action` values are `UPDATED`, `CREATED`, and `DELETED`.

### Advanced Usage

#### Subscribe to groups
By default, subscriptions are grouped by the primary key: you send one message to the websocket to get updates for
a single Task with a given primary key. But in the todo list example, you'd generally be interested in an entire
list of tasks, including being notified of any tasks which have been created since the page was first loaded.

Rather than subscribe to single tasks individually, you want to subscribe to a list: an entire group of tasks.
This is where group keys come in. Pass in the `group_key` you'd like to group tasks by
to the `@subscribable` decorator to register subscriptions for an entire list:


```python
# todolist/serializers.py
from rest_live.decorators import subscribable
...
@subscribable(group_key="list_id")
class TaskSerializer(serializers.ModelSerializer):
    ...
```

On the client side, subscription requests will no longer be `pk`, but the `list_id` of the list you'd like to get updates
from:

```json5
{
  "model": "todolist.Task",
  "list_id": 1
}
```

This will subscribe you to updates for all Tasks where `list_id` is `1`.

What's important to remember here is that while the field is defined as a `ForeignKey` called `list` on the model,
the underlying integer field in the database that links together Tasks and Lists is called `list_id`, or, more generally,
`<fieldname>_id` for any related fieldname on the model.

The subscribable decorator can be stacked. If you want to enable subscriptions by both `list_id` for entire lists and
`pk` for individual tasks, add two decorators:

```python
# todolist/serializers.py
from rest_live.decorators import subscribable
...
@subscribable()
@subscribable(group_key="list_id")
class TaskSerializer(serializers.ModelSerializer):
    ...
```

Just note that clients which subscribe to list updates and individual pk updates will receive two messages when a task
updates.

## Limitations
This package works by listening in on model lifecycle events sent off by Django's [signal dispatcher](https://docs.djangoproject.com/en/3.1/topics/signals/).
Specifically, the [`post_save`](https://docs.djangoproject.com/en/3.1/ref/signals/#post-save)
and [`post_delete`](https://docs.djangoproject.com/en/3.1/ref/signals/#post-delete) signals. This means that REST live
can only pick up changes that Django knows about. Bulk operations, like `filter().update()`, `bulk_create`
and `bulk_delete` do not trigger Django's lifecycle signals, so updates will not be sent.

## TODO

- [ ] Permissions 
- [ ] Conditional Serializers
- [ ] Error handling and reporting