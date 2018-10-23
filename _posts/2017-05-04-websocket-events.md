---
layout: post
title:  WebSocket Server Events
tags: javaee websocket
icon: images/java.png
---

I have documented a small tutorial on how to push events from the server to the clients using WebSockets.

Using Websockets with Java EE 7 is extremely easy. You just need two things:
* a **ServerEndpoint** class
* a **Client** to send messages or receive events

To demonstrate several events I have created a base ServerEndpoint class and a simple client in Javascript.

***

#### ServerEndpoint class

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
We declare the annotations `@ServerEndpoint` for class scope and `@onOpen`, `@onClose` and `@onMessage`. The annotations and method responsibility is quite intuitive.

The full code for the encoders and decoders is:

***

#### Message class

```java
public class ChatMessage {

    private String content;
    private String sender;
    private LocalDateTime received;

    public ChatMessage(String content, String sender) {
        this.content = content;
        this.sender = sender;
        this.received = LocalDateTime.now();
    }

    // getters
}
```


#### Encoder class

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


#### Decoder class

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

#### Startup Singleton

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

#### Batch Event

```java
    @Schedule(hour = "*", minute = "*", second = "*/10")
    public void batch() {
        broadcastMessage(new ChatMessage("server event", SYSTEM));
    }
```

This creates a timer scheduled process that will run every ten seconds and send messages to the connected clients.

***

#### External Event

What if we want to broadcast messages based on events occurring in the system? With Java EE this becomes extremely simple.

```java
    @Asynchronous
    public void event(@Observes String resourceMessage) {
        broadcastMessage(new ChatMessage(resourceMessage, RESOURCE));
    }
```

The external event is fired and the method `event` acts as listener. 
Notice we can even leverage asynchronous behaviour with a simple annotation `@Asynchronous`  because we are using EJB (via `@Singleton`).

Now all that's left is to create a simple client.

***

#### Client

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


**HTML**

```xml

     <div class="row">
        <div class="col-lg-6">
            <h3 class="title">Chat Room</h3>
            <div id="chat-messages"/>
        </div>
        <div class="col-lg-6">
            <form role="form">
                <legend>My Details</legend>
                <div class="form-group">
                    <label for="sender">Sender</label>
                    <input type="text" class="form-control" id="sender" ng-model="form.sender" placeholder="Name...">
                </div>

                <div class="form-group">
                    <label for="content">Message</label>
                    <input type="text" class="form-control" id="content" ng-model="form.content" placeholder="Message...">
                </div>

                <button class="btn btn-primary" ng-click="sendDetails()">Submit</button>
            </form>
        </div>
    </div>
```

This is using AngularJS but standard Javascript will do.

Hope this helps!

****


