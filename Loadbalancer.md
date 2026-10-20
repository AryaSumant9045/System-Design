# Incident Post-Mortem: Load Balancer Session Loss, Empty Cart & Logout Bug

> **Scenario:** System par Load Balancer (LB) lagate hi customers achanak logout hone lage, product cart empty/change hone laga, aur multi-step forms clear/blank hone lage.

---

## 1. Root Cause Analysis

Yeh **Stateless Server Architecture vs Stateful In-Memory Sessions** ki classic system design problem hai.

### Why Did This Happen?
Load Balancer se pehle system mein **ek hi application server** tha. User ka Auth Token, Shopping Cart, aur Form Drafts sab ussi single server ki **RAM (In-Memory Session)** mein save ho rahe the.

Load Balancer add hone ke baad, requests multiple servers (**Server A, Server B, Server C**) mein distribute hone lagi:

```
[ Customer Request ]
                               │
                               ▼
                     [ Load Balancer (LB) ]
                     (Round-Robin Routing)
                      /        │        \
                     /         │         \
                    ▼          ▼          ▼
              ┌──────────┐┌──────────┐┌──────────┐
              │ Server A ││ Server B ││ Server C │
              └──────────┘└──────────┘└──────────┘
              [RAM: Auth] [RAM: Empty][RAM: Empty]

```

### Symptom Breakdown:

| Issue | Root Cause |
| :--- | :--- |
| **Achanak Logout** | **Request 1 (Login)** Server A par gayi $\rightarrow$ Session Server A ki RAM mein save hua.<br>**Request 2 (Next Click)** LB ne Server B par bheji $\rightarrow$ Server B ki RAM mein session nahi tha $\rightarrow$ User logged out! |
| **Cart Empty / Shuffle** | User ne item add kiya jab request Server A par thi (Cart = 2 Items in Server A RAM). Checkout/Refresh par request Server C par gayi $\rightarrow$ Cart blank ho gaya. |
| **Form Clear / Blank** | Multi-step form ka draft state Server A ke process memory mein tha. Request Server B par switch hone se state match nahi hui aur form reset ho gaya. |

---

## 2. Solutions & Architectural Mitigations

---

### Solution 1: Quick Hotfix — Sticky Sessions / Session Affinity

Load Balancer level par **Sticky Sessions** enable karein.

* **Mechanism:** Load Balancer pehli request par browser mein ek Cookie (`sticky-cookie`) set kar deta hai. Uske baad uss specific user ki saari requests hamesha **same server (e.g., Server A)** par hi route hoti hain.
* **Pros:** 
  * Code level par **zero changes** (LB console se 1-click enable).
* **Cons (Why it fails in scale):**
  * **Single Point of Failure:** Agar Server A crash/restart hua, toh uss server par jude sabhi users ka session permanent lost ho jayega.
  * **Uneven Auto-Scaling:** Traffic distribution uneven ho jata hai (kuch servers over-utilized ho jate hain).

---

### Solution 2: Permanent Fix — Centralized Session Store (Stateless Architecture)

Application Servers ko **Stateless** banayein. Koi bhi application server user ka state/session apni local RAM mein store nahi karega.
```


                       [ Customer ]
                            │
                            ▼
                  [ Load Balancer (LB) ]
                   /        │        \
                  ▼         ▼         ▼
             [ Server A ] [ Server B ] [ Server C ]
             (Stateless)  (Stateless)  (Stateless)
                  │         │         │
                  └─────────┼─────────┘
                            ▼
               ┌─────────────────────────┐
               │ Central Redis Cluster   │
               │ (Shared Session Store)  │
               └─────────────────────────┘
```
#### Step-by-Step Implementation:

1. **Centralized Redis / Memcached Store:**
   * Session data ko Server A, B, ya C ki local RAM mein rakhne ke bajaye ek central **Redis Cache Cluster** mein rakhein.
   * Jab Server B par request aayegi, woh cookie se `SessionID` read karke Central Redis se live data query kar lega:
     ```
     Server B -> Query Redis(SessionID_123) -> Found Auth & Cart Data!
     ```
2. **Stateless Authentication (JWT - JSON Web Tokens):**
   * Auth session tracking ke liye **JWT** use karein.
   * User login par signed JWT return hota hai (containing User ID, Role, Expiry) jo browser `HTTP-Only Cookie` mein store karta hai.
   * Any server (A, B, C) can independently decrypt/verify the token using a shared `SECRET_KEY` without making a database lookup.
3. **Cart & Draft Persistence:**
   * Shopping Cart items aur Form Drafts ko central Database (PostgreSQL/MongoDB) ya Redis mein User ID ke against key-value pair mein persist karein.

---

## 3. Architecture Comparison

| Parameter | Sticky Sessions (Hotfix) | Centralized Redis (Stateless Fix) |
| :--- | :--- | :--- |
| **Setup Effort** | Low (5 Mins LB Config) | Medium (Code Integration + Redis Instance) |
| **Server Crash Resilience** | ❌ Data Lost | ✅ Safe in Redis Cluster |
| **Auto-Scaling Compatibility** | ❌ Poor Traffic Distribution | ✅ Perfect Load Balancing |
| **Production Grade** | ⚠️ Legacy / Quick Patch | 🚀 Enterprise Standard |

---

## 4. Key Takeaways

1. **Never store user state in server process memory** when running multiple application instances behind a Load Balancer.
2. **Decouple State from Compute:** Compute nodes (servers) should be disposable and stateless; state belongs in dedicated stores (Redis/Database).
3. **Use JWTs or Centralized Caches** for auth sessions across distributed microservices/nodes.

