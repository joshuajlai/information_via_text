# Rabbit Blocked Connections
## What is it?
RabbitMQ implemented a system for automatically blocking connections/requests in order to prevent the
system from going down. By default, a RabbitMQ node will self monitor its resources and set itself
into a "Blocked State" where it returns a `connection.blocked` function callback trigger. This
is documented here: https://www.rabbitmq.com/connection-blocked.html

The "Blocked State" is used to prevent the server from crashing. A dead server is less ideal
than a server that's rejecting requests temporarily. However, if your RabbitMQ client code
is not designed to accept a `connection.blocked` callback, it's basically the same thing. The
less than ideal situation is that RabbitMQ is configured for this behavior by default which
can lead to unexpected failures after upgrading clusters or using certain out of the box
RabbitMQ clients.

The most common situation I've encountered is where RabbitMQ enters a "Blocked State" when
running out of memory. This has been caused by having too many un-acknowledge/un-processed
messages in queues. Not a usual situation because people should not be using RabbitMQ as
a database but that's a different problem.


## How do you solve it?
To solve the situation, you need to implement proper `connection.block` callbacks. For example,
when using an old version of php-amqplib, you need to do the following:
	* Disable disconnect on destructor of the connection object
		* This is a flag/configuration
	* Register the necessary callbacks for `connection.block`
	* Mark a flag to specify that connections will be blocked
	* Do not attempt to send requests if the connection is marked as blocked
	* Do not attempt to disconnect if the connection is marked as blocked
Thankfully the latest version of php-amqplib has been updated to have registration functions
for setting a callback. If it is absolutely necessary for the current request to push data
into RabbitMQ, the callback should fail the request if the RabbitMQ server is in a blocked state.

The main issue that I saw was that if no callback is set for `connection.block`, the client code
will send a request for a new message, receive a `connection.block` response, not know how to process
that response, and sit there waiting for a message created response. Eventually, the php process hits
the request timeout limit and errors out. Making sure you have a proper callback/behavior configured
for a blocked scenario will at least prevent this detrimental behavior.


## Why are we here?
I have no idea. Maybe an alien created us.
