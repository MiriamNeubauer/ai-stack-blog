# The AI Stack Is on Fire. Here's What Actually Matters.

*May 2026 — synthesised from 19 LinkedIn signals*

---

Fall in love with the problem. Not the model. Not the benchmark. The problem.

And right now the problem is this: **we are building on top of infrastructure that is fundamentally miscalibrated.** Memory is the bottleneck. Identity is missing. The tooling is a mess. Security is an afterthought. The trust layer doesn't exist. And the cost curve is about to break in ways most teams haven't modelled.

Nineteen LinkedIn posts this week pointed at the same iceberg from different angles. Here's the synthesis — with the numbers that actually matter.

---

## Problem 1: You're Paying to Forget

**The KV cache is the bill. Everything else is noise.**

Here's the Fermi estimate nobody talks about at demos:

- 70B model, 80 layers, 64 heads, 128-dim, FP16
- Each token = **320 KB per layer**
- 128K context = **40 GB of KV cache**
- Model weights at 4-bit = 35 GB
- **The cache is bigger than the model**

You're spending more GPU memory on *remembering the conversation* than on *running the intelligence*. That's the wrong ratio. That's the problem.

The serving stack has converged on five attacks against it: **GQA** (Llama 3, 8x reduction, zero quality loss), **MLA** (DeepSeek, another 4x on top), **PagedAttention** (vLLM, fragmentation drops from 50% to under 5%), **prompt caching** (one system prompt, 1,000 users, compute prefix once), and **INT8 KV quantisation** (5x concurrent users per H100). DeepSeek's latest hybrid attention hits **7% of original cache size** at **$0.14/million tokens** — 100x cheaper than comparable APIs.

And LLM inference is fundamentally memory-bound, not compute-bound. When you generate a token, you're loading the entire model weight to produce it. The compute costs almost nothing once the weights are loaded. Every optimisation that matters is about reducing how much you move, not how much you calculate.

**First principles:** don't pay to recompute what hasn't changed. Cache once, reuse forever.

**What to test:** put static content (system prompt, tools, persona) first. Dynamic content last. Log your prompt/completion token ratio. If prompts >50% of spend, turn on prefix caching today and measure the delta.

---

## Problem 2: Your LLM Is Slow Because You Don't Understand Where the Bottleneck Is

**Most people blame the model. The bottleneck is the system around it.**

A complete map of LLM inference optimisation techniques — and the numbers that justify each:

- **Quantisation**: 4-bit or 8-bit weights instead of 32-bit → 4–8x less memory, 2–4x higher token generation
- **Speculative decoding**: small draft model predicts K tokens, large model validates → 2–3x faster generation
- **KV cache** (see above): 10–100x faster token generation vs. naive recompute
- **PagedAttention**: dynamic block allocation → 2–4x higher throughput
- **Flash Attention**: tiles data into SRAM instead of HBM → 2–4x faster compute, room for much longer context
- **Continuous batching**: pack sequences back-to-back, slot new requests when one finishes → 10–20x higher throughput
- **Model pruning**: remove unnecessary connections → 25% size reduction, 1.3–3x faster generation

These stack **multiplicatively on throughput** but **additively on engineering cost**. The real skill is knowing which three to combine for your workload — not chasing all seven.

One underappreciated addition: **consistent-hash routing** in front of your LLM gateway for prefix caching. Without it, thundering herds land on different replicas and everyone pays the recompute. One team dropped p99 latency by a third with this single change.

**Fermi check:** if you're running 1,000 requests/day on a 70B model without any of these optimisations, you're likely paying 5–20x more than necessary. That's not a rounding error. That's a strategic liability.

**What to test:** instrument each layer of your stack separately. Identify where your actual bottleneck is. Most teams haven't done this and are optimising the wrong thing.

---

## Problem 3: We're Stacking Layers When We Should Be Looping

**50% more parameter efficient. Same quality. The architecture insight is brutally simple.**

**Hyperloop Transformer** ([arXiv:2504.21254](https://arxiv.org/abs/2604.21254)): three blocks — begin, middle, end. The middle block runs recurrently. Augmented with DeepSeek's Manifold-Constrained Hyper-Connections. Result: same benchmark performance at half the parameter count.

And Claude Code's architecture makes the same point from a different angle. Pop the hood of the dominant CLI code agent and you find: **a while loop.**

```
while(tool_call) → reason → act → observe → repeat
```

No DAG. No classifier. No RAG pipeline. 5 modules: agent loop, tools (Read/Write/Bash/Search/Web), context (auto-compaction at ~85% window), safety (18 lifecycle hooks), extensions (MCP, sub-agents). That's it.

Compare with Manus (CodeAct, Python in cloud sandbox, +20% success rate vs function calling) and Codex (kernel sandboxing via Seatbelt/seccomp). All three architectures are different. But the pattern that keeps emerging: **the simpler the loop, the more room the model has to reason.**

**First principles:** complexity is drag, not capability. Every framework layer you add is a place the model can't think. Keep the loop tight. Let the model do the work.

**What to test:** strip your agent to its minimal viable loop. Scheduler + LLM call + filesystem operations. Is it more reliable than your current stack? Almost certainly yes.

---

## Problem 4: Your Production Data Is Worth More Than Any Open-Source Dataset

**172,000 training samples. 35 systems. 8 domains. One Mac Studio. Zero API costs.**

A DevOps engineer fine-tuned Qwen2.5-7B using LoRA — 40 million trainable parameters out of 7.6 billion. The training data: 264 real production incidents (symptoms, root cause, fix, prevention) plus 171,634 curated security samples covering prompt injection patterns, MITRE ATT&CK mappings, and vulnerable-to-fixed code pairs.

Result: a local LLM that answers incident response calls, security analyses, and network diagnoses. For zero API cost per request. On Apple Silicon. In an office.

```
You say: "Service not responding, logs show D-State"
It answers: "vzdump without I/O limit causes kernel cascade.
Here are the 4 fix steps and how to prevent it on the next backup."
```

The key insight: **confidence, not accuracy, is the right metric.** After training, every inference gets a confidence score + source cluster. If confidence drops below 0.75 on new incident types, that's a retrain signal. That's how you go from interesting experiment to production grade.

**Fermi estimate of the data advantage:** your company has 3 AM SSH logs that no foundation model has ever seen. Every production incident is a training sample. If you have 100+ incidents per year and you're not fine-tuning on them, you're donating your competitive edge to the public model.

**What to test:** document your next 10 production incidents in the format: symptoms, root cause, fix, prevention. That's the beginning of a fine-tuning dataset. LoRA on a 7B model on Mac Studio hardware is a $2–3K experiment, not a $200K data center project.

---

## Problem 5: You Built an Insider Threat and Called It an AI Agent

**Single service account. Admin credentials. No audit log. No kill switch. That's not an AI agent. That's an insider threat you built on purpose.**

82% of 10,000+ MCP server repos interact with sensitive APIs. 34% are prone to command injection. Most enterprise agents hit Simon Willison's lethal trifecta on day one:
1. Access to private data
2. Exposure to untrusted input
3. Ability to take actions

The 7-point agent security audit every team needs to run before production:

1. **Identity:** does every agent have scoped, least-privilege access? Can you tell whether an API call came from a human or an agent?
2. **Memory:** can you audit what the agent remembers and who put it there? Poisoned memory is the new poisoned training data — except it's happening in production, in real time, no retraining required.
3. **Tools:** 82% of MCP server repos interact with sensitive APIs. 34% are prone to command injection. Assume your tool audit has an expiration date. An attacker doesn't need to compromise your tool — they just need malicious instructions in any content the agent reads. A poisoned PDF. A crafted ticket. A calendar invite.
4. **Blast radius:** if this agent goes rogue, what's the worst case? A compromised orchestrator can instruct sub-agents to take actions they'd never accept from a human. They trust whoever called them.
5. **Human checkpoints:** where are the approval gates on irreversible actions?
6. **Output validation:** treat agent output as untrusted until validated. Are outputs scanned for PII, credentials, or embedded instructions?
7. **Observability:** agents drift. Memory accumulates context you didn't put there. Can you detect anomalous tool call patterns? Can you replay what an agent did during an incident?

**Red flags heard every week:** "We trust the model" (not a security strategy). "Our agents only access internal data" (that's what attackers want). "We'll add security later" (you won't).

**Fermi estimate of your blast radius:** how many systems does your most privileged agent have HTTP access to? Multiply by the number of data records accessible. That's your breach surface. Has anyone calculated it?

**What to run now:** AAuth ([aauth.dev](https://www.aauth.dev)) is an emerging protocol from the author of OAuth 2.0, built specifically for agent authorisation: cryptographic identity per agent, no pre-registration, no bearer tokens. The [AAuth Knowledge Graph](https://mcp-shark.github.io/aauth-explorer/) maps 72 flows across signing, access, mission, and advanced layers. Start there.

---

## Problem 6: The Trust Layer Has No Trust

**The gatekeeper never verified itself.**

Sumsub: KYC giant, 4,000+ crypto exchange and fintech clients, Gartner Magic Quadrant Leader, WEF Unicorn member, INTERPOL partner. Used by Bybit, Vodafone, Duolingo. Millions of biometric scans and government IDs.

Now: opaque ownership (beneficial owner of controlling Cyprus entity not publicly disclosed), an unnamed Series B investor described only as "a corporate VC fund", and an 18-month undetected breach (July 2024 – January 2026).

And the AI doing the passport OCR? Smart Engines — a Russian science-driven AI company simultaneously used by major Russian banks and public sector.

**The structural problem:** every exchange embedded Sumsub because they couldn't afford to trust strangers. None of them applied the same standard to the tool doing the trusting.

This is the supply chain problem in its purest form. You can't trust the output of a system if you haven't verified the system. KYC your KYC vendor. KYB your KYB provider. The trust chain is only as strong as its most opaque link.

**Fermi check on your own stack:** how many third-party SaaS tools does your product rely on? How many have you run through a KYB process? For most teams: zero. Every unverified vendor in your chain is a Sumsub waiting to happen.

---

## Problem 7: The Model That Just Replaced Every Bespoke ML Pipeline

**24 billion banking events. 207 billion tokens. One model. Every downstream task, beaten.**

Revolut published PRAGMA — not an LLM, not a chatbot. A foundation model for financial behavior, co-authored with NVIDIA. The results:

- Credit scoring: **+130%**
- Fraud recall: **+65%**
- Marketing engagement prediction: **+79%**
- Product recommendation: **+40%**
- AML: **-47%** (honest result, and here's why it matters)

The key architectural insight: financial data has a structure that text tokenization destroys. Dollar amounts become digit fragments. Magnitude disappears. Revolut built a **key-value-time tokenization scheme** with two encoder branches (static profile + event sequences) fused by a history encoder. They preserved the semantics that LLMs would have destroyed.

The AML failure is as important as the wins. AML is a *network* problem, not a *you* problem. What matters is who you transact with, not your individual event history. PRAGMA focuses on individual user behavior. Revolut's own exec confirmed: a graph neural network layer is next. Then fusion.

**First principles:** every industry has a behavioral language. Text. Code. Financial transactions. DNA sequences. Medical events. The teams that build domain-specific foundation models — not fine-tuned LLMs, but models designed from the ground up for the data structure of that domain — will replace entire generations of bespoke ML pipelines.

**Fermi estimate of the moat:** if PRAGMA replaces 6 bespoke ML pipelines and each pipeline cost $500K–2M to build and maintain annually, that's $3–12M/year in saved engineering for Revolut, plus compounding advantage on every new use case. The bank with the best foundation model gets a structural edge across every decision it makes.

**What to consider:** do you have domain-specific behavioral data at scale? If yes, a transformer built for your data structure — not a fine-tuned GPT — may be the right architecture. [The PRAGMA paper](https://arxiv.org/html/2604.08649v1) is worth reading.

---

## Problem 8: Knowledge Work Has No GitHub

**Coding with AI is solved because all context is in the git repo. Knowledge work is unsolved because context is everywhere.**

Engineering has been ready for AI for a decade: deterministic code, centralised versioning, tools built by engineers for engineers. For the rest of white-collar work? The context problem is brutal.

Knowledge is: **distributed** (transcripts in Granola, documents in Notion, customer data in HubSpot), **unstructured** (no schema, no version control, no canonical source), and **unverifiable** (code either works or it doesn't; a good proposal is subjective).

The company that builds the enterprise "brain" — an ingestion engine that connects disparate sources, auto-updates based on data shelf-life, self-organises into a coherent schema, and continuously improves based on user feedback — will unlock AI automation for the 80% of knowledge work that hasn't moved yet.

The hard part isn't ingestion. It's **compaction and cleaning at scale**. As the corpus grows, the needle-in-haystack problem gets worse, not better. The brain has to be as good at forgetting the irrelevant as it is at remembering the useful.

YC's latest RFS lists this as #5 on their priority list. The whitespace is visible and closing fast.

**Fermi estimate of the market:** global knowledge worker headcount ~1 billion. Average fully-loaded cost ~$80K/year. If the enterprise brain automates 20% of knowledge work, that's **$16 trillion of labor value** unlocked. Someone will capture a fraction of that. That fraction will be enormous.

**What to build toward:** the interface that wins isn't a better search or a better chatbot. It's a system that can tell you what you need to know before you know you need it, grounded in your company's actual context, not a generic model's priors.

---

## Problem 9: Preventive Health Has the Same Problem as Enterprise Knowledge

**More inputs than ever. Not enough clarity. More products. No trusted layer above them all.**

Sleep from Oura. Strain from Whoop. Blood work from Levels. Medical records from your GP. Behaviour from your phone. Doctor conversations that never make it into any system. Every signal is siloed. No layer connects them.

Apple Health and Google Health aggregate the data. Ada Health, K Health, Humanity, Superpower try to interpret it. Most still sit on one part of the stack. No clear winner exists for the layer that sits *above* all of them and explains your health back to you.

The structural barrier: this is not a data problem. It's a **trust problem, an insight problem, a guidance problem, and a practicality problem**, all at once. Governments tried to build this for years and failed. The difference now: consumer health and medical data are finally converging. AI can actually do something with the combined signal. But the gap between capability and trust is where this layer gets valuable — or fails.

**The Revolut parallel is exact.** Revolut proved that financial behavior has transferable semantics across credit, fraud, and marketing. The preventive health equivalent: physiological behavior has transferable semantics across sleep quality, metabolic health, cardiovascular risk, and cognitive performance. The foundation model for human health hasn't been built yet.

**Fermi estimate of the opportunity:** global preventive health market ~$400B. Current fragmentation means the average person uses 3–5 health apps that don't talk to each other. The platform that connects them and provides guidance — not just reporting — captures a winner-take-most dynamic in a market where no incumbent has meaningful lock-in.

---

## The Complete Supply Chain Map

| Layer | The Problem | The Number | The Bet |
|---|---|---|---|
| **Hardware** | Edge agents too expensive | Pi Zero + API = €20, 0.5W | Cost floor is gone |
| **Memory** | KV cache bigger than model weights | 40 GB cache vs 35 GB weights at 128K | Compression ratios, not context specs |
| **Inference** | Memory-bound, not compute-bound | 10–20x throughput headroom untapped | Stack 3 optimisations for your workload |
| **Architecture** | Parameter count is not quality | 50% fewer params, same benchmark | Depth reuse + simple loops beat complexity |
| **Fine-tuning** | Generic models vs. production data | 264 incidents > 171K public samples | Your data is the moat |
| **Multimodal** | Natural language can't point | 25 hrs/week lost to fuzzy references | Coordinates in the reasoning chain |
| **Agent Tooling** | 40 MCP schemas, 200K tokens/day overhead | ~€600/month just in tool definitions | Filesystem > MCP sprawl |
| **Agent Security** | Insider threat by design | 82% of MCP repos touch sensitive APIs | AAuth + 7-point audit before prod |
| **Trust Infrastructure** | KYC vendors aren't KYC'd | 18-month undetected breach at KYC giant | Verify your verifiers |
| **Domain AI** | Bespoke ML pipelines vs foundation models | +130% credit scoring, +65% fraud recall | Domain-native tokenization wins |
| **Knowledge Context** | No GitHub for knowledge work | $16T labor value locked in unstructured data | Enterprise brain = next platform |
| **Health Intelligence** | Health data is fragmented and siloed | $400B market, no trusted aggregation layer | Foundation model for human health |
| **Security** | PQC adds 20–30% handshake RTT | DNS TCP fallback = global infra problem | Measure now, not when mandated |
| **Policy** | EU deploying instead of regulating | €70B/year at 1% industrial efficiency | Auditable, sovereign AI wins Europe |

---

## The Three Questions Worth Asking

**1. What is the actual cost of the problem, not the solution?**
Most teams optimise for slightly cheaper inference. The real question: what is it costing you to *not* have structured agent context, *not* have prefix caching, *not* have a security audit, *not* have a production fine-tuning dataset? Quantify it. Be uncomfortable with the number.

**2. Can you survive on the minimal viable stack?**
Claude Code is a while loop. A fine-tuned 7B model on a Mac Studio beats your RAG pipeline. A €20 Pi Zero handles most agentic workflows. Can you explain why you need 10x more complexity? If not, the complexity is the bug.

**3. Are you building for the world that's coming or the one that's already here?**
PRAGMA already replaced every bespoke ML pipeline at Revolut. AAuth is already defining the agent identity standard. PQC overhead is already measurable. The EBW has a 2026 deadline. The teams benchmarking and building toward these today have a 12-month head start when it becomes mandatory. That head start is the moat.

---

**Efficiency. Trust. Deployability.**

Pick two, and you're behind. The next two years go to whoever cracks all three.

---

*Synthesised from 19 LinkedIn posts, May 2026. Original posts by Nikolas Markou, Yatin Arora, Dr. Ashish Bamania, Vlad Kuklev, Ajay Salunkhe, Eric Vyacheslav, Mitko Vasilev, Dr. Carsten Stöcker, Werner Bogula, Arthur Mensch, MCP Shark, Lan Chu, René F., Caleb Sima, Paweł Kuskowski, Simon Taylor, Alex Lieberman, Adham S., and Nicholas A. Gnan.*
