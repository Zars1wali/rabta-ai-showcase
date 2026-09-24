# 🏛️ Rabta AI — Deep Architectural Specification

## 1. High-Level Agentic Design
Rabta AI is engineered around a **dual autonomous agent topology** coordinated by a central LangGraph state graph. Unlike conventional linear conversational bots, Rabta AI separates customer sales acquisition from managerial copilot operations into two distinct cognitive agents.

```text
[Incoming Message] ──> [Gatekeeper Node]
                             │
            ┌────────────────┴────────────────┐
            ▼                                 ▼
   [Customer ReAct Agent]           [Owner Intelligence Agent]
   (Gemini 3.5 Flash-Lite)          (Gemini 3.5 Flash-Lite)
            │                                 │
            ├─ search_catalog                 ├─ update_price
            ├─ get_product_photos             ├─ toggle_stock
            ├─ check_delivery                 ├─ set_margin_preference
            └─ escalate_to_owner ─────────────┘
                             │
                             ▼
                 [Unified Output Guardrail]
                             │
                             ▼
                    [WhatsApp Delivery]
```

---

## 2. LangGraph State Schema (`RabtaGraphState`)
Every conversational turn flows through a shared, persistent state model:

```python
class RabtaGraphState(TypedDict):
    # Routing & Identity
    sender_phone: str
    sender_jid: str
    push_name: Optional[str]
    tenant_id: str
    is_boss: bool
    
    # Message Context
    raw_message: str
    image_base64: Optional[str]
    conversation_history: List[Dict[str, str]]
    
    # Customer Funnel State
    customer_state: str  # BROWSING | COLLECTING_INFO | ESCALATED | IDLE
    customer_name: Optional[str]
    customer_city: Optional[str]
    customer_address: Optional[str]
    customer_sim_phone: Optional[str]
    customer_product: Optional[str]
    escalation_id: Optional[str]
    
    # Output Delivery
    reply_text: str
    reply_chunks: List[str]
    media_url: Optional[str]
    media_urls: Optional[List[Dict[str, str]]]
    owner_alert: Optional[str]
```

---

## 3. ReAct (Reasoning + Acting) Execution Loop
The conversational agent executes a continuous iterative tool loop:
1. **Model Turn 1:** Inspects message and conversation context $\rightarrow$ emits tool call `search_catalog(query="Wireless Earbuds", category="Audio")`.
2. **Deterministic Handler:** Executes SQL query against PostgreSQL catalog with hybrid vector reranking $\rightarrow$ returns structured JSON inventory payload.
3. **Model Turn 2:** Consumes tool output $\rightarrow$ detects that customer also asked for photos $\rightarrow$ emits second tool call `get_product_photos(product_names=["Pro ANC Earbuds Black", "Pro ANC Earbuds White"])`.
4. **Deterministic Handler:** Resolves verified static assets $\rightarrow$ attaches media objects.
5. **Model Turn 3:** Emits natural Roman Urdu reply formatted with clean headings, zero asterisks, and a proactive retail closer.

---

## 4. Multi-Layer Guardrail Pipeline
Before any generated response leaves the core engine, it passes through `output_guardrail`:
1. **URL Rescue:** Any raw image URLs inadvertently output in text bodies are extracted and promoted into verified native WhatsApp media payloads.
2. **Server IP Masking:** Any internal or staging IP addresses are rewritten to secure CDN endpoints.
3. **Zero-Asterisk Discipline:** All customer messages are parsed through `strip_customer_formatting_asterisks` to eliminate raw `*` signs from list quotes and headers.
4. **Anti-Flooding:** Photo attachments are capped at 4 images per turn to prevent bandwidth and UI saturation on mobile devices.
