# Ontology Learning from Dialogue: Building Shared Understanding Dynamically

## Vision

**The Dream:** Two software agents meet for the first time. They have no shared ontology, no predefined schemas, no common vocabulary. Yet through conversation, they:

1. Discover each other's capabilities
2. Align on terminology
3. Build a shared conceptual model (ontology)
4. Execute tasks using this emergent understanding

**The Reality (2024):** Partially achievable with modern LLMs, but significant gaps remain.

---

## What is Ontology Learning?

**Classical Definition:** The automatic or semi-automatic extraction of concepts, relationships, and axioms from text or structured data to construct a formal ontology.

**Modern LLM-Based Definition:** The process by which language models identify, organize, and align conceptual structures from natural language interactions, enabling agents to develop shared understanding without predefined schemas.

---

## Why It Matters for Inter-Software Communication

### The Ontology Burden in Traditional Systems

**FIPA-ACL Example:**
```
(REQUEST
  :content "((action agent2 (sell book :title 'AI' :price 50)))"
  :ontology book-trading)
```

**Problem:** Both agents must **pre-agree** on:
- What a "book" is (concept)
- What "sell" means (relationship/action)
- What properties books have (title, price, author, ISBN, etc.)
- What data types to use (string, number, currency)

**Result:** Rigidity. Can't handle:
- Novel concepts ("audiobook subscription")
- Cross-domain requests ("trade book for consulting hours")
- Evolution (adding "digital rights" to books)

### The Ontology-Free Vision

**LLM-Based Approach:**
```
Agent A: "I have a book I'd like to sell. It's 'AI: A Modern Approach'"
Agent B: "I buy books. What's your asking price?"
Agent A: "$50"
Agent B: "Deal. How should I transfer payment?"
```

**Magic:** No predefined ontology, yet they:
- Understand "book" (even without formal definition)
- Recognize "sell" as an economic transaction
- Negotiate price (emergent protocol)
- Coordinate on next steps

**Limitation:** This only works because both LLMs were trained on internet-scale corpora containing book-selling transactions. What about novel domains?

---

## Approaches to Ontology Learning

### 1. Zero-Shot Ontology Learning with LLMs

**Approach:** Use LLM's pre-trained knowledge to infer ontologies from minimal examples.

**LLMs4OL 2024 Challenge:** First Large Language Models for Ontology Learning Challenge

**Three Core Tasks:**

#### Task 1: Term Typing
**Goal:** Classify terms into ontology categories

Example:
```
Input: "Laptop"
Output: {type: "Product", subtype: "Electronics"}

Input: "Refund"
Output: {type: "BusinessProcess", domain: "Finance"}
```

**2024 Performance:**
- GPT-4: 78% accuracy (vs 85% human agreement)
- Claude 3.5 Sonnet: 81% accuracy
- Local LLMs: 55-65%

**Key Insight:** LLMs can infer types from general world knowledge, but struggle with domain-specific classifications.

---

#### Task 2: Taxonomy Discovery
**Goal:** Organize concepts into hierarchical relationships (is-a)

Example:
```
Input: ["Laptop", "Desktop", "Computer", "Electronic Device", "MacBook"]

Output (Taxonomy):
Electronic Device
  └── Computer
       ├── Desktop
       └── Laptop
            └── MacBook
```

**2024 Performance:**
- Hierarchical accuracy: 72% (correct parent assignment)
- Depth estimation: 85% (correct level in tree)

**Challenges:**
- Multiple valid hierarchies (is "MacBook" a subtype of "Laptop" or "Apple Product"?)
- Cross-cutting concerns (a "Gaming Laptop" is both "Laptop" and "Gaming Device")

---

#### Task 3: Non-Taxonomic Relation Extraction
**Goal:** Identify relationships beyond is-a (has-part, causes, requires, etc.)

Example:
```
Input: "A laptop has a battery. The battery powers the CPU."

Output:
Laptop --[has-part]--> Battery
Battery --[powers]--> CPU
```

**2024 Performance:**
- Relation identification: 65% precision, 58% recall
- Relation type classification: 72% accuracy

**Gap:** LLMs struggle with implicit relations and domain-specific connection types.

---

### 2. Ontology Learning from Dialogue State Tracking

**Problem:** In task-oriented dialogue (e.g., booking flights), track what the user wants without predefined slots.

**Traditional Approach (Slot-Filling):**
```
Slots: {
  departure_city: null,
  arrival_city: null,
  date: null
}

User: "I want to fly from Boston to Seattle next Tuesday"

Filled: {
  departure_city: "Boston",
  arrival_city: "Seattle",
  date: "2025-11-11"  // next Tuesday
}
```

**Problem:** Fixed slots. Can't handle "I want a window seat" (seat_preference not in schema).

---

**Modern Approach (Zero-Shot DST):**

**Research:** "Semantic Parsing by Large Language Models for Intricate Updating Strategies of Zero-Shot Dialogue State Tracking" (2023, EMNLP)

**Approach:** Reformulate dialogue state as semantic parsing to JSON

```
User: "I want to fly from Boston to Seattle next Tuesday, window seat if possible"

LLM Output:
{
  "intent": "book_flight",
  "constraints": {
    "departure_city": "Boston",
    "arrival_city": "Seattle",
    "date": "2025-11-11",
    "seat_preference": "window"  // Emergent slot!
  }
}
```

**Key Innovation:** The ontology (schema) is **discovered dynamically** from dialogue, not predefined.

**2024 Performance (ParsingDST):**
- Joint Goal Accuracy: 56.3% on MultiWOZ 2.1
- Improvement: +20% over previous zero-shot methods
- Still below supervised methods (~75%), but no training data needed!

---

**Zero-Shot Open-Vocabulary Pipeline (September 2024):**

**Paper:** "A Zero-Shot Open-Vocabulary Pipeline for Dialogue Understanding" (arXiv:2409.15861)

**Innovations:**
1. **No fixed slot values:** System doesn't need predefined values like "Boston", "Seattle"
2. **Self-refining prompts:** LLM critiques its own output and corrects errors
3. **QA reformulation:** Instead of "extract slots", ask "What city is the user departing from?"

**Results:**
- 20% better Joint Goal Accuracy than previous methods
- Works across domains (flights, hotels, restaurants) without retraining

**Implication for Agents:** Agents can understand each other's requests even when capabilities/schemas differ!

---

### 3. Frame Semantic Parsing

**Idea:** Represent actions as frames (structured templates) that can be learned from examples.

**Research:** "Towards Zero-Shot Frame Semantic Parsing with Task Agnostic Ontologies and Simple Labels" (arXiv:2305.03793, 2023)

**Framework: OpenFSP**

**Goal:** Enable non-experts to define new domains using simple labels (not full ontologies).

**Example:**

Define a new "file_transfer" domain:
```
Frame: TransferFile
  Roles:
    - source_path (string)
    - destination_path (string)
    - file_name (string)
    - protocol (enum: ftp, http, scp)

  Example utterances:
    - "Copy <file_name> from <source_path> to <destination_path> via <protocol>"
    - "Transfer <file_name> using <protocol>"
```

**Zero-Shot Parsing:**
User: "Send report.pdf from /home/docs to /backup using scp"

Parsed:
```json
{
  "frame": "TransferFile",
  "roles": {
    "file_name": "report.pdf",
    "source_path": "/home/docs",
    "destination_path": "/backup",
    "protocol": "scp"
  }
}
```

**Performance:**
- F1 score: 68% on unseen domains (vs 85% for supervised)
- Requires only 3-5 example utterances per domain

**Implication:** Agents can define their capabilities using simple examples, and other agents can learn to call them!

---

### 4. Ontology Alignment (Cross-Agent Mapping)

**Scenario:** Two agents have different ontologies. How do they communicate?

**Agent A's Ontology:**
```
Product {
  name: string
  cost: number (USD)
  in_stock: boolean
}
```

**Agent B's Ontology:**
```
Item {
  title: string
  price: number (EUR)
  availability: enum(available, out_of_stock)
}
```

**Alignment Challenge:**
- `name` ↔ `title` (synonym mapping)
- `cost` ↔ `price` (synonym + currency conversion)
- `in_stock` ↔ `availability` (boolean vs enum + semantic mapping)

---

**Classical Approach:** Manual mapping or ontology matching algorithms (COMA, Falcon)

**LLM-Based Approach (2024):**

1. **Schema Description:**
   Each agent provides natural language descriptions:
   ```
   Agent A: "name is the product's official name"
   Agent A: "cost is the price in US dollars"
   Agent A: "in_stock is true if we have inventory"
   ```

2. **LLM Mapping:**
   ```
   Prompt: Given these two schemas, create a mapping.

   LLM Output:
   {
     "name": "title",
     "cost": {"target": "price", "transform": "USD_to_EUR"},
     "in_stock": {
       "target": "availability",
       "transform": {
         "true": "available",
         "false": "out_of_stock"
       }
     }
   }
   ```

3. **Validation:**
   Send test queries and verify responses match expected semantics.

**Success Rate (2024 Research):**
- Simple mappings (synonyms): 90%+
- Complex mappings (transformations): 70-80%
- Semantic conflicts (incompatible models): 40-50%

**Gap:** LLMs can suggest mappings, but can't verify correctness without test data.

---

## Knowledge Graph Construction from Conversation

**Goal:** Build structured knowledge graphs from agent dialogues.

### Recent Research (2024)

**Paper:** "Constructing Personal Knowledge Graph from Conversation via Deep Reinforcement Learning" (Springer 2024)

**Approach:**
1. **Extract entities from dialogue:**
   ```
   User: "I met John at the conference yesterday"
   Entities: {User, John, conference}
   ```

2. **Identify relationships:**
   ```
   Relationships:
   - User --[met]--> John
   - meeting --[located_at]--> conference
   - meeting --[time]--> yesterday
   ```

3. **Build knowledge graph:**
   ```
   (User) --[met {when: yesterday, where: conference}]--> (John)
   ```

4. **Reinforcement learning path reasoning:**
   - Learn to traverse graph to answer questions
   - Example: "Where did I meet John?" → traverse (User)-[met]->(John), extract [where] → "conference"

**Results:**
- 78% accuracy on personal QA tasks
- Graph construction precision: 72%
- Handles multi-turn conversations

**Implication for Agents:** As agents communicate, they can build shared knowledge graphs representing their interaction history, context, and learned facts.

---

### GraphWOZ: Dialogue Management with Knowledge Graphs

**Innovation:** Represent dialogue state as a **dynamic knowledge graph** instead of fixed slots.

**Traditional Dialogue State:**
```json
{
  "hotel_name": "Marriott",
  "check_in": "2025-11-10",
  "check_out": "2025-11-12"
}
```

**GraphWOZ State:**
```
(User) --[wants]--> (Reservation)
(Reservation) --[at]--> (Marriott:Hotel)
(Reservation) --[check_in]--> (2025-11-10:Date)
(Reservation) --[check_out]--> (2025-11-12:Date)
(Marriott) --[has_amenity]--> (Pool)
(User) --[mentioned]--> (Pool)  // from "Do they have a pool?"
```

**Advantages:**
- Can represent complex, nested entities
- Captures relationships between entities (not just slots)
- Can evolve (add nodes/edges as conversation progresses)

**Performance:**
- Better handles multi-domain dialogues (hotel + restaurant + taxi)
- 12% improvement in task completion vs slot-filling

**Challenge:** Graph reasoning is more complex than slot extraction.

---

## Building Ontologies from Agent Interactions

### The Bootstrapping Process

**Phase 1: Initial Contact (No Shared Ontology)**

```
Agent A: "Hello, I'm a data analysis agent. What can you do?"
Agent B: "I'm a weather information service. I can provide forecasts, historical data, and alerts."
```

**Extracted Ontology (Initial):**
```
Agent A: {
  type: "agent",
  capabilities: ["data_analysis"]
}

Agent B: {
  type: "service",
  domain: "weather",
  capabilities: ["forecasts", "historical_data", "alerts"]
}
```

---

**Phase 2: Capability Exploration**

```
Agent A: "What do you need to provide a forecast?"
Agent B: "I need a location (city name or coordinates) and optionally a date range."
Agent A: "What format do you return?"
Agent B: "I return JSON with temperature, precipitation, wind speed, and conditions."
```

**Extracted Ontology (Expanded):**
```
Service: WeatherForecast
  Inputs:
    - location: string | {lat: number, lon: number}
    - date_range: {start: date, end: date} (optional)
  Outputs:
    - temperature: number
    - precipitation: number
    - wind_speed: number
    - conditions: string
```

---

**Phase 3: Alignment and Refinement**

```
Agent A: "I have temperature data in Fahrenheit. Do you need Celsius?"
Agent B: "I can accept both. Just specify units."
```

**Ontology Update:**
```
Service: WeatherForecast
  Inputs:
    ...
    - units: enum(celsius, fahrenheit) (optional, default: celsius)
```

---

**Phase 4: Execution and Learning**

```
Agent A: "GET /forecast?location=Boston&units=fahrenheit"
Agent B: { "temperature": 68, "conditions": "partly cloudy", ... }
Agent A: "Thanks. For future reference, I always use Fahrenheit."
```

**Ontology Update (Preference Learning):**
```
Agent A preferences:
  - temperature_units: fahrenheit
```

**Result:** Over multiple interactions, agents build:
- **Shared vocabulary:** Both agree "location" can be city name or coords
- **Interface contracts:** A knows how to call B's forecast endpoint
- **Preferences:** Reduce negotiation overhead in future calls

---

## Challenges and Limitations (2024)

### 1. Semantic Ambiguity

**Problem:** Natural language is inherently ambiguous.

**Example:**
```
Agent A: "What's the status of order 123?"
Agent B: "It's being processed"
```

**Ambiguity:** Does "processed" mean:
- Payment is being verified?
- Items are being picked from warehouse?
- Shipment is in transit?

**LLM Response:** Varies based on training data bias.

**Solution (Partial):**
- Request clarification: "By processed, do you mean payment, fulfillment, or shipping?"
- Define status taxonomy explicitly after first interaction

---

### 2. Hallucination and Incorrect Inferences

**Problem:** LLMs may infer incorrect ontology elements.

**Example:**
```
Agent A: "I sell books"
LLM Inference: {
  products: ["physical books", "e-books", "audiobooks"]  // WRONG!
}
```

**Reality:** Agent A only sells physical books.

**Consequence:** Agent B requests "audiobook of XYZ", Agent A can't fulfill, error.

**Solution:**
- Explicit capability listing (don't infer)
- Validation through test queries
- Error feedback to correct ontology

---

### 3. Ontology Drift

**Problem:** As agents interact over time, their internal representations may diverge.

**Example:**
```
Week 1: Agent A calls temperature endpoint, gets Fahrenheit
Week 5: Agent B updates to return Celsius by default
Agent A still expects Fahrenheit → data misinterpretation!
```

**Solution:**
- Version ontologies (like API versioning)
- Explicit negotiation on each interaction (overhead)
- Change notifications (pub/sub pattern)

---

### 4. Scalability of Shared Ontologies

**Problem:** In a network of N agents, there are N(N-1)/2 potential pairwise ontologies.

**Example:**
- 10 agents → 45 pairwise mappings
- 100 agents → 4,950 mappings

**Combinatorial explosion:** Infeasible to maintain all mappings.

**Solutions:**

**Option 1: Upper Ontology (Common Core)**
- Define shared concepts all agents understand
- Example: Basic types (string, number, date), common actions (query, update, delete)
- Agents map their specific concepts to upper ontology

**Option 2: Ontology Hubs**
- Designate central agents that mediate between others
- Hub maintains mappings to all connected agents
- Reduces N² to N mappings

**Option 3: Lazy Learning**
- Only build mappings when needed (first interaction)
- Cache for reuse
- Garbage collect unused mappings

---

### 5. Trust and Verification

**Problem:** How does Agent A trust that Agent B's self-described ontology is accurate?

**Example:**
```
Agent B: "I can predict stock prices with 95% accuracy"
[Reality: 55% accuracy, barely better than random]
```

**Solutions:**
- **Reputation systems:** Track accuracy of past claims
- **Third-party verification:** Certification authorities
- **Test-driven interaction:** A sends test queries before trusting B

**Current State:** Mostly unsolved. Trust is established through repeated interactions and reputation.

---

## Hybrid Approaches: Best of Both Worlds

### Pattern: Ontology Bootstrapping with Formal Refinement

**Process:**
1. **Initial Discovery (NL):** Agents converse to discover capabilities
2. **Ontology Extraction (LLM):** Extract structured ontology from conversation
3. **Formalization (Human/Automated):** Convert to formal ontology (OWL, RDF)
4. **Validation (Testing):** Verify through example queries
5. **Refinement (Iterative):** Update based on errors and new information

**Example:**

**Step 1: Conversation**
```
Agent A: "I need to store data persistently"
Agent B: "I provide a key-value store. You can PUT, GET, and DELETE by key."
```

**Step 2: LLM Extraction**
```json
{
  "service": "KeyValueStore",
  "operations": [
    {"name": "PUT", "params": ["key", "value"]},
    {"name": "GET", "params": ["key"], "returns": "value"},
    {"name": "DELETE", "params": ["key"]}
  ]
}
```

**Step 3: Formalization (OWL)**
```turtle
:KeyValueStore a owl:Class ;
  rdfs:label "Key-Value Store" .

:PUT a owl:ObjectProperty ;
  rdfs:domain :KeyValueStore ;
  :hasParameter :Key, :Value .

:GET a owl:ObjectProperty ;
  rdfs:domain :KeyValueStore ;
  :hasParameter :Key ;
  :returns :Value .
```

**Step 4: Validation**
```
Test: PUT(key="test", value="hello")
Test: GET(key="test") → "hello" ✓
Test: DELETE(key="test")
Test: GET(key="test") → null ✓
```

**Step 5: Refinement**
```
Error: GET(key="nonexistent") → should return error, not null
Update ontology: GET returns union(Value, Error)
```

**Result:** Combines flexibility of NL discovery with rigor of formal ontologies.

---

## The Frontier: Truly Emergent Ontologies

### Research Direction: Ontology Evolution through Usage

**Vision:** Ontologies are not bootstrapped from conversation, but emerge organically from usage patterns.

**Approach:**
1. Start with minimal ontology (basic types)
2. Agents interact freely
3. System observes patterns:
   - Which terms co-occur?
   - Which operations follow which others?
   - What implicit relationships exist?
4. Automatically propose ontology extensions
5. Agents validate and adopt

**Example:**

**Initial State:**
```
Ontology: {
  types: [string, number, boolean]
}
```

**After 100 interactions:**
```
Observed Patterns:
- Term "order" often appears with "customer", "items", "total"
- Operation "place_order" usually followed by "payment"
- Numeric values after "total" are always positive

Proposed Ontology Extension:
Order {
  customer: reference(Customer)
  items: array(Item)
  total: PositiveNumber
}

Workflow:
  place_order() → payment() → confirmation()
```

**Validation:**
- Present to agents: "I noticed this pattern. Is this accurate?"
- Agents confirm or correct
- Ontology added to shared knowledge

**2024 State:** Mostly theoretical. Some research in conversational AI, but not yet applied to multi-agent systems.

---

## Practical Recommendations

### For Agent Developers

1. **Provide Rich Descriptions**
   - Don't just list capabilities, explain them
   - Include examples of inputs/outputs
   - Specify constraints and requirements

2. **Support Ontology Negotiation**
   - Be prepared to explain your concepts
   - Ask for clarification on unfamiliar terms
   - Validate understanding through test queries

3. **Version Your Ontology**
   - Track changes to your schema
   - Notify dependent agents of breaking changes
   - Support multiple versions during transition

4. **Learn from Interactions**
   - Cache successful ontology mappings
   - Update preferences based on partner agents
   - Build reputation models for accuracy

### For System Architects

1. **Design for Hybrid Communication**
   - Use NL for discovery and negotiation
   - Use structured formats (JSON, RDF) for data exchange
   - Validate LLM outputs against schemas

2. **Build Ontology Repositories**
   - Maintain shared ontology store
   - Enable agents to query and contribute
   - Version control for ontologies

3. **Implement Verification Mechanisms**
   - Test-driven interaction (validate with examples)
   - Monitor for semantic drift
   - Alert on ontology conflicts

4. **Provide Fallback Mechanisms**
   - When ontology learning fails, fall back to manual specification
   - Support human-in-the-loop for critical ontology decisions

---

## Connection Points

- See `../concepts/agent-communication-protocols.md` for how ontologies integrate with agent protocols (A2A, ANP)
- See `../reliability-tradeoffs/conversational-vs-rest-analysis.md` for trade-offs between structured and unstructured ontologies
- See `../patterns/semantic-discovery.md` for patterns of capability discovery and ontology negotiation
- See `../implementations/knowledge-graph-systems.md` for practical KG construction systems

---

## Open Research Questions

1. **Can ontologies be fully emergent?**
   - Or do we always need some seed ontology (basic types, upper ontology)?

2. **How to handle ontological conflicts?**
   - Example: Agent A models "employee" as a person, Agent B as a contract
   - Can LLMs mediate such deep semantic differences?

3. **What is the optimal granularity for shared ontologies?**
   - Too coarse → Loss of precision
   - Too fine → Overfitting, fragility

4. **How to evaluate ontology quality?**
   - Metrics: Coverage, consistency, usability
   - But how to measure these automatically?

5. **Can we prove properties of learned ontologies?**
   - Formal verification of consistency, completeness
   - Detect logical contradictions

---

## Summary

**What We Know (2024):**
- LLMs can perform zero-shot ontology learning (term typing, taxonomy, relation extraction) with 70-90% accuracy
- Dialogue state tracking can be reformulated as ontology learning, enabling open-vocabulary systems
- Knowledge graphs can be constructed from conversations with 70-80% precision
- Ontology alignment can be automated with LLMs for simple cases (synonyms, simple transformations)

**What We Don't Know:**
- How to achieve human-level accuracy (95%+) without supervision
- How to handle truly novel concepts (not in LLM training data)
- How to scale ontology learning to large multi-agent networks
- How to verify correctness of learned ontologies
- How to make ontologies evolve gracefully over time

**The Vision:**
Software agents that meet, converse, discover shared understanding, and collaborate—all without predefined ontologies. We're 50% of the way there. The next frontier is robustness, scalability, and formal guarantees.
