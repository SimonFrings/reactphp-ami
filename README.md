# clue/ami-react [![Build Status](https://travis-ci.org/clue/php-ami-react.svg?branch=master)](https://travis-ci.org/clue/php-ami-react)

Simple async, event-driven access to the Asterisk Manager Interface (AMI)

The [Asterisk PBX](http://asterisk.org/) is a popular open source telephony solution
that offers a wide range of telephony features.
The [Asterisk Manager Interface (AMI)](https://wiki.asterisk.org/wiki/display/AST/The+Asterisk+Manager+TCP+IP+API)
allows you to control and monitor the PBX.
Among others, it can be used to originate a new call, execute Asterisk commands or
monitor the status of subscribers, channels or queues.

* **Async execution of Actions** -
  Send any number of actions (commands) to the asterisk in parallel and
  process their responses as soon as results come in.
  The Promise-based design provides a *sane* interface to working with out of bound responses.
* **Event-driven core** -
  Register your event handler callbacks to react to incoming events, such as an incoming call or
  a change in a subscriber state.
* **Lightweight, SOLID design** -
  Provides a thin abstraction that is [*just good enough*](http://en.wikipedia.org/wiki/Principle_of_good_enough)
  and does not get in your way.
  Future or custom actions and events require no changes to be supported.
* **Good test coverage** -
  Comes with an automated tests suite and is regularly tested against versions as old as Asterisk 1.8+

> Note: This project is in beta stage! Feel free to report any issues you encounter.

## Quickstart example

Once [installed](#install), you can use the following code to access your local
Asterisk Telephony instance and issue some simple commands via AMI:

```php
$factory = new Factory($loop);

$factory->createClient('user:secret@localhost')->then(function (Client $client) {
    echo 'Client connected' . PHP_EOL;
    
    $api = new Api($client);
    $api->listCommands()->then(function (Response $response) {
        echo 'Available commands:' . PHP_EOL;
        var_dump($response);
    });
});

$loop->run();
```

See also the [examples](example).

## Usage

### Factory

The `Factory` is responsible for creating your `Client` instance.
It also registers everything with the main `EventLoop`.

```php
$loop = \React\EventLoop\Factory::create();
$factory = new Factory($loop);
```

The `createClient($amiUrl)` method can be used to create a new `Client`.
It helps with establishing a plain TCP/IP or secure SSL connection to the AMI
and issuing an initial `login` action.

```php
$factory->createClient('user:secret@localhost')->then(
    function (Client $client) {
        // client connected and authenticated
    },
    function (Exception $e) {
        // an error occured while trying to connect or authorize client
    }
);
```

> Note: The given $amiUrl *must* include a host, it *should* include a username and secret
> and it *can* include a scheme (tcp/ssl) and port definition.

### Client

The `Client` is responsible for exchanging messages with the Asterisk Manager Interface
and keeps track of pending actions.

The `on($eventName, $eventHandler)` method can be used to register a new event handler.
Incoming events and errors will be forwarded to registered event handler callbacks:

```php
$client->on('event', function (Event $event) {
    // process an incoming AMI event (see below)
});
$client->on('close', function () {
    // the connection to the AMI just closed
});
$client->on('error', function (Exception $e) {
    // and error has just been detected, the connection will terminate...
});
```

The `close()` method can be used to force-close the AMI connection and reject all pending actions.

The `end()` method can be used to soft-close the AMI connection once all pending actions are completed.

> Advanced: Creating `Action` objects, sending them via AMI and waiting for incoming
> `Response` objects is usually hidden behind the `Api` interface.
>
> If you happen to need a custom or otherwise unsupported action, you can also do so manually
> as follows. Consider filing a PR though :)
>
> The `createAction($name, $fields)` method can be used to construct a custom AMI action.
> A unique ActionID will be added automatically (needed to match incoming responses).
>
> The `request(Action $action)` method can be used to queue the given messages to be sent via AMI
> and wait for a `Response` object that matches its ActionID.

### Api

The `Api` wraps a given `Client` instance to provide a simple way to execute common actions.

```php
$api = new Api($client);

$api->ping()->then(function (Response $response) {
    // response received for ping action
});
```

All public methods resemble their respective AMI actions.
Listing all available actions is out of scope here, please refer to the class outline.

Sending actions is async (non-blocking), so you can actually send multiple action requests in parallel.
The AMI will respond to each action with a `Response` object. The order is not guaranteed.
Sending actions uses a Promise-based interface that makes it easy to react to when an action is *fulfilled*
(i.e. either successfully resolved or rejected with an error):

```php
$api->ping()->then(
    function (Response $response) {
        // response received for ping action
    },
    function (Exception $e) {
        // an error occured while executing the action
        
        if ($e instanceof ErrorException) {
            // we received a valid error response (such as authorization error)
            $response = $e->getResponse();
        } else {
            // we did not receive a valid response (likely a transport issue)
        }
    }
});
```

> Advanced: Using the `Api` is not strictly necessary, but is the recommended way to execute common actions.
>
> If you happen to need a new or otherwise unsupported action, or additional arguments,
> you can also do so manually. See the advanced `Client` usage above for details.
> A PR that updates the `API` is very much appreciated :)

### Message

The `Message` is an abstract base class for the `Response`, `Action` and `Event` value objects.
It provides a common interface for these three message types.

Each `Message` consists of any number of fields with each having a name and one or multiple values.
Field names are matched case-insensitive. The interpretation of values is application specific.

The `getFieldValue($key)` method can be used to get the first value for the given field key.
If no value was found, `null` is returned.

The `getFieldValues($key)` method can be used to get a list of all values for the given field key.
If no value was found, an empty `array()` is returned.

The `getFields()` method can be used to get an array of all fields.

The `getActionId()` method is a shortcut for accessing the value of the "ActionID" field.

#### Response

The `Response` value object represents the incoming response received from the AMI.

#### Action

The `Action` value object represents an outgoing action messages to be sent to the AMI.

#### Event

The `Event` value object represents the incoming event received from the AMI.

The `getName()` method can be used to get the name of the event. This is a shortcut to
get the value of the "Event" field.

## Install

The recommended way to install this library is [through composer](http://getcomposer.org). [New to composer?](http://getcomposer.org/doc/00-intro.md)

```JSON
{
    "require": {
        "clue/ami-react": "~0.2.0"
    }
}
```

## License

MIT
