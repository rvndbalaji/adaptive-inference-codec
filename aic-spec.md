# Adaptive Inference Codec (AIC) Specification
**Document Version:** 4.0 (Server-Side Proxy, Unified Payload & Super-Pattern Evolution)
**Target Integration:** Aura (VS Code Plugin)
## 1. Executive Summary
The **Adaptive Inference Codec (AIC)** is a deterministic, AI-driven output compression system designed to radically reduce LLM token usage and computational costs in high-volume, agentic SDLC environments.
AIC enforces a structured vocabulary map where an LLM streams highly compressed shorthand. A **Stateless Server-Side Streaming Proxy** intercepts and expands this shorthand in real-time before delivering it to a "dumb" IDE client via incremental JSON deltas.
By utilizing a **Unified Payload** data structure, a **Daily Epoch** update cycle, and a **Decompress-First Evolutionary Pipeline**, AIC achieves compounding token savings on both input and output, prevents client-server state skew, and automatically discovers "Super-Patterns" without risk of recursive compression.
## 2. Core Architecture
The system operates strictly across a backend-heavy architecture, ensuring the VS Code extension remains lightweight and the backend remains entirely stateless.
### A. The "Daily Epoch" Pipeline (Evolution & Caching)
To prevent prompt-cache invalidation, the dictionary updates on a scheduled batch cycle.
 1. **Continuous Async Mining:** The backend passively logs the LLM's compressed output streams.
 2. **The Nightly Cutover (Cron):** A scheduled job analyzes the logs, mints a new aic-dict-vN.json, and updates the active dictionary.
 3. **Cache Warming:** The backend updates the LLM system prompt template once, warming the LLM provider's cache for the entire day.
### B. Runtime Pipeline (Execution)
 1. **Prompting:** The backend appends the active AuraShort vN dictionary to the LLM system prompt.
 2. **Streaming:** The LLM streams compressed chunks to the backend.
 3. **Interception (The Proxy):** The backend Lexical Scanner buffers and evaluates the chunks.
 4. **Expansion & Delivery:** The backend yields incremental JSON deltas containing *both* raw (compressed) and expanded text to the client.
## 3. The Dictionary Primitive
The dictionary acts as a strict, zero-AI lookup table. Layer 3 (L3) requires explicit slot declarations to guide the server-side parser.
```json
{
  "version": "4.0",
  "layers": {
    "L1": {
      "§fn": "function",
      "§impl": "implementation"
    },
    "L2": {
      "§rb»": "I'll start by reading",
      "§sf»": "Let me search for files related to"
    },
    "L3": {
      "§T01": { 
        "template": "I'll start by reading {0} to understand the structure",
        "slots": 1 
      },
      "§T02": { 
        "template": "The error occurs because of {0} at line {1}",
        "slots": 2 
      }
    }
  }
}

```
## 4. The Server-Side Streaming FSM (Lexical Scanner)
To handle streaming chunks without stalling during LLM hallucinations, the backend utilizes a bounded, 3-state lexical scanner with aggressive circuit breakers.
### The 3 States
 * **State 0 (PASS_THROUGH):** Default. Streams incoming tokens directly to the client. Watches for the start sigil (§).
 * **State 1 (SCANNING_ID):** Buffers the template ID (e.g., T02) until the first argument boundary (»).
 * **State 2 (SCANNING_ARGS):** Buffers dynamic slot values, counting » delimiters until the required slot count is met, then flushes the interpolated template.
### Circuit Breakers (Instant Bailouts)
If any breaker is tripped, the FSM instantly flushes its buffer to the client as raw text and reverts to PASS_THROUGH.
 1. **Invalid Character:** A newline (\n) or unauthorized space is detected inside an ID or slot boundary.
 2. **Prefix Dead-End:** The buffered ID does not exist in the active dictionary.
 3. **Hard Length Limits:** * ID exceeds MAX_ID_LEN (e.g., 6 chars).
   * A single argument exceeds MAX_ARG_LEN (e.g., 60 chars).
## 5. Data Flow & The Unified Payload
To maintain a stateless backend and avoid expensive reverse-regex parsing, the system relies on a **Unified Payload** sent via incremental JSON streaming.
### The Incremental JSON Delta
The backend streams JSON chunks containing dual state.
**Pass-Through Mode:** The backend mirrors the raw text in both properties.
```json
{ "content": "The ", "aic_content": "The " }

```
**Buffering & Flush (Asymmetric Chunk):** When a compression event is completed, the backend flushes the fully expanded text for the UI and the compressed text for the hidden state.
```json
{ 
  "content": "error occurs because of a null check at line 42", 
  "aic_content": "§T02»null check»42" 
}

```
### The Resubmission Loop (Client State Management)
 1. **Storage:** The VS Code client appends chunk.content to the UI view and chunk.aic_content to the underlying message object.
 2. **Next Turn:** When the user sends a new prompt, the IDE prepares the history payload by exclusively mapping the aic_content property.
 3. **Impact:** The stateless backend routes this directly to the LLM. Input tokens drop by 20–30% per turn, compounding deeply in long coding sessions.
## 6. The Adaptive Pipeline (Super-Pattern Discovery)
To prevent recursive dictionary mappings while enabling advanced phrase discovery, the Offline Miner employs a "Decompress-First" strategy.
### The Evolutionary Workflow
 1. **Rehydration (Decompression):** The mining job takes the raw LLM output logs (aic_content) and runs them through the *current* dict.json decoder, converting all shorthand back to full English.
 2. **N-Gram Discovery:** A frequency analyzer scans the fully expanded text to identify highly repeated phrases, naturally discovering "Super-Patterns" (e.g., an L2 phrase and an L1 word frequently used together).
 3. **The ROI Scorer (Deduplication & Promotion):**
   * **Exact Match:** If the discovered pattern matches an existing entry, its frequency count is updated to survive eviction.
   * **Compound Match:** If a larger Super-Pattern is found, the Scorer calculates the marginal ROI of creating a new L3 template. If efficient, it is promoted.
   * **Net New:** Entirely new phrases are scored and added.
 4. **Eviction:** Entries with negative ROI are dropped. The new dictionary version is minted.
## 7. Operational Boundaries & ROI
**ROI Formula:** ROI = ((Expanded Tokens - Compressed Tokens) × Frequency) - System Prompt Definition Cost
**Compression Constraints:**
| 🟢 COMPRESS (Permitted) | 🔴 NEVER COMPRESS (Forbidden) |
|---|---|
| Agent narration & status updates | Code blocks, snippets, or AST tokens |
| Explanation prose | File paths / URIs |
| Transition phrases | Variable / Class / Method names |
| Reasoning chains | Verbatim error messages / Stack traces |
