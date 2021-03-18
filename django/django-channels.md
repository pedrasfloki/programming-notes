
source: I took those notes from the official django-channels documentation

# what is channels :
- Channels is a project that takes Django and extends its abilities beyond HTTP - to handle WebSockets, chat protocols, IoT protocols, and more.
- Channels builds upon the -native ASGI support- available in Django since v3.0, and provides an implementation itself for Django v2.2.
- Channels wraps Django's native asynchronous view support, allowing Django projects to handle not only HTTP, but protocols that require long-running connections too - WebSockets, MQTT, chatbots, amateur radio, and more.
- It does this while preserving Django's synchronous and easy-to-use nature, allowing you to choose how you write your code - synchronous in a style like Django views, fully asynchronous, or a mixture of both. On top of this, it provides integrations with Django's auth system, session system, and more, making it easier than ever to extend your HTTP-only project to other protocols.
- We treat HTTP and the existing Django application as part of a bigger whole. Traditional Django views are still there with Channels and still useable - with Django's native ASGI support, or a Channels provided version for Django 2.2 - but you can now also write custom HTTP long-polling handling, or WebSocket receivers, and have that code sit alongside your existing code. URL routing, middleware - they are all just ASGI applications.
- Our belief is that you want the ability to use safe, synchronous techniques like Django views for most code, but have the option to drop down to a more direct, asynchronous interface for complex tasks.

## Scopes and Events
- Channels and ASGI split up incoming connections into two components: a scope, and a series of events.
- The scope is a set of details about a single incoming connection - such as the path a web request was made from, or the originating IP address of a WebSocket, or the user messaging a chatbot - and persists throughout the connection.
- For HTTP, the scope just lasts a single request. For WebSockets, it lasts for the lifetime of the socket (but changes if the socket closes and reconnects). For other protocols, it varies based on how the protocol's ASGI spec is written; for example, it's likely that a chatbot protocol would keep one scope open for the entirety of a user�s conversation with the bot, even if the underlying chat protocol is stateless.
- During the lifetime of this scope, a series of events occur. These represent user interactions - making a HTTP request, for example, or sending a WebSocket frame. Your Channels or ASGI applications will be instantiated once per scope, and then be fed the stream of events happening within that scope to decide what to do with.

### An example with -HTTP-:
- The user makes an HTTP request.
- We open up a new http type scope with details of the request's path, method, headers, etc.
- We send a http.request event with the HTTP body content
- The Channels or ASGI application processes this and generates a http.response event to send back to the browser and close the connection.
- The HTTP request/response is completed and the scope is destroyed.
### An example with a -chatbot-:
- The user sends a first message to the chatbot.
- This opens a scope containing the user's username, chosen name, and user ID.
- The application is given a chat.received_message event with the event text. It does not have to respond, but could send one, two or more other chat messages back as chat.send_message events if it wanted to.
- The user sends more messages to the chatbot and more chat.received_message events are generated.
- After a timeout or when the application process is restarted the scope is closed.

    Note: Within the lifetime of a scope - be that a chat, an HTTP request, a socket connection or something else - you will have one application instance handling all the events from it, and you can persist things onto the application instance as well. You can choose to write a raw ASGI application if you wish, but Channels gives you an easy-to-use abstraction over them called consumers.

## What is a Consumer?
- A consumer is the basic unit of Channels code. We call it a consumer as it consumes events, but you can think of it as its own tiny little application. When a request or new socket comes in, Channels will follow its routing table - we'll look at that in a bit - find the right consumer for that incoming connection, and start up a copy of it.
- This means that, unlike Django views, consumers are long-running. They can also be short-running - after all, HTTP requests can also be served by consumers - but they're built around the idea of living for a little while (they live for the duration of a scope, as we described above).

- A basic consumer looks like this:

```
class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.username = "Anonymous"
        self.accept()
        self.send(text_data="[Welcome %s!]" % self.username)

    def receive(self, *, text_data):
        if text_data.startswith("/name"):
            self.username = text_data[5:].strip()
            self.send(text_data="[set your username to %s]" % self.username)
        else:
            self.send(text_data=self.username + ": " + text_data)

    def disconnect(self, message):
        pass
```

- Each different protocol has different kinds of events that happen, and each type is represented by a different method. You write code that handles each event, and Channels will take care of scheduling them and running them all in parallel.
- Underneath, Channels is running on a fully asynchronous event loop, and if you write code like above, it will get called in a synchronous thread. This means you can safely do blocking operations, like calling the Django ORM:
```
class LogConsumer(WebsocketConsumer):
    def connect(self, message):
        Log.objects.create(
            type="connected",
            client=self.scope["client"],
        )
```

- However, if you want more control and you're willing to work only in asynchronous functions, you can write fully asynchronous consumers:
```
class PingConsumer(AsyncConsumer):
    async def websocket_connect(self, message):
        await self.send({
            "type": "websocket.accept",
        })
    async def websocket_receive(self, message):
        await asyncio.sleep(1)
        await self.send({
            "type": "websocket.send",
            "text": "pong",
        })
```

## Routing and Multiple Protocols:
You can combine multiple consumers (which are, remember, their own ASGI apps) into one bigger app that represents your project using routing:
```
application = URLRouter([
    url(r"^chat/admin/$", AdminChatConsumer.as_asgi()),
    url(r"^chat/$", PublicChatConsumer.as_asgi(),
])
```
- Channels is not just built around the world of HTTP and WebSockets - it also allows you to build any protocol into a Django environment, by building a server that maps those protocols into a similar set of events. For example, you can build a chatbot in a similar style:
```
class ChattyBotConsumer(SyncConsumer):
    def telegram_message(self, message):
        """    
        Simple echo handler for telegram messages in any chat.
        """
        self.send({
            "type": "telegram.message",
            "text": "You said: %s" % message["text"],
        })
```
- And then use another router to have the one project able to serve both WebSockets and chat requests:
```
    application = ProtocolTypeRouter({
        "websocket": URLRouter([
            url(r"^chat/admin/$", AdminChatConsumer.as_asgi()),
            url(r"^chat/$", PublicChatConsumer.as_asgi()),
        ]),
        "telegram": ChattyBotConsumer.as_asgi(),
    })
```

    Note: The goal of Channels is to let you build out your Django projects to work across any protocol or transport you might encounter in the modern web, while letting you work with the familiar components and coding style you're used to.

