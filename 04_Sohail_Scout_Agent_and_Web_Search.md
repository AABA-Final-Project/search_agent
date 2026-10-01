# Person 4: Disruption Scout Agent and Web Search Tool

**Your repo:** `scm-scout-agent`
**Your role in one line:** you build the first AI agent, the "news reporter". It searches the live web for storms, port closures, strikes and trade problems affecting our suppliers, and it keeps working even when the search service fails.
**Deadline:** search tool working by **Thursday 1 Oct, 9 pm**. Agent tuned and tagged `v1.0` by **Friday 2 Oct, 8 pm**.
**Estimated effort:** 5 hours.

## What you deliver

| File | What it is |
|---|---|
| `tools/search_tool.py` | `web_search` CrewAI tool with retries, a fallback search engine and a "simulate failure" switch |
| `agents/scout.py` | `build_scout_agent(llm)` and `build_scout_task(agent)`, plus a stand-alone test run |
| `tests/test_search_tool.py` | Tests for the tool, including the failure and fallback path |
| `docs/scout.md` | How it works, plus **2 saved sample Scout outputs** (Persons 5 and 6 use them as test input) |

This part earns much of the **35% "Agent Autonomy & Tool Calling"** marks: the agent picks its own searches and recovers from errors by itself.

---

## Part A: Shared rules for all 6 members (identical in every file)

These rules keep the six repos from conflicting when Person 6 merges them. **Do not change anything in Part A on your own.** If something here needs to change, ask Person 6, who updates it for everyone.

### A1. The team and who owns what

Every file in the final project has exactly one owner. You may only create or edit files in **your own** rows below. The paths are the final paths inside the combined repo, so build your repo with exactly this folder layout.

| Person | Repo name | Owns these paths (and nothing else) |
|---|---|---|
| 1 | `scm-erp-infra` | `docker-compose.yml`, `db/init.sql`, `n8n/supply_chain_workflow.json`, `tests/test_erp.py`, `docs/erp_infra.md` |
| 2 | `scm-knowledge-base` | `knowledge_base/SOP-SC-014_Supply_Disruption_Response.pdf`, `knowledge_base/Vendor_Contracts_and_AVL.pdf`, `knowledge_base/source/` (editable originals), `docs/knowledge_base.md`, `docs/demo_script.md` |
| 3 | `scm-rag-tool` | `tools/rag_tool.py`, `tests/test_rag.py`, `docs/rag.md` |
| 4 | `scm-scout-agent` | `tools/search_tool.py`, `agents/scout.py`, `tests/test_search_tool.py`, `docs/scout.md` |
| 5 | `scm-analyst-agent` | `tools/sql_tools.py`, `agents/analyst.py`, `tests/test_sql_tools.py`, `docs/analyst.md` |
| 6 (lead) | `scm-coordinator` | `schemas/payload.py`, `tools/n8n_tool.py`, `agents/coordinator.py`, `main.py`, `requirements.txt`, `.env.example`, `.gitignore`, `README.md`, `tests/test_n8n_tool.py`, `docs/coordinator.md` |

Rules that prevent merge conflicts:

1. **Create your GitHub repo empty.** Do not tick "Add a README", ".gitignore" or "license". Only Person 6 creates `README.md`, `.gitignore` and `requirements.txt`.
2. **Do not create `__init__.py` files.** Python treats `tools/`, `agents/` and `schemas/` as packages without them. This avoids six people creating the same file.
3. **Keep secrets and junk out of git** without a `.gitignore`. After `git init`, run this once:

   ```bash
   printf ".venv/\n.env\n__pycache__/\n.rag_store/\noutbox/\nreports/\n.pytest_cache/\n" >> .git/info/exclude
   ```

4. **Work on the `main` branch** of your own repo and push often.
5. **Never commit an API key.** Keys live only in your local `.env` file.

### A2. Everyone's software setup (do this first, about 20 minutes)

| Tool | Version | Used by | How to get it |
|---|---|---|---|
| Git + GitHub account | any recent | all | git-scm.com; accept the organisation invite |
| Python | 3.11 (3.10–3.12 also fine) | all | python.org |
| VS Code | any | all | code.visualstudio.com, plus the "Python" extension |
| Docker Desktop | any recent | all (to run the database and n8n locally) | docker.com/products/docker-desktop |
| OpenAI API key | n/a | 4, 5, 6 | platform.openai.com → API keys (or the team key from Person 6) |
| Serper API key | free tier | 4, 6 | serper.dev (free 2,500 searches) |

Create the same Python environment on every laptop:

```bash
python -m venv .venv
# Windows: .venv\Scripts\activate      Mac/Linux: source .venv/bin/activate
pip install crewai==1.15.23 langchain-community psycopg2-binary chromadb pypdf ddgs requests python-dotenv reportlab pytest
```

Everyone uses the **same crewai version (1.15.23)**. Do not upgrade it on your own.

To run the shared database and n8n on your laptop before Person 1 has finished, wait for Person 1's push (due Thursday 6 pm), then copy their `docker-compose.yml` and `db/` folder into a scratch folder **outside your repo** and run `docker compose up -d` there.

### A3. Settings (environment variables)

Each person keeps a local `.env` file in their repo root (never committed). Read values with `os.getenv()` after `load_dotenv()`. Use exactly these names:

| Variable | Example value | Used by |
|---|---|---|
| `LLM_MODEL` | `openai/gpt-4o-mini` | 4, 5, 6 |
| `OPENAI_API_KEY` | `sk-...` | 4, 5, 6 (and 3 only if testing memory) |
| `SERPER_API_KEY` | `...` | 4, 6 |
| `DATABASE_URL` | `postgresql+psycopg2://scm:scm@localhost:5432/erp` | 1, 5, 6 |
| `N8N_WEBHOOK_URL` | `http://localhost:5678/webhook/supply-chain-rfq` | 1, 6 |
| `KB_DIR` | `knowledge_base` | 3, 6 |
| `SIMULATE_SEARCH_FAILURE` | `false` | 4, 6 |

### A4. Business codes everyone must use exactly

The database (Person 1), the documents (Person 2) and every prompt (Persons 4–6) must use these exact IDs and names.

**Company:** Aravalli Mobility Pvt Ltd, an electric scooter maker. Plant and warehouse: Manesar (`Manesar-WH1`).

| Supplier ID | Name | Country | Export port | Role | AVL status |
|---|---|---|---|---|---|
| S101 | Formosa Microchip Co. | Taiwan | Kaohsiung | Primary: P-1001, P-1002 | APPROVED |
| S102 | Keelung Precision PCB Ltd. | Taiwan | Keelung | Primary: P-2001 | APPROVED |
| S103 | Hsinchu SensorTech Inc. | Taiwan | Kaohsiung | Primary: P-3001, P-3002 | APPROVED |
| S201 | Penang Semicon Sdn. Bhd. | Malaysia | Penang | Backup: P-1001, P-1002 | APPROVED |
| S202 | Saigon Circuit Works JSC | Vietnam | Cat Lai | Backup: P-2001 | APPROVED |
| S301 | Bengaluru Embedded Systems | India | Domestic (road) | Backup: P-1001, P-3001, P-3002 | APPROVED |
| S302 | Chennai Interconnect Ltd. | India | Domestic (road) | Primary: P-4001 | APPROVED |
| S401 | Shenzhen PowerCell Co. | China | Yantian | Primary: P-5001 | APPROVED |
| S402 | Pune CellWorks Pvt. Ltd. | India | Domestic (road) | Backup: P-5001 | CONDITIONAL |

| Part ID | Name | Critical | Days of stock (cover) |
|---|---|---|---|
| P-1001 | 32-bit Automotive MCU | Yes | 12 |
| P-1002 | Motor Controller Power MOSFET Module | Yes | 21 |
| P-2001 | BMS PCB Assembly (6-layer) | Yes | 18 |
| P-3001 | Hall-Effect Throttle Sensor | No | 35 |
| P-3002 | 6-axis IMU Sensor Module | Yes | 8 |
| P-4001 | IP67 Harness Connector | No | 40 |
| P-5001 | Li-ion 21700 Cell | Yes | 25 |

### A5. The company rules (the numbers everyone codes and tests against)

| Rule | Exact value |
|---|---|
| Severity | L1: delay < 7 days (watch only, no RFQ). L2: 7–21 days. L3: > 21 days, or port closure lasting beyond 21 days, blockade, export ban, supplier force majeure. Always use the **worst-case (upper) delay**. |
| RFQ trigger | RFQ is mandatory when `days_of_cover < worst_case_delay + 5` |
| Shortfall | `shortfall_units = (worst_case_delay + 5 − days_of_cover) × daily_consumption` |
| RFQ quantity | `shortfall_units × 1.2`, rounded **up to the nearest multiple** of the backup supplier's MOQ |
| Eligible suppliers | Only `APPROVED_BACKUP` with AVL status `APPROVED`. `CONDITIONAL` needs Head of SCM sign-off first. |
| Dual RFQ | L3 + critical part: RFQ at least 2 backups where 2 exist |
| Contingency | Air freight if premium < 15% of PO value · safety stock +30% during L2/L3 · ask primary to re-route via alternate port · open POs → HOLD-REVIEW · notify Production Planning if stock-out within 7 days |
| Approval (on the **total** value of all RFQs in one action record) | ≤ ₹25,00,000: Operations Manager – Procurement · ₹25,00,001–₹1,00,00,000: Head of Supply Chain Management · > ₹1,00,00,000: Chief Financial Officer |

**The demo scenario (everyone tests against this):** a typhoon closes Kaohsiung and Keelung ports in Taiwan, with shipments expected 14–21 days late. The worst case is 21 days, so the severity is L2. Expected result:

| Part | Cover | Delay + 5 | Gap days | Shortfall | RFQ qty | Supplier | Value (₹) |
|---|---|---|---|---|---|---|---|
| P-1001 | 12 | 26 | 14 | 16,800 | 22,000 | S201 | 99,44,000 |
| P-1002 | 21 | 26 | 5 | 12,000 | 15,000 | S201 | 98,25,000 |
| P-2001 | 18 | 26 | 8 | 9,600 | 12,000 | S202 | 1,11,60,000 |
| P-3002 | 8 | 26 | 18 | 21,600 | 26,000 | S301 | 1,93,70,000 |
| P-3001 | 35 | 26 | −9 | 0 | none (watch) | none | none |

Additional expected results:
- The total is ₹5,02,99,000, which is above ₹1 crore, so the approver is the **Chief Financial Officer**.
- P-3002 needs 26,000 but S301 can only make 15,000 a month. The Coordinator should flag this and add contingency steps (air freight or re-routing).
- P-3002 runs out in 8 days, so Production Planning must be notified.

### A6. How the pieces connect (function names are fixed)

```
main.py (P6)
 ├─ get_watchlist()                       ← tools/sql_tools.py (P5)
 ├─ build_index()                         ← tools/rag_tool.py (P3)
 ├─ build_scout_agent(llm), build_scout_task(agent)            ← agents/scout.py (P4)
 ├─ build_analyst_agent(llm), build_analyst_task(agent, ctx)   ← agents/analyst.py (P5)
 ├─ build_coordinator_agent(llm), build_coordinator_task(agent, ctx) ← agents/coordinator.py (P6)
 └─ Crew(sequential).kickoff(inputs={today, watchlist, focus, payload_schema})

Tools each agent gets:
 Scout       → web_search                              (tools/search_tool.py, P4)
 Analyst     → list_erp_tables, describe_erp_tables, query_erp (tools/sql_tools.py, P5)
 Coordinator → sop_search (P3) + trigger_n8n (tools/n8n_tool.py, P6)
Infrastructure → Postgres + n8n (P1) · PDFs (P2)
```

Task input placeholders (CrewAI fills these from `kickoff(inputs=...)`): `{today}`, `{watchlist}`, `{focus}`, `{payload_schema}`. Use only these names.

### A7. Coding conventions

- **Tools use the CrewAI decorator:** `from crewai.tools import tool`. Every tool needs a docstring, because the AI reads it to decide when to use the tool. Every tool takes string input and returns a string.
- **Tools must never crash.** Catch exceptions and return a clear message starting with `ERROR:`, `SQL ERROR:`, `RAG ERROR:`, `VALIDATION ERROR` or `SEARCH UNAVAILABLE:`, telling the AI what to do next.
- **Log with this exact helper** (copy it into your tool file). Tags: `TOOL`, `RECOVERY`, `RAG`, `N8N`, `BOOT`, `WARN`.

  ```python
  from datetime import datetime
  def log(tag: str, msg: str) -> None:
      print(f"\033[96m[{datetime.now():%H:%M:%S}] [{tag}]\033[0m {msg}", flush=True)
  ```

- **Agent builders** take `llm` as a parameter and never create their own LLM, except inside `if __name__ == "__main__":` test blocks.
- Every agent is created with `verbose=True, allow_delegation=False, max_iter=15, max_retry_limit=3`.
- Run tests from your repo root with `python -m pytest -q`.

### A8. Timeline (presentation Monday 5 Oct 2026)

| When | What | Who |
|---|---|---|
| Wed 30 Sep, night | Organisation and 6 empty repos created, invites accepted, A2 setup done | P6 creates; all set up |
| Thu 1 Oct, 6 pm | Database + n8n pushed and working · PDFs pushed | P1, P2 |
| Thu 1 Oct, 9 pm | Each tool works on its own (tested) | P3, P4, P5, P6 |
| Fri 2 Oct, 8 pm | Every module finished, tests pass, `docs/<area>.md` written, repo tagged `v1.0` | all |
| Sat 3 Oct | P6 merges all repos; full run on a group call; fixes | P6 leads, all attend |
| Sun 4 Oct | 3 practice runs, Loom recorded, README final | P6 records, P2 directs, all review |
| Mon 5 Oct | Presentation | all |

After the merge (Saturday onwards), everyone works in the combined repo `supply-chain-crew` on a branch named `fix/<your-name>`, still touching **only your own paths**. Open a Pull Request, and Person 6 merges it.

### A9. Tagging your finished work

```bash
git add . && git commit -m "v1.0: <your module> complete"
git tag v1.0 && git push origin main --tags
```

Then post in the team group: "Person N done, v1.0 tagged, tests pass", plus anything the others need to know.

---

## Part B: Your step-by-step instructions

### Step 1: Create your repo and get keys (20 min)

```bash
mkdir scm-scout-agent && cd scm-scout-agent
git init -b main
printf ".venv/\n.env\n__pycache__/\n.rag_store/\noutbox/\nreports/\n.pytest_cache/\n" >> .git/info/exclude
mkdir tools agents tests docs
git remote add origin https://github.com/<ORG>/scm-scout-agent.git
```

1. Sign up at **serper.dev**. It is free, gives 2,500 searches and needs no card. Copy the API key.
2. Create `.env` in the repo root (never commit it):

   ```
   SERPER_API_KEY=your_key
   OPENAI_API_KEY=your_key
   LLM_MODEL=openai/gpt-4o-mini
   SIMULATE_SEARCH_FAILURE=false
   ```

### Step 2: Understand your tools (15 min)

| Tool | What it does | Key calls |
|---|---|---|
| **LangChain `GoogleSerperAPIWrapper`** (from `langchain_community.utilities`) | The main search: Google News results through Serper. This is the "LangChain Web Search Tool" named in the course brief. | `GoogleSerperAPIWrapper(type="news", k=8).results(query)` returns a dict with a `"news"` list of `{title, link, snippet, date, source}` |
| **ddgs** (DuckDuckGo) | Free backup search, no key needed | `from ddgs import DDGS; DDGS().news(query, max_results=8)` returns a list of `{title, url, body, date, source}` |
| **CrewAI** | `Agent`, `Task`, `Crew`, `LLM`, `@tool` | see the code below |

### Step 3: Write `tools/search_tool.py` (1.5 hours)

**Required behaviour (the contract):**

| Item | Requirement |
|---|---|
| Tool name / function | `"Web Search"` / `web_search(query: str) -> str` |
| Primary search | Serper news, **3 attempts** with waits of 2, 4 and 8 seconds |
| Fallback 1 | DuckDuckGo news |
| Fallback 2 | Return `SEARCH UNAVAILABLE: ...` telling the agent to retry once with a shorter query, then report "no verified signal". **Never invent news.** |
| Simulated failure | If env `SIMULATE_SEARCH_FAILURE=true`, **or** `enable_simulated_failure()` was called, the **first attempt of the first search** raises a fake timeout. This shows the retry on camera. |
| Output format | One block per result: `- <title> \| <date> \| <source>`, then the snippet, then the URL |
| Logs | `[TOOL]` on success, `[RECOVERY]` on every retry or fallback |

```python
"""Web search tool for the Disruption Scout: Serper (LangChain) -> DuckDuckGo -> graceful message."""
import os, time
from datetime import datetime
from crewai.tools import tool
from dotenv import load_dotenv
from langchain_community.utilities import GoogleSerperAPIWrapper

load_dotenv()
_state = {"calls": 0, "simulate": False}


def log(tag: str, msg: str) -> None:
    print(f"\033[96m[{datetime.now():%H:%M:%S}] [{tag}]\033[0m {msg}", flush=True)


def _simulate_on() -> bool:
    # Read env dynamically (main.py may set it after import) + explicit toggle for tests/demo
    return _state["simulate"] or os.getenv("SIMULATE_SEARCH_FAILURE", "false").lower() == "true"


def enable_simulated_failure(on: bool = True) -> None:
    """Called by main.py when run with --simulate-search-failure."""
    _state["simulate"] = on
    _state["calls"] = 0  # reset so the NEXT search demonstrates the retry on camera


def _fmt(items: list) -> str:
    return "\n".join(
        f"- {i.get('title')} | {i.get('date', '')} | {i.get('source', '')}\n"
        f"  {i.get('snippet') or i.get('body', '')}\n  {i.get('link') or i.get('url', '')}"
        for i in items) or "NO RESULTS for this query."


@tool("Web Search")
def web_search(query: str) -> str:
    """Search live news for supply-chain disruption signals: port congestion or closures, typhoons
    and extreme weather, strikes, sanctions, export bans, geopolitical tension.
    Input: ONE focused query, e.g. 'Kaohsiung port typhoon closure'.
    Returns headline | date | source, a snippet and the URL for each result."""
    q = (query or "").strip()[:300]
    if not q:
        return "SEARCH UNAVAILABLE: empty query. Retry with e.g. 'Kaohsiung port closure'."
    _state["calls"] += 1
    if os.getenv("SERPER_API_KEY"):
        serper = GoogleSerperAPIWrapper(type="news", k=8)
        for attempt in range(1, 4):
            try:
                if _simulate_on() and _state["calls"] == 1 and attempt == 1:
                    raise TimeoutError("simulated Serper timeout (demo)")
                news = serper.results(q).get("news", [])
                log("TOOL", f"Serper OK ({len(news)} hits): {q}")
                return _fmt(news)
            except Exception as e:
                log("RECOVERY", f"Serper attempt {attempt}/3 failed: {e}")
                time.sleep(2 ** attempt)
    else:
        log("RECOVERY", "No SERPER_API_KEY - skipping to DuckDuckGo fallback")
    try:
        try:
            from ddgs import DDGS
        except ImportError:  # older images still ship duckduckgo_search
            from duckduckgo_search import DDGS
        res = list(DDGS().news(q, max_results=8)) or []
        log("RECOVERY", f"Fell back to DuckDuckGo ({len(res)} hits): {q}")
        return _fmt(res)
    except Exception as e:
        log("RECOVERY", f"DuckDuckGo failed: {e}")
    return ("SEARCH UNAVAILABLE: all providers failed. Try ONE shorter, different query. "
            "If still unavailable, report 'no verified signal' - never invent events.")
```

### Step 4: Write `agents/scout.py` (1 hour)

**Required behaviour (the contract):**

| Item | Requirement |
|---|---|
| `build_scout_agent(llm) -> Agent` | role `"Disruption Scout"`, tools `[web_search]`, settings from A7 |
| `build_scout_task(agent) -> Task` | Uses **only** the placeholders `{today}`, `{watchlist}`, `{focus}` |
| Output format | Must follow the **Scout handover format** below. Person 5's Analyst reads it. |

**Scout handover format** (put this in `expected_output`):

```
## Disruption events
1. Location/port: <port or region> | Country: <country>
   Type: <weather | port congestion/closure | strike | geopolitical/trade>
   Date: <YYYY-MM-DD>
   Delay range: <a>-<b> days | Worst-case delay: <b> days
   Confidence: <high | medium | low>
   Sources: <url1>, <url2>
2. ...
## Verdict
<MATERIAL DISRUPTION FOUND | NO MATERIAL DISRUPTION FOUND>
```

```python
"""Disruption Scout agent + task."""
from crewai import Agent, Task
from tools.search_tool import web_search

SCOUT_OUTPUT_FORMAT = """## Disruption events
1. Location/port: <port or region> | Country: <country>
   Type: <weather | port congestion/closure | strike | geopolitical/trade>
   Date: <YYYY-MM-DD>
   Delay range: <a>-<b> days | Worst-case delay: <b> days
   Confidence: <high | medium | low>
   Sources: <url1>, <url2>
## Verdict
<MATERIAL DISRUPTION FOUND | NO MATERIAL DISRUPTION FOUND>"""


def build_scout_agent(llm) -> Agent:
    return Agent(
        role="Disruption Scout",
        goal="Detect and verify current disruptions (ports, weather, geopolitics, logistics) "
             "that threaten our supplier lanes",
        backstory="Former logistics control-tower analyst. You only report events you can back "
                  "with sources and dates, and you give conservative (worst-case) delay estimates.",
        tools=[web_search], llm=llm, verbose=True, allow_delegation=False,
        max_iter=15, max_retry_limit=3)


def build_scout_task(agent) -> Task:
    return Task(
        description=(
            "Today is {today}. Our primary suppliers and export lanes are: {watchlist}.\n{focus}\n"
            "1. Run AT LEAST 3 distinct searches: (a) port congestion/closures on these lanes, "
            "(b) typhoons/extreme weather in these countries, (c) geopolitical, trade-restriction "
            "or strike news.\n"
            "2. Keep only events from the last 30 days that plausibly affect these countries/ports.\n"
            "3. For each event give a delay range in days and state the worst-case (upper) value.\n"
            "4. If a search fails, reformulate and retry. If nothing material is found, say so. "
            "Never invent events or URLs."),
        expected_output=SCOUT_OUTPUT_FORMAT,
        agent=agent)


if __name__ == "__main__":   # stand-alone test: python -m agents.scout
    import os
    from datetime import date
    from dotenv import load_dotenv
    from crewai import Crew, LLM
    load_dotenv()
    llm = LLM(model=os.getenv("LLM_MODEL", "openai/gpt-4o-mini"), temperature=0.1)
    a = build_scout_agent(llm)
    crew = Crew(agents=[a], tasks=[build_scout_task(a)], verbose=True)
    out = crew.kickoff(inputs={
        "today": date.today().isoformat(),
        "watchlist": "Formosa Microchip Co. (Taiwan, port Kaohsiung); Keelung Precision PCB Ltd. "
                     "(Taiwan, port Keelung); Hsinchu SensorTech Inc. (Taiwan, port Kaohsiung); "
                     "Shenzhen PowerCell Co. (China, port Yantian); Chennai Interconnect Ltd. (India, Domestic (road))",
        "focus": os.getenv("SCOUT_FOCUS", "")})
    print("\n=== SCOUT OUTPUT ===\n", out.raw)
```

Run it with `python -m agents.scout` from the repo root. With a focus:
- Mac/Linux: `SCOUT_FOCUS="Priority focus: typhoon Kaohsiung port" python -m agents.scout`
- Windows: set the variable in `.env` instead.

### Step 5: Tune the agent (1 hour)

Run the agent at least 5 times and check each point. Fix problems by editing the **backstory** and **description** wording, never by hard-coding news.

| Check | Good behaviour | If it fails, add to the description |
|---|---|---|
| Number of searches | 3 or more different queries | "Each search must use different keywords." |
| Relevance | Only Taiwan, China or India lanes | "Ignore events outside the watchlist countries." |
| Evidence | Every event has at least 1 real URL copied from search results | "Only use URLs that appear in search results." |
| Worst-case delay | A single number stated | "Always state 'Worst-case delay: N days'." |
| Format | Matches the handover format exactly | Paste the format again at the end of the description |
| No news day | Says `NO MATERIAL DISRUPTION FOUND` instead of inventing | Strengthen the "never invent" line |

Also run it once with `SIMULATE_SEARCH_FAILURE=true` and confirm the log shows `[RECOVERY] Serper attempt 1/3 failed: simulated...`, followed by success.

### Step 6: Write `tests/test_search_tool.py` (45 min)

**Tool: pytest** with `monkeypatch`, which swaps real services for fakes so tests don't use your search credits.

```python
import tools.search_tool as st

def test_formats_serper_results(monkeypatch):
    monkeypatch.setenv("SERPER_API_KEY", "x")
    class FakeSerper:
        def __init__(self, **kw): pass
        def results(self, q): return {"news": [{"title": "Typhoon shuts Kaohsiung", "link": "https://n/1",
                                                "snippet": "Port closed", "date": "1 day ago", "source": "Reuters"}]}
    monkeypatch.setattr(st, "GoogleSerperAPIWrapper", FakeSerper)
    st._state.update(calls=0, simulate=False)
    out = st.web_search.run(query="Kaohsiung typhoon")
    assert "Typhoon shuts Kaohsiung" in out and "https://n/1" in out

def test_simulated_failure_then_recovers(monkeypatch):
    monkeypatch.setenv("SERPER_API_KEY", "x")
    monkeypatch.setattr(st.time, "sleep", lambda s: None)
    class FakeSerper:
        def __init__(self, **kw): pass
        def results(self, q): return {"news": [{"title": "OK", "link": "u"}]}
    monkeypatch.setattr(st, "GoogleSerperAPIWrapper", FakeSerper)
    st._state.update(calls=0); st.enable_simulated_failure(True)
    assert "OK" in st.web_search.run(query="q")
    st.enable_simulated_failure(False)

def test_all_providers_down_returns_message(monkeypatch):
    monkeypatch.delenv("SERPER_API_KEY", raising=False)
    import ddgs
    class Broken:
        def news(self, *a, **k): raise RuntimeError("down")
    monkeypatch.setattr(ddgs, "DDGS", Broken)
    out = st.web_search.run(query="q")
    assert out.startswith("SEARCH UNAVAILABLE")

def test_empty_query_never_crashes():
    assert st.web_search.run(query="").startswith("SEARCH UNAVAILABLE")
```

Run it with `python -m pytest -q`; all 4 should pass.

> Data-quality fix: `enable_simulated_failure(True)` now resets the call counter so the
> `--simulate-search-failure` demo always fails the *next* search on camera, and the env var
> is read dynamically (setting it in `.env` after import works). The `ddgs` /
> `duckduckgo_search` dual import prevents `ModuleNotFoundError` across laptops.

### Step 7: Write `docs/scout.md` (30 min)

Include:
1. How the tool and agent work, in 5 lines.
2. The fallback chain: Serper ×3 → DuckDuckGo → message.
3. How to turn on the simulated failure.
4. **Two saved sample outputs**, pasted from real runs:
   - **(a)** a run with the typhoon focus
   - **(b)** a normal run

   Person 5 uses these to test the Analyst, and Person 6 uses them to test the Coordinator. If there is no real typhoon, write sample (a) by hand in the exact handover format: Kaohsiung and Keelung closed, 14–21 days, worst case 21.

   **(a) Canonical typhoon sample (copy-paste for P5/P6 tests — worst-case 21 drives all downstream maths):**
   ```
   ## Disruption events
   1. Location/port: Kaohsiung port | Country: Taiwan
      Type: weather (typhoon) - port closure
      Date: 2026-10-01
      Delay range: 14-21 days | Worst-case delay: 21 days
      Confidence: high
      Sources: https://example.com/typhoon-kaohsiung-closure, https://example.com/taiwan-ports-shut
   2. Location/port: Keelung port | Country: Taiwan
      Type: port closure
      Date: 2026-10-01
      Delay range: 14-21 days | Worst-case delay: 21 days
      Confidence: high
      Sources: https://example.com/keelung-port-typhoon
   ## Verdict
   MATERIAL DISRUPTION FOUND
   ```
   **(b) No-disruption sample (Analyst must return empty tables):**
   ```
   ## Disruption events
   ## Verdict
   NO MATERIAL DISRUPTION FOUND
   ```

### Step 8: Hand over (5 min)

Tag `v1.0` (A9). Then message Person 5 and Person 6: "Scout ready; sample outputs in `docs/scout.md`; `enable_simulated_failure()` is in `tools.search_tool`."

## Part B+: Research & improve (your ownership — no copy-paste solution here)

1. **Confidence calibration.** "High/medium/low" is vague. Research how news sources signal reliability (2 independent sources per SOP L1? recency? official port authority vs aggregator?). Propose your own rubric in `docs/scout.md` and test it over 5 runs — do not hard-code news.
2. **Geofencing to watchlist.** The agent must ignore events outside Taiwan/China/India lanes. Research prompt phrasing that reduces off-watchlist hits without listing every city. Experiment with "ignore events outside {watchlist} countries" variants.
3. **No-news behaviour.** On a quiet news day the correct output is `NO MATERIAL DISRUPTION FOUND`. Research how to prompt against hallucination (explicit "never invent URLs", require copied URLs). Save one real no-event run as sample (b).

## Part C: Your done checklist

- [ ] `web_search` returns formatted news with URLs
- [ ] Simulated failure shows `[RECOVERY]` and then succeeds
- [ ] With no Serper key, DuckDuckGo is used
- [ ] `python -m agents.scout` gives output in the exact handover format
- [ ] 5 tuning runs done; no invented URLs seen
- [ ] `python -m pytest -q` passes
- [ ] `docs/scout.md` has 2 sample outputs
- [ ] Only your paths committed, no `.env`; `v1.0` tagged

## Part D: Common problems

| Problem | Fix |
|---|---|
| `ModuleNotFoundError: tools` | Run from the repo root with `python -m agents.scout`, not `python agents/scout.py` |
| Agent does only 1 search | Make the "AT LEAST 3 distinct searches" line more prominent; lower temperature to 0.1 |
| DuckDuckGo "rate limit" | Wait a minute; it's only the fallback |
| Serper 403 | Wrong key or the credits are used up; check the serper.dev dashboard |
| Agent loops too long | Keep `max_iter=15`; make sure queries are short |
