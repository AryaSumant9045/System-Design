
                    Student
                       ↓
               Load Balancer
              ↓        ↓        ↓
          Server 1  Server 2  Server 3   (stateless)
                       ↓
          ┌────────────┴─────────────┐
          ↓                          ↓
   [Chat flow]                 [Upload flow]
          ↓                          ↓
   1. Semantic Cache            Queue (BullMQ)
      ↓ miss                         ↓
   2. Embedding                  Workers (checkpoint + idempotent)
      ↓                              ↓
   3. Vector DB                  transcribe → chunk → embed
      ↓                              ↓
   4. LLM (circuit breaker       Vector DB + status in DB
      + fallback LLM)
      ↓
   5. Stream answer
      ↓
   Chat history → DB (indexed by user_id, replicated, shard later)