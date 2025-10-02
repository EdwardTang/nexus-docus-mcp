---
name: nexus-docs-mcp
status: backlog
created: 2025-10-02T20:45:01Z
progress: 0%
prd: nexus-docs-mcp-architecture.md
github: https://github.com/EdwardTang/nexus-docus-mcp/issues/1
---

# Epic: Nexus Docs MCP Tool

## Overview

Build an MCP (Model Context Protocol) server that provides Qumulo support engineers with intelligent documentation access and real-time telemetry analysis through a chatbot interface. The system combines RAG (Retrieval-Augmented Generation) for semantic documentation search with live cluster telemetry to deliver contextual, actionable support guidance with citations to official Qumulo documentation.

**Core Value Proposition**: Transform static documentation into intelligent, context-aware support responses that combine docs with real-time system state.

## Architecture Decisions

### Technology Stack
- **MCP Server Runtime**: Node.js 20+ with TypeScript for proven ecosystem and MCP SDK support
- **MCP Framework**: `@modelcontextprotocol/sdk` for protocol compliance and tool/resource handling
- **Vector Database**: ChromaDB for MVP (embedded, zero-config), Qdrant for production (distributed, scalable)
- **Embeddings**: OpenAI `text-embedding-3-small` (1536 dims) for quality, with local `sentence-transformers` as fallback
- **Cache Layer**: Redis 7+ for telemetry caching and query result optimization
- **RAG Framework**: LangChain for document chunking and retrieval orchestration

### Design Patterns
- **Stateless MCP Server**: Horizontal scaling with shared state in Redis/Vector DB
- **Tool-Based Architecture**: Each capability (search_docs, get_telemetry, analyze_api, diagnose_issue) as discrete MCP tool
- **Resource-Based Access**: Documentation and telemetry exposed as MCP resources with URI patterns
- **Async Processing**: Non-blocking telemetry fetches with connection pooling and batching
- **Graceful Degradation**: Core documentation search works even if telemetry unavailable

### Key Technical Decisions
1. **Start Simple, Scale Later**: MVP with ChromaDB embedded database, migrate to Qdrant when proven
2. **Leverage Managed Services**: Use OpenAI embeddings initially, transition to local models only if cost prohibitive
3. **Polling Over Streaming**: Telemetry via polling (30s-5min) for simplicity, add streaming if real-time requirements emerge
4. **Citation-First**: Every response must include verifiable links to docs.qumulo.com with section context

## Technical Approach

### Backend Services

#### MCP Server Core (Node.js/TypeScript)
- **Protocol Handler**: JSON-RPC 2.0 over stdio/HTTP/SSE for MCP client communication
- **Tool Registry**: Dynamic tool registration and invocation routing
- **Resource Manager**: URI-based resource access for docs and telemetry
- **Authentication**: API key validation for chatbot clients, encrypted Qumulo cluster credentials

#### RAG Engine
- **Document Ingestion**: Parse qumulo-docs.txt, extract sections, tag with version/category metadata
- **Chunking Strategy**: RecursiveCharacterTextSplitter (1024 tokens, 128 overlap) preserving semantic boundaries
- **Embedding Pipeline**: Batch embed chunks (100 per batch), store with metadata in vector DB
- **Query Processing**: Embed query → vector similarity search (k=5-10, cosine >0.7) → optional cross-encoder rerank → citation generation
- **Citation Mapping**: Link chunk metadata to docs.qumulo.com URLs with section headers and relevance scores

#### Telemetry Integration
- **API Client Pool**: Connection pooling (max 50/cluster) with auth token caching (1h TTL)
- **Metrics Aggregation**: Fetch capacity, performance, health, alerts from Qumulo REST APIs
- **Time-Series Cache**: Redis with 1-hour retention for fast lookback and pre-aggregated stats
- **Correlation Engine**: Match telemetry anomalies with documentation troubleshooting sections

#### API Analyzer
- **Endpoint Documentation**: Map API calls to documentation with parameters, schemas, examples
- **Version Compatibility**: Track API availability across Qumulo versions (introduced/deprecated/removed)
- **Code Generation**: Provide Python/JavaScript/cURL examples for API usage

### Infrastructure

#### Development Environment
- **Docker Compose**: MCP server + ChromaDB + Redis + Nexus chatbot for local development
- **Volume Persistence**: Separate volumes for vector DB data and Redis cache

#### Production Deployment (Kubernetes)
- **Horizontal Scaling**: 3-10 MCP server pods with HPA based on CPU/memory (70%/80% thresholds)
- **Vector DB**: Qdrant deployment with sharding and replication for read scaling
- **Cache Layer**: Redis cluster with persistence for distributed telemetry caching
- **Load Balancing**: nginx/HAProxy for request distribution across MCP instances
- **Health Checks**: Liveness probe on `/health`, readiness probe on `/ready`

#### Monitoring & Observability
- **Metrics**: Prometheus for query latency (p50/p95/p99), tool invocations, cache hit rates, error rates
- **Dashboards**: Grafana for real-time monitoring of system health and performance
- **Alerting**: Alert on error rate spikes, latency degradation, telemetry API failures

## Implementation Strategy

### Phase 1: MVP (4-6 weeks) - Core Documentation Search
**Goal**: Basic MCP server with semantic documentation search and citation generation

**Tasks**:
1. **MCP Server Scaffold**: Set up Node.js project with TypeScript, MCP SDK, and basic tool registry
2. **Document Ingestion Pipeline**: Parse qumulo-docs.txt, chunk with LangChain, embed with OpenAI, store in ChromaDB
3. **search_docs Tool**: Implement semantic search with vector similarity, citation generation, and docs.qumulo.com linking
4. **Nexus Chatbot Integration**: Connect chatbot client to MCP server, test end-to-end query flow

**Success Criteria**: Support engineer can search documentation via chatbot and receive cited answers in <2s

### Phase 2: Telemetry Integration (4-6 weeks)
**Goal**: Add real-time cluster monitoring and correlation with documentation

**Tasks**:
5. **Qumulo API Client**: Build connection pool, auth management, metrics fetching (capacity/performance/health/alerts)
6. **get_telemetry Tool**: Implement telemetry retrieval with Redis caching and formatting for chat display
7. **diagnose_issue Tool**: Correlate telemetry anomalies with troubleshooting documentation
8. **Multi-Cluster Support**: Handle concurrent connections to multiple Qumulo clusters

**Success Criteria**: Chatbot can fetch live cluster metrics and suggest relevant troubleshooting docs

### Phase 3: API Intelligence (2-3 weeks)
**Goal**: Provide API documentation and usage examples

**Tasks**:
9. **analyze_api_call Tool**: Map API endpoints to documentation, show parameters/schemas/examples
10. **Version Compatibility Tracking**: Document API availability across Qumulo versions

**Success Criteria**: Engineers can query API usage and get versioned documentation with code examples

### Risk Mitigation
- **Vector DB Scaling**: Load test ChromaDB early, have Qdrant migration plan ready if limits hit
- **OpenAI Dependency**: Cache embeddings aggressively, have local model (sentence-transformers) as backup
- **Qumulo API Rate Limits**: Implement exponential backoff, request batching, and cache-first strategy
- **Documentation Drift**: Automate daily re-indexing with change detection and version tracking

### Testing Approach
- **Unit Tests**: Tool handlers, RAG engine components, telemetry client with mocked APIs
- **Integration Tests**: End-to-end MCP protocol flow, vector search accuracy, citation correctness
- **Performance Tests**: Query latency under load (100+ concurrent sessions), cache hit rate validation
- **User Acceptance**: Beta test with 5-10 support engineers, gather feedback on answer quality and speed

## Task Breakdown Preview

High-level task categories (8 total):

1. **MCP Server Foundation**: Set up Node.js/TypeScript project, MCP SDK integration, tool/resource registry, health endpoints
2. **RAG Engine Implementation**: Document parsing, chunking pipeline, embedding generation, ChromaDB integration, citation mapping
3. **Documentation Search Tool**: Implement search_docs with vector similarity, filtering, reranking, citation generation
4. **Telemetry Integration**: Qumulo API client, connection pooling, metrics aggregation, Redis caching
5. **Correlation & Diagnostics**: diagnose_issue tool connecting telemetry anomalies with troubleshooting docs
6. **API Documentation Tool**: analyze_api_call with version tracking, schema documentation, code examples
7. **Chatbot Client Integration**: MCP client setup in Nexus chatbot, query routing, response rendering
8. **Production Deployment**: Kubernetes manifests, Docker images, CI/CD pipeline, monitoring setup

## Dependencies

### External Services
- **OpenAI API**: For embedding generation (fallback: local sentence-transformers)
- **Qumulo Cluster APIs**: For real-time telemetry (graceful degradation if unavailable)
- **Nexus Chatbot Platform**: For client integration and user interface

### Internal Prerequisites
- **qumulo-docs.txt**: Complete, up-to-date Qumulo documentation corpus
- **Qumulo API Credentials**: Encrypted cluster connection credentials with appropriate permissions
- **Infrastructure**: Kubernetes cluster or Docker environment for deployment

### Team Dependencies
- **Support Engineering**: For user stories, acceptance criteria, and beta testing feedback
- **Qumulo Product Team**: For API documentation accuracy and version compatibility data
- **DevOps**: For production deployment, monitoring setup, and CI/CD pipeline

## Success Criteria (Technical)

### Performance Benchmarks
- **Query Response Time**: p95 < 2s for documentation search (from query to formatted response)
- **Vector Search Latency**: p95 < 500ms for similarity search
- **Telemetry Fetch**: p95 < 300ms per cluster metric retrieval
- **Cache Hit Rate**: >80% for telemetry queries (reducing API load)
- **Concurrent Sessions**: Support 100+ simultaneous chat sessions without degradation

### Quality Gates
- **Answer Accuracy**: >90% of responses rated "helpful" by support engineers
- **Citation Correctness**: 100% of citations link to valid docs.qumulo.com pages
- **Telemetry Reliability**: <1% error rate for cluster API calls (with retry logic)
- **System Uptime**: 99.9% availability for MCP server

### Acceptance Criteria
- Support engineer can find relevant documentation in <2s with citations
- Live cluster health/alerts visible in chatbot with context-aware troubleshooting suggestions
- API usage questions answered with versioned docs and code examples
- System gracefully handles documentation updates (auto-reindex) and API unavailability

## Estimated Effort

### Overall Timeline
- **Phase 1 (MVP)**: 4-6 weeks
- **Phase 2 (Telemetry)**: 4-6 weeks
- **Phase 3 (API Intelligence)**: 2-3 weeks
- **Total**: 10-15 weeks (2.5-3.5 months)

### Resource Requirements
- **Backend Engineer**: 1 FTE (MCP server, RAG engine, API integration)
- **DevOps Engineer**: 0.5 FTE (infrastructure, deployment, monitoring)
- **Support Engineering Liaison**: 0.25 FTE (requirements, testing, feedback)

### Critical Path Items
1. **Document Ingestion Pipeline**: Blocks all search functionality (Week 1-2)
2. **MCP Tool Implementation**: search_docs must work before telemetry integration (Week 2-4)
3. **Chatbot Client Integration**: Blocks end-to-end testing and user feedback (Week 5-6)
4. **Production Deployment**: Blocks beta rollout to support engineers (Week 10-12)

### Risk Buffer
- Add 20% contingency (2-3 weeks) for:
  - Vector DB scaling issues requiring Qdrant migration
  - Qumulo API integration complexity (auth, rate limits, error handling)
  - User feedback-driven iterations and refinements

## Tasks Created
- [ ] #2 - MCP Server Foundation Setup (parallel: true)
- [ ] #3 - Document Ingestion & RAG Pipeline (parallel: false)
- [ ] #4 - Search Docs MCP Tool Implementation (parallel: false)
- [ ] #5 - Qumulo API Client & Telemetry Integration (parallel: true)
- [ ] #6 - Get Telemetry & Diagnose Issue MCP Tools (parallel: false)
- [ ] #7 - API Documentation Tool (parallel: true)
- [ ] #8 - Nexus Chatbot MCP Client Integration (parallel: false)
- [ ] #9 - Production Deployment & Monitoring (parallel: false)

Total tasks: 8
Parallel tasks: 3
Sequential tasks: 5
Estimated total effort: 138-170 hours (17-21 days for single developer)
