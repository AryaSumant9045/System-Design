# AI System Design: Timeouts, Retry Budgets & Idempotency

> **Overview:** Distributed systems (Microservices, AI Agents, REST APIs) mein network requests unreliable hoti hain. Transient failures ko gracefully handle karne ke 3 fundamental pillars hain: **Timeouts**, **Retries (with Retry Budgets)**, aur **Idempotency**.

---

## 1. Timeouts (The Circuit Safeguard)

### Problem Without Timeouts: Thread Starvation
Jab Service A Service B ko call karti hai aur Service B hang/freeze ho jati hai, agar Service A ke paas timeout set na ho:
* Service A ka worker thread infinitely wait karega.
* Naye incoming requests ke sath Service A ke saare threads block ho jayenge.
* **Result:** Cascading Failure & Total System Outage (Thread Pool Starvation).

```

[ User Request ] ──► [ Service A ] ──(No Timeout)──► [ Service B (Hangs 🐢) ]
│
▼
(Thread Blocked Infinitely) ──► Server A Out of Memory / Crash!
```
---

### Types of Timeouts

1. **Connect Timeout:** Client aur Server ke beech TCP Handshake ($SYN \rightarrow SYN-ACK \rightarrow ACK$) establish hone ka maximum waiting time (e.g., $500\text{ms}$).
2. **Read / Socket Timeout:** Connection establish hone ke baad server se data packets receive hone ka maximum waiting time (e.g., $2\text{s}$).

---

### Sahi Timeout Value Kaise Calculate Karein?

* **Too Short Timeout:** False failures trigger honge (Healthy slow requests interrupt ho jayengi).
* **Too Long Timeout:** System threads waste honge aur request queues fill honge.

#### Production Rule:
Timeout hamesha downstream service ke **p99 ya p99.9 Latency** ke basis par safety buffer add karke set kiya jata hai:

$$\text{Timeout} = \text{Downstream Service p99 Latency} + \text{Network Jitter Buffer}$$

* *Example:* Agar Service B ka p99 latency $150\text{ms}$ hai, toh Service A ka timeout $\approx 300\text{ms} - 500\text{ms}$ set hoga.

---

## 2. Retries & Retry Budgets (The Traffic Regulator)

Jab koi request timeout ya network glitch ke karan fail ho jati hai, toh system usse **Retry** karta hai. Par naive retries production ko tabaah kar sakti hain!

---

### The Danger: Naive Retries $\rightarrow$ Retry Storms

Maan lo **Service B** overload hone ki wajah se slow ho gayi hai:
1. 1,000 Clients request bhejte hain.
2. Requests timeout ho kar fail hoti hain.
3. Har client 3-5 baar naive retry karta hai.
4. Service B par load $1,000 \rightarrow 3,000+$ requests ho jata hai!

```

[ Overloaded Service B ] ◄── (1,000 Initial Requests)
│ (Starts Failing)
▼
[ Retry Storm Triggered! ] ◄── (3,000 Retry Requests Bombardment) ──► Total Crash 💥
```

Isse **Retry Storm / Cascading Outage** kehte hain. Slowness ki vajah se system aur zyada load under dab kar crash ho jata hai.


---

### Solution A: Exponential Backoff with Jitter

Retries ko instantly bhejney ke bajaye interval badha kar aur randomness add karke bheja jata hai.

#### 1. Exponential Backoff
Har failure ke baad wait time exponentially badhaya jata hai:

$$\text{Wait Time} = \text{Base Delay} \times 2^{\text{Attempt Number}}$$

* Attempt 1: Wait $1\text{s}$
* Attempt 2: Wait $2\text{s}$
* Attempt 3: Wait $4\text{s}$
* Attempt 4: Wait $8\text{s}$

#### 2. Adding Jitter (Randomness)
Agar 10,000 clients ek hi time par fail hue, toh Exponential Backoff ke sath bhi wo saare exact $2\text{s}$ baad ek sath retry karenge (**Thundering Herd Problem**).

Random Jitter add karne se retries time mein spread-out ho jati hain:
```

$$\text{Actual Wait Time} = \text{Random}(0, \text{Exponential Backoff Wait Time})$$

```
WITHOUT JITTER : All 1000 Clients retry at exact t = 2.0s ──► Spike! 📈
WITH JITTER    : Clients retry randomly between t = 0.5s to 2.5s ──► Smooth Load 📉

---

### Solution B: Retry Budgets

Exponential Backoff ke baad bhi agar downstream system completely down hai, toh retries continue karna waste hai. Iske liye **Retry Budget** lagaya jata hai.

* **Mechanism:** System track karta hai ki total traffic ka kitna percentage retries hain.
* **Rule:** Client/Gateway allow karta hai ki total outgoing requests mein **max 10% traffic hi retries ho sakta hai**.
* **Token Bucket Algorithm:**
  * Har successful request 1 Retry Token add karti hai.
  * Har retry request 1 Retry Token consume karti hai.
  * Agar retry budget (10%) exhaust ho jaye, toh client retries stop kar deta hai aur **Fail-Fast** mode mein chala jata hai.

---

## 3. Idempotency (The Safety Shield)

Retries ka sabse bada khatra: **"Double Execution / Double Charging"**.

### The Timeout Dilemma

Jab Client ko Timeout error milta hai, toh Client ko nahi pata hota ki failure kahan hua:

```
Scenario A (Failed Before Processing):
Client ──► (Network Drops Request) ──X Server (Server never saw request)

Scenario B (Failed After Processing):
Client ◄──X (Network Drops Response) ── Server (Server ALREADY executed payment!)

```
Dono scenarios mein Client ko **Timeout Error** hi milta hai. Agar Client Scenario B mein retry kar de, toh customer **do baar charge** ho jayega!

---

### What is Idempotency?

An operation is **Idempotent** if executing it multiple times produces the **exact same result** as executing it once:

$$f(f(x)) = f(x)$$

* **Non-Idempotent:** `POST /api/v1/charge-wallet` (Executes payment every time).
* **Idempotent:** `PUT /api/v1/users/101` (Updates user profile to same data $N$ times).

---

### How to Implement Idempotency (The Idempotency Key Pattern)

Payment systems (Stripe, Razorpay, PayPal) aur state-changing APIs `X-Idempotency-Key` header use karti hain.


```
[ 1. Client ]
│
│ Generates Unique UUID: Idempotency-Key = "key_abc_123"
│ Header: X-Idempotency-Key: key_abc_123
▼
[ 2. API Gateway / Server ]
│
├─► Query Redis Cache / DB: Check if "key_abc_123" exists?
│
├──► CASE 1: KEY EXISTS in Cache (Duplicate Retry Request)
│    └─► Return Cached Previous Response Instantly! (Do NOT re-execute)
│
└──► CASE 2: KEY DOES NOT EXIST (First Time Request)
├─► Acquire Distributed Lock on "key_abc_123" (Redis Redlock)
├─► Execute Core Business Logic (DB Write / Charge Card)
├─► Save Result in Redis with TTL (e.g., 24 Hours)
└─► Return Response to Client

```
---

### HTTP Methods & Idempotency Matrix

| HTTP Method | Natural Idempotency | Reason | Handling Strategy |
| :--- | :---: | :--- | :--- |
| **GET** | ✅ Yes | Only reads data, no state change. | Safe to retry unconditionally. |
| **PUT** | ✅ Yes | Overwrites complete resource state. | Safe to retry. |
| **DELETE** | ✅ Yes | Deleting an already deleted resource changes nothing. | Safe to retry. |
| **POST** | ❌ No | Creates new resource / appends state. | **MUST use Idempotency Keys!** |

---

## 4. Summary Architecture Cheat Sheet

```markdown
1. Timeouts:
   - Always set Connect and Read Timeouts on all HTTP/gRPC clients.
   - Base Timeout on Downstream p99 Latency + Safety Buffer.

2. Retries & Retry Budgets:
   - Never use instant or infinite retries.
   - Use Exponential Backoff + Jitter to prevent Thundering Herd Spikes.
   - Enforce a Retry Budget (Max 10% of total traffic) to avoid Retry Storms.

3. Idempotency:
   - Mandatory for all state-changing operations (POST requests, Payments, Orders).
   - Use Client-generated Unique Idempotency Keys (UUIDs).
   - Cache key-response pairs in Redis to return identical responses on retries.

