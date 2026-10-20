# Incident Post-Mortem & System Design: Chatbot Stale Knowledge Problem

> **Scenario:** Chatbot customer ko batata hai ki "Return policy 60 days ki hai", jabki 1 week pehle business team ne policy update karke **30 days** kar di thi.

---

## 1. Problem & Root Cause Analysis

Yeh **Stale Knowledge & Cache Invalidation** ki classic system design problem hai.

| Possible Point of Failure | Technical Reason |
| :--- | :--- |
| **Outdated Vector DB (RAG)** | Knowledge base ke chunks re-index nahi hue. Vector database mein purana embedding `[return_policy_v1]` hi retrievable tha. |
| **Hardcoded System Prompt** | Policy numbers system prompt ya hardcoded context mein direct likhe the, jise developer ne update nahi kiya. |
| **Model Fine-Tuning** | Informational facts fine-tuning weights mein embed the. Without retraining, model purana weight use kar raha tha. |
| **Response Cache** | Redis ya API Response Cache expire/flush nahi hua tha (Stale Cache Hit). |
| **Missing Pipeline Sync** | CMS (where policy is edited) aur Chatbot Knowledge Base ke beech **Automated Sync** missing tha. |

---

## 2. Immediate Action Plan

### A. Business & Operational Handling (Customer First)
* **Honor the Chatbot Statement:** Company ki taraf se galti hui hai, isliye affected customer ko **60 days return allow** karna chahiye to protect brand trust.
* **Audit Logs Scan:** Chatbot conversational logs run karein aur pichle 1 week ke un sabhi users ko filter karein jinhe chatbot ne wrong policy inform ki.

### B. Immediate Technical Hotfix
* **Update Vector Store / Prompt:** Purana 60-day policy chunk delete karke new 30-day document embed/index karein.
* **Flush Cache:** Redis / Response Caches ko immediately flush/invalidate karein.

---

## 3. Permanent System Design Fixes

Aise incidents ko permanently prevent karne ke do primary engineering patterns hain:

```
[ Central CMS / Admin Portal ]
                (Policy Updated: 30 Days)
                           │
        ┌──────────────────┴──────────────────┐
        ▼                                     ▼
[ Approach 1: Event Sync ]           [ Approach 2: Live API ]
│                                     │
(Webhook / Event Bus)                 (LLM Function Call)
│                                     │
▼                                     ▼
[ Auto-Reindex Vector DB ]           [ Direct Real-Time DB Query ]
```


### Approach 1: Live Function Calling (Recommended for Dynamic Rules)
Dynamic business logic (Pricing, Stock, Return Limits) ko static text files ya embeddings mein rakhne ki bajaye Chatbot ko **Function Calling / Tools** provide karein.

* **Flow:** 
  1. User asks: *"Return policy kya hai?"*
  2. LLM detects intent $\rightarrow$ Invokes Tool: `get_policy_details(category="returns")`
  3. API live database se current active policy (30 Days) query karke LLM ko deti hai.
  4. LLM response format karke user ko bhejta hai.

### Approach 2: Event-Driven Vector DB Sync Pipeline
Agar document-based RAG use kar rahe hain, toh human manual updates par depend mat rahiye.

* **Automated Webhooks:** CMS mein document save/publish hone par automatically Webhook trigger ho.
* **Pipeline:** Webhook $\rightarrow$ Message Queue (RabbitMQ/Kafka) $\rightarrow$ Ingestion Service $\rightarrow$ Old Chunk Delete $\rightarrow$ New Chunk Embed & Upsert to Vector DB.

### Approach 3: Post-Generation Guardrails & Fact Verification
* Response send karne se pehle ek deterministic check/guardrail layer ho jo numbers, policies, aur critical facts ko live database rules ke against validate kare.

---

## 4. Key Engineering Takeaways

1. **Don't store highly volatile logic in static prompts/embeddings.** Use Function Calling for volatile business rules.
2. **Single Source of Truth (SSOT):** Business policies hamesha ek hi central DB/CMS mein honi chahiye, jahan se chatbot runtime par query kare.
3. **Automate Invalidation:** Cache ya Vector DB ka Invalidation manual process nahi hona chahiye; event-driven webhooks use karein.
