# 🏛️ Rabta AI — Deep Architectural Specification & Engineering Blueprint

<div align="center">

[![Built With](https://img.shields.io/badge/Built%20With-Google%20Antigravity%20%2B%20Gemini%204%20(Argon)-4285F4?style=flat-square&logo=google&logoColor=white)](https://ai.google.dev/)
[![Runtime](https://img.shields.io/badge/Runtime-Gemini%203.5%20Flash--Lite%20%2B%20Gemini%203.8%20TTS-8E75B2?style=flat-square&logo=googlegemini&logoColor=white)](https://ai.google.dev/)
[![LangGraph](https://img.shields.io/badge/Orchestration-3--Node%20Guardrailed%20ReAct%20LangGraph-1C3C3C?style=flat-square&logo=langgraph&logoColor=white)](https://langchain-ai.github.io/langgraph/)
[![Pilot](https://img.shields.io/badge/Production%20Pilot-Haider%20Arms%20(246%2B%20SKUs)-brightgreen?style=flat-square)](https://github.com/Zars1wali/rabta-ai-showcase)

</div>

---

## 1. Architectural Philosophy: Why a 3-Node Guardrailed ReAct Graph?

In production conversational commerce on WhatsApp, customer messages are highly compound and code-switched (e.g., *"Salam bhai, Glock 19X Austria ka final rate kya hai, tasweer bhejo aur Peshawar se Lahore delivery ho jayegi?"*).

Industry research from **Cognition AI** (*"Don't Build Multi-Agents"*), **Anthropic** (*"Building Effective Agents"*), and **OpenAI** (*"A Practical Guide to Building Agents"*) demonstrates that decomposing a single user turn across 6–8 micro-nodes (separate NLU classifier node $\rightarrow$ sub-router node $\rightarrow$ price lookup node $\rightarrow$ writer node) causes **context fragmentation** and **cascading misclassifications**.

Rabta AI implements the industry-standard **3-Node Guardrailed Single-Context ReAct Topology**:

```text
[Incoming WhatsApp Turn (Text / Voice .ogg / Photo / PDF)]
                          │
                          ▼
        ┌───────────────────────────────────┐
        │   Node 1: Gatekeeper Router       │
        │   (route_message)                 │
        │   • Tenant & Role Resolution      │
        │   • State Hydration               │
        └─────────────────┬─────────────────┘
                          │
         ┌────────────────┴────────────────┐
         │ is_boss == False                │ is_boss == True
         ▼                                 ▼
┌──────────────────────────────┐  ┌──────────────────────────────┐
│ Node 2A: Customer Sales Core │  │ Node 2B: Owner Copilot Core  │
│ (customer_react_node)        │  │ (owner_react_node)           │
│ • Gemini 3.5 Flash-Lite      │  │ • Gemini 3.5 Flash-Lite      │
│ • TPU CachedContent Prefix   │  │ • PDF/Image Vision Parser    │
│ • Full ReAct Tool Loop       │  │ • Bulk Price & Stock Tools   │
│ • Margin-Aware Negotiation   │  │ • HITL Approval Relay        │
└──────────────┬───────────────┘  └──────────────┬───────────────┘
               └────────────────┬────────────────┘
                                ▼
        ┌───────────────────────────────────┐
        │   Node 3: Unified Output Guardrail│
        │   (output_guardrail)              │
        │   • Raw URL Rescue to Media Array │
        │   • Server IP Masking to HTTPS    │
        │   • Zero-Asterisk Sanitization    │
        │   • Anti-Flooding (Max 4 Photos)  │
        │   • Transient Blob Pruning        │
        └─────────────────┬─────────────────┘
                          │
                          ▼
     [AsyncPostgresSaver Checkpoint + WhatsApp Delivery]
```

---

## 2. LangGraph State Contract & Checkpoint Hygiene (`RabtaGraphState`)

Every turn executes within a typed state dictionary persisted per thread (`{tenant_id}:{sender_phone}`) via `AsyncPostgresSaver`:

```python
class RabtaGraphState(TypedDict, total=False):
    # 1. Routing & Multi-Tenant Identity
    tenant_id: str
    sender_phone: str
    owner_phone: str
    business_phone: str
    business_name: str
    industry: str
    is_boss: bool

    # 2. Multimodal Turn Inputs (Transient per turn)
    raw_message: str
    image_base64: Optional[str]        # Pruned in output_guardrail before DB commit
    document_base64: Optional[str]     # Pruned in output_guardrail before DB commit
    document_mime: Optional[str]
    document_name: Optional[str]
    catalog_context: str               # Pruned in output_guardrail before DB commit
    is_voice_note: bool

    # 3. Persistent Customer Funnel & Negotiation Memory
    customer_state: str                # BROWSING | COLLECTING_INFO | ESCALATED | IDLE
    customer_name: Optional[str]
    customer_city: Optional[str]
    customer_address: Optional[str]
    customer_sim_phone: Optional[str]
    customer_product: Optional[str]
    negotiation_round: int
    quoted_prices: Dict[str, float]
    escalation_id: Optional[str]

    # 4. Guardrailed Output Payload
    reply_text: str
    reply_chunks: List[str]
    media_urls: Optional[List[Dict[str, str]]]
    send_as_voice: bool
    voice_gender: Optional[str]
    owner_alert: Optional[str]
```

### Checkpoint Storage Optimization (Transient Payload Pruning)
To prevent multi-megabyte base64 images, supplier PDFs, and raw catalog dumps from bloating PostgreSQL `checkpoint_blobs` across multi-turn conversations on a cloud VM, `output_guardrail` explicitly resets:
```python
"catalog_context": "",
"image_base64": None,
"document_base64": None,
```
immediately before the terminal `END` edge commits the state to `AsyncPostgresSaver`.

---

## 3. Google TPU Context Caching (`CachedContent`)

Implemented in `backend/app/services/cache_manager.py`:
1. **Compilation**: Combines the static Master Sales Intelligence System Prompt with the tenant's verified active catalog (246+ SKUs for Haider Arms, including caliber, mechanism, condition, stock status, and confirmed PKR pricing).
2. **TPU Pinning**: Uploads the compiled prefix to Google's `client.aio.caches.create()` API using `gemini-2.5-flash-lite` with a 2-hour TTL (`7200s`).
3. **Zero-Stale Invalidation**: Any call to `update_item_price`, `bulk_update_catalog_prices`, or stock mutation immediately calls `invalidate_tenant_cache(tenant_id)`, ensuring customers never see outdated pricing.
4. **Fallback Resilience**: If a tenant's catalog is below Google's minimum caching token threshold (32,768 tokens) or if the cache is warming up, the engine seamlessly falls back to standard prompt injection with zero downtime.

---

## 4. Multimodal RAG (MM-RAG) & Deterministic Tool Registry

Rabta AI combines **Semantic Vector Search**, **Cross-Modal Image Vectors**, and **Exact SKU Lexical Boosting**:

| Tool Name | Target Agent | Execution Mechanism |
| :--- | :--- | :--- |
| `search_catalog` | Customer & Owner | Queries PostgreSQL using 768-dim `gemini-embedding-001` / `gemini-embedding-2` cosine similarity + exact caliber/brand token boosting + synonym expansion. |
| `get_product_photos` | Customer | Resolves up to 4 studio photos and companion showcase videos per SKU with strict model-number boundary checks (preventing e.g. `Glock 19` vs `Glock 19X` cross-contamination). |
| `check_delivery_policy` | Customer | Enforces city-specific logistics terms and 100% advance payment rules before booking. |
| `get_payment_bank_details` | Customer | Dispatches verified merchant bank/JazzCash/EasyPaisa details once order intent is confirmed. |
| `escalate_to_owner` | Customer | Creates a persistent `Escalation` record and dispatches an actionable WhatsApp alert to the owner's copilot. |
| `update_item_price` | Owner Copilot | Updates a single SKU's price, records a `PriceChangeLog` audit entry, and invalidates the TPU cache. |
| `bulk_update_catalog_prices` | Owner Copilot | Parses supplier PDF/image price sheets and executes atomic batch updates across dozens of SKUs in one turn. |

---

## 5. Voice & Dialect Pipeline (STT + Progressive TTS)

1. **Inbound Voice Note (`.ogg` Opus)**:
   - Gateway captures audio buffer and forwards to the backend.
   - Gemini Native Multimodal Audio (with Deepgram Nova-3 support) transcribes **Urdu, Roman Urdu, Pashto, Punjabi, and English** speech, preserving domain-specific terms (*"bore"*, *"mag"*, *"zigana"*, *"beretta"*, *"gunjaish"*).
2. **Outbound Progressive Voice Synthesis**:
   - Triggered when a customer sends a voice note or explicitly requests a voice reply (`[RABTA_VOICE_NOTE: MALE/FEMALE]`).
   - Applies a specialized **Pakistani Roman Urdu Phonetic Lexicon** (`voice_service.py`) so model numbers, calibers (`9mm`, `.30 bore`, `12 gauge`), and Urdu courtesies (*"Assalam o Alaikum"*, *"Jee zaroor"*, *"InshaAllah"*) are pronounced with authentic cadence.
   - Streams synthesized `.ogg` audio chunks to WhatsApp while broadcasting live `"recording audio..."` presence.

---

## 6. LLM-as-a-Judge Quality Flywheel & CI/CD Gatekeeper

Located in `backend/eval/evaluate_rabta.py`:
- Executes **124+ multi-turn test scenarios** covering catalog lookup, out-of-stock alternatives, adversarial prompt injection, price negotiation boundaries, voice-note switching, and owner bulk PDF updates.
- Grades every response using a deterministic assertion layer plus an **LLM-as-a-Judge** rubric evaluating:
  1. **Factual Grounding (Zero Hallucination)**
  2. **Tool Invocation Accuracy**
  3. **WhatsApp Formatting Compliance (Zero Asterisks, Clean Chunks)**
  4. **Commercial Persona & Closing Effectiveness**
