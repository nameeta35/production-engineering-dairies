**Why I Started Looking at HTTP Through an Application's Perspective**

HTTP/1.1, HTTP/2, and HTTP/3 are often explained through their individual features: multiplexing, connection reuse, TCP head-of-line blocking, QUIC, and so on.I could understand each concept individually, but understanding how these differences actually affect an application wasn't as intuitive.

Things started clicking when I decided to use an e-commerce checkout flow to explore how the same application behaves under HTTP/1.1, HTTP/2, and HTTP/3.

**Quick peek at HTTP/1.1 features**
- **TCP-based communication:** Provides reliable, ordered delivery of data between clients and servers.
- **Persistent connections (Keep-Alive):** Allows multiple requests to reuse the same TCP connection instead of establishing a new connection for every request.
- **Sequential request-response exchanges:** HTTP/1.1 typically handles requests sequentially on a connection. Although pipelining is supported by the specification, browser implementations generally rely on multiple connections for concurrency instead.
- **Multiple TCP connections:** Browsers commonly open multiple connections to the same server to support concurrent requests, subject to per-origin connection limits.

**How HTTP/1.1 loads the checkout page?**
- Browser requests /checkout.
- Server returns HTML, CSS, and JavaScript.
- Browser parses the page and discovers additional resources.
- Browser initiates API requests for order details, items, shipping, and payment methods.
- Multiple TCP connections are opened to support concurrency.
- Each connection handles its own request-response exchange.

```
 Bottleneck - Browsers commonly impose per-origin connection limits.This limits how many requests can be in flight concurrently using separate connections. Connections get reused for next requests using keep-alive.
```
**Quick peek at HTTP/2 features**
- **Multiplexing:** Supports multiple concurrent request-response streams over a single TCP connection.
- **Binary framing:** Converts HTTP messages into smaller binary frames that can be interleaved across multiple streams.
- **Independent stream processing:** Server processes the requests and sends responses on their corresponding streams, not necessarily in the same order as the requests were initiated.
- **TLS support:** HTTP/2 can technically operate without TLS, but browsers generally require HTTPS.

**How HTTP/2 loads the same checkout page?**

- Browser establishes one TCP connection
- It creates a separate stream for each of the 4 API requests.
- It assigns each stream an ID.
- It converts requests into HTTP/2 frames.
- Frames from different streams can be interleaved and sent over the same TCP connection.
- Server processes the requests and sends responses on their corresponding streams not necessarily in the same order as they were passed on the connection.

```
Bottleneck - Although HTTP/2 supports multiplexing at the application layer, it still relies on TCP for reliable, ordered delivery. If a TCP packet is lost, delivery of data for all streams can be delayed until TCP recovers the missing bytes.
```
**Quick peek at HTTP/3 features**
- **QUIC-based communication:** Uses QUIC over UDP instead of TCP for transport
- **Multiplexing:** Supports multiple independent streams over a single QUIC connection.
- **Independent stream delivery:** Packet loss in one stream does not block delivery of data on other streams.
- **Faster connection establishment:** Integrates transport and cryptographic handshakes, reducing connection setup latency.
- **Connection migration:** Uses connection IDs to maintain connections across network changes, such as switching from Wi-Fi to cellular.
- **Built-in encryption:** Requires TLS 1.3 for secure communication.

**How HTTP/3 loads the same checkout page?**

- Browser establishes one QUIC connection
- It creates a separate stream for each of the 4 API requests.
- It assigns each stream an ID.
- It converts requests into frames.
- Frames from different streams can be interleaved and sent over the same QUIC connection.
- Server processes the requests and sends responses on their corresponding streams not necessarily in the same order as they were passed on the connection.
- If one of the packets from the order request/response gets lost due to network error, only the order stream gets blocked.

```
For eg:
Order stream: waits for its missing data to be retransmitted.
Items stream: can continue receiving data.
Shipping stream: can continue receiving data.
```

- If the client's device switched network, the connection doesn't need to be established again.

```
Bottlenecks resolved
- Packet loss in one stream doesn't block delivery on other streams unlike HTTP/2.
```

**Deep Dive: How network latency impacts HTTP performance?**

Even if the backend processes all four requests quickly, the browser still needs to communicate with the server over a network.

Network latency is the time it takes for data to travel between two endpoints. Round-trip time (RTT) measures the time for a message to travel to the destination and for a response to return. For example, suppose the browser is communicating with a server over a network with an RTT of 100 ms.

**How would the three protocols perform?**

HTTP/1.1: Establishing a new HTTPS connection would involve a TCP handshake followed by a TLS handshake. Multiple connections can repeat connection-establishment overhead.

HTTP/2: Multiple requests can reuse one TCP connection, avoiding the need to establish separate connections for each request. However, the initial TCP and TLS handshakes still contribute to connection latency.

HTTP/3: QUIC combines transport and TLS 1.3 handshake functionality, typically allowing a new connection to be established in one RTT. Resumed connections can also support 0-RTT data under specific conditions. Instead of waiting for a full handshake round trip before sending requests, the client uses cached session parameters to send the very first packet when reconnecting.

**Final Thoughts**
HTTP/1.1, HTTP/2, and HTTP/3 solve the same fundamental problem: enabling communication between clients and servers. However, the way they manage connections, multiplex requests, and handle network conditions has evolved significantly.

What helped me understand these differences was looking beyond individual protocol features and following the actual requests made by an application.

Understanding what's happening beneath the HTTP APIs we use every day can help backend engineers make better architectural decisions. However, protocol evolution doesn't mean applications can switch to newer versions overnight. Compatibility, infrastructure, deployment constraints, and real-world network conditions all influence how much of a performance improvement a protocol upgrade actually delivers.
