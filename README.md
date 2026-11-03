# RAG Compliance Reviewer

A single-notebook, production-style agent that answers policy/compliance questions **only from approved documents**, cites verbatim evidence for every claim, and **refuses** when evidence is missing, conflicting, stale, or out of scope.

- **File:** `rag_compliance_reviewer.ipynb` (36 cells, runs top to bottom)
- **Pattern:** Master–Worker · ReAct · Adaptive RAG (with a corrective round) · MCP-style tools · sandboxed execution · auto-evaluation
- **Default dataset:** OWASP ASVS v4.0.3 (authentication + session chapters), loaded **by URL only** (pinned git tag). No dataset file lives in the project.

## Quick start (Google Colab, ~3 minutes)

1. Open the notebook in Colab (upload it, or `File → Upload notebook`).
2. Get a free key at <https://aistudio.google.com/apikey>.
3. In Colab, add it under **Secrets** (key icon) as `GEMINI_API_KEY` and enable notebook access (or just paste it when prompted; input is hidden and never stored).
4. **Runtime → Run all.** Cell 11 runs 29 offline tests first; if any fail the run stops.

Local: `jupyter lab`, `export GEMINI_API_KEY=...`, run all cells (Python 3.10+).

## Free resources

| Need | Recommendation | Notes |
|---|---|---|
| LLM (free key) | **Google Gemini API** via AI Studio, `gemini-2.5-flash-lite` (falls back to `gemini-2.5-flash`) | Free tier, no card. Limits change often and apply per project: check AI Studio. The client throttles to stay under ~15 requests/min. |
| LLM alternative | **Groq** (`CFG.provider = "groq"`, `GROQ_API_KEY`) | Free tier is token-limited (e.g. 12K tokens/min on the 70B model); the 8B model is the fallback. Expect more 429 waits. |
| MCP | **Official MCP Python SDK** (open source, free), `pip install "mcp<2"` | The tool layer already uses MCP message shapes (`tools/list`, `tools/call`, resources). `serve_mcp_http(INDEX)` exposes the same tools as a real MCP server. No paid MCP service is needed; the corpus is closed, so a web-search MCP would be counterproductive. |
| Dataset | URL(s) in `CFG.dataset_urls` | Host must be in `CFG.allowed_hosts`. Supports `.md`, `.txt`, `.html`, `.pdf`. |

## Architecture

```
question ─► SessionGuard (rate limit, strikes) ─► InputGuard (normalise, rules, base64/rot13/leet views)
        ─► [grey zone only] LLM security classifier ─► PII redaction ─► cache
        ─► Planner (adaptive router)
              ├─ small talk / legal / off-topic  → refuse (no LLM call)
              ├─ simple                           → 1 sub-question (no planner call)
              └─ complex / comparative            → LLM plan, ≤3 sub-questions
        ─► Workers (ReAct: Thought → Action → Observation, ≤4 steps, ≤3 searches)
              tools via sandbox: list_sources · search_policy · get_chunk · verify_citation · check_policy_version
              weak/insufficient → corrective round (broader k, new wording) → else refuse
        ─► Master verification
              deterministic: chunk was retrieved · quote occurs in chunk · numbers exist in source · claim ≈ quote
              LLM judge: entailment per claim · version-conflict / inactive-policy check
              confidence = 0.4·retrieval + 0.3·citation validity + 0.3·faithfulness
        ─► OutputGuard (canary, secrets, prompt fragments, URL/image stripping) ─► concise JSON
```

**Retrieval:** structure-aware chunking (headings → list items/paragraphs → sentences; table rows are atomic and rendered as `[2.1.1] Description: …; L1: ✓`), BM25 + dense (sentence-transformers `all-MiniLM-L6-v2`, TF-IDF fallback) fused with Reciprocal Rank Fusion, heuristic rerank with an exact-clause-id bonus (optional cross-encoder), metadata filters (status, clearance, jurisdiction, date) applied before ranking.

## Output

```json
{
  "status": "ANSWERED",
  "answer": "User-set passwords must be at least 12 characters long [1].",
  "citations": [{"id": 1, "chunk_id": "0x11-v2-authentication::chunk_07", "source": "0x11-v2-authentication",
                 "section": "V2 Authentication › V2.1 Password Security Requirements",
                 "quote": "Verify that user set passwords are at least 12 characters in length",
                 "supports_claim": "User-set passwords must be at least 12 characters long"}],
  "confidence": 0.9,
  "metrics": {"llm_calls": 3, "tool_calls": 1, "latency_s": 14.2, "citation_validity": 1.0, "faithfulness": 1.0},
  "disclaimer": "Informational policy lookup based only on the indexed documents; not legal advice."
}
```
Other statuses: `INSUFFICIENT_EVIDENCE` (answer is literally that string, with `reason`, empty `citations`), `CONFLICTING_EVIDENCE`, `REFUSED`, `BLOCKED`, `ERROR`. (Illustrative chunk ids/numbers above; yours come from the run.)

## Security model

| Threat | Control |
|---|---|
| Direct prompt injection / role inversion | Unicode normalisation (zero-width, bidi, homoglyph, leetspeak views), weighted rules, base64/rot13 decoding, fake `system:` markers, chat-template tokens; blocked inputs **never reach the LLM** |
| Ambiguous inputs | Optional LLM classifier, only for the grey zone (saves quota) |
| Indirect injection (poisoned documents) | Ingestion scan → suspicious chunks **quarantined**; evidence is HTML-escaped and wrapped in nonce-tagged `<evidence>` blocks, declared untrusted in the system prompt |
| Prompt / secret exfiltration | Canary token in the system prompt, output guard blocks leaks, strips URLs and markdown images, redacts PII/keys |
| Hallucinated policy | Verbatim-quote verification, number check, entailment judge, refusal on failure |
| Abuse | Per-session rate limits, strike-based lockout, per-request budgets (LLM calls, tool calls, deadline), length caps |
| SSRF via dataset URL | https only, host allow-list, no credentials/IP literals/odd ports, DNS result must be public, manual redirect re-validation, size and content-type caps |
| Tool misuse | Allow-listed read-only tools, JSON-schema argument validation, timeouts, exception containment, output caps; clearance comes from the server, never from the model |

## Evaluation

Cell 15 runs 16 golden cases (direct, paraphrase, exact-number, table-cell, multi-document, missing evidence, out-of-corpus, out-of-scope, 5 attack types) and reports status accuracy, answer correctness, citation validity, faithfulness, refusal and attack-block rates, p50/p95 latency, average LLM calls and tokens. Cell 12 reports retrieval recall@5, MRR and a hybrid/BM25/dense ablation without any LLM.

## Customising

- Your own policies: set `CFG.dataset_urls`. Optional front-matter in a document (`version: v3`, `effective_date: 2025-01-01`, `jurisdiction: eu`, `status: draft`) is picked up automatically; draft/expired documents are excluded from retrieval.
- Multiple versions of one policy (e.g. `policy-v2.md`, `policy-v3.md`) are detected as a family; citing both yields `CONFLICTING_EVIDENCE`.
- Quota tuning: `CFG.llm_judge=False` and `CFG.max_corrective_rounds=0` cut LLM calls per question to about 2–3.

## Known limitations (honest notes)

- **Sandbox is thread-level**, not a container. Tools are read-only with no shell, file, or network access, which is what makes this adequate here. For untrusted third-party tools, run them in a subprocess/container.
- **Rule-based injection detection is not exhaustive.** It is one layer among several; novel paraphrased attacks may reach the grey zone or the model, where structural isolation, output guard and citation verification still apply. Extend `INJECTION_RULES` for your threat model.
- Verification confirms a claim is supported by the quoted text; it cannot judge whether the *document itself* is correct or complete.
- Not legal advice. The agent is a retrieval aid, and a human should review high-risk outputs.
- Free-tier quotas and model names change; check your provider console if you hit 429/404 errors.
- The notebook's LLM calls were not exercised against live APIs in the build environment; the LLM client, retry logic, agents and security layers were tested offline with scripted doubles (29 tests), and retrieval was checked on the real ASVS data.
