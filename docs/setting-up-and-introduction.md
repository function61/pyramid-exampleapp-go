Setting up & introduction
=========================

Setup streams etc.
------------------

Our app will listen to events in `/example`.
[Enter Horizon CLI](https://github.com/function61/eventhorizon/blob/master/docs/enter-horizon-cli.md)
and create the stream:

```
$ horizon stream-create /example
```

We'll need to create a subscription (`example-app`) and subscribe it to `/example`:

```
$ horizon stream-create /_sub/example-app
$ horizon stream-subscribe /example /_sub/example-app
```

A subscription is just a regular stream under the covers. The only difference is
that you cannot subscribe to subscriptions, because that would create an infinite loop.

Now when taking a peek at our subscription stream, we should see the tip of the
`/example` having been advertised:

```
$ horizon reader-read /_sub/example-app:0:0:? 10
/Created {"subscription_ids":[],"ts":"2017-03-22T14:55:49.557Z"}
/SubscriptionActivity {"activity":["/example:0:135:1.2.3.4"],"ts":"2017-03-22T14:55:59.597Z"}
```

Now, let's enter some The Office -themed sample data into the stream so our
database will not be empty:

```
$ horizon stream-appendfromfile /example example-dataimport/import.txt
2017/03/22 14:55:59 Appending 21 lines
2017/03/22 14:55:59 Done. Imported 21 lines in 135.305288ms.
```

If you'd now read the example-app subscription again, we'd notice that there are
new notifications on the stream:

```
$ horizon reader-read /_sub/example-app:0:0:? 10
/Created {"subscription_ids":[],"ts":"2017-03-22T14:55:49.557Z"}
/SubscriptionActivity {"activity":["/example:0:135:1.2.3.4"],"ts":"2017-03-22T14:55:59.597Z"}
/SubscriptionActivity {"activity":["/example:0:2417:1.2.3.4"],"ts":"2017-03-22T14:56:04.601Z"}
/SubscriptionActivity {"activity":["/example:0:2535:1.2.3.4"],"ts":"2017-03-22T15:16:00.62Z"}
```

Now you understand the mechanism for how Pusher knows which streams to push to
the endpoint - it just monitors the subscription stream which subscribed streams
have stuff the endpoint should be aware of! There's a bit more but that's the basic idea.

You can now exit from the CLI.


Now start the example application
---------------------------------

First, build the application:

```
$ docker build -t eventhorizon-exampleapp-go .
```

Then, run your application. Like with the Horizon CLI, you have to specify the
`STORE` so the Pusher component can push events to your application:

```
$ docker run -it --rm -e STORE='...' eventhorizon-exampleapp-go
2017/03/22 15:04:37 App: listening at :8080
2017/03/22 15:04:37 pusherchild: starting
2017/03/22 15:04:37 configfactory: downloading discovery file
...
2017/03/22 15:04:37 Pusher: reached the top for /_sub/example-app
```

Ok it's succesfully started.


Interacting with the example application
----------------------------------------

This service has an internal projection of all the events it saw, i.e. it has
a database and you can query the database via its REST endpoint:

```
$ curl http://localhost:8080/users
[
...
   {
        "ID": "e1dd2e26",
        "Name": "Kelly Kapoor",
        "Company": "629cfead"
    }
]
```

We can modify the data by invoking commands which will raise new events. Commands
are something where you do authorization (is this user allowed to do this), validate
input, foreign key references etc.

Normally the service itself
would probably have a UI with forms on how to change the data, which would post
the changes as events to Event Horizon and it would notify your app of those events.

But here's the interesting bit: we can skip that application altogether, and use
Event Horizon to directly change Kelly's (`id=e1dd2e26`) name!

If you look at [events/usernamechanged.go](../events/usernamechanged.go), you'll
learn that we can do this (in the CLI):

```
$ horizon stream-append /example 'UserNameChanged {"user_id": "e1dd2e26", "new_name": "Kelly Kaling", "reason": "Married", "ts": "2017-03-22 00:00:00"}'
```

And now inspect the data:

```
$ curl http://localhost:8080/users
[
...
   {
        "ID": "e1dd2e26",
        "Name": "Kelly Kaling",
        "Company": "629cfead"
    }
]
```
