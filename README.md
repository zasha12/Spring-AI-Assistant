# 🤖 Spring AI Assistant

**LLM-powered backend with Spring Boot + Spring AI — OpenAI, Ollama, RAG, Chat Memory, Tool Calling & MCP**

`ChatClient` • `Prompt Templates` • `Structured Output` • `Advisors` • `Qdrant` • `MCP`

![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1-6DB33F?logo=springboot&logoColor=white)
![Spring AI](https://img.shields.io/badge/Spring%20AI-2.0-6DB33F)
![Java](https://img.shields.io/badge/Java-17%2B-ED8B00?logo=openjdk&logoColor=white)
![Ollama](https://img.shields.io/badge/Ollama-Llama%203.2-000000)
![Status](https://img.shields.io/badge/status-in%20progress-yellow)

---

## 📌 Overview

Spring AI Assistant is a Spring Boot backend that adds LLM features to a normal Java service: chat, prompt engineering, typed responses, conversation memory, document Q&A (RAG), tool calling, and MCP. It works with **OpenAI** and with **local models served by Ollama**, and the provider can be switched through configuration instead of code changes.

🎓 Built while completing *From Java Dev to AI Engineer: Spring AI Fast Track* by Madan Reddy (Eazy Bytes). The code was written by me as I learned each concept, then extended with my own pieces such as the HR-policy assistant and the token-usage audit advisor (see Roadmap).

## 🏗️ Architecture

```
                    ┌───────────────────────────┐
   HTTP request ──▶ │  REST Controllers         │
                    │  /api/chat, /api/...      │
                    └────────────┬──────────────┘
                                 ▼
                    ┌───────────────────────────┐
                    │  ChatClient (Spring AI)   │
                    │  prompt • options • tools │
                    └────────────┬──────────────┘
                                 ▼
        ┌─────────────── Advisor chain ───────────────┐
        │ TokenUsageAuditAdvisor → MemoryAdvisor →    │
        │ RetrievalAugmentationAdvisor → Tool calling │
        └──────┬───────────────┬──────────────┬───────┘
               ▼               ▼              ▼
        ┌────────────┐  ┌────────────┐  ┌────────────┐
        │ Ollama     │  │ Qdrant     │  │ JDBC DB    │
        │ / OpenAI   │  │ (vectors)  │  │ (memory)   │
        └────────────┘  └────────────┘  └────────────┘
                                 │
                                 ▼
                       MCP servers / clients
                       (e.g. GitHub MCP, STDIO)
```

## 📦 Features

| Feature | What it does | Spring AI building block |
|---|---|---|
| Chat | Sends a user message to the model and returns the reply | `ChatClient`, `ChatModel` |
| Prompt engineering | System prompts, prompt templates, prompt stuffing (HR-policy assistant limited to its topic) | `PromptTemplate`, message roles |
| Structured output | Maps the LLM answer to Java Beans, Lists, and Maps | `.entity(...)`, output converters |
| Streaming | Returns tokens as they are generated | Project Reactor `Flux` |
| Token audit | Logs token usage for every call | custom `CallAdvisor` |
| Chat memory | Remembers the conversation per user | `MemoryAdvisor`, conversation ID, JDBC |
| RAG | Answers from your own documents | Qdrant, embeddings, `RetrievalAugmentationAdvisor` |
| Tool calling | Lets the model call Java methods (time, database) | `@Tool` methods |
| MCP | Exposes and uses tools through the Model Context Protocol | MCP client and server, STDIO |

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Framework | Spring Boot 4.1 |
| AI framework | Spring AI 2.0 |
| Models | Ollama (Llama 3.2), OpenAI API |
| Vector store | Qdrant (Docker) |
| Chat memory | Spring AI JDBC chat memory |
| Reactive | Project Reactor (streaming) |
| Resilience | Spring Retry |
| Build | Maven |
| Language | Java 17+ |

## 🚀 Getting Started

### Prerequisites
- JDK 17+
- Maven 3.9+ (or the included `mvnw`)
- [Ollama](https://ollama.com) installed
- Docker (for Qdrant)
- An OpenAI API key (optional, only for the OpenAI profile)

### 1. Pull a local model
```bash
ollama pull llama3.2
ollama list
```

### 2. Configure — placeholders only, NEVER commit real keys!
In `src/main/resources/application.properties`:

```properties
# Ollama (local)
spring.ai.ollama.base-url=http://localhost:11434
spring.ai.ollama.chat.options.model=llama3.2
spring.ai.ollama.chat.options.num-ctx=4096

# OpenAI (optional)
spring.ai.openai.api-key=${OPENAI_API_KEY}
```

### 3. Start Qdrant (for RAG)
```bash
docker run -p 6333:6333 -p 6334:6334 qdrant/qdrant
```

### 4. Run the app
```bash
./mvnw spring-boot:run
```

### 5. Verify
- Chat → `GET http://localhost:8080/api/chat?message=hello`
- Structured output → `GET http://localhost:8080/api/chat-bean?message=Cities in India`

> 📝 Adjust endpoints, ports, and property names to match your actual configuration.

## 📡 Endpoints

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/chat?message=` | Plain chat with the model |
| GET | `/api/chat-bean?message=` | Returns the answer as a typed Java object (`CountryCities`) |
| GET | `/api/prompt-stuffing?message=` | HR-policy assistant using a system prompt and stuffed context |

## 🧠 Prompt Engineering & Structured Output

Prompts are built from message roles (system and user), templates, and extra context. The HR-policy assistant uses a system prompt to refuse anything outside HR topics:

```java
String answer = chatClient.prompt()
        .system(hrSystemPrompt)
        .user(message)
        .call()
        .content();
```

Structured output converts the model's JSON into a Java record:

```java
CountryCities result = chatClient.prompt()
        .user(message)
        .call()
        .entity(CountryCities.class);
```

**Interview talking points:**

| Observation | Why it matters |
|---|---|
| Small models sometimes ignore format instructions | Structured output needs validation and a capable model |
| Prompt stuffing adds context to every request | More tokens, higher cost, and a bigger context window |
| System prompts limit the topic | Basic guardrail against off-topic answers |

## 📊 Advisors (Token Usage Audit)

A custom `CallAdvisor` wraps every model call and logs the token usage:

```java
public class TokenUsageAuditAdvisor implements CallAdvisor {
    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {
        ChatClientResponse response = chain.nextCall(request);
        // read usage from response metadata and log it
        return response;
    }
}
```

Advisors run in order of `getOrder()`, so logging, memory, and RAG can be stacked around the same call.

## 🧵 Chat Memory

LLM calls are stateless. Memory makes them stateful:
- Each user gets a **conversation ID**
- A `MemoryAdvisor` loads the previous messages and adds them to the prompt
- History is stored through **JDBC** and limited by `maxMessages`

## 📚 RAG (Retrieval-Augmented Generation)

```
Documents → split → embeddings → Qdrant
Question  → embedding → similar chunks → added to the prompt → LLM answer
```

`RetrievalAugmentationAdvisor` retrieves the most relevant chunks from Qdrant and feeds them to the model, so answers come from your documents instead of only the model's training data.

## 🛠️ Tool Calling

Java methods are exposed to the model, which decides when to call them:

```java
@Tool(description = "Get the current time")
public String currentTime() { ... }
```

Tool errors can be customized, and results are returned to the user through the model.

## 🔗 MCP (Model Context Protocol)

- Build an **MCP server** that exposes tools
- Build an **MCP client** that connects to servers
- Connect to the **GitHub MCP server** using **STDIO** transport

## ⚠️ Troubleshooting

| Problem | Fix |
|---|---|
| `404 model 'x' not found` | Run `ollama list`; the model name in properties must match exactly |
| `cudaMalloc failed: out of memory` | Use `llama3.2:1b` or lower `num-ctx` |
| `Port 8080 was already in use` | Stop the old process or set `server.port` |
| Structured output parse error | Use a larger model or set the output format to JSON |

## 🎓 Concepts Practised

- Integrating LLMs into Spring Boot with `ChatClient`
- Provider-agnostic configuration (OpenAI and Ollama)
- Prompt templates, system prompts, and prompt stuffing
- Mapping LLM output to Java objects
- Advisors for cross-cutting behaviour (logging, memory, RAG)
- Embeddings and vector search
- Tool calling and the Model Context Protocol

## 🛣️ Roadmap

- [x] Chat, prompt engineering, structured output
- [x] Custom token-usage advisor
- [ ] Chat memory with JDBC
- [ ] RAG with Qdrant
- [ ] Tool calling
- [ ] MCP client and server
- [ ] Evaluators (RelevancyEvaluator, FactCheckingEvaluator)
- [ ] Observability (Actuator, Prometheus, Grafana, OpenTelemetry)
- [ ] Docker Compose for the whole system

## 📄 License

This project is licensed under the MIT License — see the `LICENSE` file for details.

## 👤 Author

**Kartikeya Dhasmana**

- GitHub: [@zasha12](https://github.com/zasha12)
- LinkedIn: [Kartikeya Dhasmana](https://linkedin.com/in/kartikeya-dhasmana-b91389245)
- Email: kartikeyadhasmana@gmail.com

⭐ If this helped you understand Spring AI, star it!
