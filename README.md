# Notifications (InApp, Email, SMS, CustomBackend) for Django.

django-notifs is a notifications app for Django. Basically it allows you to notify users about events that occur in your application E.g

1. Your profile has been verified
2. User xxxx sent you a message

It also has built in support to deliver these notifications via Emails (if you want). It's also very easy to extend the delivery backends with a Custom Adapter.

## Installation
Get it from pip with

```bash
pip install django-notifs
```

Include it in `INSTALLED_APPS`

```python
INSTALLED_APPS = (
    'django.contrib.auth',
    ...
    'notifications',
    ...
)
```

Include the urls in urls.py

```python
urlpatterns = [
    ...
    url('^notifications/', include('notifications.urls', namespace='notifications')),
    ...
]
```

Finally don't forget to run the migrations with

```bash
python manage.py migrate notifications
```

### Sending Notifications
django-notifs uses Django signals to make it as decoupled from the rest of your application as possible. This also enables you to listen to it's signals and carry out other custom actions if you want to.

To Create/Send a notification import the notify signal and send it with the following arguments (Hypothetical example).

```python
from notifications.signals import notify

args = {
    'source': self.request.user,
    'source_display_name': self.request.user.get_full_name(),
    'recipient': recipent_user, 'category': 'Chat',
    'action': 'Sent', 'obj': message.id,
    'short_description': 'You a new message', 'url': url
}
notify.send(sender=self.__class__, **args)
```

### Notification Fields

The fields in the `args` dictionary map to the fields in the `Notification` model

- **source: A ForeignKey to Django's User model (Can be null if it's not a User to User Notification).**
- **source_display_name: A User Friendly name for the source of the notification.**
- **recipient: The Recipient of the notification. It's a ForeignKey to Django's User model.**
- **category: Arbitrary category that can be used to group messages.**
- **action: Verbal action for the notification E.g Sent, Cancelled, Bought e.t.c**
- **obj: The id of the object associated with the notification (Can be null).**
- **short_description: The body of the notification.**
- **url: The url of the object associated with the notification (Can be null).**
- **silent: If this Value is set, the notification won't be persisted to the database.**
- **extra_data: Arbitrary data as a dictionary.**

The values of the fields can easily be used to construct the notification message.

### Extra/Arbitrary Data

Apart from the standard fields, django-notifs allows you to attach arbitrary data to a notification.
Simply pass in a dictionary as the extra_data argument.

**NOTE: The dictionary is serialized using python's json module so make sure the dictionary contains objects that can be serialized by the json module**

Internally, the JSON is stored as text in django's standard `TextField` so it doesn't use database-specific JSON fields.

### Reading notifications

To read a notification simply send the read signal like this:

```python
from notifications.signals import notify

# id of the notification object, you can easily pass this through a URL
notify_id = request.GET.get('notify_id')

# Read notification
if notify_id:
    read.send(sender=self.__class__, notify_id=notify_id,
              recipient=request.user
    )
```

It's really important to pass the correct recipient to the read signal, Internally it's used to check if the user has the right to read the notification. If you pass in the wrong recipient or you omit it entirely, `django-notifs` would raise a
`NotificationError`

### Accessing Notifications in templates

django-notifs comes with a Context manager that you can use to display notifications in your templates. Include it with

 ```bash
'context_processors': [
    ...
    'notifications.notifications.notifications',
    ...
],
```

This makes a user's notifications available in all templates as a template variable named "notifications"


## Sending Emails in the background

That is beyond the scope of this project, though it can easily be achieved with a third party extension [django-celery-email](https://github.com/pmclanahan/django-celery-email) after the installation it's as easy as setting:

```bash
EMAIL_BACKEND = 'djcelery_email.backends.CeleryEmailBackend'
```
in your application's settings.

## Websockets

This is the coolest part of this library, Unlike other django notification libraries that provide a JavaScript API that the client call poll at regular intervals, django-notifs supports websockets (thanks to `uWSGI`). This means you can easily alert your users when events occur in your application, You can use it to build a chat application etc

To actually deliver notifications, `django-notifs` uses rabbitmq (No Redis support yet) as a message queue to store notifications which are then consumed and sent over the websocket to the client.


### Running the websocket server

To enable the Websocket functionality simply set

```bash
NOTIFICATIONS_WEBSOCKET = True
```

and set the URL to your RabbitMQ Server with:

```bash
NOTIFICATIONS_RABBIT_MQ_URL = 'YOUR RABBIT MQ SERVER'
```

This will tell django-notifs to publish messages to the rabbitmq queue.

Due to the fact that Django itself doesn't support websockets, The Websocket server has to be started separately from your main application with uwsgi. e.g

```bash
uwsgi --http :8080 --http-websockets --wsgi-file websocket.py --threads 2 --workers 2 --processes 2 --thunder-lock --master
```

There is a sample implementation of a websocket server in `websocket.py` and there's a `websocket.html` file that you can use to test the websocket.

### How to listen to notifications

At the backend, A Rabbitmq queue is created for each user based on the username, so when you're connecting to the websocket server you have to pass the username in the websocket url. For example to listen to messages for a username `danidee` connect to this url (Assuming the websocket server is running on `localhost` and port `8080`)

```JavaScript
var websocket = new WebSocket('ws://localhost:8080/danidee')
```

