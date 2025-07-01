---
title: "Using AI to generate repetitive code automatically"
seoTitle: "Using AI to generate code"
seoDescription: "GreyhoundDash automates service integrations for a self-hosted monitoring dashboard with LLM-generated code and batch GitHub PRs—zero manual boilerplate."
datePublished: Tue Jul 01 2025 20:54:42 GMT+0000 (Coordinated Universal Time)
cuid: cmcl0alu1000j02l83hve9h6q
slug: using-ai-to-generate-repetitive-code-automatically
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/UopUfxghnWo/upload/e387920592d49e8e8b53dc29687af678.jpeg
tags: ai, tooling, boilerplate, ci-cd, llm

---

[https://github.com/JTSG1/GreyhoundDash](https://github.com/JTSG1/GreyhoundDash)

I’m building **GreyhoundDash**, a monitoring dashboard that can run either self-hosted or as SaaS.

To recognise and health-check the long tail of open-source, self-hostable services, the dashboard needs a machine-readable definition for each one.

Hand-authoring those definitions doesn’t scale; the Awesome-Self-Hosted index alone lists hundreds of candidates.

Instead of grinding through them manually, I’m using large-language-model code generation to create the boilerplate classes automatically. This series documents the experiment—why I chose the approach, how the tooling is wired together, and what still needs tightening up.

## Requirements

**Core workflow**

1. Detect new service
    
    Scan the curated list (JSON produced from Awesome-Self-Hosted).
    
2. Generate code
    
    Produce a Python subclass of `ServiceBase` that performs a simple “is-alive” check.
    
3. Create PR
    
    Commit the file on a new branch, push, and open a pull-request back to `main`. Branch/PR names must be traceable (`feat/basic-svc-<slug>`, etc.).
    

| Step | What must happen | Notes |
| --- | --- | --- |
| 1\. Detect new service | Scan the curated list (JSON produced from Awesome-Self-Hosted). |  |
| 2\. Generate code | Produce a Python subclass of `ServiceBase` that performs a simple “is-alive” check. |  |
| Must compile and lint clean. |  |  |
| 3\. Create PR | Commit the file on a new branch, push, and open a pull-request back to `main`. Branch/PR names must be traceable (`feat/basic-svc-<slug>`, etc.). |  |

**Determinism & safety**

* Same input → same output. If the model drifts, retry up to *n* times, then quarantine the service.
    
* Validate the JSON schema with *Pydantic* before writing to disk; on failure, log and skip.
    

**Cost & noise controls**

* **Batching:** group *10 services per PR* (`PR_FREQUENCY`) so reviewers aren’t flooded.
    
* **Budget guard:** stop once the estimated OpenAI spend or token quota for the run is hit.
    
* **Checkpointing:** persist a `completed_services.json` so reruns pick up where they left off.
    

These constraints keep the process affordable and the repository maintainable while still letting the LLM do the grunt work. This article will focus on the first and third points.

## Harvesting candidate services

**Source:** [https://awesome-selfhosted.net/#software](https://awesome-selfhosted.net/#software)  
**Goal**: Iterate over every entry and convert to a JSON format for easy parsing

This bit was relatively straight forwards, I decided to parse the content on this page [https://awesome-selfhosted.net/#software](https://awesome-selfhosted.net/#software) (awesome project that I fully endorse) using the BS4 lib. I won’t go into specifics here but I will share the code:

**Extraction pipeline:**

1: Fetch the HTML

```python
def fetch_awesome_selfhosted():
    url = "https://awesome-selfhosted.net/#software"
    response = requests.get(url)
    if response.status_code != 200:
        raise Exception(f"Failed to fetch the page: {response.status_code}")
    return response.text
```

2: Parse with Beautiful Soup

BS is a great Python library used for parsing data from web pages, often used for web scraping. Parsing over HTML can be quite delicate, as any changes to the structure of the page can throw off the code.

```python
def extract_services(html: str) -> dict[str, dict]:
    soup = BeautifulSoup(html, "html.parser")
    services, skipped = {}, []

    for s in soup.select("#software section"):
        name = s.h3.get_text(strip=True)
        link = s.select_one("a.external-link")
        if not link:
            skipped.append(name); continue
        services[to_camel(name)] = {
            "name": name,
            "description": (s.p or "").get_text(strip=True),
            "homepage": link["href"],
        }

    if skipped:
        logging.warning("Skipped %d services without a homepage: %s",
                        len(skipped), ", ".join(skipped))
    return services
```

3: Persist

The list should be persisted so it can be used as input to the Language Model.

```python
with open("tooling/awesome_selfhosted_services.json", "w") as f:
    import json
    json.dump(result, f, indent=2)
```

The scraper gives me a one-shot JSON cache with exactly the three fields the generator needs—`name`, `description`, `homepage`. Whenever Awesome-Self-Hosted changes, I just rerun the script and the new entries appear in the file.

It’s deliberately minimal:

* **No retry/back-off** and no DOM diffing—acceptable for tooling, not for prod.
    
* Relies on the current page structure; if the maintainers rename a class, the selector fails and I’ll fix it then.
    
* Runs on demand, outside the dashboard’s runtime path, so a crash can’t hurt users.
    

I also persist a `awesome_selfhosted_services_completed.json` checkpoint. Each code-gen run looks at that file first, skips anything already merged, and works on the remainder. Preventing duplicate processing and allows the code to pick what hasn’t been done only.

## Generating the service stubs

The LLM must return something I can deserialize without guess-work, so the interaction is *schema-first*:

1. **Schema contract**  
    A single `ServiceResponseSchema` (Pydantic) drives both the OpenAI call and local validation:
    
    ```python
    class ServiceResponseSchema(BaseModel):
        file_name: str      # e.g. "service_grafana.py"
        service: str        # Human-readable name
        code: str           # The Python subclass itself
        logo: str           # URL or data-URI (future feature)
        web_app: bool       # True ⇢ include, False ⇢ skip
    ```
    
    Any field mismatch raises immediately; no fragile `json.loads()` spelunking.
    
2. **Prompt template**  
    Stored in `tooling/service_generation_prompt.txt` with Jinja-like tokens:
    
    ````plaintext
    You are ChatGPT. Generate a subclass of {{ base_class_name }}.
    
    ``` python
    {{ base_class_file }}
    ```
    
    Sample implementations of this base class:
    
    ``` python
    {{ implementation_samples }}
    ```
    Response must comply with the below schema:
    
    ``` json
    {{ json_schema }}
    ```
    ````
    
    At runtime the script simply `.replace()`s the placeholders—no template engine needed.
    

*Why JSON over code blocks?*  
Without a rigid envelope, the model may prepend helper text or miss imports, which breaks CI. JSON gives me a single thing to parse—and Pydantic enforces it at runtime.

*Why a bare string replace instead of Jinja?*  
The prompt is *configuration*, not *application* code; simplicity beats another dependency.

Future cleanup: swap the naïve `.replace()` for `str.format_map()` so accidental `{{ }}` inside comments don’t expand.

## Wiring the pipeline

The automation lives in `/tooling` and is split into two well-defined actors:

| Actor | Responsibility | Why separate? |
| --- | --- | --- |
| `ServiceGenerator` | \- Iterate over the JSON cache |  |
| \- Call OpenAI, validate the `ServiceResponseSchema` |  |  |
| \- Write the new `service_*.py` files |  |  |
| \- Update `awesome_selfhosted_services_completed.json` | Pure “business logic” & prompt-handling; no Git side-effects. |  |
| `GithubPRCreator` | \- Spawn a branch |  |
| \- `git add/commit/push` |  |  |
| \- Open a pull-request via GitHub API | Keeps VCS concerns isolated; lets you swap GitHub for GitLab later. |  |

#### `GithubPRCreator` (excerpt)

```python
pythonCopyEditdef create_branch_and_raise_pr(self, branch, services):
    # 1. git checkout -b <branch>
    # 2. git add -A && git commit -m "feat: ..."
    # 3. git push -u origin <branch>
    pr = self.repo.create_pull(
        title=f"gen: {services[0]} → {services[-1]}",
        body="\n".join(f"{n}: {ok}" for n, ok in services),
        head=branch,
        base="main",
    )
```

*Guard rails*

* Refuses to reuse an existing branch (fast-fail).
    
* Logs the PR URL so CI can post a comment or Slack alert later.
    

#### `ServiceGenerator` (key points)

```python
PR_FREQUENCY = 10   # services per PR
BATCH_SIZE   = 10   # services per run (keeps token spend predictable)
GPT_MODEL    = "gpt-4o-mini"
```

1. **Prompt fill** → deterministic JSON response (temperature = 0).
    
2. **Schema validation** with Pydantic; on failure retry ≤ 5 then skip.
    
3. **Write file** only when `web_app is True`.
    
4. **Book-keeping** — append to `completed.json` so reruns resume cleanly.
    
5. Every `PR_FREQUENCY` successes → delegate to `GithubPRCreator`.
    

#### Entrypoint

```python
if __name__ == "__main__":
    gh = GithubPRCreator(Github(os.getenv("GH_TOKEN")), "JTSG1/GreyhoundDash")
    ServiceGenerator(openai_client=OpenAI(), github_pr_creator=gh).iterate_services()
```

The `main` block wires the two actors together; nothing else runs in global scope, which makes the module import-safe for future unit tests.

---

*Why this structure?*  
A tight, two-class design keeps the surface area small: one side talks to the LLM, the other to Git. Each part can be mocked in isolation, batteries included for retries, cost limits, and PR hygiene.Wrapping up

### Wrap-up

* **What’s solved?**  
    GreyhoundDash can now turn a raw list of open-source services into ready-to-review Python stubs, batch-commit them, and open pull-requests—all without a human typing the same boilerplate thirty times.
    
* **What’s still brittle?**
    
    * Scraper depends on Awesome-Self-Hosted’s HTML staying stable.
        
    * Generated code is unit-tested only for syntax, not runtime behaviour.
        
    * Logo retrieval and licence compliance are TODOs.
        
    * OpenAI cost control is coarse (batch ceiling rather than per-token budgeting).
        
* **Immediate next steps**
    
    1. Add a CI job that lints each generated file and executes a dummy health-check.
        
    2. Swap the HTML scrape for the project’s YAML source to decouple from DOM churn.
        
    3. Enforce a spend cap via the OpenAI usage API and abort runs that hit it.
        
    4. Store prompt + schema version in git so regressions are traceable.
        
* **Calling for help**
    
    * Spot a service the scraper missed? Open an issue.
        
    * See a way to harden the GitHub automation (e.g., branch naming, PAT scope)? Let me know.
        

The repo is at [**github.com/JTSG1/GreyhoundDash**](http://github.com/JTSG1/GreyhoundDash). Star it and keep up to date on the development!