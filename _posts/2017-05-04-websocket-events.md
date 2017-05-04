---
layout: post
title:  WebSocket - Server Side Events
tags: javaee websocket
---

I have documented a little tutorial on how to push events to the clients from the server.

Using Websockets with Java EE 7 is extremely easy. You just need two things:
* a *ServerEndpoint* class
* a *Client* to send messages or receive events

To demonstrate several events i have created a base ServerEndpoint class and a simple client in Javascript.

***

### ServerEndpoint class

```java
@ServerEndpoint(value = "/wss", decoders = ChatMessageDecoder.class, encoders = ChatMessageEncoder.class)
public class ChatEndpoint {

    private static final Set<Session> peers = Collections.synchronizedSet(new HashSet<>());

    @OnOpen
    public void onOpen(Session session) {
        peers.add(session);
    }

    @OnMessage
    public void onMessage(ChatMessage message, Session session) {
        broadcastMessage(message);
    }

    @OnClose
    public void onClose(Session session) {
        peers.remove(session);
    }

    private void broadcastMessage(ChatMessage message) {
        synchronized (peers) {
            peers.stream().filter(Session::isOpen).forEach(session -> wrappedMessage(session, message));
        }
    }
}
```
In it's simplest form this is all that is needed to have a "chatty" endpoint.
We declare the annotations `@ServerEndpoint` for class scope and `@onOpen`, `@onClose` and `@onMessage`. The annotations and method responsability is quite intuitive.

The full code for the encoders and decoders is:

***

### Encoder class

```java
public class ChatMessageEncoder implements Encoder.Text<ChatMessage> {

    @Override
    public String encode(ChatMessage chatMessage) throws EncodeException {
        return Json.createObjectBuilder()
                .add("content", chatMessage.getContent())
                .add("sender", chatMessage.getSender())
                .add("received", chatMessage.getReceived().format(DateTimeFormatter.ofPattern("dd-MM-yyyy kk:mm:ss")))
                .build().toString();
    }
}
```

### Decoder class

```java
public class ChatMessageDecoder implements Decoder.Text<ChatMessage> {

    @Override
    public ChatMessage decode(String message) throws DecodeException {
        JsonReader reader = Json.createReader(new StringReader(message));
        JsonObject jsonObject = reader.readObject();
        String content = jsonObject.getString("content");
        String sender = jsonObject.getString("sender");

        return new ChatMessage(content, sender);
    }
}
```

Now that we have a way to encode and decode the messages, let's move  on to the server side events..

First to create a batch process lets modify our endpoint class to become a Singleton.

***

### Startup Singleton

```java
@Singleton
@Startup
@ConcurrencyManagement(ConcurrencyManagementType.BEAN)
@ServerEndpoint(value = "/wss", decoders = ChatMessageDecoder.class, encoders = ChatMessageEncoder.class)
public class ChatEndpoint
```

This creates a startup singleton EJB. We control the synchronization behaviour, hence the  `@ConcurrencyManagement(ConcurrencyManagementType.BEAN)`.
We could have used Container managed concurrency with @Lock(LockType.READ) as well.

Now that we have a singleton EJB let's create a batch process. *This will all be done inside the class `ChatEndpoint`.*

***

### Batch Event

```java
    @Schedule(hour = "*", minute = "*", second = "*/10")
    public void batch() {
        broadcastMessage(new ChatMessage("server event", SYSTEM));
    }
```

This creates a timer scheduled process that will run every ten seconds and send messages to the connected clients.

***

### External Event

What if we want to broadcast messages based on events ocurring in the system? With Java EE this becomes extremely simple.

```java
    @Asynchronous
    public void event(@Observes String resourceMessage) {
        broadcastMessage(new ChatMessage(resourceMessage, RESOURCE));
    }
```

The external event is fired and the method `event` acts as listener. 
Notice we can even leverage asynchronous behaviour with a simple annotation `@Asynchronous`  because we are using EJB (via `@Singleton`).

Now all that's left is create a simple client.

***

### Javascript Client

```javascript
    var app = angular.module("myApp", []);

    app.controller("myCtrl", function ($scope, $http) {
        $scope.form = {
            sender: '',
            message: ''
        };

        var websocket = new WebSocket("ws://localhost:8080/wss");

        $scope.sendDetails = function () {
            websocket.send(JSON.stringify($scope.form));
        }

        websocket.onmessage = function(msg) {
            var msg = JSON.parse(msg.data);
            var myEl = angular.element( document.querySelector( '#chat-messages' ) );
            var cssClass = '';
            switch (msg.sender){
                case 'system': cssClass = 'text-danger'; break;
                case 'resource': cssClass = 'text-success'; break;
                default:
            }
            myEl.append('<span class="'+cssClass+'"><b><i>' + msg.received + '</i>' + '  ' + msg.sender + '</b>: ' + msg.content + '<br/></span>');
        };
    });
```

This is using AngularJS but standard Javascript will do.

Hope this helps!

****


