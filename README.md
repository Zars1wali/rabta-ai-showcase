# 🌐 RABTA AI (رابطہ) — Multimodal Autonomous Dual-Agent Conversational Platform

[![Python 3.11+](https://img.shields.io/badge/Python-3.11%2B-blue.svg)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/Framework-FastAPI-009688.svg)](https://fastapi.tiangolo.com/)
[![LangGraph](https://img.shields.io/badge/Orchestration-LangGraph-FF6F00.svg)](https://langchain-ai.github.io/langgraph/)
[![Google Gemini](https://img.shields.io/badge/LLM-Gemini%203.5%20Flash--Lite-4285F4.svg)](https://ai.google.dev/)
[![Gemini Vision](https://img.shields.io/badge/Computer%20Vision-Gemini%20Multimodal%20Vision-34A853.svg)](https://ai.google.dev/)
[![Deepgram](https://img.shields.io/badge/Voice%20STT-Deepgram%20Nova--3-13EF93.svg)](https://deepgram.com/)
[![PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL%20%2B%20Vector-336791.svg)](https://www.postgresql.org/)
[![Meta WhatsApp](https://img.shields.io/badge/Messaging-Meta%20WhatsApp%20Cloud%20Platform-25D366.svg)](https://developers.facebook.com/docs/whatsapp/)

> **Case Study & Technical Showcase:** This repository serves as the public architectural portfolio, technical specification, and system showcase for **Rabta AI**. The core production backend engine, proprietary prompts, and enterprise databases remain confidential and proprietary.

---

## Executive Summary

**Rabta AI** (رابطہ — meaning *"Connection"* in Urdu) is an enterprise-grade, multimodal autonomous dual-agent conversational AI platform engineered for high-stakes, regulated retail and commerce. Designed and battle-tested in live production for **Haider Arms** (a premier licensed firearms dealership in Peshawar, Pakistan), Rabta AI autonomously navigates 24/7 customer sales inquiries, real-time inventory discovery, physical weapon verification via computer vision, advance payment intake funnels, and store-owner copilot operations.

---

## 🏗️ System Architecture & Dataflow

```mermaid
flowchart TD
    subgraph ClientChannels ["Omnichannel Ingestion & Messaging"]
        WA["Meta WhatsApp Business Platform / Web Client"]
        VN["WhatsApp Voice Notes (Audio Stream)"]
    end

    subgraph IngestionGateway ["Gateway & Real-Time Presence Layer"]
        GW["WhatsApp Gateway Bridge"]
        DB["15s Sliding Debounce & Presence Detector\n(composing / recording state tracking)"]
        STT["Deepgram Nova-3 API\n(Real-Time Urdu & English STT)"]
    end

    subgraph LangGraphCore ["Rabta AI Orchestration Core (LangGraph)"]
        GK["Node 1: Entry Gatekeeper & Route Dispatcher\n(Context-Aware Role Resolution)"]
        subgraph DualAgentCores ["Specialized Autonomous Cores"]
            CustomerAgent["Node 2A: Customer Sales ReAct Agent\n(Gemini 3.5 Flash-Lite @ temp=0.5)"]
            OwnerAgent["Node 2B: Owner Intelligence Copilot\n(Gemini 3.5 Flash-Lite @ temp=0.2)"]
        end
        OG["Node 3: Unified Output Guardrail\n(Zero-Asterisk Cleaner, URL Rescue & IP Masking)"]
    end

    subgraph AgenticTools ["Native ReAct Tool Registry & RAG Knowledge Base"]
        SearchTool["search_catalog\n(Hybrid Vector + Keyword Filter)"]
        PhotoTool["get_product_photos\n(Verified Media Asset Engine)"]
        PaymentTool["get_payment_bank_details"]
        DeliveryTool["check_delivery_policy & escalate_delivery_quote"]
        MarginTool["manage_payment_details & owner_preferences"]
        VisionEngine["Gemini Vision Engine\n(Physical Rollmark & SKU Verification)"]
    end

    subgraph PersistenceLayer ["Enterprise Persistence Layer"]
        PG[("PostgreSQL\n(Neon / Local Container)")]
        Vectors[("1536-dim Gemini Embeddings\n(postgres_jsonb / pgvector)")]
        AuditLogs[("PriceChangeLog & Conversation Checkpoints")]
    end

    WA --> GW
    VN --> GW
    GW --> DB
    DB --> STT
    STT --> GK
    DB --> GK

    GK -->|Customer Flow| CustomerAgent
    GK -->|Owner / Boss Flow| OwnerAgent

    CustomerAgent <--> SearchTool
    CustomerAgent <--> PhotoTool
    CustomerAgent <--> PaymentTool
    CustomerAgent <--> DeliveryTool

    OwnerAgent <--> SearchTool
    OwnerAgent <--> MarginTool
    OwnerAgent <--> VisionEngine

    SearchTool <--> PG
    SearchTool <--> Vectors
    MarginTool <--> AuditLogs

    CustomerAgent --> OG
    OwnerAgent --> OG
    OG --> GW
    GW --> WA
```

---

## 🧠 Core Engineering Pillars

### 1. LangGraph State Machine & Orchestration
- **Cyclic StateGraph Engine:** Engineered on **LangGraph** (`StateGraph`) with persistent checkpointing. Replaces fragile chain-of-custody pipelines with a deterministic 3-node graph:
  - **Node 1 (Gatekeeper):** Performs sender verification, role classification (Customer vs. Store Owner), and session state restoration.
  - **Node 2A & 2B (Dual Autonomous ReAct Agents):** Parallel, decoupled cognitive agents with isolated tools, memory states, and personas.
  - **Node 3 (Output Guardrail):** Enforces quality, formatting discipline, anti-flooding rules, and privacy protections before delivery.
- **Sliding 15-Second Debounce Window:** Integrates an asynchronous sliding timer with real-time WhatsApp presence updates (`composing` / `recording`). Eliminates fragmented, erratic responses by buffering multi-message customer thoughts into a unified reasoning context.

### 2. Native ReAct Agent Architecture (Reasoning + Acting)
- Powered by **Google Gemini 3.5 Flash-Lite** utilizing native ReAct tool/function calling loops:
  $$\text{Thought} \longrightarrow \text{Action (Tool Call)} \longrightarrow \text{Observation (SQL/API)} \longrightarrow \text{Final Response}$$
- Enables autonomous multi-step reasoning: an inquiry like *"Canik k konse models hain aur unki pics bhejo"* autonomously executes `search_catalog(query="Canik")`, evaluates available inventory, calls `get_product_photos(product_names=[...])`, and crafts a tailored response in a single execution graph turn.

### 3. Deterministic Tool & Function Calling Registry
- Uses strict **JSON Schema declarations** via the modern `google-genai` SDK:
  - `search_catalog`: Multi-attribute inventory discovery across category, caliber, manufacturer, origin, and price limits.
  - `get_product_photos`: Fetches verified studio assets by product SKU or batch model queries.
  - `check_delivery_policy`: City-specific logistics validation and terms enforcement.
  - `escalate_delivery_quote`: Triggers shipping calculation once customer Name, City, and Address are collected.
  - `get_payment_bank_details`: Dispatches verified dealership bank/EasyPaisa/JazzCash accounts.
  - `recommend_alternative`: Discovers budget-friendly or in-stock alternatives when requested models are out of stock.

### 4. Hybrid Retrieval-Augmented Generation (RAG)
- **1536-Dimensional Vector Embeddings:** Generated via `gemini-embedding-002` (Matryoshka-truncated from 3072 dims for maximum vector search performance).
- **Hybrid Retrieval Strategy:** Combines high-speed SQL token matching with cosine similarity vector scoring in PostgreSQL (`postgres_jsonb` / `pgvector`). Customers can describe firearms conceptually (e.g., *"Turkish compact concealed carry 9mm"*) and receive precise matches even when exact model names are omitted.

### 5. Multimodal Agentic AI & Gemini Vision
- **Physical SKU & Rollmark Verification:** Gun models frequently share nearly identical visual appearances. Gemini Vision cross-references uploaded catalog imagery against physical receiver rollmarks, calibers, and serials (e.g. distinguishing *Palmetto State Armory PA-15* from an *Anderson M4*, or *Glock 19X Standard* from an optics-ready *Glock 19X MOS*).
- **Multimodal Visual Search:** Customers can snap and send a photo of a firearm on WhatsApp to identify the model and immediately retrieve in-stock availability and confirmed pricing.

### 6. Voice Processing with Deepgram Nova-3
- **Bilingual Speech-to-Text Pipeline:** Deepgram Nova-3 API processes streaming audio bytes (OGG Opus) sent via WhatsApp voice notes.
- **Code-Switching Comprehension:** Natively handles fluid transitions between Urdu, English, and Roman Urdu speech, instantly transcribing spoken audio into structured text for downstream agentic reasoning.

### 7. Human-in-the-Loop (HITL) & Escalation Protocol
- **Intelligent Delegation Gating:** Routine inventory, specs, and shop location queries are resolved 100% autonomously.
- **Escalation Triggers:** Automatically routes to the store owner's private WhatsApp copilot only when human judgment is mandatory:
  - Explicit customer price negotiations (*"gunjaish"* / discount demands).
  - Custom shipping calculations once Name, City, and Street Address are verified.
  - High-value B2B dealer inquiries and bulk orders.
- **Bilateral Decision Relay:** The owner replies naturally to their copilot (*"dedo 400k mein"*), and Rabta AI automatically translates and relays the decision back to the customer.

### 8. Multi-Tier Guardrails & Safety Architecture
- **The Cardinal Grounding Law (Zero Hallucination):** The LLM possesses zero firearm specs or prices in its internal memory weights. All data quoted to customers must originate from verified tool output.
- **Zero-Asterisk Output Formatting:** Guarantees that customer-bound messages are devoid of raw markdown asterisks (`*` or `**`), outputting clean, natural headings and lists.
- **IP Masking & URL Rescue:** Intercepts internal server addresses (`65.20.90.130`) and converts them to public HTTPS CDN endpoints.
- **Anti-Flooding Controls:** Enforces a strict cap (max 4 images) per turn to prevent mobile WhatsApp session clutter.

### 9. Advanced Prompt Engineering Discipline
- **Hybrid Prompting Architecture:**
  - **Zero-Shot Task Constraints:** Strict negative prompting rules and JSON schema boundaries for NLU intent classification.
  - **Few-Shot Exemplars:** Contextual demonstrations of conversational Roman Urdu retail tone, sales closing questions, and good vs. bad message pairs.
- **Dynamic In-Context Injection (Part A / Part B):**
  - *Part A (Static System Prompt):* Permanent business identity, policies, and behavioral instructions.
  - *Part B (Live Runtime Context):* Injects confirmed daily prices, in-stock SKUs, owner margin directives, and customer interaction history on every single turn.

---

## 🤖 The Dual Agents Explained

| Dimension | Customer Sales Agent (`customer_react_node`) | Owner Copilot Agent (`owner_react_node`) |
| :--- | :--- | :--- |
| **Persona** | Warm, respectful, commercially aggressive salesperson | Discrete, exceptionally capable executive assistant |
| **LLM Configuration** | **Gemini 3.5 Flash-Lite** (`temp=0.5`) | **Gemini 3.5 Flash-Lite** (`temp=0.2`) |
| **Primary Goal** | Drive sales, answer specs, book orders, collect advance | Execute owner commands, adjust inventory, confirm rates |
| **Input Style** | Conversational Roman Urdu & English | Casual owner shorthand (*"490k"*, *"sold out"*, *"push this"*) |
| **Special Policies** | 100% advance payment required; COD strictly prohibited | Daily 9:00 AM automated price confirmation workflows |
| **Output Style** | Crisp 1–3 short sentences, clean text, zero asterisks | Instant confirmation, short bullet points, actionable alerts |

---

## 🛠️ Complete Technology Stack

| Layer | Component | Description |
| :--- | :--- | :--- |
| **Agentic Framework** | **LangGraph** (`0.2.x`) | Stateful multi-turn conversation graph, routing & checkpoints |
| **Cognitive Engine** | **Google Gemini 3.5 Flash-Lite** | Core reasoning, ReAct function calling & sales intelligence |
| **Vision Verification** | **Gemini Vision** | Physical rollmark, receiver stamping & SKU image verification |
| **Voice STT Engine** | **Deepgram Nova-3** | Real-time audio streaming speech-to-text (Urdu/English) |
| **Vector Embeddings** | **Gemini Embedding 2** (`1536-dim`) | Multimodal and text semantic embeddings for catalog RAG |
| **Backend Framework** | **FastAPI** (Python 3.11+, AsyncIO) | High-throughput asynchronous backend server |
| **Database & ORM** | **PostgreSQL** + **SQLAlchemy Asyncpg** | Multi-tenant persistent store, audit history, session state |
| **Messaging Channel** | **Meta WhatsApp Cloud Platform** | Official WhatsApp Business API & bidirectional gateway |
| **Reverse Proxy & SSL**| **Caddy** | Automated TLS/SSL termination, WebSocket & HTTP reverse proxy |
| **Containerization** | **Docker** & **Docker Compose** | Production container packaging and microservices |

---

## 📈 Production Metrics & Impact

- **Live In-Production Catalog:** 114+ distinct firearm SKUs, verified studio imagery, and daily dynamic price confirmations.
- **Automated Regression Suite:** 28 end-to-end integration and behavior scenarios tested and passing with zero regressions.
- **Latency Performance:** Sub-1.5s end-to-end ReAct tool resolution and response generation on live WhatsApp channels.
- **Bilingual Accuracy:** 100% resolution of Roman Urdu colloquial expressions and phonetic variations without hallucination.

---

## 📄 License & Intellectual Property

Copyright © 2026 **ZeroPointIntel / Rabta AI Engineering Team**. All rights reserved.  
This software, prompt engineering architecture, database schemas, and associated algorithms are **proprietary and confidential**. Unauthorized reproduction, reverse-engineering, commercial deployment, or redistribution is strictly prohibited.
