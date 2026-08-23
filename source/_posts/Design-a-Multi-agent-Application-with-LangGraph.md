---
title: Design a Multi-agent Application with LangGraph
date: 2026-08-20 00:18:58
tags: [system-design, Agent, AI]
categories: [Agent]
---

# System Design Document: Multi-Agent Personal Loan Approval System

**Author:** AI Systems Architecture Team  
**Date:** August 19, 2026  
**Frameworks & Technologies:** LangGraph, DeepAgents, Agent Skills, Model Context Protocol (MCP), FastAgent / FastAPI  

---

## Executive Summary

The **Finance Personal Loan Approval System** is an enterprise-grade multi-agent architecture built on top of the **LangGraph** framework and **DeepAgents** platform. The system streamlines personal loan applications by combining conversational AI with automated workflows: human-in-the-loop validation, credit scoring via Model Context Protocol (MCP) integrations, multi-modal document extraction (salary slips, real estate contracts), and automated loan underwriting.

---

## 1. High-Level System Architecture

The overall system architecture consists of a frontend Chat UI layer, an API Gateway/Agent Service layer, and specialized agentic sub-systems backed by skill registries and MCP tools.

{% mermaid %}
graph TD
    subgraph Frontend Layer
        UI[Chat UI / Web Portal]
        Upload[Document Upload Module]
    end

    subgraph API & Gateway Layer
        Gateway[FastAPI / Agent Service Endpoints]
        SessionMgr[Session & State Manager]
    end

    subgraph LangGraph Core Orchestration
        Router[Router Agent / Intent & Named Entity Extractor]
        StateStore[(LangGraph Checkpointer / Postgres)]
    end

    subgraph DeepAgents & Sub-Agent Ecosystem
        CreditAgent[Credit Report Sub-Agent]
        LoanAgent[Personal Loan Processing Sub-Agent]
        DocAgent[Document Extraction Skill Sub-Agent]
    end

    subgraph Human-in-the-Loop & Tools
        HITL[HITL Interruption Node / Review & Edit UI]
        MCP_Hub[MCP Server Hub / Tool Provider]
    end

    subgraph External Systems & Data Sources
        CreditBureau[Credit Bureau API / Experian]
        CoreBanking[Core Banking System]
        OCR[Multi-modal Vision LLM / Document Parser]
    end

    UI -->|1. Chat & Upload| Gateway
    Upload --> Gateway
    Gateway --> SessionMgr
    Gateway --> Router
    Router <--> StateStore
    
    Router -->|Intent: Credit Check| CreditAgent
    Router -->|Intent: Loan Application| LoanAgent
    
    CreditAgent -->|Fetch Financial Data| MCP_Hub
    LoanAgent -->|Parse Docs| DocAgent
    LoanAgent -->|Trigger Approval Workflow| HITL
    
    DocAgent --> OCR
    MCP_Hub --> CreditBureau
    HITL --> CoreBanking
{% endmermaid %}

---

## 2. Router Agent Design & Intent Classification

The **Router Agent** serves as the entry point and orchestrator in the LangGraph graph. It classifies user intent, performs Named Entity Recognition (NER) to extract required fields, and decides whether to route the conversation to a sub-agent or prompt the user for missing data.

### Workflow Sequence

{% mermaid %}
sequenceDiagram
    autonumber
    actor User
    participant Router as Router Agent
    participant GraphState as LangGraph State
    participant SubAgent as Specialized Sub-Agent
    
    User->>Router: Send Message / Upload File
    Router->>GraphState: Read Current State (Session Data)
    Router->>Router: Run Intent Classification & Schema Extraction
    alt Intent Needs Clarification
        Router-->>User: Ask for missing required fields
    else Intent Recognized: Credit Report
        Router->>SubAgent: Route to Credit Report Agent
    else Intent Recognized: Loan Processing
        Router->>SubAgent: Route to Loan Processing Agent
    end
{% endmermaid %}

### Extracted Field Schemas

| Intent Target | Mandatory Fields Extracted | Optional Fields Extracted |
| :--- | :--- | :--- |
| `Credit_Report_Gen` | Full Name, SSN / ID, Date of Birth | Preferred Bureau, Target Loan Amount |
| `Personal_Loan_Process` | Loan Amount, Loan Term, Annual Income, Employment Status | House Buying Contract (Doc), Salary Slip (Doc) |

---

## 3. Sub-Agent Detailed Workflows

### 3.1 Credit Report Generation Sub-Agent (MCP Tools)

This sub-agent connects to external credit scoring infrastructure using **Model Context Protocol (MCP)** tools to gather credit history, existing debt ratios, and risk profiles.

{% mermaid %}
graph LR
    subgraph Credit Agent Workflow
        Start([Start Credit Gen]) --> ValidateInfo[Validate Identity Schema]
        ValidateInfo --> MCPCall[Call MCP Tool: fetch_bureau_data]
        MCPCall --> CalcScore[Skill: Calculate Debt-to-Income DTI]
        CalcScore --> RenderReport[Skill: Generate Markdown Credit Summary]
        RenderReport --> End([Return Report to User])
    end
{% endmermaid %}

### 3.2 Personal Loan Processing Sub-Agent & HITL Workflow

The **Personal Loan Sub-Agent** coordinates document ingestion, income verification, and human-in-the-loop confirmation before submitting the loan application for final bank clearance.

{% mermaid %}
graph TD
    A[Start Loan Process] --> B{User Uploaded Docs?}
    B -- Yes --> C[Skill: Vision OCR & Doc Extraction]
    B -- No --> D[Prompt User for Documents/Inputs]
    D --> B
    C --> E[Aggregate Information: Income, Collateral, Loan Details]
    E --> F[Generate Pre-approval Summary]
    F --> G[LangGraph Interrupt: Human-in-the-Loop]
    
    subgraph HITL Confirmation Boundary
        G --> H{User Action}
        H -- Edit Details --> I[Update LangGraph State]
        I --> F
        H -- Confirm & Submit --> J[Submit to Core Banking API]
    end
    
    J --> K[Return Final Decision & Tracking ID]
{% endmermaid %}

---

## 4. DeepAgents, Skills & Tool Architecture

The system utilizes **DeepAgents** to manage agent lifecycle and dynamic skill loading. **Skills** are modular capabilities wrapped as standard tool schemas that sub-agents dynamically invoke based on execution context.

```
+-------------------------------------------------------------------+
|                        DEEPAGENTS RUNTIME                         |
|                                                                   |
|  +---------------------+        +------------------------------+  |
|  | Router Skill Set   |        | Loan Processing Skill Set    |  |
|  | - Intent Classifier |        | - Doc Extraction Skill (OCR) |  |
|  | - Entity Extractor  |        | - Risk Assessment Model      |  |
|  +---------------------+        | - Application Form Schema    |  |
|                                 +------------------------------+  |
|  +---------------------+                                          |
|  | Credit Skill Set    |        +------------------------------+  |
|  | - MCP Client Adapter|        | Human-in-the-Loop Skill      |  |
|  | - DTI Calculation   |        | - State Interruption Node    |  |
|  | - Score Normalizer  |        | - Edit Form Schema Builder   |  |
|  +---------------------+        +------------------------------+  |
+-------------------------------------------------------------------+
```

---

## 5. System State & LangGraph Schema Design

```python
from typing import List, Optional, Dict, Any, TypedDict
from langchain_core.messages import BaseMessage

class LoanApplicationState(TypedDict):
    # Base Conversation
    messages: List[BaseMessage]
    session_id: str
    current_agent: str
    
    # Extracted User Details
    user_id: Optional[str]
    full_name: Optional[str]
    ssn_last_four: Optional[str]
    annual_income: Optional[float]
    
    # Loan Application Specifics
    requested_loan_amount: Optional[float]
    loan_purpose: Optional[str]
    documents_uploaded: List[Dict[str, str]]  # Metadata: type, path, extracted_data
    
    # Financial Assessment (MCP Populated)
    credit_score: Optional[int]
    debt_to_income_ratio: Optional[float]
    existing_liabilities: Optional[float]
    
    # Human-in-the-Loop State Flags
    requires_user_confirmation: bool
    user_approved: Optional[bool]
    edited_fields: Optional[Dict[str, Any]]
```

---

## 6. Interview Questions for Multi-Agent System Design

When interviewing candidates or evaluating architecture teams for designing multi-agent LLM systems, the following categorized interview questions assess core competency:

### Category 1: Graph State & Framework Design (LangGraph Specific)
1. **How do you handle state persistence and concurrent user updates in a multi-agent graph like LangGraph?**  
   *Target answer:* Use LangGraph checkpointers (e.g., Postgres checkpointer) with session thread locking and schema-enforced state transitions.
2. **When should you choose a centralized router topology versus a decentralized peer-to-peer agent network?**  
   *Target answer:* Router topologies excel in structured enterprise domain workflows with clear user intents, whereas peer-to-peer fits open-ended research or debate workflows.

### Category 2: Human-in-the-Loop (HITL) & Interruption Patterns
3. **How do you implement non-blocking Human-in-the-Loop interrupts in LangGraph during an active API web stream?**  
   *Target answer:* Use `interrupt()` points in LangGraph to halt state execution, return a JSON payload to the UI, persist the state ID, and resume state via a dedicated approval endpoint (`Command(resume=...)`).
4. **How do you handle user corrections when a user edits AI-extracted metadata (e.g., salary extracted from an OCR document)?**  
   *Target answer:* Maintain both `raw_extracted_data` and `validated_data` in the state schema to allow tracking edits without re-triggering expensive extraction runs.

### Category 3: Model Context Protocol (MCP) & Skill Tool Design
5. **What advantages does Model Context Protocol (MCP) offer over standard function calling in enterprise loan architectures?**  
   *Target answer:* Decouples tool execution from agent definitions, standardizes auth/security contexts, and allows hot-swapping backend credit services without changing agent prompt logic.
6. **How do you prevent agent infinite loops when sub-agents invoke skills dynamically?**  
   *Target answer:* Implement max recursion limits in graph execution, enforce strict schema validation on tool outputs, and include exit condition fallback branches.

### Category 4: Security, Multi-modality & Robustness
7. **How do you securely handle sensitive PII (SSN, income data, contracts) uploaded via Chat UI?**  
   *Target answer:* Redact PII prior to sending text to external LLMs, use local vision models or secure enterprise API endpoints, and encrypt checkpointer database state at rest.
