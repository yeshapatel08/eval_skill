# AI Agent Hackathon Evaluation Report

## 1. Overall Score

| Parameter | Maximum Marks | Awarded Marks | Percentage |
|---|---:|---:|---:|
| Problem Statement Alignment | 100 | 64 | 64% |
| Code Quality | 100 | 58 | 58% |
| Innovation | 100 | 39 | 39% |
| Security | 100 | 61 | 61% |
| Grounding and Evals | 50 | 16 | 32% |
| **Total Score** | **450** | **238** | **52.9%** |

---

## 2. Executive Summary

- **Overall Assessment:** TripPilot is a useful travel-planning prototype with a complete Gradio flow, deterministic constraints, live public data lookups, and an optional LLM-written explanation. It addresses many requested outcomes, but its free-text budget parser mishandles the built-in example, its prices are heuristic rather than sourced, and itinerary generation is mostly fixed templates. It is not a tool-using autonomous agent; the LLM is optional and limited to presentation.
- **Main Strengths:**
  - [IMPLEMENTED] A coherent request-to-itinerary path considers duration, traveler count, budget, pace, interests, and weather (`TripPilot_Colab_OneFile (1).py:L206-L258`, `L315-L447`).
  - [IMPLEMENTED] Live geocoding, forecast, routing, and OpenStreetMap POI integrations have timeouts and fallback behavior (`L64-L165`, `L315-L329`).
  - [IMPLEMENTED] Budget warnings, packing suggestions, budget scenarios, plan diffs, and booking handoffs give the UI useful breadth (`L407-L464`, `L467-L581`).
  - [IMPLEMENTED] Most displayed dynamic values are HTML-escaped, and the optional LLM cannot directly change planner state (`L545-L550`, `L604-L625`, `L738-L846`).
- **Significant Weaknesses:**
  - The default request’s `5-day` token is parsed as a budget of 5, then clamped to ₹5,000 instead of recognizing ₹50K (`L223-L258`).
  - Costs are hand-entered estimates and transport is capped at 40% of budget, potentially hiding a real budget overrun (`L171-L203`, `L331-L354`).
  - The planner always schedules nature and food blocks regardless of selected interests; several listed interests do not affect destination scoring or activity selection (`L264-L297`, `L380-L403`).
  - There are no target-project tests, automated evals, dependency manifest, or reproducible environment specification.
- **Key Technical Observations:**
  - The application, integrations, planning logic, and UI are delivered in one 966-line script; helper functions provide some local separation, but the entrypoint installs dependencies and launches the app at import/run time (`L10-L32`, `L206-L966`).
  - Replanning is a fresh deterministic calculation with a displayed diff, not an agent state machine or learned adaptation (`L449-L464`, `L695-L710`, `L932-L942`).
- **Important Security Concerns:**
  - `demo.launch(share=True)` publishes a Gradio share endpoint without application authentication when run (`L966`).
  - Prompt-injection matching only displays a warning; when Gemini is enabled, request data and fixed instructions are concatenated into one content string rather than passed as distinct roles (`L35-L57`, `L604-L625`). The model has no planner tools, limiting but not eliminating this concern.
- **Alignment with Problem Statement:** The submission implements a personalized travel-planning prototype with live context and change handling, but reliability gaps in parsing, pricing, and interest adherence prevent it from consistently satisfying the sample request and practical-plan requirements.

---

## 3. Detailed Parameter Evaluations

### 3.1 Problem Statement Alignment (Awarded: 64 / 100)
- **Assessment:** The core workflow is present: request constraints feed destination choice, weather and POI lookups, budget estimation, a day-by-day plan, warnings, and re-planning. It is only moderately aligned in practice because a key example request is misparsed, costs are not live or calibrated, and selected interests do not consistently control the itinerary. The destination catalog is small for automatic recommendations, although custom destinations can be geocoded.
- **Evidence:**
  - Files Inspected: `[TripPilot_Colab_OneFile (1).py:L206-L297]`, `[L315-L447]`, `[L695-L966]`.
  - Implementation Findings: [IMPLEMENTED] Duration/traveler limits and interest/destination parsing are in `L210-L258`; destination ranking and geocoding fallback are in `L264-L297`; weather-based morning substitutions and budget warnings are in `L380-L425`. [PARTIALLY IMPLEMENTED] The cost model uses fixed destination estimates (`L171-L203`, `L341-L354`), and the itinerary always draws nature and food activities (`L380-L403`). [CONFIRMED DEFECT] In the default “5-day ... under ₹50K” request, the generic money regex matches `5` before the intended budget (`L223-L230`); the verified regex match was `5`, with no unit, so the returned budget becomes ₹5,000 after the minimum clamp (`L258`).
- **Strengths:**
  - Duration, traveler count, pace, interests, and destination have explicit inputs and bounded values (`L206-L258`, `L873-L899`).
  - Weather-aware alternatives and visible over-budget warnings are implemented (`L315-L425`).
- **Weaknesses & Gaps:**
  - Free-text amount parsing is not tied to “under” and fails on the sample request; the structured budget control does not fix the parser when a matching number occurs in the prompt (`L223-L258`, `L873-L899`).
  - The activity selection ignores most interest values and repeats a small set of POIs or generic fallbacks across days (`L380-L403`).
  - Live prices, availability, opening hours, and transport fares are not obtained; the UI itself tells users to recheck booking details (`L523-L581`, `L650-L660`).
- **Recommendations:**
  - Parse currency amounts only in a currency/budget context and add regression cases for the built-in request, `₹50K`, `under 50000`, and numbers in trip duration or party size.
  - Filter and rank activities against all selected interests, avoid repetitive itinerary entries, and include travel-time feasibility in day scheduling.
  - Label estimates with their assumptions and include a safety margin based on destination-specific, maintainable data.

### 3.2 Code Quality (Awarded: 58 / 100)
- **Assessment:** The script is readable and divided into named helpers, with fallback paths around external calls and a successful static Python AST parse. However, it combines setup, integrations, planning, rendering, and launch in one large file; has no type annotations or target tests; broadly swallows exceptions; and installs packages dynamically. The import-time execution and absent dependency lock reduce reproducibility.
- **Evidence:**
  - Files Inspected: `[TripPilot_Colab_OneFile (1).py:L10-L32]`, `[L64-L165]`, `[L206-L966]`; repository inventory and test/config file search.
  - Implementation Findings: [IMPLEMENTED] Functions separate parsing, tools, plan building, rendering, and booking (`L64-L966`). [PARTIALLY IMPLEMENTED] External calls have timeouts and return fallbacks (`L76-L165`, `L315-L329`), but several broad exception handlers hide error causes (`L76-L89`, `L91-L109`, `L583-L629`). Startup installs Gradio/requests/pandas (`L21-L32`), and Gemini use installs another package at runtime (`L593-L600`). No separate target tests, evals, or dependency manifest were present. `ast.parse` completed successfully; this was not a runtime test.
- **Strengths:**
  - The planning and rendering flow is split into named functions rather than one monolithic callback (`L206-L966`).
  - External HTTP operations set timeouts and degrade to fallback data or estimates (`L64-L165`, `L315-L329`).
- **Weaknesses & Gaps:**
  - A one-file design couples application startup, UI, integrations, and business logic; the script launches a public app at the top level (`L10-L32`, `L855-L966`).
  - There are no type annotations, automated tests, structured logs, dependency lock, or explicit setup/configuration beyond runtime package installation.
  - Broad exception handling can make failed live integrations indistinguishable from legitimate empty results (`L76-L109`, `L123-L137`, `L153-L165`).
- **Recommendations:**
  - Separate pure planning logic, provider clients, and UI; guard launch behind an explicit entrypoint.
  - Add pinned dependency metadata and deterministic unit tests for parsing, costs, destination selection, and replanning; preserve actionable diagnostics without returning sensitive details to users.

### 3.3 Innovation (Awarded: 39 / 100)
- **Assessment:** The project has practical product features beyond a bare LLM wrapper, including deterministic constraint handling, a budget shock simulator, weather substitution, plan diffs, and a trace of data lookups. Its planning is handcrafted and static, with no dynamic tool selection, agent loop, memory, retrieval pipeline, or novel orchestration. The optional Gemini call generates explanatory prose only.
- **Evidence:**
  - Files Inspected: `[TripPilot_Colab_OneFile (1).py:L264-L447]`, `[L449-L520]`, `[L583-L629]`, `[L932-L966]`.
  - Implementation Findings: [IMPLEMENTED] Rule-based destination scoring and weather-aware planning (`L264-L297`, `L380-L425`); deterministic budget scenarios and trip diffs (`L449-L464`, `L475-L492`); optional LLM rationale (`L583-L629`). No LLM tool schema or agent execution loop is present.
- **Strengths:**
  - Budget shock testing and explicit before/after changes are useful, explainable interactions (`L449-L464`, `L475-L492`).
  - The Python planner remains authoritative when the optional language model is used (`L604-L625`).
- **Weaknesses & Gaps:**
  - The core is a fixed rules engine and does not dynamically reason over tools or replan in response to conversation history.
  - “Trip DNA” and the tool trace are presentation features, not measured personalization or provenance mechanisms (`L435-L447`, `L467-L474`).
- **Recommendations:**
  - Make planning decisions reflect validated activity constraints and route feasibility, with explainable evidence attached to each recommendation.
  - Add a bounded, explicit conversational revision flow only if needed; keep deterministic validation authoritative and evaluate any LLM decisions against cases.

### 3.4 Security (Awarded: 61 / 100)
- **Assessment:** There is useful baseline hygiene: no hardcoded API token was found in the inspected target file, Gemini credentials are read from an environment variable, HTTP requests use timeouts, outbound URLs are mostly fixed/allowlisted, dynamic HTML text is escaped, and no `eval`, `exec`, or shell execution was found. The main gaps are the unauthenticated public share deployment, weak prompt/data separation when Gemini is enabled, and runtime package installation. No active exploitation was performed.
- **Evidence:**
  - Files Inspected: `[TripPilot_Colab_OneFile (1).py:L21-L73]`, `[L111-L165]`, `[L545-L550]`, `[L583-L629]`, `[L695-L966]`.
  - Findings: [IMPLEMENTED] `safe_get` checks a fixed host allowlist and supplies a timeout (`L43-L73`); POI and routing requests also specify timeouts (`L123-L160`); user-facing dynamic values are generally escaped (`L545-L550`, `L738-L846`). [PARTIALLY IMPLEMENTED] `injection_flags` recognizes a few strings but only adds a warning (`L35-L57`, `L738-L744`); Gemini receives the request payload appended to instruction text in one content string (`L604-L625`). [CONFIRMED DEPLOYMENT BEHAVIOR] `demo.launch(share=True)` exposes a share endpoint without an application authentication layer in this script (`L966`).
- **Strengths:**
  - Gemini credentials are not embedded in source and the optional LLM call falls back to deterministic text on errors (`L593-L629`).
  - HTTP destinations and request timeouts are constrained, and the integration surface is read-only (`L43-L73`, `L76-L165`).
- **Weaknesses & Gaps:**
  - Public share hosting has no authentication or rate limiting in the application code; if a Gemini key is configured, public users can trigger calls billed to that key (`L593-L602`, `L966`).
  - Injection detection does not block or neutralize the input; the user prompt is included in data sent to Gemini without a separate API role or robust untrusted-data boundary (`L604-L625`, `L738-L744`).
  - Runtime pip installation uses open-ended minimum versions and modifies the execution environment (`L21-L32`, `L593-L600`).
- **Recommendations:**
  - Disable `share=True` by default or require authentication, and apply request/concurrency limits before exposing the app.
  - Keep user and retrieved data in explicitly untrusted fields/messages, constrain Gemini output, and avoid sending unnecessary prompt data to an external provider.
  - Pin dependencies in a reviewed environment and remove runtime installation from application request paths.

### 3.5 Grounding and Evals (Awarded: 16 / 50.0)
- **Subcategory Breakdown:**
  - **Grounding Score:** 16 / 25.0
  - **Evals Score:** 0 / 25.0
  - **Total Grounding and Evals:** 16 / 50.0
- **Assessment:**
  - Grounding: Open-Meteo, OSRM, and OpenStreetMap provide live weather, routing, and nearby-place data, and the planner has fallbacks. The application does not show record-level citations, timestamps, or source links for recommendations; much of the budget and destination ranking comes from hardcoded estimates. The optional LLM is instructed not to invent live prices, but there is no output faithfulness check.
  - Evals: No target unit tests, semantic judge, trajectory checks, adversarial dataset, benchmark, or reproducible evaluation runner was found. No tests were run; static syntax parsing and a focused parser reproduction were performed instead.
- **Evidence:**
  - Files Inspected: `[TripPilot_Colab_OneFile (1).py:L76-L165]`, `[L171-L203]`, `[L315-L447]`, `[L583-L629]`; repository file inventory.
  - Implementation Findings: [IMPLEMENTED] Live provider lookups and fallback paths (`L76-L165`, `L315-L329`); a human-readable provider/tool trace is returned (`L435-L447`). [PARTIALLY IMPLEMENTED] External data is reduced to names and categories without per-item provenance in the output (`L111-L150`, `L380-L403`). [CLAIMED BUT UNVERIFIED] UI language calls the plan “grounded” and prints readiness messages, but those messages are not eval results (`L855-L966`). No evaluation harness exists in the target project.
- **Strengths:**
  - Live data enriches a locally computed plan, and provider failure has fallbacks (`L76-L165`, `L315-L329`).
  - The UI disclaims live booking prices and availability and directs users to verify them with providers (`L523-L581`, `L650-L660`).
- **Weaknesses & Gaps:**
  - No citations or timestamps let users trace which source supported an individual itinerary recommendation; estimated prices are not grounded in those live sources.
  - No automated tests or evaluations demonstrate parsing accuracy, budget correctness, resilience, or prompt-injection behavior.
- **Recommendations:**
  - Attach provider, retrieval time, and relevant source links to live facts; clearly distinguish sourced data from assumptions and estimates.
  - Build deterministic regression and edge-case tests first, then add trajectory/adversarial checks and an automated, reproducible eval command.

---

## 4. Cross-Cutting Findings

- **Architecture & Modularity:** A standalone one-file app with named helpers; no separate service, agent framework, MCP integration, RAG, or persistent memory.
- **Reliability & Resilience:** Provider failures generally degrade gracefully, but broad exception handling masks causes and the free-text parser can corrupt a core constraint.
- **Security Posture:** Read-only external data access and HTML escaping are positives. Public unauthenticated sharing, runtime installs, and weak optional-LLM input separation need attention.
- **Evaluation Maturity:** No tests or eval runner for TripPilot. The repository’s `.agents/skills/hack-eval/` material is evaluator guidance, not a test harness for TripPilot.
- **Maintainability & Extensibility:** Named functions help, but UI, integrations, and domain logic in a single script make independent testing and replacement difficult.
- **Reproducibility:** The file documents a Colab flow and installs packages itself, but dependencies are not locked; live APIs and a public Gradio tunnel make results environment-dependent.

---

## 5. Critical Issues & Vulnerabilities

| Issue | Severity (Critical/High/Medium/Low) | Affected Component | Confirmation Status (Confirmed/Potential) | Evidence | Potential Impact |
|---|---|---|---|---|---|
| Sample budget is parsed from the trip duration | High | `TripPilot_Colab_OneFile (1).py:L223-L258` | Confirmed | Exact regex match on the built-in example is `5` with no unit; clamp changes intended ₹50,000 to ₹5,000. | Destination selection and feasibility warnings use the wrong ceiling; the default demonstration misrepresents the user's constraint. |
| Transport estimate is forcibly capped at 40% of budget | Medium | `TripPilot_Colab_OneFile (1).py:L331-L354` | Confirmed | The computed transport estimate is replaced by `min(transport, budget * 0.40)` before total cost calculation. | Total and buffer can be understated, yielding a misleading within-budget plan. |
| Public unauthenticated Gradio share endpoint | High | `TripPilot_Colab_OneFile (1).py:L966` | Confirmed | App launches with `share=True`; no application authentication is defined in the script. | Anyone with the share URL can submit requests; with Gemini configured, requests may consume the owner’s API quota. |
| Optional LLM prompt boundary is not robust | Medium | `TripPilot_Colab_OneFile (1).py:L604-L625` | Potential | Request JSON is concatenated to instruction text in one `contents` string; injection detection only renders a warning (`L55-L57`, `L738-L744`). | A malicious prompt may influence the optional explanation. The LLM has no planner tools and the returned text is HTML-escaped, which limits impact. |
| Dynamic package installation at runtime | Low | `TripPilot_Colab_OneFile (1).py:L21-L32`, `L593-L600` | Confirmed | Dependencies are installed with open-ended minimum versions during execution. | Environment reproducibility and supply-chain review are weakened; running the notebook requires package-index access. |

---

## 6. Final Summary & Judging Verdict
- **Final Score Breakdown:**
  - Problem Statement Alignment: 64 / 100
  - Code Quality: 58 / 100
  - Innovation: 39 / 100
  - Security: 61 / 100
  - Grounding and Evals: 16 / 50.0
  - **Total Score: 238 / 450.0**
- **Strongest Aspects:**
  - A usable end-to-end travel-planning UI combines several live data sources with deterministic constraints.
  - The planner includes visible budget warnings, weather substitutions, fallbacks, and a replanning diff.
- **Major Gaps:**
  - A confirmed parser bug breaks the built-in example’s budget, and a transport cap can mask cost overruns.
  - No tests or evals support reliability claims; pricing and several personalization behaviors remain heuristic.
- **Improvement Priorities:**
  1. Correct and test currency parsing; remove the budget-based transport cap and report uncertain estimates conservatively.
  2. Disable unauthenticated public sharing by default and strengthen the Gemini data boundary.
  3. Add pinned dependencies, focused tests, and a reproducible evaluation suite.
- **Evaluation Limitations:** The application was not launched, no live services were called, and no existing target tests were available. Python AST parsing succeeded; a source-equivalent regex check reproduced the sample budget defect. Security findings are based on static inspection; no active exploitation was performed.