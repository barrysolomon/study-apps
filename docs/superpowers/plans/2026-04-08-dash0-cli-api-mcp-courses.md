# Dash0 CLI / API / MCP Crash Courses — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build three hands-on crash course apps (CLI, API, MCP) with full content at three depth levels (Sales Rep, Solutions Architect, Engineer), plus companion study guides.

**Architecture:** Each course is a single self-contained HTML file built from the v3.1 template. Content is populated with flashcards, quiz questions, scenarios, rapid fire, lessons with exercises, schedule, and checkpoints. The three courses are independent but cross-linked.

**Tech Stack:** Vanilla JS, CSS classes in `<style>` block, single-file HTML apps, localStorage persistence.

**Spec:** `docs/superpowers/specs/2026-04-08-dash0-cli-api-mcp-courses-design.md`

**Depends on:** Plan A (Platform v3.1 Engine Upgrades) must be completed first — courses are built from the upgraded template.

---

## File Map

| File | Action |
|------|--------|
| `public-courses/dash0-cli-crash-course.html` | Create: CLI crash course |
| `public-courses/dash0-api-crash-course.html` | Create: API crash course |
| `public-courses/dash0-mcp-crash-course.html` | Create: MCP crash course |
| `public-courses/dash0-cli-study-guide.md` | Create: CLI study guide |
| `public-courses/dash0-api-study-guide.md` | Create: API study guide |
| `public-courses/dash0-mcp-study-guide.md` | Create: MCP study guide |

---

## Content Research Sources (for all three courses)

Before writing content, the implementer MUST research current documentation to ensure accuracy:

**CLI:**
- `github.com/dash0hq/dash0-cli` README and help output
- `dash0.com/changelog/dash0-cli-ga`
- `dash0.com/changelog/cli-manage-assets`
- `dash0.com/changelog/cli-agent-mode`

**API:**
- `dash0.com/documentation/dash0/api`
- `dash0.com/documentation/dash0/key-concepts/endpoints`
- `dash0.com/documentation/dash0/key-concepts/auth-tokens`
- `dash0.com/documentation/dash0/metrics/promql-query-patterns`
- `dash0.com/documentation/dash0/dashboards/managing-dashboards-as-code`

**MCP:**
- `github.com/dash0hq/mcp-dash0`
- `dash0.com/docs/dash0/ai/mcp`
- `dash0.com/changelog/dash0-mcp-server`
- `dash0.com/hub/integrations?tag=AI`
- `github.com/dash0hq/dash0-examples` (kagent agent-observability.yaml)

---

### Task 1: Build CLI Course — Scaffold and Configuration

**Files:**
- Create: `public-courses/dash0-cli-crash-course.html`

- [ ] **Step 1: Copy template**

Copy `setup/crash-course-TEMPLATE.html` to `public-courses/dash0-cli-crash-course.html`.

- [ ] **Step 2: Update metadata**

Replace all TODO markers with CLI-specific content:

```javascript
// Title and subtitle in HTML header:
// <h1>Dash0 CLI Crash Course</h1>
// <small>From install to GitOps mastery</small>

const APP_VERSION='v1.0';
const PLATFORM_VERSION='v3.1';
const STORAGE_KEY='dash0-cli-cc';
```

- [ ] **Step 3: Set categories**

```javascript
const CATEGORIES=["All","Fundamentals","Installation & Setup","Asset Management","GitOps & Apply","Querying","Sending Telemetry","Agent Mode & CI/CD","Pitch & Positioning"];
```

- [ ] **Step 4: Set ROLE_CONFIG**

```javascript
const ROLE_CONFIG={
role1:{label:'Sales Rep',icon:'💼',desc:'Conversational fluency. Pitch points, competitive positioning, objection handling.',
coreCategories:['Fundamentals','Pitch & Positioning'],
coreLessons:['lesson-1','lesson-3','lesson-8'],
categoryOrder:['All','Fundamentals','Pitch & Positioning','Asset Management','GitOps & Apply','Installation & Setup','Querying','Sending Telemetry','Agent Mode & CI/CD']},
role2:{label:'Solutions Architect',icon:'🏗️',desc:'Demo-ready working knowledge. Architecture, workflows, customer Q&A.',
coreCategories:['Fundamentals','Installation & Setup','Asset Management','GitOps & Apply','Querying','Pitch & Positioning'],
coreLessons:['lesson-1','lesson-2','lesson-3','lesson-4','lesson-5','lesson-6','lesson-8'],
categoryOrder:['All','Fundamentals','Installation & Setup','Asset Management','GitOps & Apply','Querying','Sending Telemetry','Agent Mode & CI/CD','Pitch & Positioning']},
role3:{label:'Engineer / DevOps',icon:'⚡',desc:'Deep technical. Hands-on with commands, YAML, APIs, troubleshooting.',
coreCategories:['Fundamentals','Installation & Setup','Asset Management','GitOps & Apply','Querying','Sending Telemetry','Agent Mode & CI/CD','Pitch & Positioning'],
coreLessons:['lesson-1','lesson-2','lesson-3','lesson-4','lesson-5','lesson-6','lesson-7','lesson-8'],
categoryOrder:['All','Installation & Setup','Asset Management','GitOps & Apply','Querying','Sending Telemetry','Agent Mode & CI/CD','Fundamentals','Pitch & Positioning']}
};
```

- [ ] **Step 5: Set RELATED_COURSES**

```javascript
const RELATED_COURSES=[
{title:"Dash0 API Crash Course",url:"dash0-api-crash-course.html",icon:"🌐",desc:"What happens under the hood — HTTP endpoints, OTLP, PromQL"},
{title:"Dash0 MCP Crash Course",url:"dash0-mcp-crash-course.html",icon:"🧠",desc:"AI-native access to the same platform via MCP tools"}
];
```

- [ ] **Step 6: Set schedule and checkpoints**

Use the default schedule from the spec (section 5.5) with CLI-specific titles:

```javascript
const DEFAULT_SESSIONS=[
{id:"s1",date:"2026-04-14",time:"09:00",dur:30,title:"CLI Foundations",desc:"Lessons 1-2. Install the CLI, set up profiles. Review fundamentals flashcards.",phase:1},
{id:"s2",date:"2026-04-15",time:"09:00",dur:30,title:"Asset Management",desc:"Lesson 3. CRUD dashboards, check rules, views, synthetic checks.",phase:1},
{id:"s3",date:"2026-04-16",time:"09:00",dur:30,title:"Flashcards + Scenarios",desc:"All flashcards by category. Practice first 4 scenarios.",phase:1},
{id:"s4",date:"2026-04-21",time:"09:00",dur:45,title:"GitOps + Querying",desc:"Lessons 4-5. dash0 apply, log/metric/span queries.",phase:2},
{id:"s5",date:"2026-04-22",time:"09:00",dur:45,title:"Sending Telemetry + Rapid Fire",desc:"Lesson 6. Send logs/spans. Rapid fire round. Review weak cards.",phase:2},
{id:"s6",date:"2026-04-23",time:"09:00",dur:30,title:"Agent Mode + Competitive",desc:"Lessons 7-8. CI/CD integration, competitive positioning.",phase:3},
{id:"s7",date:"2026-04-24",time:"09:00",dur:45,title:"Final Review",desc:"All scenarios, rapid fire 85%+, final exam. Review weak areas.",phase:3},
];
const DEFAULT_CHECKPOINTS=[
{id:"cp1",target:"Score 80%+ on Fundamentals quiz",by:"2026-04-16",metric:"bestQuiz",threshold:80},
{id:"cp2",target:"Complete all Phase 1 exercises",by:"2026-04-17",metric:"exercises",threshold:40},
{id:"cp3",target:"Rapid fire 85%+",by:"2026-04-23",metric:"bestRapid",threshold:85},
{id:"cp4",target:"Score 90%+ on Final Exam",by:"2026-04-24",metric:"bestQuiz",threshold:90},
];
```

- [ ] **Step 7: Commit scaffold**

```bash
git add public-courses/dash0-cli-crash-course.html
git commit -m "feat(cli-course): scaffold from v3.1 template with config"
```

---

### Task 2: Build CLI Course — Flashcards

**Files:**
- Modify: `public-courses/dash0-cli-crash-course.html`

- [ ] **Step 1: Research current CLI commands**

Run `brew install dash0` or check `github.com/dash0hq/dash0-cli` README for the latest command reference. Web search for recent changelog entries.

- [ ] **Step 2: Write 65 flashcards**

Populate the `FLASHCARDS` array with 65 flashcards distributed across categories and difficulty levels:

| Category | Target Count | Difficulty Mix |
|----------|-------------|----------------|
| Fundamentals | 8 | 5 beginner, 3 intermediate |
| Installation & Setup | 7 | 4 beginner, 3 intermediate |
| Asset Management | 10 | 3 beginner, 5 intermediate, 2 advanced |
| GitOps & Apply | 9 | 2 beginner, 5 intermediate, 2 advanced |
| Querying | 9 | 2 beginner, 4 intermediate, 3 advanced |
| Sending Telemetry | 7 | 2 beginner, 3 intermediate, 2 advanced |
| Agent Mode & CI/CD | 8 | 2 beginner, 4 intermediate, 2 advanced |
| Pitch & Positioning | 7 | 4 beginner, 3 intermediate |

Each flashcard: `{f:"question", b:"answer", c:"Category", u:"https://source-url", d:1|2|3}`

Every card must have a valid `u` URL pointing to the specific documentation page covering that topic (not a homepage or generic page).

Example cards per category:

**Fundamentals:**
```javascript
{f:"What is the Dash0 CLI?",b:"A command-line tool for managing Dash0 observability assets, querying telemetry, and sending data — the human-native interface to the Dash0 platform.",c:"Fundamentals",u:"https://github.com/dash0hq/dash0-cli",d:1},
{f:"What are the three Dash0 platform interfaces?",b:"CLI (human-native terminal), API (machine-native HTTP), and MCP Server (AI-native Model Context Protocol).",c:"Fundamentals",u:"https://www.dash0.com/changelog/dash0-cli-ga",d:1},
```

**Installation & Setup:**
```javascript
{f:"How do you install the Dash0 CLI via Homebrew?",b:"brew tap dash0hq/dash0-cli https://github.com/dash0hq/dash0-cli && brew install dash0",c:"Installation & Setup",u:"https://github.com/dash0hq/dash0-cli#installation",d:1},
{f:"How do you create a named CLI profile?",b:"dash0 config profiles create <name> --api-url <url> --auth-token <token>",c:"Installation & Setup",u:"https://github.com/dash0hq/dash0-cli#configuration",d:2},
```

**Pitch & Positioning:**
```javascript
{f:"What's the 30-second elevator pitch for the Dash0 CLI?",b:"One tool for all your observability operations — manage dashboards, query logs/metrics/traces, deploy config as code, and integrate with CI/CD. Works in terminal, in pipelines, and even in AI agents via agent mode.",c:"Pitch & Positioning",u:"https://www.dash0.com/changelog/dash0-cli-ga",d:1},
```

Continue for all 65 cards. All URLs must be verified.

- [ ] **Step 3: Commit flashcards**

```bash
git add public-courses/dash0-cli-crash-course.html
git commit -m "feat(cli-course): add 65 flashcards across 8 categories"
```

---

### Task 3: Build CLI Course — Quiz Questions

**Files:**
- Modify: `public-courses/dash0-cli-crash-course.html`

- [ ] **Step 1: Write 45 quiz questions**

Populate the `QUIZ` array with 45 questions distributed across categories:

| Category | Count | Difficulty Mix |
|----------|-------|----------------|
| Fundamentals | 6 | 3B, 2I, 1A |
| Installation & Setup | 5 | 2B, 2I, 1A |
| Asset Management | 7 | 2B, 3I, 2A |
| GitOps & Apply | 6 | 1B, 3I, 2A |
| Querying | 6 | 1B, 3I, 2A |
| Sending Telemetry | 5 | 1B, 2I, 2A |
| Agent Mode & CI/CD | 5 | 1B, 3I, 1A |
| Pitch & Positioning | 5 | 2B, 2I, 1A |

Each: `{q:"question", o:["A","B","C","D"], a:correctIndex, cat:"Category", u:"url", e:"explanation", d:1|2|3}`

Include `Dash0 angle` in explanations where natural.

Example:
```javascript
{q:"Which command applies observability assets from a YAML file?",o:["dash0 create -f file.yaml","dash0 apply -f file.yaml","dash0 deploy -f file.yaml","dash0 push -f file.yaml"],a:1,cat:"GitOps & Apply",u:"https://github.com/dash0hq/dash0-cli#apply",e:"dash0 apply -f auto-detects create vs update, supports multi-doc YAML and directory recursion. Similar to kubectl apply.",d:2},
```

- [ ] **Step 2: Commit quiz**

```bash
git add public-courses/dash0-cli-crash-course.html
git commit -m "feat(cli-course): add 45 quiz questions across 8 categories"
```

---

### Task 4: Build CLI Course — Scenarios, Rapid Fire, and Lessons

**Files:**
- Modify: `public-courses/dash0-cli-crash-course.html`

- [ ] **Step 1: Write 12 scenario drills**

Mix of sales objections, SA demos, and deep technical:

```javascript
const SCENARIOS=[
// Sales (4):
{q:"A prospect says: 'We already manage everything through the Datadog UI. Why would we need a CLI?'",a:"Great that you have a workflow! The CLI unlocks something the UI can't: version-controlled observability. Your dashboards and alert rules live in YAML files checked into git, applied via CI/CD — same GitOps workflow your engineering team uses for infrastructure. If someone accidentally deletes a dashboard, you just re-apply from git. Try that with a UI-only tool.",u:["https://www.dash0.com/changelog/cli-manage-assets"]},
// ... 11 more
];
```

- [ ] **Step 2: Write 15 rapid fire questions**

```javascript
const RAPID=[
{q:"Command to list all dashboards?",a:"dash0 dashboards list",t:6,u:"https://github.com/dash0hq/dash0-cli"},
{q:"How do you install the CLI on macOS?",a:"brew tap dash0hq/dash0-cli && brew install dash0",t:10,u:"https://github.com/dash0hq/dash0-cli#installation"},
// ... 13 more
];
```

- [ ] **Step 3: Write 8 lessons with exercises**

Each lesson: `{id, title, time, week, icon, body (HTML), sources:[], exercises:[{task,hint,url,steps:[]}]}`

Lessons follow the structure from the spec (Section 5.1 CLI Course Lessons):

1. What is the Dash0 CLI? (Phase 1)
2. Installation & Profiles (Phase 1)
3. Asset Management CRUD (Phase 1)
4. GitOps with dash0 apply (Phase 2)
5. Querying Telemetry (Phase 2)
6. Sending Telemetry (Phase 2)
7. Agent Mode & CI/CD (Phase 3)
8. Competitive Positioning (Phase 3)

Each lesson body uses proper CSS classes: `.lesson-section`, `.lesson-text`, `.lesson-highlight`, `.code-block` with `codeBlock()`, `.lesson-compare` where appropriate.

Exercises include step-by-step guides with actual CLI commands. Example exercise for Lesson 3:

```javascript
{task:"Create a dashboard from a YAML file using the CLI.",
hint:"Use dash0 dashboards create -f <file>",
steps:[
"Create a file called my-dashboard.yaml with a Perses dashboard definition",
"Run: dash0 dashboards create -f my-dashboard.yaml",
"Run: dash0 dashboards list -o table — your dashboard should appear",
"Run: dash0 dashboards get <id> -o yaml — verify the full definition",
"Modify the YAML and run: dash0 dashboards update <id> -f my-dashboard.yaml",
"Run: dash0 dashboards delete <id> to clean up"
]}
```

- [ ] **Step 4: Update Resources tab**

Add CLI-specific resources to `renderResourcesTab`:
- GitHub repo
- CLI GA changelog
- Asset management changelog
- Agent mode changelog
- Dash0 documentation

- [ ] **Step 5: Commit**

```bash
git add public-courses/dash0-cli-crash-course.html
git commit -m "feat(cli-course): add scenarios, rapid fire, lessons, and resources"
```

---

### Task 5: Build API Course — Full Content

**Files:**
- Create: `public-courses/dash0-api-crash-course.html`

- [ ] **Step 1: Copy template and configure**

Copy `setup/crash-course-TEMPLATE.html` to `public-courses/dash0-api-crash-course.html`.

Set metadata:
```javascript
// <h1>Dash0 API Crash Course</h1>
// <small>From auth tokens to observability-as-code</small>
const APP_VERSION='v1.0';
const PLATFORM_VERSION='v3.1';
const STORAGE_KEY='dash0-api-cc';
```

Categories:
```javascript
const CATEGORIES=["All","Fundamentals","Auth & Endpoints","OTLP Ingestion","Prometheus Queries","Management API","Observability-as-Code","Integrations","Pitch & Positioning"];
```

ROLE_CONFIG: same structure as CLI but with API-specific core categories:
- Sales Rep: core = Fundamentals, Pitch & Positioning
- SA: core = all except Agent Mode (no Agent Mode category in API)
- Engineer: all core

RELATED_COURSES:
```javascript
const RELATED_COURSES=[
{title:"Dash0 CLI Crash Course",url:"dash0-cli-crash-course.html",icon:"🖥️",desc:"Developer-friendly terminal interface to the API"},
{title:"Dash0 MCP Crash Course",url:"dash0-mcp-crash-course.html",icon:"🧠",desc:"AI-native access via Model Context Protocol"}
];
```

- [ ] **Step 2: Write 65 flashcards**

Same distribution as CLI course but API-specific content: auth tokens, Bearer vs Basic auth, OTLP endpoints, gRPC vs HTTP, PromQL API paths, management endpoints, Perses format, Prometheus CRDs.

- [ ] **Step 3: Write 45 quiz questions**

API-specific: endpoint URLs, auth scopes, OTLP payload formats, PromQL API parameters, dashboard JSON structure, check rule format.

- [ ] **Step 4: Write 12 scenarios**

API-specific objections and technical Q&A: "Why should I use your API instead of Prometheus directly?", "How do I migrate my Grafana dashboards?", "Can I use your API with my existing Prometheus alerting rules?"

- [ ] **Step 5: Write 15 rapid fire questions**

API-specific: port numbers, endpoint paths, auth header format, metric types supported.

- [ ] **Step 6: Write 8 lessons with exercises**

Following spec Section 5.1 API Course Lessons. Exercises include curl commands for API calls, YAML examples for dashboards-as-code, PromQL queries via the API.

- [ ] **Step 7: Set schedule, checkpoints, resources**

- [ ] **Step 8: Commit**

```bash
git add public-courses/dash0-api-crash-course.html
git commit -m "feat(api-course): complete API crash course with full content"
```

---

### Task 6: Build MCP Course — Full Content

**Files:**
- Create: `public-courses/dash0-mcp-crash-course.html`

- [ ] **Step 1: Copy template and configure**

Set metadata:
```javascript
// <h1>Dash0 MCP Crash Course</h1>
// <small>From setup to AI-powered observability</small>
const APP_VERSION='v1.0';
const PLATFORM_VERSION='v3.1';
const STORAGE_KEY='dash0-mcp-cc';
```

Categories:
```javascript
const CATEGORIES=["All","Fundamentals","Setup & Config","Service Discovery","Logs & Traces","Metrics & PromQL","Synthetic & Web","K8s & Dashboards","Pitch & Positioning"];
```

RELATED_COURSES:
```javascript
const RELATED_COURSES=[
{title:"Dash0 CLI Crash Course",url:"dash0-cli-crash-course.html",icon:"🖥️",desc:"Terminal workflows for CI/CD and GitOps"},
{title:"Dash0 API Crash Course",url:"dash0-api-crash-course.html",icon:"🌐",desc:"Direct HTTP access for custom integrations"}
];
```

- [ ] **Step 2: Write 65 flashcards**

MCP-specific: 23 MCP tools, setup for each IDE, Streamable HTTP, remote hosted, natural language patterns, tool names and what they return.

- [ ] **Step 3: Write 45 quiz questions**

MCP-specific: which tool does X, setup configuration, tool parameters, when to use MCP vs CLI vs API.

- [ ] **Step 4: Write 12 scenarios**

MCP-specific: "A customer asks how MCP differs from just using the CLI in agent mode", "Show me how to debug a service degradation using only natural language", "A security team wants to know what data the MCP server can access".

- [ ] **Step 5: Write 15 rapid fire**

Tool name recall: "What tool lists all services?", "What's the MCP endpoint URL format?", "How many tools does the Dash0 MCP server expose?"

- [ ] **Step 6: Write 8 lessons with exercises**

Following spec Section 5.1 MCP Course Lessons. Exercises include setting up MCP in Claude Code, querying services with natural language, investigating incidents.

- [ ] **Step 7: Set schedule, checkpoints, resources**

- [ ] **Step 8: Commit**

```bash
git add public-courses/dash0-mcp-crash-course.html
git commit -m "feat(mcp-course): complete MCP crash course with full content"
```

---

### Task 7: Write CLI Study Guide

**Files:**
- Create: `public-courses/dash0-cli-study-guide.md`

- [ ] **Step 1: Write study guide**

Structure:
1. Course overview and prerequisites
2. Phase 1: Foundations (CLI basics, install, profiles, first commands)
3. Phase 2: Intermediate (asset CRUD, GitOps, querying, sending telemetry)
4. Phase 3: Advanced (agent mode, CI/CD, competitive positioning)
5. Command reference table (all commands with synopsis)
6. Scenario drills with model answers (spoiler-tagged with `<details><summary>`)
7. Exercises with step-by-step guides
8. Competitive cheat sheet (CLI vs Datadog/Grafana/Dynatrace CLIs)
9. Quick reference card

- [ ] **Step 2: Commit**

```bash
git add public-courses/dash0-cli-study-guide.md
git commit -m "docs(cli-course): add companion study guide"
```

---

### Task 8: Write API Study Guide

**Files:**
- Create: `public-courses/dash0-api-study-guide.md`

- [ ] **Step 1: Write study guide**

Same structure as CLI guide, adapted for API content:
- Endpoint reference table (all API paths with methods)
- Auth token setup walkthrough
- curl examples for common operations
- PromQL query patterns via API
- Dashboard-as-code YAML examples

- [ ] **Step 2: Commit**

```bash
git add public-courses/dash0-api-study-guide.md
git commit -m "docs(api-course): add companion study guide"
```

---

### Task 9: Write MCP Study Guide

**Files:**
- Create: `public-courses/dash0-mcp-study-guide.md`

- [ ] **Step 1: Write study guide**

Same structure, adapted for MCP content:
- All 23 MCP tools reference table
- Setup guides for each supported IDE
- Natural language query patterns
- MCP vs CLI vs API decision matrix
- Incident investigation workflows using MCP

- [ ] **Step 2: Commit**

```bash
git add public-courses/dash0-mcp-study-guide.md
git commit -m "docs(mcp-course): add companion study guide"
```

---

### Task 10: Link Validation

**Files:**
- All three course HTML files
- All three study guide MD files

- [ ] **Step 1: Extract all URLs from the three HTML files**

Use grep or a script to find all URLs in flashcard `u` fields, quiz `u` fields, scenario `u` arrays, lesson `sources` arrays, exercise `url` fields, resource tab links, and `RELATED_COURSES` URLs.

- [ ] **Step 2: Validate each URL**

Navigate to each URL and verify it returns a valid page (not 404). For Dash0 URLs, verify against current site navigation since Dash0 restructures frequently.

- [ ] **Step 3: Fix broken URLs**

Replace any broken URLs with the correct current equivalent. If content was removed, link to the closest parent page.

- [ ] **Step 4: Commit fixes**

```bash
git add public-courses/dash0-cli-crash-course.html public-courses/dash0-api-crash-course.html public-courses/dash0-mcp-crash-course.html
git commit -m "fix: validate and fix all URLs across CLI, API, MCP courses"
```

---

### Task 11: Final Verification

- [ ] **Step 1: Open each course in browser**

Verify for each of the three courses:
- Role picker appears on first load with Sales Rep / SA / Engineer
- Dashboard shows all 8 stat cards including Weak Cards
- Cards: category pills, difficulty pills, rating buttons all work
- Quiz: section picker shows all categories with difficulty distribution
- Scenarios: questions and answers render, source links work
- Rapid Fire: timer works, scoring works
- Learn: all 8 lessons render with exercises and step-by-step guides
- More: font size toggle, schedule, checkpoints, related courses in Resources

- [ ] **Step 2: Test cross-course links**

From each course's Resources tab, click the related course links — they should navigate to the other courses in the same directory.

- [ ] **Step 3: Test the index page**

Open `index.html` — all three new courses should appear (via API auto-discovery or fallback).

- [ ] **Step 4: Verify study guides render**

Open each `.md` file — verify the study guide link appears on the index page next to the course.
