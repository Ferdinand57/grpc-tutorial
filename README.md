### 1. What are the key differences between unary, server streaming, and bi-directional streaming RPC (Remote Procedure Call) methods, and in what scenarios would each be most suitable?

unary RPC is a one to one model where the client sends one request message and in return the server sends one response message back, this is most suitable for action like making a payment, for example

    Client sends: PaymentRequest { user_id: "user_123", amount: 100.0 }
    Server replies: PaymentResponse { success: true }

server streaming is a one to many model where the client sends one request message and in return the server sends multiple response messages back, this is most suitable when requesting for large list of items, like a bank statement for example

    Client sends: TransactionRequest { user_id: "user_123" }
    Server replies with a stream:
        TransactionResponse { transaction_id: "trans_0", ... }
        TransactionResponse { transaction_id: "trans_1", ... }
        .
        .
        .

bi-directional streaming is a many to many model where the client and the server can send multiple responses to each other over the same connection, this is suitable for real-time interaction like a chat room

    a
    Server says: ChatMessage { user_id: "user_123", message: "Terima kasih telah melakukan chat kepada CS virtual, Pesan anda akan dibalas pada jam kerja. pesan anda : a" }
    a
    Server says: ChatMessage { user_id: "user_123", message: "Terima kasih telah melakukan chat kepada CS virtual, Pesan anda akan dibalas pada jam kerja. pesan anda : a" }
    a
    Server says: ChatMessage { user_id: "user_123", message: "Terima kasih telah melakukan chat kepada CS virtual, Pesan anda akan dibalas pada jam kerja. pesan anda : a" }
    tidak bisa login
    Server says: ChatMessage { user_id: "user_123", message: "Terima kasih telah melakukan chat kepada CS virtual, Pesan anda akan dibalas pada jam kerja. pesan anda : tidak bisa login" }

### 2. What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption?

- regarding authentication, does the server know who the client is before processing the request? The client can have some identifying information like a username in the metadata of their gRPC request and in our rust code on the server will check this metadata to verify the client, usually this would be placed in a place like "MyPaymentService::process_payment" before actually processing the payment

- regarding authorization, once the server know who the client is, it should check if the client have permission to request the requested function. For example, we don't want some random person accessing our bank statement.

- regarding data encryption, making sure the data that is sent between the client and the server is unreadable to anyone/anything other than the client and the server itself to ensure privacy.

### 3. What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications?

- the server need to be ready to receive multiple messages from the client and sending multiple messages to the client at the same time, our server do this by listening to request asyncronously by using "tokio::spawn" in "MyChatService::chat", this way the server won't be confused by having to listen to multiple responses at the same time

- what if the internet is down and the client lose their connection with the server and vice versa. Our code need to handle this scenario with grace by closing the chat or informing the user instead of just crashing. Our server do this by using unwrap_or_else for example "while let Some(message) = stream.message().await.unwrap_or_else(|_| None) {" in here when unwrap_or_else receive an error, it will return a None as closure, this let's the while function to close gracefully instead of a screaming error message

- managing resources used by each instances of active chat in ChatService like the MPSC channel "(let (tx, rx) = mpsc::channel(10);)", after one active chat is inactive the server need to be sure to clean up the chat memory so that it can be used by another chat room

- if the client is sending a lot of messages very,very quickly or the server tries to sends a lot of responses at the same time, the other end will be overwhelmed. The way our rust application deals with this is by using a buffer like "mpsc::channel(10)", this makes it so when if one send tries to send messages and the buffer is full, they will have to wait until there's space, which makes sure the other side doesn't get overwhelmed

### 4. What are the advantages and disadvantages of using the tokio_stream::wrappers::ReceiverStream for streaming responses in Rust gRPC services?

"The ReceiverStream is a utility from tokio_stream::wrappers that converts a tokio receiver into a stream that tonic can work with. This stream is then wrapped in a Response and returned."

"...converts the receiving end of the channel (rx) into a ReceiverStream that can be returned as part of a gRPC stream response."

- Advantages:

the main advantage of using receieverStream is it's simplicity in connectng a standard Tokie MPSC channel "(like let (tx, rx) = mpsc::channel(4);)" directly to a gRPC streaming response, and the tutorial explains it as a "utility... that converts a tokio receiver (rx) into a stream that tonic can work with," which allow us to easily generate data in one asynchronous task for example the spawned task sending transaction records, and have "ReceiverStream::new(rx)" make that data available for gRPC to send to the client

- Disadvantages:

relying on RecieverStream means we are introducing the slight overhead of an MPSC channel for message passing and are tied to using a Tokio Reciever which mean if our data stream comes from anything other than MPSC channel then RecieverStream might be less ideal than a custom stream implementation

### 5. In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time?

currently, a lot of the logic is hard coded in the gRpc service itself, such as in MyPayentService, therefore to promote maintainability and extensibility, we can do the following:

- following separation of concern philosophy, we should extract the core processing logic such as payment validation or transaction recording into a separate rust modules, or at least dedicated structs, for example, a payment processor module could contain the actual payment handling function, transferring responsibility from the gRPC, effectively transforming gRPC request into call to the business logic which is followed by mapping the result back to the gRPC responses. This separation of concerns allows the core logic to be reused by other interfaces if needed

- refactoring, functionality that is shared across multiple gRPC services such as custom error handling routines could be separated into a stand alone utility modules to avoid code duplication ad promote consistency



### 6. In the MyPaymentService implementation, what additional steps might be necessary to handle more complex payment processing logic?

In my opinion, the existing MyPaymentService implementation is the bare minimum for an implementation, only returning PaymentResponse { success: true }, to implement more complex payment processing logic i think we need to develop the following:

- Input Validation: we need to make sure that our input validation can validate the more complex input that comes with a more complex payment processing, this includes verifying the format of fields like user_id and ensuring the amount is valid and a positive value

- Database Interaction: with a more complex input, comes more complex database to store it, we need to make sure our database can handle a more complex input

- Comprehensive Error Handling: instead of a binary success/failure, our service need to be upgraded into returning a more detailed error information to indicate spesific issues like "not enough funds" or "client gateway timeout"

- Anti Duplicate Mechanism: we need to implement a unique request identifier to make sure taht if the service receive multiple request at the same time that it knows if the request is a duplicate due to lag or two different request sent consecutively

### 7. What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms?

Adopting gRPC have significant influence, for example, gRPC require us to define the services and message structures in .proto files beforehand and this approach forces us to establish a clear and strongly typed interface between component before we commit to implementation, which reduces integration ambiguity, also, the fact that gRPC uses HTTP2 and protocol buffer generally result in lower latency for inter-service communication compared to text-based protocols like REST/JSON, more than that, gRPC also capable of suporting diverse streaming pattern, as we can see in the tutorial, gRPC allows us to implement server streaming (like the one in TransactionService) and bi-directional streaming (like the one for ChatService), this build-in support for various streaming models is simplifying our design that require continuous data flow

### 8. What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs?

Advantages: 
- HTTP2 allow multiple gRPC calls to be transmitted at the same time over a single TCP connection which improve efficiency and reduced latency, meanwhile HTTP1.1 often suffers from head-of-line blocking or requires multiple connection to do what HTTP2 can do in one

- HTTP2 transmits data in a binary format, which is more efficient for a machine to parse and process compared to the text-based nature of HTTP1.1

- HTTP2 uses HPACK to compress request and response headers, which reduces bandwitdth consumption especially for application with many small requests

Disadvantages: 
- The binary nature of HTTP2 make raw network traffic less human-readable, which makes it harder to debug, often requiring specialised tooling compared to text-based HTTP1.1

- HTTP2 is a more complex protocol than HTTP1.1, luckyly this complexity is largely managed by libraries like Tonic

### 9. How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness?

Rest APIs follow a strict one request one response pattern, which is slow for real-time updates because the client might need to send a request to the server for new data repeatedly, and each interaction might involve a new connection which can result in higher latency and less efficient real-time updates for highly iteractive applicatio.

gRPC bi-directional streaming capability, as seen in the ChatService part of the tutorial, allows both the client and server to independently and asynchronously send a stream of messages to each other over a single gRPC connection which is establishes by client.chat(...)

- Initiating Chat with Server:
    
        let mut response_stream = client.chat(request).await?.into_inner();: Initiates a chat with the server by calling the chat method of the ChatService client. This method returns a stream of responses from the server.
    
- Processing Server Responses:
    
        while let Some(response) = response_stream.message().await? { ... }: Loops to continuously receive and print responses from the server.


### 10. What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads?

The schema-based approach using protocol buffers gRPC uses requires a strict definition of the message structures and service interface in a .proto file just like the structure of ChatMessage. This strict definition provides a clear and unambiguous contract between client and server which enables compile-time checks and code generation, effectively reducing runtime errors due to mismatched data structures. Not only that, because protocol buffers uses binary format, the resulting payload will be way smaller and have faster parsing/serialization compared to text-based JSON. Unfortunately, this strict requirement means modifying the data structure requres us to update the .proto file and regenerating our code, making it less suitable for scenarios where data schemans are highly dynamic or unknown at design time

On the otherhand, JSON itself doesn't enforce a strcit schema, which allows for easy transmission of varied or evolving data structure without needing to define every field beforehand, which is really well suited for rapidly changing requirements or handling unstrcuturedd data, not only that, JSON is text-based which means generally it is way easier for developers to read and debug. But, without a strictly enforced schema, it will be easier for the client and server to have mismatched data structure expectation, which may lead to parsing errors or unexpected behavior. Not only that, JSON payloads are typically larger and slower to parse than binary protocol buffers, and also more effort is often required in application code to validate the structure and types of received JSON data