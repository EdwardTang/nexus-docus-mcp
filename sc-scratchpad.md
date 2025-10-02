/sc:brainstorm "

====================================================================== 
## 📊 Rule of thumb for thinking: Three-Lens Analysis 

### 🎯 Graham Lens (Product-Market Fit): **CAUTION**

**Key Question**: *Does this create user delight or user confusion?*

**Research Evidence:**
{You need do gather evidence in the code and authentic research to prove it}

**Assessment**: {Speak out your assessment step by step}.

**Recommendation**: {Your verdicts}

### 🚀 Hassabis Lens (AI Evolution): **STRATEGIC OPPORTUNITY**

**Key Question**: *Does this align with AI capability development direction?*

**Research Evidence:**
{You need do gather evidence in the code and authentic research to prove it}

**Assessment**: {Speak out your assessment step by step}.

**Recommendation**: {Your verdicts}


### ⚡ Musk Lens (Execution Minimalism): **STRONG CAUTION**

**Key Question**: *What's the minimum viable mechanism to achieve 80% of the value?*

**Research Evidence:**
{You need do gather evidence in the code and authentic research to prove it}

**Assessment**: {Speak out your assessment step by step}.

**Recommendation**: {Your verdicts}
=====================================================================

Product Requirements Document (PRD) for the Nexus Docs MCP tool

@agent-requirements-analyst
@agent-system-architect
@agent-technical-writer

**Objective:** Brainstorm and generate a comprehensive Product Requirements Document (PRD) for a new tool called the "Nexus Docs MCP tool".

**Background:**
We are building a tool to assist Qumulo support engineers. This tool will be integrated into a chatbot on the Nexus platform and will be powered by an MCP server. The primary goal is to help support engineers with customer support scenarios, especially those involving configuration changes to a running Qumulo system.

**Core Requirements:**
The tool must be able to:
1.  Understand natural language queries from support engineers.
2.  Identify the correct API calls needed to perform configuration changes on a Qumulo system.
3.  Provide detailed information about the behavior of these API calls.
4.  Highlight any unexpected results or relevant edge cases.
5.  Generate responses that are grounded in the official Qumulo documentation, and near-realtime Qumulo System Telemetry, providing citations for all information.
    5.1 As to the citations, we want them to linked to the actual production doc sites.

**Knowledge Base:**
The tool's knowledge will be based on the attached Qumulo documentation (`qumulo-docs.txt`). This document contains the flattened codebase of Qumulo's documentation and should be treated as the single source of truth.

**PRD Structure:**
Please structure the output as a formal PRD with the following sections:

1.  **Introduction & Vision:** A high-level overview of the product and its goals.
2.  **Problem Statement:** What problem are we solving for the Qumulo support engineers?
3.  **User Personas:**
    * Primary Persona: Qumulo Support Engineer.
    * Describe their goals, frustrations, and how this tool will help them.
    * I want to help the end user with a permissions issue -- the end user cannot access a file that they think they should. 
    * I want to guide the user through networking configuration -- what settings need to be changed given the user's network.
    * I want to help set up shares, exports, replication relationships, snapshots, etc.
4.  **Features & Functionality:**
    * **Natural Language Query Processing:** How the tool will understand user requests.
    * **API Call Identification:** The process of mapping a user's request to specific Qumulo API calls from the documentation.
    * **Behavioral & Edge Case Analysis:** How the tool will extract and present information about API behavior and edge cases.
    * **RAG (Retrieval-Augmented Generation) & Citations:** Detail the mechanism for retrieving information from the provided `qumulo-docs.txt` and citing the sources in the response. As to the citations, we want them to linked to the actual production doc sites: docs.qumulo.com 
5.  **Technical Requirements:**
    * **MCP Server Architecture:** High-level architecture of the MCP server.
    * **Integration with Nexus Chatbot:** How the MCP tool will interface with the chatbot.
    * **API & Documentation Parsing:** How the system will parse and index the `qumulo-docs.txt` to make it searchable.
6.  **Success Metrics:** How will we measure the success of this tool?
7.  **Future Considerations:** Potential future enhancements and features.

Please use your expertise as a requirements analyst, system architect, and technical writer to fill out these sections with as much detail as possible, drawing insights from the provided context.
"
--focus "user personas, features, technical requirements, API integration, RAG system with citations" --ultrathink --tavily --context7 --deepwiki
