# Hybrid Communication Patterns: Bridging Structured and Conversational Interfaces

## The Synthesis

Neither pure REST APIs nor pure conversational interfaces are optimal for most systems. The future of inter-software communication lies in **hybrid architectures** that strategically combine both paradigms.

This document catalogs proven patterns, emerging approaches, and design principles for hybrid systems.

---

## Pattern Catalog

### Pattern 1: NL2API (Natural Language to API Gateway)

**Intent:** Enable natural language access to existing REST APIs without modifying them.

**Architecture:**

```
┌─────────────┐
│   User/     │
│   Agent     │ "Get weather for San Francisco"
└──────┬──────┘
       │ (Natural Language)
       ↓
┌─────────────────────┐
│  LLM Orchestrator   │ Intent: get_weather(location="San Francisco")
│   (Intent Parser)   │
└──────┬──────────────┘
       │ (Structured API Call)
       ↓
┌─────────────────────┐
│   REST API Layer    │ GET /weather?city=san_francisco&units=metric
│  (Weather Service)  │
└──────┬──────────────┘
       │ (JSON Response)
       ↓
┌─────────────────────┐
│  LLM Orchestrator   │ "It's 62°F and partly cloudy in San Francisco"
│ (Response Formatter)│
└──────┬──────────────┘
       │ (Natural Language)
       ↓
┌─────────────┐
│   User/     │
│   Agent     │
└─────────────┘
```

**Components:**

1. **Intent Parser (LLM):**
   - Input: Natural language query
   - Output: Structured function call
   - Example: "weather in Boston" → `get_weather(location="Boston")`

2. **API Registry:**
   - Maps intents to REST endpoints
   - Maintains schemas for validation
   - Example:
     ```json
     {
       "get_weather": {
         "endpoint": "/weather",
         "method": "GET",
         "params": {
           "location": {"type": "string", "required": true},
           "units": {"type": "enum", "values": ["metric", "imperial"], "default": "metric"}
         }
       }
     }
     ```

3. **Validator:**
   - Checks LLM output matches schema
   - Prevents malformed requests
   - Retries with LLM if validation fails

4. **REST Executor:**
   - Makes actual API call
   - Handles errors (retries, timeouts)
   - Returns structured response

5. **Response Formatter (LLM):**
   - Input: JSON response from API
   - Output: Natural language summary
   - Example: `{temp: 62, conditions: "cloudy"}` → "It's 62°F and partly cloudy"

**Advantages:**
- ✅ Leverage existing APIs without modification
- ✅ Deterministic execution (REST layer)
- ✅ Flexible input (natural language)
- ✅ Easy to add new APIs (update registry)

**Disadvantages:**
- ❌ Latency: LLM parsing + REST call + LLM formatting
- ❌ Intent parsing can fail (ambiguous queries)
- ❌ Requires maintaining schema mappings

**Best For:**
- Chatbots for existing services
- Human-friendly interfaces to APIs
- Exploratory access to data

**Real-World Examples:**
- Microsoft's NL2API research project
- Anthropic's Claude Code (this system!) accessing tools
- OpenAI's ChatGPT plugins (before function calling API)

---

### Pattern 2: Natural Language Command Interfaces (NLCI)

**Intent:** Ensure reliability by constraining LLM outputs to structured commands.

**Architecture:**

```
┌─────────────┐
│   User/     │ "Transfer $100 from checking to savings"
│   Agent     │
└──────┬──────┘
       │ (Natural Language)
       ↓
┌──────────────────────┐
│  LLM Command Parser  │
│  (Constrained Output)│
└──────┬───────────────┘
       │ (Structured Command - JSON)
       ↓
┌──────────────────────────┐
│  {"command": "transfer", │
│   "params": {            │
│     "from": "checking",  │
│     "to": "savings",     │
│     "amount": 100        │
│   }}                     │
└──────┬───────────────────┘
       │
       ↓
┌──────────────────────┐
│   Command Validator  │ Check: amount > 0, accounts exist, balance sufficient
│   (Schema + Rules)   │
└──────┬───────────────┘
       │ (Validated Command)
       ↓
┌──────────────────────┐
│ Deterministic        │ Execute transfer (database transaction)
│ Executor             │
└──────┬───────────────┘
       │ (Result)
       ↓
┌──────────────────────┐
│   User/Agent         │ "Transfer complete. New checking balance: $400"
└──────────────────────┘
```

**Key Principles:**

1. **Constrained Generation:**
   - Use LLM's JSON mode or structured output features
   - Provide schema to LLM:
     ```json
     {
       "command": "transfer | withdraw | deposit | check_balance",
       "params": {
         "from": "string (account name)",
         "to": "string (account name)",
         "amount": "number (positive)"
       }
     }
     ```

2. **Validation Before Execution:**
   - **Syntactic:** Does output match schema?
   - **Semantic:** Do params make sense? (e.g., amount > 0)
   - **Business Logic:** Is action allowed? (e.g., sufficient balance)
   - **CRITICAL:** Validation happens *before* execution, not after!

3. **Deterministic Execution:**
   - Once validated, execution is pure function
   - No further LLM involvement
   - Same command → Same result (idempotency if possible)

4. **Audit Trail:**
   - Log every command (for debugging and compliance)
   - Store: timestamp, user, command, validation result, execution result

**Example Flow:**

```
User: "Pay my rent ($1500) from checking"

LLM Output:
{
  "command": "transfer",
  "params": {
    "from": "checking",
    "to": "landlord_account",
    "amount": 1500
  }
}

Validator Checks:
✓ Schema valid (has command, params)
✓ Amount > 0
✓ "checking" account exists
? "landlord_account" not found → REJECT

Response to User:
"I couldn't find 'landlord_account'. Did you mean 'landlord_rent_payment'?"

User: "Yes"

[Retry with corrected account]
```

**Advantages:**
- ✅ Reliable execution (validated commands)
- ✅ Auditable (structured logs)
- ✅ Testable (unit test command validators)
- ✅ Secure (validation prevents injection attacks)

**Disadvantages:**
- ❌ Less flexible than free-form conversation
- ❌ Requires defining command schemas upfront
- ❌ Can't handle truly novel requests

**Best For:**
- Financial transactions
- Administrative actions (user management, config changes)
- Any critical operation requiring auditability

**Real-World Examples:**
- Banking chatbots (high-stakes operations)
- DevOps assistants (infrastructure changes)
- Healthcare systems (medication orders)

**Implementation Note:**
```python
# Pseudocode for NLCI pattern

def handle_request(user_input: str) -> str:
    # Step 1: LLM generates structured command
    command_json = llm.generate(
        prompt=user_input,
        schema=COMMAND_SCHEMA,
        temperature=0  # Maximum determinism
    )

    # Step 2: Validate command
    validation_result = validator.validate(command_json)
    if not validation_result.valid:
        return f"Error: {validation_result.error_message}"

    # Step 3: Execute deterministically
    result = executor.execute(command_json)

    # Step 4: Log for audit trail
    audit_log.record(command_json, result)

    # Step 5: Format response (optional LLM)
    return format_response(result)
```

---

### Pattern 3: Progressive Disclosure (Conversation → Contract)

**Intent:** Start with flexible conversation, progressively formalize into structured contracts.

**Philosophy:** Don't force structure prematurely. Let agents explore freely, then solidify what works.

**Phases:**

#### Phase 1: Discovery (Pure Natural Language)

```
Agent A: "Hello, what can you help with?"
Agent B: "I provide stock market data. I can give you prices, historical charts, and news."
Agent A: "Great! What do you need from me to get a price quote?"
Agent B: "Just a stock ticker symbol, like AAPL or GOOGL."
```

**State:** Unstructured conversation, building mental model.

---

#### Phase 2: Capability Negotiation (NL + Examples)

```
Agent A: "Can you show me an example?"
Agent B: "Sure. If you ask 'What's the price of AAPL?', I'll return something like:
          {
            'symbol': 'AAPL',
            'price': 150.25,
            'currency': 'USD',
            'timestamp': '2025-11-05T10:30:00Z'
          }"
Agent A: "Perfect. Do you support batch queries?"
Agent B: "Yes, you can request multiple symbols: ['AAPL', 'GOOGL', 'MSFT']"
```

**State:** Understanding structure, negotiating features.

---

#### Phase 3: Contract Establishment (Structured Agreement)

```
Agent A: "Let's formalize this. Here's the interface I'll use:

Interface: StockPriceQuery
  Endpoint: /stock/price
  Method: GET
  Parameters:
    - symbols: array<string> (required)
    - currency: enum(USD, EUR) (optional, default: USD)
  Response:
    - array<{symbol: string, price: number, currency: string, timestamp: ISO8601}>

Does this match your service?"

Agent B: "Yes, that's correct. I'll also add:
  - Rate limit: 100 requests/minute
  - Authentication: API key in header 'X-API-Key'
  - Errors: 404 if symbol not found, 429 if rate limit exceeded"

Agent A: "Agreed. Let's proceed with this contract."
```

**State:** Formalized contract (like OpenAPI spec), ready for deterministic use.

---

#### Phase 4: Execution (REST-like, Deterministic)

```
# All future calls use the established contract

Request:
GET /stock/price?symbols=AAPL,GOOGL
Headers: X-API-Key: abc123

Response:
[
  {symbol: "AAPL", price: 150.25, currency: "USD", timestamp: "2025-11-05T10:30:00Z"},
  {symbol: "GOOGL", price: 2800.50, currency: "USD", timestamp: "2025-11-05T10:30:01Z"}
]
```

**State:** Efficient, reliable communication via established contract.

---

#### Phase 5: Evolution (Renegotiation as Needed)

```
[Weeks later, Agent A needs a new feature]

Agent A: "I now need historical data, not just current prices. Can you support this?"
Agent B: "Yes, I can add a /stock/history endpoint. What time range do you need?"
[... negotiation ...]
[Contract updated with new endpoint]
```

**State:** Contracts evolve through conversation when needs change.

---

**Advantages:**
- ✅ Flexible discovery (no upfront schema required)
- ✅ Converges to reliability (established contracts)
- ✅ Adaptable (renegotiate when needed)
- ✅ Documented (conversation = specification)

**Disadvantages:**
- ❌ Slow startup (negotiation overhead)
- ❌ Requires agents to "remember" contracts
- ❌ Ambiguity during transition (when is contract "established"?)

**Best For:**
- Long-lived agent relationships
- Domains with evolving requirements
- Exploratory integrations

**Implementation Considerations:**

1. **Contract Storage:**
   - Store negotiated contracts in database
   - Version contracts (track changes over time)
   - Share contracts across agent instances

2. **Transition Detection:**
   - Recognize when conversation has converged to agreement
   - Prompt: "Should we formalize this as a contract?"
   - Require explicit confirmation

3. **Fallback to Conversation:**
   - If contract fails (e.g., API change), fall back to negotiation
   - Don't fail silently—ask for clarification

**Real-World Analogy:**
This mirrors how human business relationships work:
- First meeting: Informal discussion
- Second meeting: Sketch out agreement
- Third meeting: Sign contract
- Ongoing: Execute per contract, renegotiate if needed

---

### Pattern 4: Semantic API Discovery

**Intent:** Agents discover each other's capabilities through machine-readable descriptions.

**Architecture:**

```
┌────────────────────────────────────────┐
│         Agent Registry Service         │
│  (Like DNS for Agents)                 │
│                                        │
│  Stores: Agent Cards / Capabilities    │
└────────────┬───────────────────────────┘
             │
       ┌─────┴─────┐
       │           │
       ↓           ↓
┌─────────────┐ ┌─────────────┐
│  Agent A    │ │  Agent B    │
│  (Consumer) │ │  (Provider) │
└─────────────┘ └─────────────┘
```

**Agent Card (A2A Protocol Format):**

```json
{
  "agentName": "WeatherService",
  "version": "1.0",
  "description": "Provides real-time weather data and forecasts for cities worldwide",
  "tags": ["weather", "forecast", "climate"],
  "capabilities": [
    {
      "name": "getCurrentWeather",
      "description": "Get current weather conditions for a specified location",
      "inputSchema": {
        "type": "object",
        "properties": {
          "location": {
            "type": "string",
            "description": "City name (e.g., 'San Francisco') or coordinates (e.g., '37.7749,-122.4194')"
          },
          "units": {
            "type": "string",
            "enum": ["metric", "imperial"],
            "default": "metric"
          }
        },
        "required": ["location"]
      },
      "outputSchema": {
        "type": "object",
        "properties": {
          "temperature": {"type": "number"},
          "humidity": {"type": "number"},
          "conditions": {"type": "string"},
          "windSpeed": {"type": "number"}
        }
      },
      "examples": [
        {
          "input": {"location": "Boston", "units": "imperial"},
          "output": {"temperature": 68, "humidity": 45, "conditions": "partly cloudy", "windSpeed": 12}
        }
      ]
    },
    {
      "name": "getForecast",
      "description": "Get multi-day weather forecast",
      "inputSchema": { /* ... */ },
      "outputSchema": { /* ... */ }
    }
  ],
  "authentication": {
    "type": "api-key",
    "headerName": "X-Weather-API-Key"
  },
  "rateLimit": {
    "requestsPerMinute": 60
  },
  "endpoint": "https://weather-service.example.com"
}
```

**Discovery Flow:**

```
# Agent A needs weather data

1. Query Registry:
   A → Registry: "Find agents with capability: weather"
   Registry → A: [WeatherService, ClimateAPI, MeteoAgent]

2. Evaluate Options:
   A reads agent cards for each
   A selects WeatherService (best match for needs)

3. Establish Connection:
   A → WeatherService: "I'd like to use your getCurrentWeather capability"
   WeatherService → A: "Provide API key for authentication"
   A → WeatherService: [Provides key]

4. Execute Calls:
   A → WeatherService: getCurrentWeather(location="Boston", units="imperial")
   WeatherService → A: {temperature: 68, humidity: 45, ...}
```

**Key Components:**

1. **Agent Cards:**
   - Structured metadata about agent capabilities
   - Includes schemas (like OpenAPI, but for agents)
   - Natural language descriptions for LLM understanding

2. **Registry Service:**
   - Central or federated registry
   - Search by tags, capabilities, natural language queries
   - Version management (agents can publish updates)

3. **Capability Matching:**
   - Semantic matching (not just keyword search)
   - Example: Agent needs "temperature data" → Matches "weather" and "climate" agents
   - Uses LLM or embedding-based similarity

4. **Negotiation:**
   - Even with agent cards, may need negotiation
   - Example: Agent card says "supports 100 cities", but A needs a city not listed
   - Fall back to conversation: "Can you support Melbourne, Australia?"

**Advantages:**
- ✅ Discoverability (no hardcoded endpoints)
- ✅ Scalability (registry handles routing)
- ✅ Evolution (agents update cards independently)
- ✅ Semantic search (find agents by capability, not just name)

**Disadvantages:**
- ❌ Registry is single point of failure (unless federated)
- ❌ Agent cards can become stale (agents change without updating cards)
- ❌ Schema validation complexity (many formats)

**Best For:**
- Large multi-agent ecosystems
- Dynamic environments (agents come and go)
- Open-ended tasks (don't know which agent you need upfront)

**Real-World Examples:**
- Google's Agent2Agent (A2A) protocol
- Agent Network Protocol (ANP)
- Microservice discovery (Consul, Eureka) + semantic layer

---

### Pattern 5: Dual-Mode Interfaces

**Intent:** Expose both structured and conversational interfaces for the same service.

**Architecture:**

```
┌──────────────────────────────────────┐
│         Unified Service              │
│                                      │
│  ┌────────────┐   ┌──────────────┐  │
│  │ REST API   │   │ Conversational│  │
│  │ Interface  │   │ Interface     │  │
│  └─────┬──────┘   └───────┬──────┘  │
│        │                  │         │
│        └────────┬─────────┘         │
│                 ↓                   │
│         ┌───────────────┐           │
│         │  Core Logic   │           │
│         │  (Shared)     │           │
│         └───────────────┘           │
└──────────────────────────────────────┘
```

**Example Service: Customer Database**

**Interface 1: REST API (for machines)**

```
GET /customers?name=John
GET /customers/123
POST /customers
  Body: {name: "Jane", email: "jane@example.com"}
PUT /customers/123
  Body: {email: "newemail@example.com"}
DELETE /customers/123
```

**Interface 2: Conversational (for humans/LLMs)**

```
User: "Find customers named John"
Service: [Calls internal function] → Returns natural language summary

User: "Create a new customer named Jane with email jane@example.com"
Service: [Parses intent] → [Validates] → [Calls same internal function as POST /customers]

User: "Update customer 123's email to newemail@example.com"
Service: [Same internal function as PUT /customers/123]
```

**Shared Core Logic:**

```python
class CustomerService:
    # Core business logic (used by both interfaces)

    def find_customers(self, filters: dict) -> list[Customer]:
        # Database query
        pass

    def create_customer(self, data: CustomerData) -> Customer:
        # Validation, database insert
        pass

    def update_customer(self, id: str, updates: dict) -> Customer:
        # Validation, database update
        pass

# REST API Controller
class RESTController:
    def get_customers(self, request):
        filters = parse_query_params(request)
        customers = service.find_customers(filters)
        return jsonify(customers)

    def post_customer(self, request):
        data = parse_json_body(request)
        customer = service.create_customer(data)
        return jsonify(customer), 201

# Conversational Controller
class ConversationalController:
    def handle_message(self, user_input: str):
        # Parse intent with LLM
        intent = llm.parse_intent(user_input)

        if intent.action == "find_customers":
            customers = service.find_customers(intent.params)
            return llm.format_response(customers)

        elif intent.action == "create_customer":
            customer = service.create_customer(intent.params)
            return f"Created customer {customer.name} with ID {customer.id}"
```

**Key Principle:** **One core, two interfaces**

**Advantages:**
- ✅ Support both humans and machines
- ✅ Code reuse (shared business logic)
- ✅ Consistency (same behavior regardless of interface)
- ✅ Incremental adoption (add conversational layer to existing API)

**Disadvantages:**
- ❌ Maintenance overhead (two interfaces to test)
- ❌ Feature parity challenges (one interface may lag)
- ❌ Different error handling (REST: status codes, NL: explanations)

**Best For:**
- Public-facing services (diverse users)
- Internal tools (developers use API, business users use chat)
- Gradual migration (from API-only to hybrid)

**Design Considerations:**

1. **Shared Validation:**
   - Both interfaces use same validation logic
   - Errors formatted differently:
     - REST: `{error: "Invalid email format", code: 400}`
     - NL: "The email address format is invalid. Please use format like user@example.com"

2. **Rate Limiting:**
   - Apply limits fairly across interfaces
   - May have different limits (API: 1000/hour, Chat: 100/hour for humans)

3. **Analytics:**
   - Track which interface is used more
   - Optimize popular flows
   - Deprecate underused interfaces

---

### Pattern 6: Semantic Middleware Layer

**Intent:** Translate between different semantic representations without forcing standardization.

**Problem:**
- Agent A uses ontology X
- Agent B uses ontology Y
- They need to communicate, but ontologies are incompatible

**Traditional Solution:** Force both to use common ontology Z (painful migration)

**Semantic Middleware Solution:** Transparent translation layer

**Architecture:**

```
┌──────────────┐                      ┌──────────────┐
│   Agent A    │                      │   Agent B    │
│ (Ontology X) │                      │ (Ontology Y) │
└──────┬───────┘                      └───────┬──────┘
       │                                      │
       │ Message in X                 Message in Y
       ↓                                      ↓
┌────────────────────────────────────────────────────┐
│            Semantic Middleware                     │
│                                                    │
│  ┌──────────────┐         ┌──────────────┐        │
│  │  Ontology X  │←────────│   Mapping    │        │
│  │  Registry    │  learns │   Engine     │        │
│  └──────────────┘         │  (LLM-based) │        │
│                           └──────────────┘        │
│  ┌──────────────┐                ↑                │
│  │  Ontology Y  │────────────────┘                │
│  │  Registry    │  learns                         │
│  └──────────────┘                                 │
└────────────────────────────────────────────────────┘
```

**Example:**

**Agent A (E-commerce Ontology):**
```json
{
  "product": {
    "sku": "PROD-123",
    "name": "Laptop",
    "price": 999.99,
    "in_stock": true
  }
}
```

**Agent B (Inventory Ontology):**
```json
{
  "item": {
    "item_id": "PROD-123",
    "description": "Laptop",
    "cost": 999.99,
    "availability": "available"
  }
}
```

**Middleware Mapping:**
```json
{
  "product": "item",
  "sku": "item_id",
  "name": "description",
  "price": "cost",
  "in_stock": {
    "target": "availability",
    "transform": {
      "true": "available",
      "false": "out_of_stock"
    }
  }
}
```

**Message Flow:**

```
1. Agent A sends to Agent B:
   {product: {sku: "PROD-123", name: "Laptop", price: 999.99, in_stock: true}}

2. Middleware intercepts, translates:
   {item: {item_id: "PROD-123", description: "Laptop", cost: 999.99, availability: "available"}}

3. Agent B receives message in its native ontology
   (Unaware that translation occurred)

4. Agent B responds:
   {item: {item_id: "PROD-123", availability: "shipped"}}

5. Middleware translates back:
   {product: {sku: "PROD-123", in_stock: false}}  // "shipped" → false

6. Agent A receives response in its native ontology
```

**Mapping Learning (Automatic):**

```
# Agents converse, middleware observes

Agent A: "I have products with SKUs and prices"
Agent B: "I track items with item IDs and costs"

Middleware (LLM):
"Likely mapping: sku ↔ item_id, price ↔ cost"

# Middleware generates tentative mapping, tests with sample messages

Test Message: {product: {sku: "TEST", price: 10}}
Translated: {item: {item_id: "TEST", cost: 10}}

Agent B responds successfully → Mapping confirmed

# Middleware stores mapping for future use
```

**Advantages:**
- ✅ No forced standardization (agents use native ontologies)
- ✅ Transparent to agents (unaware of translation)
- ✅ Automatic learning (LLM infers mappings)
- ✅ Handles evolution (mappings update as ontologies change)

**Disadvantages:**
- ❌ Translation errors (semantic loss)
- ❌ Performance overhead (translation layer)
- ❌ Complex mappings hard to learn (nested structures, conditional logic)
- ❌ Single point of failure (middleware must be highly available)

**Best For:**
- Heterogeneous systems (many ontologies)
- Legacy integration (can't change old systems)
- Cross-domain communication (different conceptual models)

**Research Frontiers:**
- **Bidirectional learning:** Improve mappings from observing successful vs failed interactions
- **Conflict resolution:** What if A's "price" could mean B's "cost" or "retail_price"?
- **Semantic verification:** Prove translations preserve meaning (formal methods)

---

### Pattern 7: Capability-Based Composition

**Intent:** Build complex behaviors by composing agent capabilities dynamically.

**Inspiration:** Unix pipes: `cat file.txt | grep "error" | sort | uniq`

**Vision:** `agent.weather("Boston") | agent.analyze(metric="temperature") | agent.plot(type="line")`

**Architecture:**

```
┌────────────────────────────────────────────────────┐
│          Orchestrator Agent                        │
│  (Understands capabilities, plans compositions)    │
└────────────┬───────────────────────────────────────┘
             │
             │ Plans: weather → analyze → plot
             │
       ┌─────┴──────┬────────────┬─────────────┐
       ↓            ↓            ↓             ↓
┌─────────────┐ ┌────────────┐ ┌──────────┐ ┌────────┐
│  Weather    │ │  Analysis  │ │  Plotting│ │  ...   │
│  Agent      │ │  Agent     │ │  Agent   │ │  Agents│
└─────────────┘ └────────────┘ └──────────┘ └────────┘
```

**Example Task:** "Show me temperature trends in Boston over the past week"

**Orchestrator Planning:**

```
1. Understand intent:
   - Need: Temperature data
   - Location: Boston
   - Time range: Past week
   - Output: Trend visualization

2. Identify required capabilities:
   - get_historical_weather(location, date_range)
   - extract_time_series(data, metric)
   - plot_trend(time_series)

3. Find agents with these capabilities:
   - WeatherAgent has get_historical_weather
   - AnalysisAgent has extract_time_series
   - PlottingAgent has plot_trend

4. Compose execution plan:
   weather_data = WeatherAgent.get_historical_weather("Boston", last_7_days)
   temp_series = AnalysisAgent.extract_time_series(weather_data, "temperature")
   chart = PlottingAgent.plot_trend(temp_series)

5. Execute plan sequentially:
   [Call agents in order, pass outputs as inputs to next]

6. Return final result (chart) to user
```

**Capability Cards:**

**WeatherAgent:**
```json
{
  "capabilities": [
    {
      "name": "get_historical_weather",
      "inputs": {
        "location": "string",
        "date_range": {"start": "date", "end": "date"}
      },
      "outputs": {
        "data": "array<{date: date, temp: number, humidity: number, ...}>"
      }
    }
  ]
}
```

**AnalysisAgent:**
```json
{
  "capabilities": [
    {
      "name": "extract_time_series",
      "inputs": {
        "data": "array<object>",
        "metric": "string (field name to extract)"
      },
      "outputs": {
        "time_series": "array<{timestamp: date, value: number}>"
      }
    }
  ]
}
```

**Key Principles:**

1. **Composability:**
   - Outputs of one capability can be inputs to another
   - Type matching: Output type must match next input type
   - Transformations: Middleware can adapt formats if needed

2. **Discoverability:**
   - Agents advertise capabilities with rich descriptions
   - Orchestrator searches capabilities semantically
   - Example: "temperature data" matches "weather" capabilities

3. **Plannable:**
   - Orchestrator uses planning algorithms (or LLM reasoning)
   - Handles branching: If weather data unavailable, try alternative source
   - Optimization: Choose fastest/cheapest composition

4. **Error Handling:**
   - If any step fails, orchestrator can:
     - Retry with different agent
     - Skip step (if optional)
     - Abort and report error

**Advantages:**
- ✅ Reusability (same agents in different compositions)
- ✅ Flexibility (novel compositions for new tasks)
- ✅ Scalability (add new agents without changing existing ones)
- ✅ Discoverability (agents don't need to know about each other)

**Disadvantages:**
- ❌ Complexity (orchestrator is sophisticated)
- ❌ Latency (sequential composition, network overhead)
- ❌ Debugging (hard to trace failures through pipeline)
- ❌ Type mismatches (outputs may not perfectly match next inputs)

**Best For:**
- Complex, multi-step workflows
- Exploratory tasks (don't know exact steps upfront)
- Heterogeneous agent ecosystems

**Real-World Examples:**
- LangChain's sequential chains
- Microsoft Semantic Kernel's plan execution
- AutoGPT's task decomposition

---

## Design Principles for Hybrid Systems

### Principle 1: Structured Core, Flexible Periphery

**Rule:** Use structure for critical operations, flexibility for discovery and orchestration.

**Rationale:**
- **Core operations** (data storage, financial transactions) need reliability
- **Periphery** (user interfaces, exploration) benefits from flexibility

**Example Architecture:**

```
┌──────────────────────────────────────────────┐
│          Flexible Periphery                  │
│  (NL queries, exploration, discovery)        │
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │       Structured Core                  │ │
│  │  (REST APIs, databases, transactions)  │ │
│  └────────────────────────────────────────┘ │
│                                              │
└──────────────────────────────────────────────┘
```

**Application:**
- Users interact via natural language
- LLM translates to structured API calls
- Core executes reliably
- Results formatted back to NL

---

### Principle 2: Validate Early, Execute Deterministically

**Rule:** Use LLMs for intent understanding, but validate before execution.

**Anti-Pattern:**
```python
user_input = "Transfer $100 to Alice"
llm_response = llm.chat(user_input)  # LLM both understands AND executes
# DANGER: LLM might hallucinate account details, amounts, etc.
```

**Correct Pattern:**
```python
user_input = "Transfer $100 to Alice"
structured_command = llm.parse_to_command(user_input)  # LLM only understands
validated_command = validator.check(structured_command)  # Explicit validation
if validated_command.valid:
    result = executor.execute(validated_command)  # Deterministic execution
else:
    return f"Error: {validated_command.error}"
```

---

### Principle 3: Conversation for Negotiation, Contracts for Execution

**Rule:** Use natural language to establish understanding, then formalize for repeated use.

**Workflow:**
1. First interaction: Conversational (discovery)
2. Establish contract: Formalize agreed-upon interface
3. Subsequent interactions: Use contract (efficient, reliable)
4. Renegotiate: If needs change, return to conversation

**Analogy:** Dating → Marriage
- Dating: Flexible, exploratory
- Marriage: Committed contract
- But can renegotiate (renew vows, divorce)!

---

### Principle 4: Graceful Degradation

**Rule:** If structured communication fails, fall back to conversation.

**Example:**
```
# Normal operation: Use established REST contract
try:
    response = agent_b.get_weather(location="Boston")
except APIError:
    # Fallback: Ask in natural language
    response = agent_b.chat("What's the weather in Boston?")
```

**Rationale:**
- APIs change, become unavailable
- Conversation is more resilient (LLMs can adapt)
- Users shouldn't see failures—system adapts

---

### Principle 5: Observability Across Both Modes

**Rule:** Log and monitor both structured and conversational interactions.

**Metrics to Track:**

| Structured (REST) | Conversational (NL) |
|------------------|---------------------|
| Latency (ms) | End-to-end latency (including LLM) |
| Error rate (%) | Intent parsing accuracy (%) |
| Throughput (req/sec) | Conversations/hour |
| Status codes (404, 500) | Misunderstanding rate (%) |

**Unified Dashboard:**
- Show both side-by-side
- Alert on anomalies in either
- Trace requests across both layers

---

## Decision Framework: Choosing the Right Pattern

### Decision Tree

```
START: What type of system are you building?

├─ Exposing existing APIs to humans/LLMs
│  └─> Pattern 1: NL2API Gateway
│
├─ Critical operations (financial, medical)
│  └─> Pattern 2: Natural Language Command Interfaces (NLCI)
│
├─ Long-term agent partnerships
│  └─> Pattern 3: Progressive Disclosure
│
├─ Large multi-agent ecosystem
│  └─> Pattern 4: Semantic API Discovery
│
├─ Serving both humans and machines
│  └─> Pattern 5: Dual-Mode Interfaces
│
├─ Integrating heterogeneous systems
│  └─> Pattern 6: Semantic Middleware Layer
│
└─ Complex, multi-step workflows
   └─> Pattern 7: Capability-Based Composition
```

### Compatibility Matrix

Patterns can be combined:

| Pattern | Compatible With | Incompatible With |
|---------|----------------|------------------|
| NL2API | 2, 4, 5, 7 | 6 (different abstraction levels) |
| NLCI | 1, 3, 5 | 7 (different execution models) |
| Progressive Disclosure | All | None |
| Semantic Discovery | 1, 5, 7 | 6 (different discovery mechanisms) |
| Dual-Mode | All | None |
| Semantic Middleware | 4, 7 | 1, 2 (different translation approaches) |
| Capability Composition | 1, 4, 5, 6 | 2 (different control flows) |

---

## Implementation Roadmap

### Phase 1: Foundation (Months 1-3)

1. **Choose core pattern** based on decision tree
2. **Implement structured layer** (REST APIs, databases)
3. **Add basic conversational interface** (simple intent parsing)
4. **Validate with test users**

**Deliverable:** Working prototype with both structured and conversational access

---

### Phase 2: Reliability (Months 4-6)

1. **Add validation layer** (schema checks, business logic)
2. **Implement error handling** (retries, fallbacks)
3. **Set up monitoring** (track both structured and conversational metrics)
4. **Load testing** (ensure performance under load)

**Deliverable:** Production-ready system with reliability guarantees

---

### Phase 3: Scalability (Months 7-9)

1. **Add agent discovery** (registry, semantic search)
2. **Implement capability composition** (orchestrator)
3. **Optimize performance** (caching, async processing)
4. **Multi-agent testing** (stress test with many agents)

**Deliverable:** Scalable multi-agent system

---

### Phase 4: Intelligence (Months 10-12)

1. **Add ontology learning** (automatic schema alignment)
2. **Implement semantic middleware** (transparent translation)
3. **Progressive disclosure** (conversation → contract transitions)
4. **Self-improvement** (learn from interactions)

**Deliverable:** Intelligent system that adapts and evolves

---

## Open Research Questions

1. **How to formalize hybrid architectures?**
   - Need formal models that capture both structured and conversational modes
   - Verification techniques for hybrid systems

2. **What is the optimal division between structured and conversational?**
   - Task-dependent? Domain-dependent?
   - Can we learn this automatically?

3. **How to test hybrid systems?**
   - Unit tests work for structured layer
   - What about conversational layer?
   - Integration testing across both?

4. **Can we prove correctness of conversational interfaces?**
   - Formal verification techniques
   - Temporal logic for conversation sequences?

5. **How to version hybrid systems?**
   - REST API versioning is well-understood
   - How to version conversational contracts?

---

## Summary

**Key Insight:** The future is not "structured OR conversational"—it's **both**, composed strategically.

**Proven Patterns:**
1. **NL2API:** Natural language gateway to existing APIs
2. **NLCI:** Structured commands from natural language
3. **Progressive Disclosure:** Conversation → Contract evolution
4. **Semantic Discovery:** Agent cards + registry
5. **Dual-Mode:** Both interfaces for same service
6. **Semantic Middleware:** Transparent ontology translation
7. **Capability Composition:** Dynamic workflow construction

**Design Principles:**
- Structured core, flexible periphery
- Validate early, execute deterministically
- Conversation for negotiation, contracts for execution
- Graceful degradation
- Unified observability

**Path Forward:**
Start with simple hybrids (NL2API, Dual-Mode), gradually add intelligence (Discovery, Composition, Middleware) as needs grow.

The goal: Software that can **converse like humans** but **execute like machines**.

---

**Last Updated:** 2025-11-05
**Related Documents:**
- `../reliability-tradeoffs/conversational-vs-rest-analysis.md`
- `../ontologies/ontology-learning-from-dialogue.md`
- `../concepts/agent-communication-protocols.md`
