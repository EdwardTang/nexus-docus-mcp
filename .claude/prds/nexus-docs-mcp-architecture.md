# Nexus Docs MCP Tool - Technical Architecture

## Executive Summary

The Nexus Docs MCP tool provides Qumulo support engineers with intelligent documentation access through a chatbot interface. It combines RAG (Retrieval-Augmented Generation) for documentation search with real-time telemetry analysis to deliver contextual, actionable support guidance.

**Three-Lens Analysis Applied:**

- **Graham Lens (User Experience)**: Instant, contextual answers with citations; zero friction for engineers needing fast information
- **Hassabis Lens (AI Capability)**: Designed for future enhancements—multimodal support, predictive diagnostics, and continuous learning from support interactions
- **Musk Lens (First Principles)**: Simplest architecture for 80% value—stateless MCP server + vector search + real-time telemetry, no unnecessary complexity

---

## 1. Technical Requirements

### 1.1 Functional Requirements

**FR-1: Documentation Retrieval**
- Parse and index qumulo-docs.txt (assumed to be comprehensive Qumulo product documentation)
- Support semantic search across documentation corpus
- Return relevant passages with citations to docs.qumulo.com
- Handle versioned documentation if applicable

**FR-2: Real-Time Telemetry Integration**
- Connect to Qumulo system APIs for live cluster status
- Retrieve metrics: performance, capacity, health, alerts
- Correlate telemetry data with documentation context
- Support multiple cluster connections simultaneously

**FR-3: API Call Identification**
- Analyze user queries for API-related questions
- Map API calls to documentation sections
- Provide code examples and usage patterns
- Track API version compatibility

**FR-4: Behavioral Analysis**
- Identify patterns in support queries
- Surface common issues based on telemetry signatures
- Suggest proactive diagnostics
- Learn from historical support interactions

### 1.2 Non-Functional Requirements

**NFR-1: Performance**
- Query response time: <2s for documentation retrieval
- Telemetry fetch: <500ms per cluster
- Support 100+ concurrent chat sessions
- Index refresh: <5 minutes for documentation updates

**NFR-2: Scalability**
- Horizontal scaling for MCP server instances
- Vector database capable of 10M+ embeddings
- Support 1000+ Qumulo clusters monitored
- Handle 10K+ queries per day

**NFR-3: Reliability**
- 99.9% uptime for MCP server
- Graceful degradation if telemetry unavailable
- Automatic retry for failed API calls
- Health monitoring and alerting

**NFR-4: Security**
- Encrypted communication (TLS 1.3)
- API key rotation for Qumulo cluster access
- Role-based access control (RBAC) for MCP tools
- Audit logging for all queries and data access

---

## 2. MCP Server Architecture

### 2.1 Component Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Nexus Chatbot                           │
│                    (Client Application)                         │
└──────────────────┬──────────────────────────────────────────────┘
                   │ MCP Protocol (JSON-RPC)
                   │
┌──────────────────▼──────────────────────────────────────────────┐
│                    MCP Server Gateway                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Tool       │  │  Resource    │  │   Prompt     │         │
│  │  Registry    │  │  Manager     │  │  Templates   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└──────────────────┬──────────────────────────────────────────────┘
                   │
         ┌─────────┼─────────┐
         │         │         │
         ▼         ▼         ▼
    ┌────────┐ ┌──────┐ ┌──────────┐
    │  RAG   │ │Telem │ │   API    │
    │ Engine │ │etry  │ │ Analyzer │
    └────┬───┘ └───┬──┘ └────┬─────┘
         │         │         │
    ┌────▼─────────▼─────────▼──────┐
    │    Service Orchestration       │
    └────────────────────────────────┘
```

### 2.2 Core Components

#### 2.2.1 MCP Server Gateway

**Responsibility**: Protocol handling, authentication, request routing

**Implementation**:
```typescript
interface MCPServerConfig {
  name: "nexus-docs-mcp";
  version: "1.0.0";
  capabilities: {
    tools: true;
    resources: true;
    prompts: true;
  };
}

// MCP Tools Exposed
const tools = [
  {
    name: "search_docs",
    description: "Search Qumulo documentation with semantic understanding",
    inputSchema: {
      query: "string",
      filters: { version?: string, category?: string },
      limit: "number (default: 5)"
    }
  },
  {
    name: "get_telemetry",
    description: "Retrieve real-time metrics from Qumulo cluster",
    inputSchema: {
      clusterId: "string",
      metrics: "string[] (capacity, performance, health, alerts)",
      timeRange?: "string (e.g., '1h', '24h')"
    }
  },
  {
    name: "analyze_api_call",
    description: "Get documentation and examples for Qumulo API calls",
    inputSchema: {
      apiEndpoint: "string",
      version?: "string",
      includeExamples: "boolean"
    }
  },
  {
    name: "diagnose_issue",
    description: "Correlate telemetry with known issues and solutions",
    inputSchema: {
      clusterId: "string",
      symptoms: "string",
      includeHistory: "boolean"
    }
  }
];

// MCP Resources Exposed
const resources = [
  {
    uri: "docs://qumulo/{version}/{category}/{topic}",
    name: "Documentation sections",
    mimeType: "text/markdown"
  },
  {
    uri: "telemetry://{clusterId}/{metric}",
    name: "Real-time cluster metrics",
    mimeType: "application/json"
  },
  {
    uri: "api-spec://{version}/{endpoint}",
    name: "API specifications and schemas",
    mimeType: "application/json"
  }
];
```

**Technology Stack**:
- **Runtime**: Node.js 20+ (TypeScript)
- **Framework**: `@modelcontextprotocol/sdk` for MCP protocol
- **Server**: Express.js or Fastify for HTTP endpoints
- **IPC**: JSON-RPC 2.0 over stdio/HTTP/SSE

#### 2.2.2 RAG Engine

**Responsibility**: Document indexing, semantic search, citation generation

**Architecture**:
```
┌──────────────────────────────────────┐
│         Document Ingestion           │
│  ┌────────────────────────────────┐  │
│  │  qumulo-docs.txt Parser        │  │
│  │  • Markdown/text extraction    │  │
│  │  • Section identification      │  │
│  │  • Version tagging             │  │
│  └────────────┬───────────────────┘  │
│               ▼                      │
│  ┌────────────────────────────────┐  │
│  │    Text Chunking & Splitting   │  │
│  │  • Semantic chunking (512-1024)│  │
│  │  • Overlap: 128 tokens         │  │
│  │  • Preserve context boundaries │  │
│  └────────────┬───────────────────┘  │
│               ▼                      │
│  ┌────────────────────────────────┐  │
│  │    Embedding Generation        │  │
│  │  • Model: text-embedding-3     │  │
│  │  • Dimensions: 1536            │  │
│  │  • Batch size: 100             │  │
│  └────────────┬───────────────────┘  │
│               ▼                      │
│  ┌────────────────────────────────┐  │
│  │    Vector Database Storage     │  │
│  │  • Chroma / FAISS / Qdrant     │  │
│  │  • Index: HNSW                 │  │
│  │  • Metadata: version, url, etc │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
         │
         │ Query Path
         ▼
┌──────────────────────────────────────┐
│         Query Processing             │
│  ┌────────────────────────────────┐  │
│  │  Query Embedding               │  │
│  │  • Same model as indexing      │  │
│  └────────────┬───────────────────┘  │
│               ▼                      │
│  ┌────────────────────────────────┐  │
│  │  Vector Similarity Search      │  │
│  │  • k=5-10 nearest neighbors    │  │
│  │  • Cosine similarity           │  │
│  │  • Score threshold: >0.7       │  │
│  └────────────┬───────────────────┘  │
│               ▼                      │
│  ┌────────────────────────────────┐  │
│  │  Re-ranking & Filtering        │  │
│  │  • Cross-encoder rerank (opt)  │  │
│  │  • Version filtering           │  │
│  │  • Diversity selection         │  │
│  └────────────┬───────────────────┘  │
│               ▼                      │
│  ┌────────────────────────────────┐  │
│  │  Citation Generation           │  │
│  │  • Map chunk → docs URL        │  │
│  │  • Add section headers         │  │
│  │  • Include relevance score     │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

**Implementation Details**:

```python
# Document Chunking Strategy
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1024,
    chunk_overlap=128,
    separators=["\n\n", "\n", ". ", " ", ""],
    length_function=len
)

# Metadata Schema
class DocumentChunk:
    content: str
    metadata: {
        source_file: str          # "qumulo-docs.txt"
        version: str              # "7.2.5"
        category: str             # "API", "Administration", "Performance"
        url: str                  # "https://docs.qumulo.com/..."
        section_title: str        # "Cluster Configuration"
        chunk_index: int          # 0, 1, 2...
        total_chunks: int         # Total chunks in this section
        timestamp: datetime       # When indexed
    }

# Vector Store Configuration
vector_store_config = {
    "type": "chroma",  # or "qdrant", "faiss"
    "collection_name": "qumulo_docs",
    "embedding_function": "text-embedding-3-small",
    "distance_metric": "cosine",
    "index_params": {
        "type": "hnsw",
        "M": 16,              # Max connections per layer
        "ef_construction": 200 # Size of dynamic candidate list
    }
}

# Citation Linking Logic
def generate_citation(chunk: DocumentChunk, score: float) -> Citation:
    return {
        "text": chunk.content,
        "url": chunk.metadata.url,
        "section": chunk.metadata.section_title,
        "version": chunk.metadata.version,
        "relevance_score": score,
        "snippet": extract_snippet(chunk.content, max_length=200)
    }
```

**Technology Choices**:

- **Vector Database**: ChromaDB (lightweight, embedded option) or Qdrant (production scale)
- **Embedding Model**: OpenAI `text-embedding-3-small` (1536 dims, cost-effective) or `text-embedding-3-large` (3072 dims, higher accuracy)
- **Alternative**: Local embeddings with `sentence-transformers` (all-MiniLM-L6-v2) for cost savings
- **Chunking**: LangChain's `RecursiveCharacterTextSplitter` with semantic preservation

#### 2.2.3 Telemetry Integration

**Responsibility**: Real-time cluster monitoring, metrics aggregation, alert correlation

**Architecture**:
```
┌────────────────────────────────────────────────┐
│         Telemetry Collector                    │
│  ┌──────────────────────────────────────────┐  │
│  │  Qumulo REST API Client Pool             │  │
│  │  • Connection pooling (max 50/cluster)   │  │
│  │  • Auth token caching (TTL: 1h)          │  │
│  │  • Rate limiting (100 req/min)           │  │
│  └──────────────────┬───────────────────────┘  │
│                     ▼                          │
│  ┌──────────────────────────────────────────┐  │
│  │  Metrics Aggregation                     │  │
│  │  • Capacity: used, free, total           │  │
│  │  • Performance: IOPS, throughput, latency│  │
│  │  • Health: node status, drive status     │  │
│  │  • Alerts: active, recent, severity      │  │
│  └──────────────────┬───────────────────────┘  │
│                     ▼                          │
│  ┌──────────────────────────────────────────┐  │
│  │  Time-Series Caching (Redis)             │  │
│  │  • 1-hour retention for fast lookback    │  │
│  │  • Aggregated stats (avg, max, p95)      │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

**API Integration Example**:
```typescript
interface QumuloClusterClient {
  // Core metrics endpoints
  getCapacity(clusterId: string): Promise<CapacityMetrics>;
  getPerformance(clusterId: string, timeRange: string): Promise<PerfMetrics>;
  getHealth(clusterId: string): Promise<HealthStatus>;
  getAlerts(clusterId: string, severity?: string[]): Promise<Alert[]>;

  // API call introspection
  getApiVersion(clusterId: string): Promise<string>;
  validateApiCall(endpoint: string, version: string): Promise<boolean>;
}

// Metrics response format
interface CapacityMetrics {
  clusterId: string;
  timestamp: Date;
  totalCapacity: number;    // bytes
  usedCapacity: number;     // bytes
  freeCapacity: number;     // bytes
  utilizationPercent: number;
  growthRate: number;       // bytes/day (trend)
}

interface Alert {
  id: string;
  severity: "critical" | "warning" | "info";
  category: "capacity" | "performance" | "hardware" | "network";
  message: string;
  timestamp: Date;
  clusterId: string;
  nodeId?: string;
  resolved: boolean;
}
```

**Correlation Engine**:
```typescript
// Correlate telemetry with documentation
async function diagnoseIssue(
  clusterId: string,
  symptoms: string
): Promise<Diagnosis> {
  // 1. Fetch current telemetry
  const telemetry = await getTelemetrySnapshot(clusterId);

  // 2. Identify abnormal patterns
  const anomalies = detectAnomalies(telemetry);

  // 3. Search documentation for related issues
  const query = `${symptoms} ${anomalies.join(' ')}`;
  const docs = await ragEngine.search(query, {
    category: "troubleshooting",
    limit: 5
  });

  // 4. Return diagnosis with citations
  return {
    summary: generateSummary(anomalies, docs),
    telemetry: telemetry,
    relatedDocs: docs.map(d => d.citation),
    suggestedActions: extractActions(docs),
    confidence: calculateConfidence(anomalies, docs)
  };
}
```

#### 2.2.4 API Analyzer

**Responsibility**: API call documentation, version compatibility, code examples

**Features**:
- Parse API endpoint requests and map to documentation
- Version compatibility checking (e.g., API available in v7.2+ only)
- Code example generation for common languages (Python, JavaScript, cURL)
- Parameter validation and schema documentation

**Implementation**:
```typescript
interface ApiAnalyzer {
  // Analyze API call and retrieve docs
  async analyzeApiCall(request: {
    endpoint: string;
    method: "GET" | "POST" | "PUT" | "DELETE" | "PATCH";
    version?: string;
  }): Promise<ApiDocumentation>;
}

interface ApiDocumentation {
  endpoint: string;
  description: string;
  parameters: Parameter[];
  requestSchema: JSONSchema;
  responseSchema: JSONSchema;
  examples: CodeExample[];
  versionInfo: {
    introducedIn: string;    // "7.0.0"
    deprecatedIn?: string;   // "8.0.0"
    removedIn?: string;      // "9.0.0"
    alternatives?: string[]; // Suggested replacements
  };
  relatedDocs: Citation[];
  rateLimits?: RateLimit;
}

interface CodeExample {
  language: "python" | "javascript" | "curl" | "go";
  code: string;
  description: string;
}
```

---

## 3. Data Flow Architecture

### 3.1 Query Processing Flow

```
User Query
    │
    ▼
┌───────────────────────────────────┐
│ Nexus Chatbot (Client)            │
│ • Parse user intent               │
│ • Determine required MCP tools    │
└───────────┬───────────────────────┘
            │ MCP JSON-RPC Request
            ▼
┌───────────────────────────────────┐
│ MCP Server Gateway                │
│ • Authenticate request            │
│ • Route to appropriate tool       │
└───────────┬───────────────────────┘
            │
    ┌───────┴───────┐
    │               │
    ▼               ▼
┌──────────┐   ┌──────────┐
│ RAG Tool │   │ Telem    │
│          │   │ Tool     │
└────┬─────┘   └────┬─────┘
     │              │
     │              │ (Parallel execution)
     │              │
     ▼              ▼
┌──────────┐   ┌──────────┐
│ Vector   │   │ Qumulo   │
│ Search   │   │ API      │
└────┬─────┘   └────┬─────┘
     │              │
     │              │
     └──────┬───────┘
            │
            ▼
┌───────────────────────────────────┐
│ Response Aggregation              │
│ • Merge documentation + telemetry │
│ • Generate citations              │
│ • Format response                 │
└───────────┬───────────────────────┘
            │ MCP JSON-RPC Response
            ▼
┌───────────────────────────────────┐
│ Nexus Chatbot (Client)            │
│ • Display formatted answer        │
│ • Render citations with links     │
│ • Show telemetry visualizations   │
└───────────────────────────────────┘
```

### 3.2 Documentation Indexing Flow

```
qumulo-docs.txt
    │
    ▼
┌───────────────────────────────────┐
│ Document Parser                   │
│ • Split by sections               │
│ • Extract metadata (version, URL) │
│ • Preserve formatting             │
└───────────┬───────────────────────┘
            │
            ▼
┌───────────────────────────────────┐
│ Chunking Engine                   │
│ • Semantic chunking (1024 tokens) │
│ • Overlap: 128 tokens             │
│ • Preserve context boundaries     │
└───────────┬───────────────────────┘
            │
            ▼
┌───────────────────────────────────┐
│ Embedding Service                 │
│ • Batch embed chunks              │
│ • Model: text-embedding-3-small   │
│ • Rate limit: 3000 req/min        │
└───────────┬───────────────────────┘
            │
            ▼
┌───────────────────────────────────┐
│ Vector Database (Chroma/Qdrant)   │
│ • Store embeddings + metadata     │
│ • Build HNSW index                │
│ • Enable similarity search        │
└───────────────────────────────────┘
```

### 3.3 Real-Time Telemetry Flow

```
Qumulo Cluster
    │
    ▼
┌───────────────────────────────────┐
│ Qumulo REST API                   │
│ • /v1/cluster/capacity            │
│ • /v1/cluster/performance         │
│ • /v1/cluster/health              │
│ • /v1/cluster/alerts              │
└───────────┬───────────────────────┘
            │
            ▼
┌───────────────────────────────────┐
│ Telemetry Collector               │
│ • Poll APIs (interval: 30s-5min)  │
│ • Connection pool per cluster     │
│ • Auth token management           │
└───────────┬───────────────────────┘
            │
            ▼
┌───────────────────────────────────┐
│ Time-Series Cache (Redis)         │
│ • 1-hour retention                │
│ • Pre-aggregated stats            │
│ • Fast read access                │
└───────────┬───────────────────────┘
            │
            ▼
┌───────────────────────────────────┐
│ MCP Tool: get_telemetry           │
│ • Fetch from cache                │
│ • Format for chat display         │
│ • Include trend analysis          │
└───────────────────────────────────┘
```

---

## 4. MCP Protocol Implementation

### 4.1 Server Initialization

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server(
  {
    name: "nexus-docs-mcp",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
      resources: {},
      prompts: {},
    },
  }
);

// Register tools
server.setRequestHandler("tools/list", async () => {
  return {
    tools: [
      {
        name: "search_docs",
        description: "Search Qumulo documentation with semantic understanding",
        inputSchema: {
          type: "object",
          properties: {
            query: {
              type: "string",
              description: "Search query for documentation"
            },
            filters: {
              type: "object",
              properties: {
                version: { type: "string" },
                category: { type: "string" }
              }
            },
            limit: {
              type: "number",
              description: "Max results to return",
              default: 5
            }
          },
          required: ["query"]
        }
      },
      // ... other tools
    ]
  };
});

// Tool execution handler
server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;

  switch (name) {
    case "search_docs":
      return await handleSearchDocs(args);
    case "get_telemetry":
      return await handleGetTelemetry(args);
    case "analyze_api_call":
      return await handleAnalyzeApi(args);
    case "diagnose_issue":
      return await handleDiagnoseIssue(args);
    default:
      throw new Error(`Unknown tool: ${name}`);
  }
});

// Start server
const transport = new StdioServerTransport();
await server.connect(transport);
```

### 4.2 Tool Implementation Examples

```typescript
async function handleSearchDocs(args: {
  query: string;
  filters?: { version?: string; category?: string };
  limit?: number;
}): Promise<ToolResponse> {
  // 1. Generate query embedding
  const embedding = await embedQuery(args.query);

  // 2. Vector search with filters
  const results = await vectorStore.similaritySearch(embedding, {
    filter: args.filters,
    limit: args.limit || 5,
    scoreThreshold: 0.7
  });

  // 3. Generate citations
  const citations = results.map(r => generateCitation(r));

  // 4. Format response
  return {
    content: [
      {
        type: "text",
        text: formatSearchResults(results, citations)
      }
    ],
    isError: false
  };
}

async function handleGetTelemetry(args: {
  clusterId: string;
  metrics: string[];
  timeRange?: string;
}): Promise<ToolResponse> {
  // 1. Fetch from cache or API
  const telemetry = await telemetryService.getMetrics(
    args.clusterId,
    args.metrics,
    args.timeRange
  );

  // 2. Format for display
  return {
    content: [
      {
        type: "text",
        text: formatTelemetryData(telemetry)
      }
    ],
    isError: false
  };
}
```

### 4.3 Resource Handlers

```typescript
server.setRequestHandler("resources/list", async () => {
  return {
    resources: [
      {
        uri: "docs://qumulo",
        name: "Qumulo Documentation",
        description: "Complete Qumulo product documentation",
        mimeType: "text/markdown"
      },
      {
        uri: "telemetry://clusters",
        name: "Cluster Telemetry",
        description: "Real-time cluster metrics",
        mimeType: "application/json"
      }
    ]
  };
});

server.setRequestHandler("resources/read", async (request) => {
  const { uri } = request.params;

  if (uri.startsWith("docs://")) {
    const docPath = uri.replace("docs://", "");
    const content = await fetchDocumentation(docPath);
    return {
      contents: [
        {
          uri,
          mimeType: "text/markdown",
          text: content
        }
      ]
    };
  }

  if (uri.startsWith("telemetry://")) {
    const clusterId = uri.replace("telemetry://clusters/", "");
    const data = await fetchTelemetry(clusterId);
    return {
      contents: [
        {
          uri,
          mimeType: "application/json",
          text: JSON.stringify(data, null, 2)
        }
      ]
    };
  }

  throw new Error(`Unknown resource URI: ${uri}`);
});
```

---

## 5. Integration with Nexus Chatbot

### 5.1 Chatbot Architecture

```
┌──────────────────────────────────────────────────┐
│            Nexus Chatbot Frontend                │
│  ┌────────────────────────────────────────────┐  │
│  │  Chat UI (React/Vue)                       │  │
│  │  • Message history                         │  │
│  │  • Citation rendering                      │  │
│  │  • Telemetry visualizations                │  │
│  └────────────────┬───────────────────────────┘  │
│                   │                              │
│  ┌────────────────▼───────────────────────────┐  │
│  │  MCP Client                                │  │
│  │  • Protocol handling                       │  │
│  │  • Tool invocation                         │  │
│  │  • Response parsing                        │  │
│  └────────────────┬───────────────────────────┘  │
└───────────────────┼──────────────────────────────┘
                    │ JSON-RPC over HTTP/SSE
                    ▼
┌──────────────────────────────────────────────────┐
│         Nexus Docs MCP Server                    │
└──────────────────────────────────────────────────┘
```

### 5.2 Message Flow Example

**User Query**: "Why is my cluster showing high latency? Cluster ID: prod-cluster-01"

**Step 1: Chatbot parses intent**
- Identifies: performance issue + specific cluster
- Determines tools needed: `get_telemetry` + `search_docs`

**Step 2: MCP tool invocations (parallel)**
```json
// Tool call 1: Get telemetry
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_telemetry",
    "arguments": {
      "clusterId": "prod-cluster-01",
      "metrics": ["performance", "health", "alerts"],
      "timeRange": "1h"
    }
  }
}

// Tool call 2: Search docs
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "search_docs",
    "arguments": {
      "query": "high latency troubleshooting performance",
      "filters": { "category": "troubleshooting" },
      "limit": 5
    }
  }
}
```

**Step 3: MCP server responses**
```json
// Response 1: Telemetry
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{
      "type": "text",
      "text": "Cluster: prod-cluster-01\nLatency (avg): 45ms (↑ 200% vs 24h ago)\nIOPS: 12,500\nActive Alerts: 2 (WARNING: Network congestion on node-03)"
    }]
  }
}

// Response 2: Documentation
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [{
      "type": "text",
      "text": "Found 3 relevant articles:\n1. Diagnosing High Latency Issues\n   URL: https://docs.qumulo.com/troubleshooting/latency\n   Snippet: Common causes include network saturation, drive failures...\n   Relevance: 0.89\n\n2. Network Performance Tuning\n   URL: https://docs.qumulo.com/admin/network-tuning\n   Relevance: 0.82"
    }]
  }
}
```

**Step 4: Chatbot synthesizes response**
```
Your cluster prod-cluster-01 is experiencing elevated latency (45ms, up 200% from normal).

Current Status:
• IOPS: 12,500
• Active Alerts: 2 warnings
  - Network congestion detected on node-03

Recommended Actions:
1. Check network connectivity to node-03
2. Review network configuration settings
3. Monitor drive performance metrics

Related Documentation:
📄 Diagnosing High Latency Issues
   https://docs.qumulo.com/troubleshooting/latency

📄 Network Performance Tuning
   https://docs.qumulo.com/admin/network-tuning
```

### 5.3 Client Implementation

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";

class NexusChatbotClient {
  private mcpClient: Client;

  async initialize() {
    this.mcpClient = new Client({
      name: "nexus-chatbot",
      version: "1.0.0"
    });

    await this.mcpClient.connect({
      command: "node",
      args: ["/path/to/nexus-docs-mcp/dist/index.js"]
    });
  }

  async handleUserQuery(query: string, clusterId?: string) {
    // Determine which tools to call based on query
    const tools = this.determineRequiredTools(query);

    // Execute tools in parallel
    const results = await Promise.all(
      tools.map(tool => this.mcpClient.callTool(tool))
    );

    // Synthesize response
    return this.synthesizeResponse(results);
  }

  private determineRequiredTools(query: string): ToolCall[] {
    const tools: ToolCall[] = [];

    // Always search docs
    tools.push({
      name: "search_docs",
      arguments: { query, limit: 5 }
    });

    // If cluster mentioned, get telemetry
    const clusterMatch = query.match(/cluster[:\s]+([a-z0-9-]+)/i);
    if (clusterMatch) {
      tools.push({
        name: "get_telemetry",
        arguments: {
          clusterId: clusterMatch[1],
          metrics: ["performance", "health", "alerts"]
        }
      });
    }

    // If API mentioned, analyze API
    if (query.toLowerCase().includes("api")) {
      tools.push({
        name: "analyze_api_call",
        arguments: { apiEndpoint: this.extractApiEndpoint(query) }
      });
    }

    return tools;
  }
}
```

---

## 6. Scalability & Performance

### 6.1 Horizontal Scaling Architecture

```
┌─────────────────────────────────────────────────────┐
│              Load Balancer (nginx/HAProxy)          │
└───────────────┬─────────────────────────────────────┘
                │
        ┌───────┴───────┐
        │               │
        ▼               ▼
┌──────────────┐  ┌──────────────┐
│ MCP Server 1 │  │ MCP Server 2 │  ... (N instances)
└──────┬───────┘  └──────┬───────┘
       │                 │
       └────────┬────────┘
                │
        ┌───────┴────────────┐
        │                    │
        ▼                    ▼
┌──────────────┐      ┌──────────────┐
│ Vector DB    │      │ Redis Cache  │
│ (Qdrant)     │      │ (Telemetry)  │
│ • Sharded    │      │ • Cluster    │
│ • Replicated │      │ • Persistent │
└──────────────┘      └──────────────┘
```

**Scaling Strategy**:
- **MCP Servers**: Stateless, scale to 10+ instances based on request load
- **Vector DB**: Shard by document version or category, replicate for read scaling
- **Cache**: Redis cluster for distributed caching of telemetry data

### 6.2 Performance Optimizations

**6.2.1 Query Optimization**
```typescript
// Caching strategy for embeddings
const queryEmbeddingCache = new LRUCache({
  max: 1000,  // Cache 1000 most recent queries
  ttl: 3600000 // 1 hour TTL
});

async function getQueryEmbedding(query: string): Promise<number[]> {
  const cached = queryEmbeddingCache.get(query);
  if (cached) return cached;

  const embedding = await embedModel.embed(query);
  queryEmbeddingCache.set(query, embedding);
  return embedding;
}
```

**6.2.2 Batch Processing**
```typescript
// Batch telemetry requests
class TelemetryBatcher {
  private queue: TelemetryRequest[] = [];
  private batchInterval = 100; // ms

  async request(clusterId: string, metrics: string[]): Promise<TelemetryData> {
    return new Promise((resolve, reject) => {
      this.queue.push({ clusterId, metrics, resolve, reject });

      if (this.queue.length === 1) {
        setTimeout(() => this.processBatch(), this.batchInterval);
      }
    });
  }

  private async processBatch() {
    const batch = this.queue.splice(0, 50); // Max 50 per batch
    const results = await fetchTelemetryBatch(
      batch.map(r => ({ clusterId: r.clusterId, metrics: r.metrics }))
    );

    batch.forEach((req, i) => req.resolve(results[i]));
  }
}
```

**6.2.3 Connection Pooling**
```typescript
// Qumulo API connection pool
const connectionPool = new Pool({
  max: 50,  // Max connections per cluster
  min: 5,   // Min idle connections
  acquire: 30000,  // Max wait for connection: 30s
  idle: 10000      // Idle timeout: 10s
});
```

### 6.3 Monitoring & Observability

**Metrics to Track**:
- Query latency (p50, p95, p99)
- Vector search performance
- Telemetry API response times
- Cache hit rates
- MCP tool invocation counts
- Error rates by tool/resource

**Implementation**:
```typescript
import { Histogram, Counter } from 'prom-client';

const queryLatency = new Histogram({
  name: 'nexus_query_latency_seconds',
  help: 'Query processing latency',
  labelNames: ['tool']
});

const toolInvocations = new Counter({
  name: 'nexus_tool_invocations_total',
  help: 'Total tool invocations',
  labelNames: ['tool', 'status']
});

// Usage
const timer = queryLatency.startTimer({ tool: 'search_docs' });
try {
  const result = await handleSearchDocs(args);
  toolInvocations.inc({ tool: 'search_docs', status: 'success' });
  return result;
} catch (error) {
  toolInvocations.inc({ tool: 'search_docs', status: 'error' });
  throw error;
} finally {
  timer();
}
```

---

## 7. Implementation Recommendations

### 7.1 Technology Stack Summary

| Component | Recommended | Alternative |
|-----------|-------------|-------------|
| **MCP Server** | Node.js 20+ (TypeScript) | Python 3.11+ |
| **MCP SDK** | `@modelcontextprotocol/sdk` | Custom JSON-RPC |
| **Vector DB** | ChromaDB (dev), Qdrant (prod) | FAISS, Pinecone |
| **Embeddings** | OpenAI `text-embedding-3-small` | `sentence-transformers` |
| **Cache** | Redis 7+ | Memcached |
| **RAG Framework** | LangChain | LlamaIndex, custom |
| **API Client** | Axios with retry | `fetch` + custom retry |
| **Monitoring** | Prometheus + Grafana | Datadog, New Relic |

### 7.2 Development Phases

**Phase 1: MVP (4-6 weeks)**
- ✅ Basic MCP server with `search_docs` tool
- ✅ Document parsing and vector indexing
- ✅ Simple citation generation
- ✅ Single-cluster telemetry integration
- ✅ Chatbot client integration

**Phase 2: Enhanced Features (4-6 weeks)**
- ✅ Multi-cluster telemetry support
- ✅ API analyzer tool
- ✅ Advanced citation linking
- ✅ Query result caching
- ✅ Performance optimizations

**Phase 3: Intelligence Layer (6-8 weeks)**
- ✅ Behavioral analysis and pattern detection
- ✅ Proactive diagnostics
- ✅ Historical query learning
- ✅ Predictive alerts
- ✅ Anomaly detection

**Phase 4: Production Hardening (4 weeks)**
- ✅ Horizontal scaling setup
- ✅ High availability configuration
- ✅ Comprehensive monitoring
- ✅ Security hardening
- ✅ Load testing and optimization

### 7.3 Key Implementation Decisions

**7.3.1 Vector Database Choice**

**For MVP/Small Scale (< 1M docs)**:
- **ChromaDB**: Embedded, lightweight, easy setup
- Pros: Zero infrastructure, fast iteration
- Cons: Limited scale, single-node only

**For Production/Large Scale (> 1M docs)**:
- **Qdrant**: Distributed, high-performance, feature-rich
- Pros: Horizontal scaling, filtering, multi-tenancy
- Cons: More complex setup, separate service

**7.3.2 Embedding Strategy**

**Option A: OpenAI Embeddings**
- Model: `text-embedding-3-small` (1536 dims)
- Cost: ~$0.02 per 1M tokens
- Pros: High quality, maintained by OpenAI
- Cons: API dependency, ongoing cost

**Option B: Local Embeddings**
- Model: `all-MiniLM-L6-v2` (384 dims)
- Cost: Infrastructure only
- Pros: No external dependency, zero marginal cost
- Cons: Lower quality, requires GPU for speed

**Recommendation**: Start with OpenAI for quality, migrate to local if cost becomes prohibitive

**7.3.3 Telemetry Architecture**

**Polling vs Streaming**:
- **Polling**: Simple, works with any API, easier debugging
- **Streaming**: Real-time, lower latency, more complex

**Recommendation**: Start with polling (30s-5min intervals), add streaming if real-time requirements emerge

### 7.4 Security Considerations

**7.4.1 Authentication & Authorization**
```typescript
// MCP server authentication
interface AuthConfig {
  // API key for chatbot clients
  clientApiKeys: Map<string, ClientPermissions>;

  // Qumulo cluster credentials (encrypted at rest)
  clusterCredentials: Map<string, QumuloAuth>;

  // Token rotation policy
  tokenRotationInterval: number; // milliseconds
}

// RBAC for MCP tools
interface ClientPermissions {
  allowedTools: string[];
  allowedClusters: string[];
  rateLimit: {
    requestsPerMinute: number;
    burstSize: number;
  };
}
```

**7.4.2 Data Protection**
- Encrypt Qumulo API credentials with KMS
- TLS 1.3 for all network communication
- No logging of sensitive telemetry data
- Audit trail for all cluster access

**7.4.3 Rate Limiting**
```typescript
// Per-client rate limiting
const rateLimiter = new RateLimiter({
  points: 100,        // 100 requests
  duration: 60,       // per 60 seconds
  blockDuration: 300  // block for 5 minutes if exceeded
});

// Per-cluster API rate limiting
const clusterLimiter = new RateLimiter({
  points: 1000,       // 1000 API calls
  duration: 60,       // per minute per cluster
  blockDuration: 60   // cooldown: 1 minute
});
```

---

## 8. Three-Lens Analysis Application

### 8.1 Graham Lens: User Experience Excellence

**Design Principles**:
1. **Zero Friction Access**: No login required for basic docs search, authenticated for telemetry
2. **Instant Feedback**: <2s response time for all queries
3. **Progressive Enhancement**: Show docs immediately, add telemetry if available
4. **Clear Citations**: Every answer links to source with section highlighting
5. **Visual Clarity**: Telemetry data rendered as charts/graphs, not raw JSON

**UX Optimizations**:
- **Smart Caching**: Frequently asked questions served from cache (<100ms)
- **Predictive Prefetch**: Load related docs based on query patterns
- **Graceful Degradation**: If telemetry unavailable, still provide docs
- **Feedback Loop**: "Was this helpful?" to improve future responses

### 8.2 Hassabis Lens: AI Capability Evolution

**Future Enhancements**:

**Phase 1 (Current)**: RAG + Real-time telemetry
- Semantic search over static documentation
- Real-time cluster metrics correlation

**Phase 2 (6-12 months)**: Predictive diagnostics
- Anomaly detection from telemetry patterns
- Proactive issue warnings before user asks
- Historical analysis: "Similar issues seen 3 times this month"

**Phase 3 (12-24 months)**: Autonomous support
- Auto-generate runbooks from support interactions
- Self-healing recommendations with confidence scores
- Multi-modal support: analyze screenshots, log files
- Continuous learning from support ticket outcomes

**Phase 4 (24+ months)**: Distributed intelligence
- Federated learning across customer deployments (privacy-preserving)
- Cluster-specific optimization recommendations
- Predictive capacity planning based on usage trends

**Architecture for Future AI**:
```
Current: Query → RAG → Response

Future: Query → Intent Analysis → Multi-Agent System
                                      ├─ Documentation Agent
                                      ├─ Telemetry Agent
                                      ├─ Diagnostic Agent
                                      ├─ Code Generation Agent
                                      └─ Orchestrator (synthesizes)
```

### 8.3 Musk Lens: First Principles Simplicity

**80/20 Analysis**:

**80% of Value**:
1. Fast, accurate documentation search (RAG)
2. Real-time cluster health visibility
3. Clear citations to official docs

**Unnecessary Complexity to Avoid**:
- ❌ Custom embedding models (use OpenAI)
- ❌ Complex multi-stage retrieval (start with single-stage)
- ❌ Custom auth system (use API keys initially)
- ❌ Real-time collaboration features
- ❌ Advanced NLP for query understanding (let LLM handle it)

**Simplest Possible Architecture**:
```
┌─────────────┐
│  Chatbot    │
└──────┬──────┘
       │ MCP
       ▼
┌─────────────┐     ┌──────────────┐
│ MCP Server  │────▶│ ChromaDB     │ (docs search)
│ (Node.js)   │     └──────────────┘
└─────────────┘     ┌──────────────┐
       │            │ Redis Cache  │ (telemetry)
       └───────────▶└──────────────┘
                    ┌──────────────┐
                    │ Qumulo API   │ (live data)
                    └──────────────┘
```

**Build vs Buy Decisions**:
- **Build**: MCP server, RAG engine (core value)
- **Buy**: Vector DB (ChromaDB), embeddings (OpenAI), cache (Redis)
- **Don't Build**: Auth system (use existing), monitoring (Prometheus), deployment (Kubernetes)

---

## 9. Deployment Architecture

### 9.1 Development Environment

```yaml
# docker-compose.yml
version: '3.8'

services:
  mcp-server:
    build: ./mcp-server
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - CHROMA_HOST=chromadb
    depends_on:
      - chromadb
      - redis

  chromadb:
    image: chromadb/chroma:latest
    ports:
      - "8000:8000"
    volumes:
      - chroma-data:/chroma/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  chatbot:
    build: ./nexus-chatbot
    ports:
      - "8080:8080"
    environment:
      - MCP_SERVER_URL=http://mcp-server:3000

volumes:
  chroma-data:
  redis-data:
```

### 9.2 Production Environment (Kubernetes)

```yaml
# k8s/mcp-server-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nexus-docs-mcp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nexus-docs-mcp
  template:
    metadata:
      labels:
        app: nexus-docs-mcp
    spec:
      containers:
      - name: mcp-server
        image: nexus-docs-mcp:1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: QDRANT_URL
          value: "http://qdrant:6333"
        - name: REDIS_URL
          value: "redis://redis-cluster:6379"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: nexus-docs-mcp
spec:
  selector:
    app: nexus-docs-mcp
  ports:
  - protocol: TCP
    port: 3000
    targetPort: 3000
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nexus-docs-mcp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nexus-docs-mcp
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 9.3 CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy Nexus Docs MCP

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test
      - run: npm run lint
      - run: npm run typecheck

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/build-push-action@v4
        with:
          push: true
          tags: nexus-docs-mcp:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/nexus-docs-mcp \
            mcp-server=nexus-docs-mcp:${{ github.sha }}
          kubectl rollout status deployment/nexus-docs-mcp
```

---

## 10. Cost Analysis & Optimization

### 10.1 Infrastructure Costs (Monthly Estimates)

| Component | Configuration | Cost |
|-----------|---------------|------|
| **MCP Servers** | 3x t3.medium (4 vCPU, 16GB) | $150 |
| **Vector DB (Qdrant)** | 1x m5.xlarge (4 vCPU, 16GB) | $140 |
| **Redis Cache** | 1x r6g.large (2 vCPU, 16GB) | $120 |
| **Load Balancer** | ALB with 100GB/month | $25 |
| **OpenAI Embeddings** | 10M tokens/month @ $0.02/1M | $0.20 |
| **Storage** | 100GB SSD | $10 |
| **Monitoring** | Prometheus/Grafana | $0 (self-hosted) |
| **Total** | | **~$445/month** |

**Cost Optimization Strategies**:
1. **Use local embeddings** (sentence-transformers) → Save ~$0.20/month (minimal, but scales)
2. **Spot instances for MCP servers** → Save ~40% ($60/month)
3. **Reserved instances** (1-year commitment) → Save ~30% ($130/month)
4. **Aggressive caching** → Reduce vector search load → Downsize Qdrant instance

### 10.2 Operational Costs

**Engineering Time**:
- Initial development: 16-20 weeks (Phase 1-3)
- Maintenance: 0.5 FTE ongoing
- Support: Included in existing support team

**Training**:
- Support engineer onboarding: 2-4 hours
- Documentation: 1 week to create comprehensive guides

---

## 11. Risk Assessment & Mitigation

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Vector DB scaling issues** | High | Medium | Use Qdrant with sharding; load testing before launch |
| **OpenAI API outages** | High | Low | Fallback to cached embeddings; local embedding model as backup |
| **Qumulo API rate limits** | Medium | Medium | Implement aggressive caching; batch requests; backoff strategy |
| **Documentation drift** | Medium | High | Automated daily re-indexing; version tracking; change detection |
| **Security vulnerabilities** | High | Low | Regular security audits; dependency scanning; penetration testing |
| **Performance degradation** | Medium | Medium | Continuous monitoring; auto-scaling; performance SLOs |
| **Chatbot misinterpretation** | Low | Medium | Clear error messages; feedback loop; escalation to human support |

---

## 12. Success Metrics

### 12.1 KPIs

**User Experience**:
- **Query Response Time**: p95 < 2 seconds
- **Answer Accuracy**: >90% rated "helpful" by users
- **Citation Click-Through Rate**: >40% (users find links valuable)
- **Repeat Usage**: >70% of engineers use tool weekly

**System Performance**:
- **Uptime**: 99.9%
- **Vector Search Latency**: p95 < 500ms
- **Telemetry Fetch Latency**: p95 < 300ms
- **Cache Hit Rate**: >80% for telemetry queries

**Business Impact**:
- **Support Ticket Reduction**: 20% decrease in Tier 1 tickets
- **Time to Resolution**: 30% faster for common issues
- **Engineer Satisfaction**: NPS > 50

### 12.2 Monitoring Dashboard

```
┌─────────────────────────────────────────────────────┐
│            Nexus Docs MCP Dashboard                 │
├─────────────────────────────────────────────────────┤
│ Query Metrics                                       │
│  • Total Queries (24h): 1,247                       │
│  • Avg Response Time: 1.2s                          │
│  • Error Rate: 0.3%                                 │
├─────────────────────────────────────────────────────┤
│ Tool Usage                                          │
│  • search_docs: 892 (71%)                           │
│  • get_telemetry: 245 (20%)                         │
│  • diagnose_issue: 87 (7%)                          │
│  • analyze_api_call: 23 (2%)                        │
├─────────────────────────────────────────────────────┤
│ Vector Search Performance                           │
│  • Avg Latency: 320ms                               │
│  • Index Size: 2.3M vectors                         │
│  • Cache Hit Rate: 82%                              │
├─────────────────────────────────────────────────────┤
│ Telemetry Integration                               │
│  • Active Clusters: 47                              │
│  • API Errors (24h): 3                              │
│  • Cache Hit Rate: 88%                              │
└─────────────────────────────────────────────────────┘
```

---

## 13. Conclusion & Next Steps

### 13.1 Architecture Summary

The Nexus Docs MCP tool delivers **intelligent documentation access** and **real-time telemetry analysis** through a clean, scalable MCP server architecture. By applying the three-lens analysis:

- **Graham**: Prioritizes instant, friction-free access with clear citations
- **Hassabis**: Designs for future AI enhancements (predictive diagnostics, multi-modal support)
- **Musk**: Keeps architecture simple—stateless MCP server, proven vector DB, managed services where possible

### 13.2 Recommended Next Steps

1. **Week 1-2**: Build MCP server skeleton + basic `search_docs` tool
2. **Week 3-4**: Implement RAG engine with ChromaDB + OpenAI embeddings
3. **Week 5-6**: Add telemetry integration for single cluster
4. **Week 7-8**: Integrate with Nexus chatbot frontend
5. **Week 9-10**: User testing with support engineers
6. **Week 11-12**: Production hardening + deployment
7. **Week 13+**: Iterate based on feedback; add advanced features

### 13.3 Critical Success Factors

✅ **Fast iteration**: Start simple, add complexity only when needed
✅ **User feedback**: Embed support engineers in design process
✅ **Reliable telemetry**: Ensure Qumulo API integration is rock-solid
✅ **Clear citations**: Every answer must link to authoritative docs
✅ **Performance SLOs**: <2s response time is non-negotiable

---

## Appendix A: API Specifications

### A.1 MCP Tool Schemas

```typescript
// search_docs tool
interface SearchDocsRequest {
  query: string;
  filters?: {
    version?: string;        // e.g., "7.2.5"
    category?: string;       // e.g., "API", "Administration"
  };
  limit?: number;            // default: 5, max: 20
}

interface SearchDocsResponse {
  results: Array<{
    content: string;
    citation: {
      url: string;
      section: string;
      version: string;
      relevanceScore: number;
    };
  }>;
  totalResults: number;
  searchTime: number;        // milliseconds
}

// get_telemetry tool
interface GetTelemetryRequest {
  clusterId: string;
  metrics: Array<"capacity" | "performance" | "health" | "alerts">;
  timeRange?: string;        // e.g., "1h", "24h", "7d"
}

interface GetTelemetryResponse {
  clusterId: string;
  timestamp: string;         // ISO 8601
  capacity?: {
    total: number;           // bytes
    used: number;            // bytes
    free: number;            // bytes
    utilizationPercent: number;
  };
  performance?: {
    iops: number;
    throughputMBps: number;
    latencyMs: number;
    trend: "increasing" | "stable" | "decreasing";
  };
  health?: {
    status: "healthy" | "degraded" | "critical";
    nodeCount: number;
    healthyNodes: number;
  };
  alerts?: Array<{
    severity: "critical" | "warning" | "info";
    message: string;
    timestamp: string;
  }>;
}
```

### A.2 Qumulo API Endpoints

```
# Authentication
POST /v1/session/login
  Request: { username, password }
  Response: { access_token, expires_in }

# Cluster Capacity
GET /v1/cluster/capacity
  Response: { total_capacity, used_capacity, free_capacity }

# Cluster Performance
GET /v1/cluster/performance?interval=1h
  Response: { iops, throughput, latency, timestamp }

# Node Health
GET /v1/cluster/nodes
  Response: { nodes: [{ id, status, role, disk_count }] }

# Active Alerts
GET /v1/cluster/alerts?severity=warning,critical
  Response: { alerts: [{ id, severity, message, timestamp }] }
```

---

## Appendix B: Code Examples

### B.1 Document Indexing Script

```python
#!/usr/bin/env python3
"""
Index qumulo-docs.txt into vector database
"""

from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from typing import List, Dict
import re

def parse_qumulo_docs(file_path: str) -> List[Dict]:
    """Parse qumulo-docs.txt into structured documents"""
    with open(file_path, 'r') as f:
        content = f.read()

    documents = []
    # Split by major sections (assuming ## headings)
    sections = re.split(r'\n## ', content)

    for section in sections:
        if not section.strip():
            continue

        # Extract section title and content
        lines = section.split('\n', 1)
        title = lines[0].strip()
        body = lines[1] if len(lines) > 1 else ""

        # Extract version info (if present)
        version_match = re.search(r'Version: ([\d.]+)', body)
        version = version_match.group(1) if version_match else "latest"

        # Generate docs.qumulo.com URL (simplified)
        url_slug = title.lower().replace(' ', '-').replace('/', '-')
        url = f"https://docs.qumulo.com/{url_slug}"

        documents.append({
            "content": body,
            "metadata": {
                "section": title,
                "version": version,
                "url": url,
                "category": infer_category(title)
            }
        })

    return documents

def infer_category(title: str) -> str:
    """Infer document category from title"""
    title_lower = title.lower()
    if 'api' in title_lower:
        return "API"
    elif any(kw in title_lower for kw in ['admin', 'config', 'setup']):
        return "Administration"
    elif any(kw in title_lower for kw in ['performance', 'tuning', 'optimization']):
        return "Performance"
    elif any(kw in title_lower for kw in ['troubleshoot', 'debug', 'error']):
        return "Troubleshooting"
    else:
        return "General"

def index_documents(docs: List[Dict]) -> Chroma:
    """Chunk and index documents into vector store"""
    # Initialize components
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=1024,
        chunk_overlap=128,
        separators=["\n\n", "\n", ". ", " ", ""]
    )
    embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

    # Chunk documents
    all_chunks = []
    for doc in docs:
        chunks = splitter.split_text(doc["content"])
        for i, chunk in enumerate(chunks):
            all_chunks.append({
                "content": chunk,
                "metadata": {
                    **doc["metadata"],
                    "chunk_index": i,
                    "total_chunks": len(chunks)
                }
            })

    # Create vector store
    texts = [chunk["content"] for chunk in all_chunks]
    metadatas = [chunk["metadata"] for chunk in all_chunks]

    vectorstore = Chroma.from_texts(
        texts=texts,
        metadatas=metadatas,
        embedding=embeddings,
        collection_name="qumulo_docs",
        persist_directory="./chroma_db"
    )

    print(f"Indexed {len(all_chunks)} chunks from {len(docs)} documents")
    return vectorstore

if __name__ == "__main__":
    docs = parse_qumulo_docs("qumulo-docs.txt")
    vectorstore = index_documents(docs)
    vectorstore.persist()
```

### B.2 MCP Server Main Entry Point

```typescript
// src/index.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { RAGEngine } from "./rag-engine.js";
import { TelemetryService } from "./telemetry-service.js";
import { tools, handleToolCall } from "./tools.js";

async function main() {
  // Initialize services
  const ragEngine = new RAGEngine({
    vectorDbUrl: process.env.CHROMA_URL || "http://localhost:8000",
    embeddingModel: "text-embedding-3-small"
  });

  const telemetryService = new TelemetryService({
    redisUrl: process.env.REDIS_URL || "redis://localhost:6379"
  });

  await ragEngine.initialize();
  await telemetryService.initialize();

  // Create MCP server
  const server = new Server(
    {
      name: "nexus-docs-mcp",
      version: "1.0.0",
    },
    {
      capabilities: {
        tools: {},
        resources: {},
        prompts: {}
      },
    }
  );

  // Register handlers
  server.setRequestHandler("tools/list", async () => ({ tools }));

  server.setRequestHandler("tools/call", async (request) => {
    return handleToolCall(request, {
      ragEngine,
      telemetryService
    });
  });

  // Health check endpoint
  server.setRequestHandler("ping", async () => ({
    status: "healthy",
    timestamp: new Date().toISOString()
  }));

  // Start server
  const transport = new StdioServerTransport();
  await server.connect(transport);

  console.error("Nexus Docs MCP server running");
}

main().catch(console.error);
```

---

**End of Architecture Document**
