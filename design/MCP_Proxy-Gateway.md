Here is a structured architectural overview of an **MCP Proxy & Gateway** system, formatted in Markdown for your documentation or technical design records.

---

# Architecture Design: MCP Proxy & Translation Gateway

## 1. Overview

The **MCP Proxy** acts as an intermediary layer between internal **MCP Clients** (e.g., IDEs, AI Agents) and various **Upstream Services** (External MCP Servers, REST APIs, and Private Databases). Its primary purpose is to provide a unified interface, centralize security, and translate non-native protocols into the Model Context Protocol.

## 2. Core Objectives

* **Credential Encapsulation:** Prevent API keys and secrets from being distributed to end-user machines.
* **Protocol Translation:** Expose legacy REST/SOAP/GraphQL services as native MCP Tools and Resources.
* **Unified Discovery:** Provide a single connection point (`list_tools`) for multiple downstream services.
* **Payload Packaging:** Aggregate and sanitize data from multiple sources before presenting it to the LLM.

---

## 3. System Architecture

```mermaid
graph LR
    subgraph Internal Environment
        Client[Internal MCP Client<br/>Claude Desktop/Cursor]
    end

    subgraph Security Boundary (Proxy Server)
        Proxy[MCP Proxy Server]
        Vault[(Secret Manager)]
    end

    subgraph Upstream Services
        REST[External REST API]
        RemoteMCP[Remote MCP Server]
        Payroll[Internal Payroll DB]
    end

    Client <-->|SSE / Stdio| Proxy
    Proxy <-->|Auth / HTTPS| REST
    Proxy <-->|MCP Protocol| RemoteMCP
    Proxy <-->|SQL/gRPC| Payroll
    Vault -.->|Injects Keys| Proxy

```

---

## 4. Key Functional Layers

### A. The Translation Layer (REST to MCP)

The proxy maps HTTP endpoints to MCP tool definitions.

* **Input:** The Proxy receives an `call_tool` request from the client.
* **Logic:** It maps arguments to JSON body/query parameters.
* **Output:** It executes the `fetch` request, filters the JSON response for sensitive info, and returns a `text` content block to the client.

### B. Security & Identity

* **Secret Masking:** The Proxy handles all `Authorization: Bearer <TOKEN>` headers. The client never sees the token.
* **RBAC (Role Based Access Control):** The Proxy can inspect the client's identity and prune the list of available tools (e.g., only HR-dept users see `get_payroll_details`).

### C. Payload Packaging

Instead of sending raw, verbose JSON back to the LLM (which wastes tokens), the Proxy:

1. Calls the external service.
2. **Packages** only the relevant fields into a clean Markdown table or summary.
3. Injects **Metadata** (e.g., "Data last updated at 14:00").

---

## 5. Benefits Comparison

| Feature | Without Proxy | With MCP Proxy |
| --- | --- | --- |
| **Credential Safety** | High risk (stored in local `.json` configs) | Secure (stored in environment/Vault) |
| **Tool Sprawl** | Hard to manage 10+ different servers | Single endpoint for all tools |
| **Non-Native Support** | Impossible without custom local code | Seamless via Proxy translation |
| **Audit Logging** | Fragmented across client logs | Centralized log of every AI-driven action |

---

## 6. Implementation Example (Conceptual TypeScript)

```typescript
// Example: Wrapping a REST Payroll API into an MCP Tool
server.tool(
  "get_payroll_summary",
  { employee_id: z.string() },
  async ({ employee_id }) => {
    // 1. Encapsulated Credential
    const apiKey = process.env.PAYROLL_API_KEY;

    // 2. Fetch from non-native REST server
    const response = await fetch(`https://api.internal.corp/v1/payroll/${employee_id}`, {
      headers: { 'Authorization': `Bearer ${apiKey}` }
    });
    const data = await response.json();

    // 3. Packaging the payload for the LLM
    return {
      content: [{
        type: "text",
        text: `Employee ${data.name} has a status of ${data.pay_status}.`
      }]
    };
  }
);

```

---

## 7. Next Steps

* **Standardization:** Use the `OpenAPI-to-MCP` specification to automate the generation of tools from your REST servers.
* **Transport Choice:** Use **Stdio** for local developer proxies or **SSE (Server-Sent Events)** for a centralized web-based proxy.

Would you like me to translate this architecture overview into **Traditional Chinese (繁體中文)** for your documentation?
