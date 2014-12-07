# Subscriptions (TCP)

When using the Event Store client API, clients can subscribe to a stream and be
notified when new events are written to that stream. Two types of subscription
are available:

- Regular Subscriptions - this kind of subscription calls a given function for
  events written after the subscription is established.
  
  For example, if a stream has 100 events in it when a subscriber connects, the
  subscriber can expect to see event number 101 onwards until the time the
  subscription is closed or dropped.

- Catch-Up Subscriptions - this kind of subscription specifies a starting
  point, in the form of an event number or transaction file position. The given
  function will be called for events from the starting point until the end of
  the stream, and then for subsequently written events.
  
  For example, if a starting point of 50 is specified when a stream has 100
  events in it, the subscriber can expect to see events 51 through 100, and
  then any events subsequently written until such time as the subscription is
  dropped or closed.

Both types of subscription are supported both by the [.NET Client API]() and
the [JVM Client API](). It is also possible to subscribe over the HTTP
interface using long polling. HTTP based subscriptions are covered <here>.

## Use cases

Subscriptions have many use cases. For example, when building an event-sourced
system adhering to the CQRS pattern, the denormalizers creating query models
can use catch-up subscriptions to have events pushed to them as they are
written to the Event Store. Regular subscriptions can be used where it is
important to be notified with minimal latency of new events which are written,
but where it is not necessary to process every event.

## Specifying a starting point for a catch-up subscription

Identifying a point in an individual stream in the Event Store is
straightfoward - it is sufficient to know the name of the stream and the number
of the event. This is not sufficient for identifying an individual event in the
database, however. Conseuqently, when subscribing to all events, it is
necessary to use a tuple of the logical prepare and commit positions in the
transaction file to identify an event. Thanks to the scavenging process which
garbage collects a database, it is not always the case that a logical position
matches the physical position in the transaction file.

When establishing a catch-up subscription, one of the parameters is the
starting point. All subsequent events to the starting point will be passed to
the subscription. For example, if subscribing to an individual stream, and a
starting event number of 50 is specified, the subscription will receive events
51 onwards.

This may seem slightly counter-intuitive, however it has the nice property of
allowing a subscriber to save a checkpoint when processing events, and be able
to resume from that checkpoint without having to make any calculations. This
property is extremely useful when subscribed to all events rather than to an
individual stream, since it is not possible to calculate the next event
position without knowing about event lengths and internal details of the
transaction engine which are subject to future change.

## .NET API

This section details establishing subscriptions using the .NET Client API.
Specifically it covers the behaviour of version 3.0.0, though the APIs are
unchanged since version 2.2.0 of the client.

Although the necessary method calls to establish a subscription are
straightforward, it is necessary to understand how the various elements of
connection lifecycle interact in order to understand the canonical way of
subscribing.

In order to introduce these concepts it is simplest to look at sample code.

### Connections and Connection Events

Implementations of `IEventStoreConnection` may be connected or disconnected
from an Event Store server at any given point in time. As well as in their
initial state, they can become disconnected if there is an interruption in
network service between the client and server, if the server is unavailable, or
(if set up for clustered operation) a master takeover is in progress.

Often it is neccesary to perform some actions when the client becomes connected
or disconnected from the server. Implementations of `IEventStoreConnection`
expose CLR events which enable consumers to hook into this.

In order to prevent race conditions, it is neccessary to attach any event
handlers to the connection before attempting to connect.

The code listing below shows the canonical way of attaching handlers which
simply output to the console when the client is connected, disconnected or
closed. 

```csharp
class Program
{
    static void Main()
    {
        //Naming a connection allows us to identify where log messages have come from
        // in the event that there are several serving different purposes.
        const string connectionName = "connection";

        //Connect to an instance running on the local box on the default port.
        var endpoint = new IPEndPoint(IPAddress.Loopback, 1113);

        //The only non-default setting we need is to log client messages to the console.
        var connectionSettings = ConnectionSettings.Create()
            .UseConsoleLogger();

        //Create a connection to a single node (on the defined endpoint) using the 
        // settings, endpoint and name that we defined.
        var connection = EventStoreConnection.Create(connectionSettings, endpoint, connectionName);

        //The Connected event is fired when the client establishes a connection to the server.
        connection.Connected += OnConnected;

        //The Disconnected event is fired when the server breaks communication with the client
        // (for example if the server is terminated).
        connection.Disconnected += OnDisconnected;

        //The Closed event is fired when the client disconnects from the server using the Close
        // method on the connection.
        connection.Closed += OnClosed;


        //Start attempting to establish a connection to the server. If the server is unavailable
        // this will retry, by default 10 times before failing.
        connection.ConnectAsync().Wait();

        //Block the main thread holding the connection open.
        Console.WriteLine("Press <ENTER> to close the connection");
        Console.ReadLine();

        //Cleanly close the connection (causes the Closed event to fire).
        connection.Close();

        //Keep the console window open when running in Debug mode.
        Console.WriteLine("Press <ENTER> to exit");
        Console.ReadLine();
    }

    /// <summary>
    /// Handle the connection establishing communication with the server. This can be used
    /// for setting up regular subscriptions, modifying a state machine and so forth.
    /// </summary>
    /// <param name="sender">The connection responsible for the event</param>
    /// <param name="e">The connection arguments. Allows strongly typed access to the connection.</param>
    private static void OnConnected(object sender, ClientConnectionEventArgs e)
    {
        var connection = e.Connection;
        Console.WriteLine("-- Connection {0} CONNECTED", connection.ConnectionName);
    }

    /// <summary>
    /// Handle the connection being closed cleanly by the client. This is also fired if the
    /// connection has never managed to connect to a server and reaches the specified limit
    /// of reconnection attempts.
    /// </summary>
    /// <param name="sender">The connection responsible for the event</param>
    /// <param name="e">The connection arguments. Allows strongly typed access to the connection.</param>
    private static void OnClosed(object sender, ClientClosedEventArgs e)
    {
        var connection = e.Connection;
        Console.WriteLine("-- Connection {0} CLOSED", connection.ConnectionName);
    }

    /// <summary>
    /// Handler for non-clean disconnections (e.g. server or network initiated).
    /// </summary>
    /// <param name="sender">The connection responsible for the event</param>
    /// <param name="e">The connection arguments. Allows strongly typed access to the connection.</param>
    private static void OnDisconnected(object sender, ClientConnectionEventArgs e)
    {
        var connection = e.Connection;
        Console.WriteLine("-- Connection {0} DISCONNECTED", connection.ConnectionName);
    }
}
```

A typical output from this program when an Event Store is already running is as follows:

```
$ mono-sgen connection.exe

Press <ENTER> to close the connection
[14,22:21:36.791,DEBUG] TcpPackageConnection: connected to [127.0.0.1:1113, L127.0.0.1:50100, {a8afd0f8-4101-48d9-a33d-bebabeb1f942}].
-- Connection connection CONNECTED

Press <ENTER> to exit
[12,22:21:40.057,INFO] ClientAPI TcpConnection closed [22:21:40.056: N127.0.0.1:1113, L127.0.0.1:50100, {a8afd0f8-4101-48d9-a33d-bebabeb1f942}]:
Received bytes: 22, Sent bytes: 22
Send calls: 1, callbacks: 1
Receive calls: 2, callbacks: 1
Close reason: [Success] Connection close requested by client.

[12,22:21:40.058,DEBUG] TcpPackageConnection: connection [127.0.0.1:1113, L127.0.0.1:50100, {a8afd0f8-4101-48d9-a33d-bebabeb1f942}] was closed cleanly.
-- Connection connection CLOSED
```

This shows the client connecting to the server, and then being closed. If
however the client is connected and the server process is terminated then
restarted, typical output is as follows:

```
$ mono-sgen connection.exe

Press <ENTER> to close the connection
[13,22:23:00.989,DEBUG] TcpPackageConnection: connected to [127.0.0.1:1113, L127.0.0.1:50101, {7e8c1fb8-e073-45d6-a7a1-f2ce4b968c7f}].
-- Connection connection CONNECTED
[16,22:23:06.543,INFO] ClientAPI TcpConnection closed [22:23:06.543: N127.0.0.1:1113, L127.0.0.1:50101, {7e8c1fb8-e073-45d6-a7a1-f2ce4b968c7f}]:
Received bytes: 44, Sent bytes: 44
Send calls: 2, callbacks: 2
Receive calls: 3, callbacks: 3
Close reason: [ConnectionReset] Socket receive error

[16,22:23:06.544,DEBUG] TcpPackageConnection: connection [127.0.0.1:1113, L127.0.0.1:50101, {7e8c1fb8-e073-45d6-a7a1-f2ce4b968c7f}] was closed with error: ConnectionReset.
-- Connection connection DISCONNECTED
[16,22:23:07.809,DEBUG] TcpPackageConnection: connection to [127.0.0.1:1113, L,{225d4510-e77d-48aa-b8e8-27087a78694e}] failed. Error: ConnectionRefused.
[16,22:23:09.031,DEBUG] TcpPackageConnection: connection to [127.0.0.1:1113, L,{022f1f38-62fe-4ae4-b3d2-be3bba071af7}] failed. Error: ConnectionRefused.
[16,22:23:10.255,DEBUG] TcpPackageConnection: connection to [127.0.0.1:1113, L,{44d89044-a996-4740-89af-b8626c0c9c86}] failed. Error: ConnectionRefused.
[16,22:23:11.474,DEBUG] TcpPackageConnection: connection to [127.0.0.1:1113, L,{c26b24a4-19aa-4b79-8d6c-3561a98ada51}] failed. Error: ConnectionRefused.
[16,22:23:12.692,DEBUG] TcpPackageConnection: connected to [127.0.0.1:1113, L127.0.0.1:50106, {26d511c5-2d71-4109-8b59-f7491b2d9a0f}].
-- Connection connection CONNECTED

Press <ENTER> to exit
[15,22:23:16.808,INFO] ClientAPI TcpConnection closed [22:23:16.808: N127.0.0.1:1113, L127.0.0.1:50106, {26d511c5-2d71-4109-8b59-f7491b2d9a0f}]:
Received bytes: 44, Sent bytes: 44
Send calls: 2, callbacks: 2
Receive calls: 3, callbacks: 2
Close reason: [Success] Connection close requested by client.

[15,22:23:16.810,DEBUG] TcpPackageConnection: connection [127.0.0.1:1113, L127.0.0.1:50106, {26d511c5-2d71-4109-8b59-f7491b2d9a0f}] was closed cleanly.
-- Connection connection CLOSED
```

In this output, the point at which the client is disconnected can be seen by
the output from the callback, and by the verbose logging emitting information
about reconnection attempts. By default the client will attempt to connect 10
times before giving up. This can be configured as part of the connection
settings, however.


### Establishing a Regular Subscription


