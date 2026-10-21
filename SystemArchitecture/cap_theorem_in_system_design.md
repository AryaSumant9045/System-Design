# CAP Theorem (Brewer's Theorem)

**CAP Theorem** distributed systems design ka ek fundamental concept hai. Yeh batata hai ki kisi bhi distributed system me aap ek hi waqt par teeno properties ko guarantee **nahi** kar sakte. Aapko maximum do properties choose karni padti hain:

1. **C** – Consistency
2. **A** – Availability
3. **P** – Partition Tolerance

---

## 1. Core Concepts (Teeno Properties Ka Matlab)

### **C – Consistency (Har jagah same data)**
- **Definition:** Har read request par aapko sabse **latest (updated)** successful write wala data hi milega.
- **Example:** Bank account balance. Agar aapne ₹10,000 transfer kiye hain, toh kisi bhi ATM, net banking, ya UPI app se balance verify karne par updated amount hi dikhna chahiye.

### **A – Availability (Hamesha response milega)**
- **Definition:** Network ka har non-failing node har request par response zaroor dega (error nahi dega). Ho sakta hai ki milega response thoda purana (stale) data ho, par request crash/fail nahi hogi.
- **Example:** E-commerce product reviews ya Social Media feed. Server busy hone par bhi feed load ho jayegi, chahe 1-2 latest comments na dikhein.

### **P – Partition Tolerance (Network break jhelna)**
- **Definition:** System ke nodes ke beech me agar network link toot jaye (Network Partition), tab bhi system apna kaam karta rahega.
- **Reality:** Real-world distributed systems me multiple servers alag-alag network wires/switches se jude hote hain, toh network lag ya connection drop hona tay hai. **Distributed systems me P compulsory hota hai.**

---

## 2. CP vs AP Trade-off

Chunki real-world systems me **Partition Tolerance (P)** non-negotiable hai, asal choices do hi milti hain: **CP** ya **AP**.

```
                   CAP Theorem Choice
                          / \
                         /   \
                        /     \
                       /       \
             CP System           AP System
        (Consistency + P)     (Availability + P)
               |                      |
      Trade-off: High Data   Trade-off: Fast & Always Up,
      Accuracy, temporary    might serve older data
      downtime/errors        temporarily (Eventual Consistency)
```

---

## 3. Detailed Comparison Matrix

| Property | CP Systems (Consistency + Partition Tolerance) | AP Systems (Availability + Partition Tolerance) |
| :--- | :--- | :--- |
| **Behavior during Network Failure** | Network cut hone par sync na ho pane wale nodes requests ko reject kar denge (Error show karenge) taaki galat data na dikhe. | System hamesha response dega, chahe kuch nodes par purana data hi kyo na ho. Network thik hone par data sync ho jata hai. |
| **Data Integrity** | Strict Integrity (No stale data allowed) | Eventual Consistency (Data thodi der baad sabhi nodes par match hoga) |
| **Best Used For** | Financial transactions, inventory control, authentication services | Social media feeds, live chat/comments, recommendations, logging |
| **Popular Databases** | PostgreSQL/MySQL (Replicated), MongoDB, HBase, Redis | Apache Cassandra, Amazon DynamoDB, CouchDB |

---

## 4. Summary Takeaway for Interviews

1. **CA System Real World Me Possible Kyun Nahi Hai?**  
   Distributed network me network issues (partitions) hona zaroori (inevitable) hai. Agar network disconnect hoga, toh aapko **Consistency rokni padegi (AP)** ya fir **Availability roki padegi (CP)**.
2. **Key Question to Ask in System Design:**  
   *"Kya mere system ko Data Accuracy zaroori hai (CP), ya System Downtime zero rakhna zaroori hai (AP)?"*