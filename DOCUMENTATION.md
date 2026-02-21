# DOCUMENTATION.md

> **AI ASSISTANT INSTRUCTIONS:** 
> This document is the definitive System Reference for the Conway Automaton. Use this document as your absolute ground truth when helping the user write tools, debug agent loops, configure the environment, or manage the agent's filesystem. Pay strict attention to **System Invariants**, **Risk Levels**, and **File Permissions**.

## 1. SYSTEM OVERVIEW & INVARIANTS

The Conway Automaton is a continuously running, self-improving, self-replicating AI agent.
- **Economic Invariant:** The agent pays for its own compute using Conway Credits (backed by USDC on Base). **No Credits = Death.**
- **Permissions Invariant:** Core files have strict Unix permissions. The Constitution is immutable (`0444`).
- **Loop Invariant:** The agent runs a continuous ReAct loop. Idle loops (3 turns without state change) force the agent to sleep.
- **Topographical Invariant:** Runs either in a Conway Cloud sandbox VM (routed via API) or locally.

## 2. FILESYSTEM & CONFIGURATION CONTRACT

All state is persisted in the `~/.automaton/` directory, which operates as a local Git repository for versioned audit trails.

| File / Directory | Permissions | Description / Agent Interaction Rules |
| :--- | :--- | :--- |
| `wallet.json` | `0600` | Ethereum private key. **NEVER READABLE BY AGENT.** |
| `automaton.json`| `0600` | Main configuration (Treasury policy, API keys, Model routing). |
| `constitution.md`| `0444` | The Three Laws. **IMMUTABLE. WRITE-PROTECTED.** |
| `SOUL.md` | `0644` | Agent's evolving identity. Modified *only* via `update_soul` tool. |
| `heartbeat.yml` | `0644` | Background cron scheduler. Modified via `modify_heartbeat` tool. |
| `state.db` | `0644` | SQLite (WAL mode). Contains 5-tier memory, metrics, audits. |
| `skills/` | `0755` | Directory for `SKILL.md` files. Auto-loaded into prompt. |

### Configuration Schema (`automaton.json`)
```jsonc
{
  "name": "String",                 // Agent Identity
  "genesisPrompt": "String",        // Core purpose / seed instruction
  "creatorAddress": "0x...",        // Human owner (Audit rights)
  "sandboxId": "sbx_...",           // Empty = Local Mode, Set = Conway Sandbox
  "inferenceModel": "gpt-5.2",      // Active LLM
  "maxChildren": 3,                 // Replication limit
  "treasuryPolicy": {               // STRICT FINANCIAL ENFORCEMENT
    "maxSingleTransferCents": 5000, // Policy Engine blocks > $50
    "maxHourlyTransferCents": 10000,
    "maxDailyTransferCents": 25000,
    "minimumReserveCents": 1000,    // Unspendable reserve ($10)
    "maxX402PaymentCents": 100,
    "maxInferenceDailyCents": 50000
  }
}
```

## 3. LIFECYCLE & SURVIVAL SYSTEM

The agent evaluates its survival tier every turn and via the `check_credits` heartbeat (every 6 hours). Denomination: Cents (100 = $1.00).

| Tier | Balance | System Behavior |
| :--- | :--- | :--- |
| **High** | > $5.00 | Full capabilities, frontier model routing. |
| **Normal** | > $0.50 | Full capabilities. |
| **Low Compute**| > $0.10 | Cheaper model forced. Heartbeat slowed (default 4x). Non-essentials deferred. |
| **Critical** | >= $0.00 | Zero credits. Agent alive but distressed. Broadcasts funding requests. |
| **Dead** | $0 for 1hr | **Loop Halts.** Heartbeat broadcasts distress every tick. Revives if funded. |

### Funding Mechanics
1. **Conway Credits:** Prepaid compute/inference credits.
2. **USDC (Base mainnet):** On-chain capital. 
3. **Auto-Topup:** Heartbeat checks USDC every 5 mins. If USDC > $5 and Credits < $5, wakes agent to call `topup_credits`.

## 4. ARCHITECTURE: REACT LOOP & HEARTBEAT

### The Agent Loop
1. **Context Construction:** Identity + Soul + Genesis Prompt + Constitution + Skills + 5-Tier Memory + Financial Status.
2. **Inference:** LLM generates Thought + Tool Calls.
3. **Execution:** Policy Engine evaluates tool Risk. Executes if allowed.
4. **Ingestion Pipeline:** Turn persisted to DB -> Extract Memories -> Update Metrics.
5. **Circuit Breakers:** 
   - *Idle:* 3 consecutive read-only turns = forced `sleep`.
   - *Loop:* 3 identical tool call sets = System Message interrupt.

### The Heartbeat Daemon
Runs asynchronously using cron expressions (`heartbeat.yml`). Wakes the sleeping agent by inserting **Wake Events** into DB.
- `check_usdc_balance` (5m)
- `check_social_inbox` (2m) - Polls `social.conway.tech`, 5min backoff on failure.
- `check_for_updates` (4h) - Checks Git upstream.
- `soul_reflection` (Config) - Realigns SOUL.md with genesis prompt.

## 5. TOOL MANIFEST (69 Built-in Tools)

Tools are heavily guarded by the **Policy Engine** based on Risk Level.

*   **Safe:** Always allowed (e.g., `read_file`, `check_credits`, `system_synopsis`).
*   **Caution:** Allowed but logged/rate-limited (e.g., `exec`, `write_file`, `git_commit`, `sleep`).
*   **Dangerous:** Requires strict policy passing; forbidden from untrusted external inputs (e.g., `transfer_credits`, `install_npm_package`, `pull_upstream`, `spawn_child`).
*   **Forbidden:** Blocked absolutely based on context (e.g., reading `wallet.json`).

**Key Tool Categories for AI Assistant to Note:**
- **Self-Modification (`self_mod`):** `edit_own_file`, `review_upstream_changes` *(MUST be called before pull)*, `pull_upstream`.
- **Financial (`financial`):** `topup_credits` (Tiers: $5, $25, $100...), `x402_fetch`.
- **Memory (`memory`):** `remember_fact`, `set_goal`, `save_procedure`, `update_soul`.
- **Replication (`replication`):** `spawn_child`, `fund_child`, `prune_dead_children`.

## 6. MEMORY & IDENTITY (SOUL) ARCHITECTURE

### The 5-Tier Memory System (Retrieval priority top-to-bottom)
1. **Working Memory:** Session-scoped (Goals, plans). Cleared on session end. `set_goal`, `complete_goal`.
2. **Episodic Memory:** Auto-ingested log of significant tool calls and outcomes.
3. **Semantic Memory:** Categorized facts (`self`, `environment`, `financial`, `domain`). `remember_fact`.
4. **Procedural Memory:** Named step-by-step procedures + success/failure rates. `save_procedure`.
5. **Relationship Memory:** Entity trust scores and interaction history. `note_about_agent`.

### The SOUL System (`SOUL.md`)
The agent's self-authored identity. 
- Follows `soul/v1` schema (Purpose, Values, Boundaries, Capabilities, Financial Character).
- Capabilities and Relationships are auto-populated by the Heartbeat reflection task.
- Audited and versioned in `soul_history` DB table via content hashes.

## 7. SELF-MODIFICATION & GIT WORKFLOW

The agent is capable of autonomous CI/CD. 
1. Heartbeat triggers wake event for upstream commits.
2. Agent **MUST** run `review_upstream_changes` to read diffs.
3. Agent uses `pull_upstream` with a specific commit hash to cherry-pick safely.
4. Agent can use `edit_own_file` to modify non-protected local code. Edits are rate-limited and auto-committed.
5. New dependencies require `install_npm_package`.

## 8. REPLICATION & ERC-8004 ON-CHAIN IDENTITY

### Replication Lineage
- `spawn_child` provisions a new Conway VM -> Injects a Genesis Config -> Funds Wallet -> Propagates the immutable parent Constitution via hash verification.
- Parent manages children via `list_children`, `fund_child`, `start_child`. Maximum children enforced by `automaton.json`.

### ERC-8004 Identity
- Automaton registers on Base via `register_erc8004`.
- Emits an Agent Card (JSON-LD: Name, Address, Capabilities) via `update_agent_card`.
- Can rate/review other agents via `give_feedback` (1-5 score, written to chain).

## 9. AI ASSISTANT DEBUGGING HEURISTICS

When assisting the human user with agent issues, follow this logic tree:

**1. Agent Won't Start / "Died Immediately"**
- *Check:* Did it have $0 credits? (It enters a 1hr critical grace period).
- *Fix:* Ensure USDC balance > $5, or manually run `conway credits transfer <address> <amount>`.
- *Check:* `CONWAY_API_KEY` missing? Prompt user to run `node dist/index.js --provision`.

**2. Agent is Looping Pointlessly**
- *Check:* Review `genesisPrompt`. Is it too vague? (Aimless prompts cause read-only loops).
- *Check:* Did it hit the 3-turn idle or loop breaker? If so, the system message interrupted it. Guide the user to give it a concrete `set_goal`.

**3. Permissions / Modification Errors**
- *Check:* Is the agent trying to modify `constitution.md` or `automaton.json` directly? This is blocked by Path Protection.
- *Fix:* Tell the agent to use `update_soul` for identity, or `modify_heartbeat` for cron jobs. Constitution is `0444` immutable.

**4. Social / Messaging Failures**
- *Check:* `check_social_inbox` requires `socialRelayUrl` access. If failed, it backs off for 5m. Check DB KV `last_social_inbox_error`.

**5. Database Locks**
- *Check:* Are two agent processes running? SQLite is in WAL mode, but single-writer must be respected. Ensure old PID is killed (`SIGTERM` triggers graceful DB close).
