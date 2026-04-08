# Dash0 CLI / API / MCP Crash Courses + Platform v3.1 Upgrades

**Date:** 2026-04-08
**Status:** Draft — awaiting approval
**Scope:** Three new crash courses + six platform engine upgrades backported to all apps

---

## 1. Overview

Build three hands-on, example-rich training courses for the Dash0 platform interfaces:

- **Dash0 CLI Crash Course** — the human-native terminal interface
- **Dash0 API Crash Course** — the machine-native HTTP interface
- **Dash0 MCP Crash Course** — the AI-native Model Context Protocol interface

All three are internal Dash0 team enablement (sales, SAs, engineers) published in `public-courses/` since the CLI, API, and MCP are public products.

Simultaneously, upgrade the crash course engine from `PLATFORM_VERSION v3.0` to `v3.1` with six new features, backported to all existing apps.

## 2. Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Depth levels | All three (conversational, working, deep technical) | Sales reps need conversational; engineers need hands-on |
| Audience | Internal Dash0 enablement | Competitive angles and sales talk tracks included |
| Location | `public-courses/` | CLI/API/MCP are public products |
| Roles | Sales Rep / Solutions Architect / Engineer | Maps cleanly to the three depth levels |
| Course relationship | Independent but cross-linked | Each stands alone; cross-links where topics overlap |
| Structure | Mirror layout (8 lessons each) | Predictable, easy to cross-link, fastest to build |
| Platform version | v3.1 bump across ALL apps | Six new engine features justify a minor version bump |

## 3. Deliverables

### 3.1 New Files (9)

| File | Type | Description |
|------|------|-------------|
| `public-courses/dash0-cli-crash-course.html` | Course | CLI crash course app |
| `public-courses/dash0-api-crash-course.html` | Course | API crash course app |
| `public-courses/dash0-mcp-crash-course.html` | Course | MCP crash course app |
| `public-courses/dash0-cli-study-guide.md` | Guide | Companion study guide for CLI |
| `public-courses/dash0-api-study-guide.md` | Guide | Companion study guide for API |
| `public-courses/dash0-mcp-study-guide.md` | Guide | Companion study guide for MCP |
| This spec file | Spec | Design document |

### 3.2 Files to Update (8)

| File | Change |
|------|--------|
| `setup/crash-course-TEMPLATE.html` | v3.1 engine upgrades (source of truth) |
| `public-courses/pca-crash-course.html` | v3.1 backport |
| `public-courses/dash0-terraform-crash-course.html` | v3.1 backport |
| `public-courses/gke-study-app.html` | v3.1 backport |
| `public-courses/otca-study-app.html` | v3.1 backport |
| `private-courses/pairwise-testing-crash-course.html` | v3.1 backport |
| `private-courses/dash0-value-framework-trainer.html` | v3.1 backport (React/Babel adaptation) |
| `index.html` | Update fallback array with 3 new courses |

### 3.3 No Changes Needed

- `README.md` — optional, not blocking
- `CLAUDE.md` — update after delivery to reflect new files and PLATFORM_VERSION

## 4. Platform v3.1 Engine Upgrades

Six features added to the engine. All implemented in template first, then propagated.

### 4.1 Font Size Settings (S/M/L/XL)

**Storage:** `${STORAGE_KEY}-fontsize` in localStorage (per-device, not synced).

**CSS classes on `<body>`:**
```css
.fs-s .card-text, .fs-s .lesson-text, .fs-s .quiz-option, .fs-s .scenario-box { font-size: 13px }
.fs-m .card-text, .fs-m .lesson-text, .fs-m .quiz-option, .fs-m .scenario-box { font-size: 15px }  /* default */
.fs-l .card-text, .fs-l .lesson-text, .fs-l .quiz-option, .fs-l .scenario-box { font-size: 17px }
.fs-xl .card-text, .fs-xl .lesson-text, .fs-xl .quiz-option, .fs-xl .scenario-box { font-size: 20px }
```

**UI:** Four-button toggle in Settings tab, between "Focus Level" and "Progress Summary" sections. Active button gets `#10b981` green. Selected size persisted on change.

**Init:** Read from localStorage on load, apply class to `<body>`. Default: `fs-m`.

### 4.2 Code Block Styling

**New CSS classes:**
```css
.code-block {
  position: relative;
  background: #0d1117;
  border: 1px solid #374151;
  border-radius: 10px;
  padding: 14px 16px;
  margin: 10px 0;
  font-family: 'SF Mono', 'Fira Code', 'Consolas', monospace;
  font-size: 13px;
  line-height: 1.6;
  color: #e6edf3;
  overflow-x: auto;
  white-space: pre;
  -webkit-overflow-scrolling: touch;
}
.code-inline {
  background: #1f2937;
  padding: 2px 6px;
  border-radius: 4px;
  font-family: 'SF Mono', 'Fira Code', 'Consolas', monospace;
  font-size: 0.9em;
  color: #a5f3fc;
}
```

**Lightweight syntax hints** (no full parser):
```javascript
function highlightCode(text) {
  return text
    .replace(/&/g,'&amp;').replace(/</g,'&lt;')  // escape HTML first
    .replace(/(#.*$)/gm, '<span style="color:#6b7280">$1</span>')  // comments
    .replace(/("(?:[^"\\]|\\.)*"|'(?:[^'\\]|\\.)*')/g, '<span style="color:#a5f3fc">$1</span>')  // strings
    .replace(/\b(const|let|var|function|return|if|else|for|while|true|false|null)\b/g, '<span style="color:#f97316">$1</span>');  // keywords
}
```

Used in lesson bodies via `<div class="code-block">` instead of inline `<code>` for multi-line examples.

### 4.3 Copy-to-Clipboard on Code Blocks

**Copy button:** Absolutely positioned top-right inside `.code-block`:
```css
.code-copy {
  position: absolute;
  top: 8px;
  right: 8px;
  background: #374151;
  border: none;
  border-radius: 6px;
  color: #9ca3af;
  font-size: 11px;
  padding: 4px 8px;
  cursor: pointer;
  opacity: 0;
  transition: opacity 0.2s;
}
.code-block:hover .code-copy { opacity: 1 }
.code-copy:hover { background: #4b5563; color: #e5e7eb }
```

**Behavior:** On click, copy `.code-block`'s `textContent` (stripped of HTML), show "Copied!" via existing `showToast()`, button text briefly changes to "✓".

**Implementation:** Each code block rendered as:
```html
<div class="code-block"><button class="code-copy" onclick="copyCode(this)">Copy</button>dash0 dashboards list -o json</div>
```

`copyCode(btn)` reads `btn.parentElement.textContent` (minus the button text), writes to clipboard.

### 4.4 Cross-Course Navigation

**Data structure:**
```javascript
const RELATED_COURSES = [
  { title: "Dash0 API Crash Course", url: "dash0-api-crash-course.html", icon: "🌐", desc: "HTTP endpoints, OTLP ingestion, PromQL" },
  { title: "Dash0 MCP Crash Course", url: "dash0-mcp-crash-course.html", icon: "🧠", desc: "AI-native observability via MCP" }
];
```

**Rendered in:** Resources tab as a "Related Courses" group at the top, using existing `.res-group` and `.res-link` styling.

**Inline cross-links:** In lesson body HTML, use `<a class="source-link" href="dash0-api-crash-course.html">` for same-directory linking. Displayed as the existing blue pill-style links.

**Existing courses:** Get an empty `RELATED_COURSES = []` or links to related courses where natural (e.g., PCA → OTCA, Terraform → CLI).

### 4.5 Difficulty Badges on Cards/Quiz

**New field:** `d` on flashcards and quiz items. Values: `1` (Beginner), `2` (Intermediate), `3` (Advanced). Optional — missing `d` defaults to no badge.

**Rendering:**
```javascript
const DIFF_LABELS = { 1: 'Beginner', 2: 'Intermediate', 3: 'Advanced' };
const DIFF_COLORS = { 1: '#10b981', 2: '#f59e0b', 3: '#ef4444' };
```

Small pill next to category label on cards: `<span style="background:rgba(COLOR,.15);color:COLOR;padding:2px 8px;border-radius:6px;font-size:10px">Beginner</span>`

**Filter:** New pills row on Cards view: "All Levels / Beginner / Intermediate / Advanced". Stored in `fcDiff` state variable (default "All Levels"). Interacts with category filter (AND logic).

**Quiz picker:** Each section card shows difficulty distribution as small dots or counts.

**Backward compatibility:** If `d` is undefined on a card, no badge shown, card included in "All Levels" filter. Existing courses work without modification — they just won't show badges until `d` fields are added.

### 4.6 Spaced Repetition Hints

**Self-rating UI:** After card flip, instead of just "Prev / Shuffle / Next", show a rating strip:

```
[Missed] [Unsure] [Know it]     [Prev] [Next]
```

Colors: Missed = red, Unsure = yellow, Know it = green. Rating advances to next card automatically.

If the user taps the card (existing flip behavior) but doesn't rate, the old Prev/Shuffle/Next still works — rating is optional but encouraged.

**Storage:** New `cardRatings` object in state:
```javascript
cardRatings: {
  "What command installs the Dash0 CLI?": { rating: "missed", count: 3, lastSeen: "2026-04-08T10:00:00Z" },
  ...
}
```

Key = card front text (the `f` field). Values: `"missed"`, `"unsure"`, `"known"`.

**Weighted shuffle:** When `fcShuffle()` is called:
- "missed" cards: 3x weight
- "unsure" cards: 1.5x weight
- "known" cards: 1x weight
- unrated cards: 1x weight

Implementation: build weighted array, pick random index.

**"Weak Cards" filter:** New pill alongside "Favorites": `"⚠ Weak (N)"` where N = count of missed + unsure cards. Shows only cards rated missed or unsure. Pill styled with amber border like the favorites pill.

**Dashboard stat:** New stat card: `⚠ Weak: N` showing count of weak cards. Clicking navigates to Cards with Weak filter active.

**State schema:** Bump save version from 7 → 8. Add `cardRatings` to `saveState()` data object and all `loadState()` / cloud sync merge paths. Version 7 data loads fine — missing `cardRatings` defaults to `{}`.

## 5. Course Content Structure

### 5.1 Mirror Layout — 8 Lessons Per Course

All three courses follow the same structural pattern. Content differs by tool.

#### CLI Course Lessons

| # | Title | Icon | Phase | Key Content |
|---|-------|------|-------|-------------|
| 1 | What is the Dash0 CLI? | 🖥️ | 1 | Why a CLI, architecture (CLI → API → platform), three interfaces positioning |
| 2 | Installation & Profiles | ⚙️ | 1 | Homebrew, Docker, GH Actions, binary. Named profiles for dev/staging/prod, env vars |
| 3 | Asset Management (CRUD) | 📊 | 1 | `dashboards`, `check-rules`, `views`, `synthetic-checks`. list/get/create/update/delete. YAML format, output formats (-o json/yaml/table/csv) |
| 4 | GitOps with `dash0 apply` | 🔄 | 2 | `apply -f`, multi-doc YAML, directory recursion, dry-run, PrometheusRule CRD conversion |
| 5 | Querying Telemetry | 🔍 | 2 | `logs query`, `metrics instant`, `spans query` (experimental), `traces get`, filters, time ranges |
| 6 | Sending Telemetry | 📤 | 2 | `logs send`, `spans send`, deployment event markers, CI/CD integration patterns |
| 7 | Agent Mode & CI/CD | 🤖 | 3 | `--agent-mode`, JSON output, env detection (Claude Code, Cursor, etc.), structured errors, GH Actions setup |
| 8 | Competitive Positioning | 🏆 | 3 | vs Datadog/Grafana/Dynatrace CLIs, open-source angle, GitOps differentiators |

#### API Course Lessons

| # | Title | Icon | Phase | Key Content |
|---|-------|------|-------|-------------|
| 1 | What is the Dash0 API? | 🌐 | 1 | Three API categories (ingestion, query, management), architecture, region model |
| 2 | Auth Tokens & Endpoints | 🔑 | 1 | Bearer tokens, Basic auth, token scopes, per-region URLs, endpoint discovery in UI |
| 3 | OTLP Ingestion | 📥 | 1 | gRPC (:4317), HTTP (:4318), Prometheus remote write, supported metric types, payload examples |
| 4 | Prometheus Query API | 📈 | 2 | `/api/prometheus/api/v1/query`, `query_range`, built-in variables, relative time ranges, curl examples |
| 5 | Management API | 🛠️ | 2 | REST CRUD for dashboards, check rules, views, synthetic checks. Request/response format |
| 6 | Observability-as-Code | 📝 | 2 | Perses format for dashboards, PrometheusRule CRD for alerts, Terraform provider, YAML workflow |
| 7 | Grafana & Tooling Integration | 🔗 | 3 | Grafana data source config, Perses native support, custom tooling via API, Go SDK |
| 8 | Competitive Positioning | 🏆 | 3 | Open API advantages, Prometheus compatibility, no proprietary query language, vs competitors |

#### MCP Course Lessons

| # | Title | Icon | Phase | Key Content |
|---|-------|------|-------|-------------|
| 1 | What is the Dash0 MCP Server? | 🧠 | 1 | MCP protocol, remote hosted (no local install), Streamable HTTP, 23 tools, AI-native positioning |
| 2 | Setup & Configuration | 🔧 | 1 | Claude Code, Claude Desktop, Cursor, Windsurf, Copilot, Zed, kagent. streamableHttp vs mcp-remote wrapper |
| 3 | Service Discovery & Catalog | 🗂️ | 1 | `getServiceCatalog`, `getServiceDetails`, RED metrics, natural language queries |
| 4 | Querying Logs & Traces | 📋 | 2 | `getLogRecords`, `getFullLogRecord`, `getLogCorrelations`, `getSpans`, `getTraceDetails`, `getSpanCorrelations` |
| 5 | Metrics & PromQL via MCP | 📊 | 2 | `promql`, `getMetricCatalog`, `getMetricDetails`, `getMetricQuery`, AI-assisted query building |
| 6 | Synthetic Checks & Web Events | 🌍 | 2 | `getSyntheticChecks`, `getFailedChecks`, `getFailedCheckDetails`, `getWebEvents`, proactive monitoring |
| 7 | Kubernetes & Dashboards | ☸️ | 3 | `getK8sPods`, `listDashboards`, `listDatasets`, `listWebsites`, `getAttributeKeys/Values`, `searchKnowledgeBase` |
| 8 | Competitive Positioning | 🏆 | 3 | First remote MCP for observability, vs CLI vs API (when to use which), market positioning |

### 5.2 Content Volume Per Course

| Content Type | Sales Rep | Solutions Architect | Engineer |
|---|---|---|---|
| Flashcards | 25 | 45 | 65 |
| Quiz Questions | 18 (3/section) | 30 (5/section) | 45 (7-8/section) |
| Scenarios | 8 (objection handling) | 10 (+ technical Q&A) | 12 (deep technical) |
| Rapid Fire | 8 | 10 | 15 |
| Exercises | 8 (explore UIs/docs) | 20 (explore + configure) | 35 (build + troubleshoot) |
| Lessons visible | All 8 (core: 1,3,8) | All 8 (core: 1-6,8) | All 8 (all core) |

All content exists in a single data set. The role determines which items are tagged core vs bonus.

### 5.3 Flashcard Categories Per Course

**CLI:** Fundamentals, Installation & Setup, Asset Management, GitOps & Apply, Querying, Sending Telemetry, Agent Mode & CI/CD, Pitch & Positioning

**API:** Fundamentals, Auth & Endpoints, OTLP Ingestion, Prometheus Queries, Management API, Observability-as-Code, Integrations, Pitch & Positioning

**MCP:** Fundamentals, Setup & Config, Service Discovery, Logs & Traces, Metrics & PromQL, Synthetic & Web, K8s & Dashboards, Pitch & Positioning

### 5.4 ROLE_CONFIG Per Course

All three courses use the same role keys and structure:

```javascript
const ROLE_CONFIG = {
  role1: {
    label: 'Sales Rep',
    icon: '💼',
    desc: 'Conversational fluency. Pitch points, competitive positioning, objection handling.',
    coreCategories: ['Fundamentals', 'Pitch & Positioning'],
    coreLessons: ['lesson-1', 'lesson-3', 'lesson-8'],
    categoryOrder: ['All', 'Fundamentals', 'Pitch & Positioning', /* ...remaining categories course-specific */]
  },
  role2: {
    label: 'Solutions Architect',
    icon: '🏗️',
    desc: 'Demo-ready working knowledge. Architecture, workflows, customer Q&A.',
    coreCategories: [/* first 6 topic categories */],
    coreLessons: ['lesson-1','lesson-2','lesson-3','lesson-4','lesson-5','lesson-6','lesson-8'],
    categoryOrder: ['All', /* all categories in logical order */]
  },
  role3: {
    label: 'Engineer / DevOps',
    icon: '⚡',
    desc: 'Deep technical. Hands-on with commands, YAML, APIs, troubleshooting.',
    coreCategories: [/* all categories */],
    coreLessons: ['lesson-1','lesson-2','lesson-3','lesson-4','lesson-5','lesson-6','lesson-7','lesson-8'],
    categoryOrder: ['All', /* all categories, technical-first order */]
  }
};
```

Exact `coreCategories` and `categoryOrder` arrays will be filled with the course-specific categories from section 5.3 during implementation.

### 5.5 Default Schedule

Self-paced 2-week schedule (no hard deadline):

```javascript
const DEFAULT_SESSIONS = [
  { id:"s1", date:"2026-04-14", time:"09:00", dur:30, title:"Lesson 1-2: Foundations", desc:"Read lessons 1-2. Complete exercises. Review flashcards.", phase:1 },
  { id:"s2", date:"2026-04-15", time:"09:00", dur:30, title:"Lesson 3: Core Workflows", desc:"Read lesson 3. Practice exercises. Quiz: Fundamentals.", phase:1 },
  { id:"s3", date:"2026-04-16", time:"09:00", dur:30, title:"Flashcards + Scenarios", desc:"All flashcards by category. Practice first 4 scenarios.", phase:1 },
  { id:"s4", date:"2026-04-21", time:"09:00", dur:45, title:"Lessons 4-5: Intermediate", desc:"Deeper workflows. Complete exercises. Quiz sections.", phase:2 },
  { id:"s5", date:"2026-04-22", time:"09:00", dur:45, title:"Lesson 6 + Rapid Fire", desc:"Advanced patterns. Rapid fire round. Review weak cards.", phase:2 },
  { id:"s6", date:"2026-04-23", time:"09:00", dur:30, title:"Lessons 7-8: Advanced + Competitive", desc:"Agent mode/integrations + competitive positioning.", phase:3 },
  { id:"s7", date:"2026-04-24", time:"09:00", dur:45, title:"Final Review", desc:"All scenarios, rapid fire 85%+, final exam. Review weak areas.", phase:3 },
];
```

Dates are illustrative. Adjusted per course title references.

### 5.6 Default Checkpoints

```javascript
const DEFAULT_CHECKPOINTS = [
  { id:"cp1", target:"Score 80%+ on Fundamentals quiz", by:"2026-04-16", metric:"bestQuiz", threshold:80 },
  { id:"cp2", target:"Complete all Phase 1 exercises",   by:"2026-04-17", metric:"exercises", threshold:40 },
  { id:"cp3", target:"Rapid fire 85%+",                  by:"2026-04-23", metric:"bestRapid", threshold:85 },
  { id:"cp4", target:"Score 90%+ on Final Exam",         by:"2026-04-24", metric:"bestQuiz",  threshold:90 },
];
```

### 5.7 Cross-Course Links

Each course's `RELATED_COURSES` array:

**CLI:**
```javascript
const RELATED_COURSES = [
  { title:"Dash0 API Crash Course", url:"dash0-api-crash-course.html", icon:"🌐", desc:"What's happening under the hood — HTTP endpoints, OTLP, PromQL" },
  { title:"Dash0 MCP Crash Course", url:"dash0-mcp-crash-course.html", icon:"🧠", desc:"AI-native access to the same platform via MCP tools" }
];
```

**API:**
```javascript
const RELATED_COURSES = [
  { title:"Dash0 CLI Crash Course", url:"dash0-cli-crash-course.html", icon:"🖥️", desc:"Developer-friendly terminal interface to the API" },
  { title:"Dash0 MCP Crash Course", url:"dash0-mcp-crash-course.html", icon:"🧠", desc:"AI-native access via Model Context Protocol" }
];
```

**MCP:**
```javascript
const RELATED_COURSES = [
  { title:"Dash0 CLI Crash Course", url:"dash0-cli-crash-course.html", icon:"🖥️", desc:"Terminal workflows for CI/CD and GitOps" },
  { title:"Dash0 API Crash Course", url:"dash0-api-crash-course.html", icon:"🌐", desc:"Direct HTTP access for custom integrations" }
];
```

Inline cross-links in lesson bodies where topics overlap:
- **Auth tokens:** All three courses reference the same token system. CLI Lesson 2, API Lesson 2, MCP Lesson 2 each cross-link.
- **PromQL:** CLI Lesson 5, API Lesson 4, MCP Lesson 5 each mention the others.
- **Dashboards:** CLI Lesson 3, API Lesson 5, MCP Lesson 7 cross-link.

## 6. State Schema

### Current (v7)
```javascript
{ stats, history, exercisesDone, schedule, checkpoints, sessionsDone, userRole, fcFavs, lastCard, savedAt, version:7 }
```

### New (v8)
```javascript
{ stats, history, exercisesDone, schedule, checkpoints, sessionsDone, userRole, fcFavs, cardRatings, lastCard, savedAt, version:8 }
```

**Migration:** If `version < 8` or `cardRatings` missing, default to `{}`. No destructive migration needed.

**Cloud sync merge:** Add `cardRatings` to all sync merge paths (same pattern as `exercisesDone`).

## 7. Study Guides

Each course gets a companion markdown study guide following the pattern established by `dash0-terraform-study-guide.md` and `pca-study-guide.md`.

**Structure per guide:**
1. Course overview and prerequisites
2. Phased study plan (Phase 1: Foundations, Phase 2: Intermediate, Phase 3: Advanced)
3. Key concepts with explanations
4. Command/endpoint/tool reference tables
5. Scenario drills with model answers (spoiler-tagged)
6. Exercises with step-by-step guides
7. Competitive positioning cheat sheet
8. Quick reference card (one-page summary)

**Naming convention** (matches `findStudyGuide()` in index.html):
- `dash0-cli-study-guide.md`
- `dash0-api-study-guide.md`
- `dash0-mcp-study-guide.md`

## 8. index.html Updates

The index page auto-discovers courses from the GitHub API. The three new HTML files will auto-appear. The fallback array needs updating:

```javascript
const fallback = [
  { name: 'dash0-api-crash-course.html', title: 'Dash0 API', desc: 'REST API, OTLP ingestion, PromQL queries', guide: 'dash0-api-study-guide.md' },
  { name: 'dash0-cli-crash-course.html', title: 'Dash0 CLI', desc: 'Terminal commands, GitOps, asset management', guide: 'dash0-cli-study-guide.md' },
  { name: 'dash0-mcp-crash-course.html', title: 'Dash0 MCP', desc: 'AI-native observability via Model Context Protocol', guide: 'dash0-mcp-study-guide.md' },
  { name: 'dash0-terraform-crash-course.html', title: 'Dash0 Terraform', desc: 'Terraform provider for observability-as-code', guide: 'dash0-terraform-study-guide.md' },
  { name: 'gke-study-app.html', title: 'Google Kubernetes Engine (GKE)', desc: 'Cloud-native Kubernetes on Google Cloud', guide: null },
  { name: 'otca-study-app.html', title: 'OpenTelemetry Certified Associate (OTCA)', desc: 'Observability & distributed tracing certification', guide: null },
  { name: 'pca-crash-course.html', title: 'Prometheus Certified Associate (PCA)', desc: 'Prometheus monitoring and PromQL certification', guide: 'pca-study-guide.md' },
];
```

## 9. Build Order

1. **Template v3.1** — Implement all 6 engine upgrades in `setup/crash-course-TEMPLATE.html`
2. **CLI course** — Build from upgraded template, fill content
3. **API course** — Build from upgraded template, fill content
4. **MCP course** — Build from upgraded template, fill content
5. **Study guides** — Write companion markdown guides for all three
6. **Backport v3.1** — Propagate engine changes to existing courses:
   - `public-courses/pca-crash-course.html`
   - `public-courses/dash0-terraform-crash-course.html`
   - `public-courses/gke-study-app.html`
   - `public-courses/otca-study-app.html`
   - `private-courses/pairwise-testing-crash-course.html`
   - `private-courses/dash0-value-framework-trainer.html` (React/Babel adaptation)
7. **index.html** — Update fallback array
8. **CLAUDE.md** — Update to reflect new files and PLATFORM_VERSION v3.1

### React/Babel Adaptation (Value Framework Trainer)

The value framework trainer uses React 18 + Babel CDN. Engine upgrades need adaptation:
- `class=` → `className=`
- `onclick=` → `onClick=`
- Template literals in JSX → JSX expressions
- Same CSS classes, same logic, different rendering syntax

### Content Accuracy

Per the crash course builder skill's accuracy requirements:
- All CLI commands verified against `dash0 --help` and the CLI repo README
- All API endpoints verified against Dash0 documentation
- All MCP tool names verified against the MCP server tool list
- All URLs validated (navigate and check for 404)
- Competitive claims date-stamped (April 2026)
- Internal claims flagged for learner verification

## 10. Content Research Sources

**CLI:**
- GitHub: `github.com/dash0hq/dash0-cli` (README, help output)
- Changelog: `dash0.com/changelog/dash0-cli-ga`, `dash0.com/changelog/cli-manage-assets`, `dash0.com/changelog/cli-agent-mode`
- Docs: `dash0.com/documentation/dash0/` (CLI sections)

**API:**
- Docs: `dash0.com/documentation/dash0/api`, `dash0.com/documentation/dash0/key-concepts/endpoints`, `dash0.com/documentation/dash0/key-concepts/auth-tokens`
- PromQL: `dash0.com/documentation/dash0/metrics/promql-query-patterns`
- Dashboards as code: `dash0.com/documentation/dash0/dashboards/managing-dashboards-as-code`
- Alerting: `dash0.com/documentation/dash0/alerting`

**MCP:**
- GitHub: `github.com/dash0hq/mcp-dash0`
- Docs: `dash0.com/docs/dash0/ai/mcp`, `dash0.com/docs/dash0/ai/mcp/use-mcp`
- Changelog: `dash0.com/changelog/dash0-mcp-server`
- Integration guides: `dash0.com/hub/integrations?tag=AI`
- Examples: `github.com/dash0hq/dash0-examples` (kagent agent-observability.yaml for tool list)
