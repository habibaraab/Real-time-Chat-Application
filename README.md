
# Real-time Chat Application using Spring Boot


### Introduction

This project is a real-time chat application developed using Spring Boot and WebSocket. The application allows users to sign up, log in, chat in public channels, and engage in private one-to-one conversations. A key feature is the persistence of chat history, allowing users to view their past messages.

The project leverages the Spring Boot framework for a robust and scalable backend, while WebSocket enables full-duplex, real-time communication between the server and the clients, providing a seamless and interactive chat experience.


### Technologies Used

* **WebSocket**: A communication protocol that provides a full-duplex communication channel over a single TCP connection. It is the ideal choice for chat applications as it minimizes latency compared to traditional HTTP.
* **STOMP (Simple Text Oriented Messaging Protocol)**: A simple text-based messaging protocol that operates on top of WebSocket. Since WebSocket itself doesn't define how to route messages to specific users or topics, STOMP provides these essential pub/sub functionalities.
* **SockJS**: A JavaScript library that provides a WebSocket-like object and serves as a fallback for browsers that do not support the WebSocket protocol. SockJS ensures that the application will run smoothly across all modern and legacy browsers.
* **Spring Boot & Spring Data JPA**: The framework provides a rapid development environment, while Spring Data JPA simplifies database interactions for persisting user and message data.


### Code Explanation

#### WebSocket Configuration
Path: `config/WebSocketConfig.java`
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").setAllowedOriginPatterns("*").withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setApplicationDestinationPrefixes("/app");
        registry.enableSimpleBroker("/chatroom","/user");
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void configureWebSocketTransport(WebSocketTransportRegistration registry) {
        registry.setSendTimeLimit(60 * 1000)
                .setSendBufferSizeLimit(50 * 1024 * 1024)
                .setMessageSizeLimit(50 * 1024 * 1024);
    }
}

This class configures the core WebSocket infrastructure for the application.

  * **`registerStompEndpoints`**: This method registers a primary WebSocket endpoint at `/ws`. Clients will use this URL to connect to the server. `withSockJS()` enables the SockJS fallback options.
  * **`configureMessageBroker`**: This is a critical section that sets up the message broker:
      * **`setApplicationDestinationPrefixes("/app")`**: It defines that any message sent from a client to a destination starting with `/app` (e.g., `/app/private-message`) should be routed to a message-handling method in a `@Controller`.
      * **`enableSimpleBroker("/chatroom", "/user")`**: It enables a simple, in-memory message broker to broadcast messages to subscribed clients.
          * `/chatroom`: Used for public, broadcast topics.
          * `/user`: Used for private, user-specific messages.
      * **`setUserDestinationPrefix("/user")`**: This configures the prefix for user-specific destinations, which is essential for one-to-one messaging.
  * **`configureWebSocketTransport`**: This method tunes the technical properties of the data transport, such as setting the maximum message size and send time limit, enhancing the application's stability and security.


#### Chat Controller

Path: `controller/ChatController.java`

```java
@RestController
@AllArgsConstructor
@RequestMapping("/api/users")
public class ChatController {

    private final SimpMessagingTemplate simpMessagingTemplate;
    private final UserRepository userRepository;
    private final ChatMessageRepository chatMessageRepository;
    @Autowired
    private UserService userService;

    // Endpoints for user management (Login, Signup, Search, History) ...

    @MessageMapping("/message")
    @SendTo("/chatroom/public")
    public Message receiveMessage(Message message) {
        // ... (saves message to database)
        return message;
    }

    @MessageMapping("/private-message")
    public void privateMessage(Message message) {
        simpMessagingTemplate.convertAndSendToUser(message.getReceiverName(), "/private", message);
        // ... (saves private message to database)
    }

    @GetMapping("/history/{user1}/{user2}")
    public ResponseEntity<List<ChatMessage>> getChatHistory(@PathVariable String user1, @PathVariable String user2) {
        // ... (fetches history from database)
    }
}
```

This controller acts as the application's brain, handling both standard HTTP requests and WebSocket messages.

##### REST Endpoints (For User & Data Management)

  * **`POST /api/users/signup`**: Creates a new user account.
  * **`POST /api/users/login`**: Logs a user in.
  * **`GET /api/users/search`**: Searches for a user by their username.
  * **`GET /api/users/history/{user1}/{user2}`**: Fetches the persisted chat history between two users.

##### WebSocket Endpoints (For Real-time Messaging)

  * **`@MessageMapping("/message")`**: This method handles public messages sent to the `/app/message` destination. It saves the message to the database and, thanks to the `@SendTo("/chatroom/public")` annotation, broadcasts it back to all clients subscribed to the public chatroom topic.
  * **`@MessageMapping("/private-message")`**: This method handles private messages sent to `/app/private-message`. It persists the message and then uses the `SimpMessagingTemplate` to send the message directly to the specific recipient on their unique private channel (e.g., `/user/soso/private`).

-----

#### Message Repository

Path: `repository/ChatMessageRepository.java`

```java
public interface ChatMessageRepository extends JpaRepository<ChatMessage, Long> {
    List<ChatMessage> findBySenderNameAndReceiverNameOrReceiverNameAndSenderNameOrderByTimestampAsc(
        String senderName1, String receiverName1, String senderName2, String receiverName2
    );
}
```

This interface is responsible for database interactions related to chat messages.

  * **`findBySenderNameAndReceiverNameOr...`**: This is not just a regular method; it's a powerful feature of Spring Data JPA. The framework automatically generates a complex SQL query directly from the method name. This specific query fetches all messages where the sender was `user1` and receiver was `user2`, **OR** where the sender was `user2` and receiver was `user1`, and then orders them chronologically. This is the core logic behind the "Chat History" feature.


#### Data Models

Path: `model/ChatMessage.java` (Entity) & `model/Message.java` (DTO)

The application intelligently uses two distinct models for messages:

1.  **`ChatMessage` (Entity)**: This class represents a message in the database. It contains fields like `id`, `senderName`, `receiverName`, `message`, and `timestamp`. It is used for persistence and data retrieval.
2.  **`Message` (DTO - Data Transfer Object)**: This is a simpler class used as the payload for WebSocket messages. Its purpose is to transfer only the necessary data between the client and server, leading to a more efficient and clean API.


#### Frontend Logic (JavaScript)

Path: `resources/index.html` (contains the JS code)

The client-side JavaScript brings the UI to life by interacting with the backend.

  * **Connection**: Upon successful login, the code establishes a WebSocket connection to the server's `/ws` endpoint.
  * **Subscription**: After connecting, the client subscribes to two key channels:
    1.  **`/chatroom/public`**: To receive all public broadcast messages.
    2.  **`/user/{username}/private`**: A unique, private channel for each user to receive one-to-one messages addressed specifically to them.
  * **Sending Messages**:
      * Public messages are sent to the `/app/message` destination.
      * Private messages are sent to the `/app/private-message` destination.
  * **Fetching History**: When a user opens a private chat with another user, the code sends an HTTP `GET` request to `/api/users/history/...` to retrieve past messages and display them in the chat window.

