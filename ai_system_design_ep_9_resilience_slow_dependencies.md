# AI System Design (Episode 9): Containing Slow Dependencies & System Resilience

## 📌 Problem Statement & Scenario

> **Scenario:** Ek team ne apni AI Assistant application deploy ki hai. Application mein requests session storage, order lookup, retrieval, moderation, aur LLM se ho kar guzarti hain. Users se continuously complaints aa rahi hain ki application bohot slow chal rahi hai ya 504 Gateway Timeouts de rahi hai. Halanki, jab engineering team dashboards check karti hai, toh individual microservices ke average metrics healthy dikhte hain.

---

## 🎯 What You'll Learn & Core Questions

1. Dependency latency end-to-end user experience ko kaise affect karti hai?
2. Parallel execution kahan help karta hai aur required slow dependency tab bhi kyu matter karti hai?
3. Waiting requests capacity ko kaise consume karti hain (Little's Law math)?
4. Circuit Breakers unhealthy dependencies ko repeated calls se kaise rokte hain?
5. Bulkheads workloads ko isolate karke capacity exhaustion se kaise bachate hain?
6. Hedged requests latency reduce karne ke liye kya trade-offs karti hain?
7. Side-effects waale operations ko duplicate karne se pehle kya precautions chahiye?
8. Overall request deadline budgets ke andar fallback paths kaise design karein?

---

## 1. Case Study: The Slow AI Assistant Pipeline

AI Assistant application user request complete karne ke liye 5 steps se guzarti hai:

```
[ User Request ]
       │
       ▼
┌──────────────────┐
│ 1. Session Store │ (Read User State / Auth)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 2. Order Lookup  │ (Fetch Recent Order Details)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 3. Retrieval     │ (Vector DB RAG Context Search)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 4. Moderation    │ (Safety & Guardrails Check)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 5. LLM Engine    │ (Generate Final Response)
└──────────────────┘
```

### The Discrepancy:
* **Dashboards look healthy:** Individual microservices (LLM, Session, Vector DB) ka average latency $30\text{ms} - 80\text{ms}$ dikhta hai.
* **User Experience is Broken:** Users complaint kar rahe hain ki AI Assistant $10\text{s} - 15\text{s}$ le raha hai ya randomly **Timeout / 504 Gateway Error** de raha hai.

---

## 2. Sequential vs Parallel Execution & Tail Latency

### A. Sequential Pipeline Bottleneck
Agar saare calls sequential hain, toh total latency:

$$T_{\text{total}} = T_{\text{session}} + T_{\text{order}} + T_{\text{retrieval}} + T_{\text{moderation}} + T_{\text{llm}}$$

Agar **Order Lookup** slow ho kar $50\text{ms}$ se $5000\text{ms}$ (5 seconds) par chala gaya, toh baaki saare components fast hone ke bawajood poora response **5+ seconds** le lega.

### B. Parallel Execution Limits
Data dependencies missing hone par hum independent calls ko parallel kar sakte hain (e.g., `Order Lookup` aur `Retrieval` ko ek saath chalana):

```
                       ┌──► [ Order Lookup ] ──┐
                       │                       │
[ Session Store ] ─────┤                       ├─────► [ Moderation ] ──► [ LLM ]
                       │                       │
                       └──► [ Retrieval ] ─────┘
```

$$\text{Parallel Phase Latency} = \max(T_{\text{order}}, T_{\text{retrieval}})$$

* **The Catch:** Parallelism se average latency drop hoti hai, par **"Weakest Link Rule"** khatam nahi hota. Agar `Order Lookup` required dependency hai aur woh 5 seconds le rahi hai, toh $\max(5000\text{ms}, 100\text{ms}) = 5000\text{ms}$. **Required slow dependency poore pipeline ka floor latency set kar deti hai.**

---

## 3. Worker Exhaustion, Capacity & Little's Law

Jab dependency slow hoti hai, toh problem sirf uss specific request tak simit nahi rehti—woh poore server capacity ko destroy kar deti hai. Isse samajhne ke liye **Little's Law** use hota hai.

### Little's Law Equation:

$$L = \lambda \times W$$

Where:
* $L$ = Concurrent Requests / Queue Depth (System ke andar active in-flight requests)
* $\lambda$ = Throughput / Arrival Rate (Requests per second - RPS)
* $W$ = Average Latency / Execution Time per request

---

### The Capacity Collapse Scenario

Maan lo server ki capacity $L_{\text{max}} = 200 \text{ worker threads}$ hai, aur application $200 \text{ RPS}$ ($\lambda$) handle karti hai.

#### Normal State:
* $W = 100\text{ms} = 0.1\text{s}$
* $L = 200 \text{ RPS} \times 0.1\text{s} = \mathbf{20 \text{ Active Threads}}$ (System super healthy, plenty of headroom).

#### Degradation State (Order Lookup becomes slow):
* Order Lookup Service hanging hone se $W$ badh kar $5\text{s}$ ho jata hai.
* $L = 200 \text{ RPS} \times 5\text{s} = \mathbf{1000 \text{ Active Threads Needed!}}$

#### Result:
* Server ke paas sirf 200 threads hain, lekin demand 1000 threads ki hai.
* **Worker Exhaustion:** Saare 200 threads `Order Lookup` ke response ka wait karte-karte freeze ho gaye.
* **Cascading Failure:** Ab jo naye users simple `Session Check` ya `LLM Query` bhej rahe hain, unhe bhi server **503 Service Unavailable** dega kyunki ek bhi worker thread free nahi hai!

> **Why adding more replicas fails:** Naye pods/replicas add karne se background hanging service (Order DB) par load aur badhta hai. Resultingly, naye replicas ke threads bhi instantly freeze ho jaate hain.

---

## 4. Resilience Pattern 1: Circuit Breaker

Circuit Breaker ka kaam ek unhealthy downstream dependency ko call karna **band kar dena** hai taaki caller application ke worker threads freeze na hon.

### Circuit Breaker State Machine

```
              ┌──────────────────────────────────────────┐
              │                                          │
              │  Success Threshold Met                   │
              ▼  (e.g., 5 consecutive successes)         │
     ┌─────────────────┐                       ┌─────────────────┐
     │                 │  Failure Rate > 50%   │                 │
     │     CLOSED      ├──────────────────────►│      OPEN       │
     │ (Normal Operations)                     │ (Calls Blocked) │
     └────────┬────────┘                       └────────┬────────┘
              ▲                                         │
              │         ┌─────────────────┐             │
              │         │    HALF-OPEN    │             │ Timeout Elapsed
              └─────────┤ (Probe Requests)│◄────────────┘ (e.g., 30 Seconds)
                        └─────────────────┘
```

### States Breakdown:

1. **CLOSED (Normal State):**
   * Saari requests normal downstream service ko pass hoti hain.
   * System failure rate track karta hai (e.g., last 100 requests mein kitni fail/timeout hui).
2. **OPEN (Tripped State):**
   * Agar Failure Rate threshold exceed kar jaye (e.g., $>50\%$ requests fail/timeout), Circuit Breaker **OPEN** ho jata hai.
   * Downstream service ko network call bhejni **band** kar di jaati hai.
   * Instant Fast-Fail / Fallback return kiya jata hai (0ms delay).
3. **HALF-OPEN (Testing State):**
   * $30\text{s}$ cooldown window ke baad, Circuit Breaker thodi si test/probe requests downstream service ko bhejta hai.
   * Agar probe requests successful rehti hain $\rightarrow$ State wapas **CLOSED** ho jati hai.
   * Agar probe request fail hoti hai $\rightarrow$ State wapas **OPEN** ho jati hai.

---

## 5. Resilience Pattern 2: Bulkheads (Workload Isolation)

Shipbuilding mein, ek jahaaz ke andar water-tight compartments (**Bulkheads**) hote hain. Agar ek compartment mein paani bhar jaye, toh baki compartments jahaaz ko doobne se bachate hain.

System design mein **Bulkheads** har dependency ke liye dedicated Worker Pools ya Thread Limiters allocate karte hain.

```
                  [ Incoming User Traffic ]
                              │
                              ▼
                 ┌─────────────────────────┐
                 │    API Gateway Node     │
                 └────────────┬────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│  Session Pool    │ │    Order Pool    │ │    LLM Pool      │
│ (Max 20 Threads) │ │ (Max 30 Threads) │ │(Max 150 Threads) │
└────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
         │                    │                    │
         ▼                    ▼                    ▼
   [ Session DB ]      [ Order DB (Slow) ]   [ LLM Engine ]
```

### Advantage:
* Agar `Order DB` slow ho jati hai, toh woh max-to-max **Order Pool ke 30 threads** ko hi block kar sakti hai.
* Remaining **170 threads** (Session Pool + LLM Pool) completely unaffected rahenge.
* Application non-order features (e.g., General LLM Chat) ke liye 100% operational rahegi.

---

## 6. Resilience Pattern 3: Hedged Requests & Trade-offs

Tail Latency ($p_{99}$) ko eliminate karne ke liye **Hedged Requests (Speculative Execution)** pattern ka use hota hai.

```
                      [ Client / Gateway ]
                               │
            ┌──────────────────┴──────────────────┐
            │ (1st Request at t=0ms)              │ (2nd Hedged Request at t=50ms)
            ▼                                     ▼
   [ Service Instance A ]               [ Service Instance B ]
   (Stuck in Tail Lag 🐢)               (Responds Fast in 10ms ⚡)
            │                                     │
            └───────────────┬─────────────────────┘
                            │
                            ▼
                [ FIRST WINNER TAKEN! 🏆 ]
              (Slower request gets cancelled)
```

### Flow & Strategy:
1. Client **Instance A** ko primary request bhejta hai.
2. System wait karta hai (e.g., $p_{95}$ latency threshold = $50\text{ms}$).
3. Agar $50\text{ms}$ tak Instance A se response nahi aata, toh system parallel mein **Instance B** ko exact duplicate request bhej deta hai.
4. Jo instance pehle respond karta hai, uska output consumer ko bhej diya jata hai aur doosri request cancel kar di jati hai.

---

### ⚠️ Critical Caution: Operations with Side Effects
Hedged Requests sirf **Idempotent Operations** (e.g., `GET` vector search, `GET` user profile) par safe hain.

**Non-Idempotent Operations** (e.g., `POST /charge-credit`, `POST /execute-trade`) par speculative duplication kabhi nahi karni chahiye bina strict **Distributed Idempotency Keys (Redis Locking)** ke, varna double processing ka issue hoga.

---

## 7. Graceful Degradation, Fallbacks & Deadline Budgets

System ko robust banane ke liye pure pipeline ke paas **Deadline Propagation** aur **Fallback Strategies** honi chahiye.

### A. Context Deadline Propagation (Deadline Budgets)
Har request ke saath ek total End-to-End Budget (e.g., $1000\text{ms}$) associate kiya jata hai jo network headers (gRPC Metadata / HTTP Headers) ke through pass hota hai.

```
[ Total Request Budget: 1000ms ]

Step 1: Session Store    ──► Spent: 50ms   ──► Remaining Budget: 950ms
Step 2: Order Lookup     ──► Spent: 800ms  ──► Remaining Budget: 150ms
Step 3: Vector Search    ──► Requires 300ms minimum. 
                             Budget Left = 150ms < 300ms!
                             
[ ACTION: SKIP Vector Search instantly & Trigger Fallback! ]
```

### B. Fallback Strategies Matrix

| Component | Primary Action | Fallback Strategy on Failure / Timeout |
| :--- | :--- | :--- |
| **Order Lookup** | Live DB Query | Return cached recent order or empty list. |
| **Retrieval (RAG)** | Vector DB Hybrid Search | Fallback to Keyword Search or generic prompt. |
| **Moderation** | LLM Safety Guardrail | Fast Local Regex / Heuristic Check. |
| **LLM Engine** | Full Fine-Tuned Model | Smaller Distilled Model (e.g., 7B $\rightarrow$ 1B) or Cached Answers. |

---

## 8. Architectural Summary & Resilience Matrix

| Resilience Pattern | Primary Problem Solved | Core Mechanism | Trade-off / Cost |
| :--- | :--- | :--- | :--- |
| **Circuit Breaker** | Cascading failures to unhealthy dependencies. | Trips OPEN on high error rates to fast-fail calls. | Temporarily disables features for all users. |
| **Bulkheads** | Thread pool starvation across services. | Isolates capacity via dedicated thread/semaphore pools. | Can underutilize idle resources if pools are fixed. |
| **Hedged Requests** | High tail latency ($p_{99}$ spikes). | Sends duplicate speculative request after $p_{95}$ timer. | Increases duplicate network & compute traffic (2-5%). |
| **Deadline Propagation** | Wasted execution on expired requests. | Passes remaining time budget across service bounds. | Requires strict header/context passing standards. |
| **Graceful Degradation** | Complete user application outage. | Serves degraded/cached data on dependency failures. | Lower accuracy or less personalized user experience. |

---

## 💡 Golden System Design Takeaway

> **"Design for failure. Assume every network call will eventually hang or fail, and ensure your system degrades gracefully rather than collapsing entirely."**