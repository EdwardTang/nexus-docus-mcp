# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Nexus Docs MCP** is an MCP (Model Context Protocol) server designed to assist Qumulo support engineers with customer support scenarios. The tool integrates with the Nexus platform chatbot and helps engineers:

- Understand configuration changes to running Qumulo systems
- Identify correct API calls for specific tasks
- Access detailed API behavior documentation
- Handle edge cases and unexpected results
- Generate responses grounded in official Qumulo documentation with citations to docs.qumulo.com

## Core Architecture

### Knowledge Base Strategy
- **Single Source of Truth**: Qumulo documentation (`qumulo-docs.txt`) contains flattened codebase documentation
- **RAG System**: Retrieval-Augmented Generation with citation linking to production docs (docs.qumulo.com)
- **Real-time Integration**: Near-realtime Qumulo System Telemetry integration

### MCP Server Design
- **Natural Language Processing**: Parse support engineer queries
- **API Mapping**: Match user requests to specific Qumulo API calls
- **Documentation Parsing**: Index and make `qumulo-docs.txt` searchable
- **Citation System**: Provide verifiable links to official documentation

### Primary Use Cases
1. **Permissions Troubleshooting**: Help users resolve file access issues
2. **Network Configuration**: Guide through networking settings
3. **Resource Setup**: Assist with shares, exports, replication, snapshots

## Key Personas

**Primary User**: Qumulo Support Engineer
- **Goals**: Quickly resolve customer issues with accurate API guidance
- **Pain Points**: Finding correct API calls, understanding edge cases, providing documented solutions
- **Value Proposition**: Instant access to API documentation with behavior analysis and citations

## Technical Requirements

### MCP Server Architecture
- Natural language query processing
- API call identification engine
- Behavioral & edge case analysis
- RAG-based documentation retrieval
- Citation generation linking to docs.qumulo.com

### Integration Points
- Nexus chatbot interface
- Qumulo API documentation parser
- Telemetry data streams

## Success Metrics
- Query resolution accuracy
- Time to correct API identification
- Citation accuracy and relevance
- Support engineer satisfaction
- Reduction in documentation lookup time

## Development Notes

### Brainstorming Framework
The project uses a three-lens analysis approach for decision-making:

1. **Graham Lens (Product-Market Fit)**: Does this create user delight or confusion?
2. **Hassabis Lens (AI Evolution)**: Does this align with AI capability development?
3. **Musk Lens (Execution Minimalism)**: What's the minimum viable mechanism for 80% value?

Each lens requires research evidence, step-by-step assessment, and clear recommendations.

### Documentation Standards
- All features must be grounded in official Qumulo documentation
- Citations must link to production docs (docs.qumulo.com)
- Edge cases and unexpected behaviors must be explicitly documented

## Future Considerations
- Enhanced telemetry integration
- Multi-system configuration support
- Advanced troubleshooting workflows
- Interactive API exploration tools
