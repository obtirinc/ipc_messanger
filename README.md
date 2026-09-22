# **IPCMessenger**

[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

A lightweight, high-performance, language-agnostic asynchronous Inter-Process Communication (IPC) library pattern for microservices.

Rather than replacing message brokers, **IPCMessenger structures and standardizes how you use them**. It combines the distinct strengths of **RabbitMQ** and **Redis Pub/Sub** into a single, unified architecture that delivers low-latency, point-to-point **Request-Response** messaging across polyglot microservice environments without massive boilerplate overhead.

---

## 🌐 **Language Implementations**

IPCMessenger is designed to work seamlessly across language boundaries. Official SDK implementations are maintained in the following repositories:

| Language / Environment | Repository |
| :--- | :--- |
| **Python** | [github.com/obtirinc/ipcm_py](https://github.com/obtirinc/ipcm_py) |
| **JavaScript / Node.js** | [github.com/obtirinc/ipcm_js](https://github.com/obtirinc/ipcm_js) |
| **TypeScript / Node.js** | [github.com/obtirinc/ipcm_ts](https://github.com/obtirinc/ipcm_ts) |
| **Go** | [github.com/obtirinc/ipcm_go](https://github.com/obtirinc/ipcm_go) |
| **Rust** | [github.com/obtirinc/ipcm_rs](https://github.com/obtirinc/ipcm_rs) |

---

## 💡 **Why IPCMessenger?**

### **The Problem with Raw Message Brokers**
Implementing a robust **Request-Response (RPC)** pattern between decoupled microservices using native brokers requires substantial infrastructure boilerplate:
* **RabbitMQ Complexity:** To receive a response asynchronously, the requesting service must declare a temporary, exclusive callback queue, generate a unique correlation ID, attach headers, and maintain consumer listeners. If a process crashes, orphaned queues can leak resources on the broker.
* **Redis Pub/Sub Limitations:** While Redis Pub/Sub is extremely fast for message delivery, it lacks native queue persistence, worker load balancing, and competing consumer management out of the box.

### **The IPCMessenger Solution: A Best-of-Both-Worlds Architecture**
IPCMessenger abstracts these broker mechanics away by combining them into a standardized hybrid pattern:

1. **RabbitMQ for Workload Distribution & Durability:** Used for distributing incoming requests across competing consumer queues. If your responder microservices scale up or temporarily go offline, RabbitMQ reliably queues and load-balances work without dropping messages.
2. **Redis Pub/Sub for Low-Latency Responses:** Used exclusively for routing responses directly back to the specific requesting process in-memory. This delivers high-speed, point-to-point delivery without the overhead of creating, monitoring, and destroying temporary RabbitMQ queues.

---

## 🌟 **Key Advantages**

* 🛡️ **Reliable, Durable Messaging:** Leverages RabbitMQ's persistent queueing for incoming tasks. Requests remain safely queued during worker restarts, deployment rollouts, or traffic spikes, preventing data loss.
* ⚡ **High Performance & Low Latency:** Bypasses RabbitMQ's disk/queue lifecycle overhead on the response path by utilizing Redis's ultra-fast in-memory Pub/Sub mechanism.
* 🔓 **Complete Microservice Decoupling:** Requesters and Responders operate independently. A Python API gateway can dispatch requests to a Rust or Go worker seamlessly without either service needing to know the other's implementation details.
* 🧹 **Zero Queue Leaks & Reduced Broker Load:** Eliminates the classic RabbitMQ RPC anti-pattern of creating and tearing down exclusive temporary reply queues for every request, preventing broker memory bloating.
* 🚀 **Accelerated Developer Velocity:** Replaces complex low-level connection, exchange, channel, correlation ID, and header handling with a clean async request/response interface across all supported languages.
* 🤖 **AI & LLM Friendly:** Because multi-step broker boilerplate is abstracted into high-level declarative calls, AI coding agents can reliably generate complete, bug-free microservices on the first attempt.
* ⚠️ **Transparent Error Propagation:** Exceptions raised inside remote worker callbacks are automatically captured, serialized, and re-raised locally as a remote service exception on the requester side for native error handling.
* 📈 **Effortless Horizontal Scaling:** Scale responder instances up or down seamlessly. RabbitMQ automatically handles competing-consumer load balancing across all active workers regardless of language.

---

## 📋 **Prerequisites**

To run services using `IPCMessenger`, you need:

* **RabbitMQ**: An accessible RabbitMQ broker instance (e.g., `amqp://guest:guest@localhost:5672/`)
* **Redis**: An accessible Redis server instance (e.g., `redis://localhost:6379/0`)
* Language runtime corresponding to the implementation SDK chosen (Python, Node.js, Go, Rust).

---

## 🔄 **Technical Protocol & Flow**

`IPCMessenger` language implementations strictly follow this protocol specification to ensure inter-compatibility:

```text
[ Requester Service ]                                    [ Responder Worker ]
 (e.g., Python / Node / Go)                               (e.g., Rust / Go / Python)
         │                                                        │
         ├── 1. Generate unique channel UUID ─────────────────────┤
         │     (e.g., response_channel:<UUID>)                    │
         │                                                        │
         ├── 2. Subscribe to Redis channel: response_channel:<UUID>
         │                                                        │
         ├── 3. Publish payload + channel UUID to RabbitMQ ──────►│
         │                                                        ├── 4. Consume from queue
         │                                                        ├── 5. Execute process callback
         │                                                        │
         │◄── 6. Publish result/error payload to Redis ───────────┤
         │       (channel: response_channel:<UUID>)               │
         │                                                        │
         ├── 7. Unsubscribe from Redis & return result ───────────┘
```

### **Standard Payload Structure**

#### **Request Message (sent over RabbitMQ):**
```json
{
  "response_channel": "response_channel:f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "payload": {
    "action": "process_data",
    "params": {}
  }
}
```

#### **Response Message (sent over Redis Pub/Sub):**
* **Success:**
  ```json
  {
    "status": "success",
    "data": { ... }
  }
  ```
* **Error:**
  ```json
  {
    "status": "error",
    "error_type": "ValueError",
    "message": "Invalid input parameters provided"
  }
  ```

---

## ⚙️ **Standard API Interface Design**

While SDK syntax is adapted idiomatically for each language, all `IPCMessenger` client implementations adhere to the following interface design:

* **`IPCMessenger(rabbitmq_url, redis_url)`**: Initializes client configuration.
* **`connect()`**: Asynchronously establishes connection pools to RabbitMQ and Redis.
* **`send_request(queue_name, payload, timeout=10)`**: Dispatches a payload to a target queue, subscribes to a temporary Redis response channel, awaits the result up to `timeout` seconds, and returns the response payload.
* **`start_listening(queue_name, callback)`**: Registers a persistent consumer on a RabbitMQ queue, routes incoming tasks to the provided worker callback, and publishes the callback return value or serialized exception back to Redis.
* **`close()`**: Gracefully shuts down active consumers and closes broker connection pools.

Refer to the respective language repository links above for language-specific installation instructions, SDK usage examples, and implementation details.

---

## 📄 **License**

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
