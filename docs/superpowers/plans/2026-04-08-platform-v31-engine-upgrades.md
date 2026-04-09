# Platform v3.1 Engine Upgrades — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upgrade the crash course engine from v3.0 to v3.1 with six new features (font size, code blocks, copy-to-clipboard, cross-course nav, difficulty badges, spaced repetition) in the template, then backport to all existing apps.

**Architecture:** All changes are made in `setup/crash-course-TEMPLATE.html` first (source of truth), then propagated to 6 vanilla JS apps and 1 React/Babel app. Each feature is self-contained: new CSS classes + new JS state/functions + new UI in the appropriate render function.

**Tech Stack:** Vanilla JS, CSS classes in `<style>` block, single-file HTML apps, localStorage persistence.

**Spec:** `docs/superpowers/specs/2026-04-08-dash0-cli-api-mcp-courses-design.md`

**Security note:** These are self-contained single-file HTML apps with no user-generated content and no external data sources. All rendered content comes from hardcoded JS constants (flashcards, quiz questions, etc.) defined by the course author. The existing codebase uses innerHTML throughout for rendering — this is the established pattern. All content is trusted and author-controlled.

---

## File Map

| File | Role | Action |
|------|------|--------|
| `setup/crash-course-TEMPLATE.html` | Source of truth | Modify: add all 6 features |
| `public-courses/pca-crash-course.html` | PCA cert course | Modify: backport engine |
| `public-courses/dash0-terraform-crash-course.html` | Terraform course | Modify: backport engine |
| `public-courses/gke-study-app.html` | GKE course | Modify: backport engine |
| `public-courses/otca-study-app.html` | OTCA cert course | Modify: backport engine |
| `private-courses/pairwise-testing-crash-course.html` | Pairwise testing | Modify: backport engine |
| `private-courses/dash0-value-framework-trainer.html` | Value framework (React) | Modify: backport engine (React adaptation) |

---

### Task 1: Add Font Size CSS Classes and Settings UI

**Files:**
- Modify: `setup/crash-course-TEMPLATE.html` (lines 30-244 style block, lines 270-271 version, lines 497-513 load IIFE, lines 553 after pickRole, renderManage function)

- [ ] **Step 1: Add font size CSS classes before `</style>`**

Find `.role-current:hover{border-color:#10b981;background:#263040}` (line 244) and add after it:

```css
.fs-s .card-text,.fs-s .lesson-text,.fs-s .quiz-option,.fs-s .scenario-box,.fs-s .scenario-q,.fs-s .scenario-a,.fs-s .focus-text{font-size:13px}
.fs-m .card-text,.fs-m .lesson-text,.fs-m .quiz-option,.fs-m .scenario-box,.fs-m .scenario-q,.fs-m .scenario-a,.fs-m .focus-text{font-size:15px}
.fs-l .card-text,.fs-l .lesson-text,.fs-l .quiz-option,.fs-l .scenario-box,.fs-l .scenario-q,.fs-l .scenario-a,.fs-l .focus-text{font-size:17px}
.fs-xl .card-text,.fs-xl .lesson-text,.fs-xl .quiz-option,.fs-xl .scenario-box,.fs-xl .scenario-q,.fs-xl .scenario-a,.fs-xl .focus-text{font-size:20px}
```

- [ ] **Step 2: Add font size init in load IIFE**

At the end of the load IIFE (before its closing `})();`), add:

```javascript
try{const fs=localStorage.getItem(STORAGE_KEY+'-fontsize');if(fs)document.body.className=fs;else document.body.className='fs-m'}catch(e){document.body.className='fs-m'}
```

- [ ] **Step 3: Add setFontSize function**

After `pickRole` function (line 553), add:

```javascript
function setFontSize(size){document.body.className=size;try{localStorage.setItem(STORAGE_KEY+'-fontsize',size)}catch(e){};render()}
```

- [ ] **Step 4: Add font size picker UI to Settings tab**

In `renderManage`, add a Font Size section BEFORE the Focus Level section. The section renders four buttons (S/M/L/XL) with the active one highlighted green.

- [ ] **Step 5: Verify in browser and commit**

Open template, go to More > Settings, verify font size toggle works and persists on refresh.

```bash
git add setup/crash-course-TEMPLATE.html
git commit -m "feat(template): add font size settings (S/M/L/XL) for v3.1"
```

---

### Task 2: Add Code Block Styling and Copy-to-Clipboard

**Files:**
- Modify: `setup/crash-course-TEMPLATE.html` (style block + script block)

- [ ] **Step 1: Add code block CSS classes**

Add after the font size classes (before `</style>`):

```css
.code-block{position:relative;background:#0d1117;border:1px solid #374151;border-radius:10px;padding:14px 16px;margin:10px 0;font-family:'SF Mono','Fira Code',Consolas,monospace;font-size:13px;line-height:1.6;color:#e6edf3;overflow-x:auto;white-space:pre;-webkit-overflow-scrolling:touch}
.code-block:hover .code-copy{opacity:1}
.code-copy{position:absolute;top:8px;right:8px;background:#374151;border:none;border-radius:6px;color:#9ca3af;font-size:11px;padding:4px 8px;cursor:pointer;opacity:0;transition:opacity .2s;font-family:inherit}
.code-copy:hover{background:#4b5563;color:#e5e7eb}
.code-inline{background:#1f2937;padding:2px 6px;border-radius:4px;font-family:'SF Mono','Fira Code',Consolas,monospace;font-size:.9em;color:#a5f3fc}
```

- [ ] **Step 2: Add copyCode, highlightCode, and codeBlock JS functions**

After `setFontSize`, add:

```javascript
function copyCode(btn){
  var block=btn.parentElement;
  var text=block.textContent.replace(/^Copy/,'').replace(/^Copied!/,'').trim();
  navigator.clipboard.writeText(text).then(function(){
    btn.textContent='Copied!';
    showToast('Copied to clipboard');
    setTimeout(function(){btn.textContent='Copy'},1500);
  }).catch(function(){
    var ta=document.createElement('textarea');
    ta.value=text;
    document.body.appendChild(ta);
    ta.select();
    document.execCommand('copy');
    document.body.removeChild(ta);
    btn.textContent='Copied!';
    showToast('Copied to clipboard');
    setTimeout(function(){btn.textContent='Copy'},1500);
  });
}
function highlightCode(text){
  return text
    .replace(/&/g,'&amp;').replace(/</g,'&lt;')
    .replace(/(#.*$)/gm,'<span style="color:#6b7280">$1</span>')
    .replace(/("(?:[^"\\]|\\.)*"|\'(?:[^\'\\]|\\.)*\')/g,'<span style="color:#a5f3fc">$1</span>')
    .replace(/\b(const|let|var|function|return|if|else|for|while|true|false|null|import|from|export|class|extends|new|this|async|await)\b/g,'<span style="color:#f97316">$1</span>');
}
function codeBlock(code){
  return '<div class="code-block"><button class="code-copy" onclick="copyCode(this)">Copy</button>'+highlightCode(code)+'</div>';
}
```

- [ ] **Step 3: Verify in browser console and commit**

Open template, in console run: `document.getElementById('app').textContent=''; var d=document.createElement('div'); d.innerHTML=codeBlock('dash0 dashboards list -o json'); document.getElementById('app').appendChild(d);` — verify dark code block with hover copy button.

```bash
git add setup/crash-course-TEMPLATE.html
git commit -m "feat(template): add code block styling and copy-to-clipboard for v3.1"
```

---

### Task 3: Add Cross-Course Navigation

**Files:**
- Modify: `setup/crash-course-TEMPLATE.html`

- [ ] **Step 1: Add RELATED_COURSES data structure**

After `ROLE_CONFIG` (around line 289), add:

```javascript
// TODO: Add related courses for cross-linking. Each: {title, url, icon, desc}
const RELATED_COURSES=[];
```

- [ ] **Step 2: Update renderResourcesTab to show related courses**

At the beginning of `renderResourcesTab`, add a conditional block that renders related courses if any exist, using the existing `.res-group` and `.res-link` classes. The related courses group appears at the very top of the Resources tab, before the existing resource groups.

- [ ] **Step 3: Verify and commit**

Temporarily set `RELATED_COURSES` to a test entry, verify it appears in More > Resources, then revert to `[]`.

```bash
git add setup/crash-course-TEMPLATE.html
git commit -m "feat(template): add cross-course navigation in resources tab for v3.1"
```

---

### Task 4: Add Difficulty Badges on Cards and Quiz

**Files:**
- Modify: `setup/crash-course-TEMPLATE.html`

- [ ] **Step 1: Add difficulty constants**

After `RELATED_COURSES`, add:

```javascript
const DIFF_LABELS={1:'Beginner',2:'Intermediate',3:'Advanced'};
const DIFF_COLORS={1:'#10b981',2:'#f59e0b',3:'#ef4444'};
```

- [ ] **Step 2: Add fcDiff state variable**

In the state variables, add `fcDiff` to the flashcard state line:

```javascript
let mode="dashboard",fcCat="All",fcDiff="All Levels",fcIdx=0,fcFlipped=false,fcSeen=new Set(),fcFavs=new Set();
```

- [ ] **Step 3: Update fcGetCards to filter by difficulty**

Replace `fcGetCards` with a version that applies difficulty filtering. When `fcDiff` is not "All Levels", filter cards to only those with matching `d` field value. Cards without a `d` field are included in "All Levels" only.

- [ ] **Step 4: Add difficulty filter pills to renderFlashcards**

Add a second pills row below the category pills showing: All Levels / Beginner / Intermediate / Advanced. Each pill sets `fcDiff` and re-renders.

- [ ] **Step 5: Add difficulty badge to card label**

When a card has a `d` field, show a small colored pill (Beginner=green, Intermediate=amber, Advanced=red) next to the category name in the card label.

- [ ] **Step 6: Add difficulty distribution to quiz picker**

In each quiz section card, show counts of Beginner/Intermediate/Advanced questions as colored mini-labels.

- [ ] **Step 7: Verify and commit**

Add `d:1` to first example flashcard, `d:3` to second. Verify filter pills work, badges appear, quiz picker shows distribution.

```bash
git add setup/crash-course-TEMPLATE.html
git commit -m "feat(template): add difficulty badges on cards and quiz for v3.1"
```

---

### Task 5: Add Spaced Repetition (Card Ratings + Weak Cards)

**Files:**
- Modify: `setup/crash-course-TEMPLATE.html`

- [ ] **Step 1: Add cardRatings state**

After `let userRole=null,showRolePicker=false;`, add:

```javascript
let cardRatings={};
```

- [ ] **Step 2: Update saveState and cloudSave — add cardRatings, bump version 7 to 8**

In both `saveState` and `cloudSave`, add `cardRatings` to the data object and change `version:7` to `version:8`.

- [ ] **Step 3: Update all load/merge paths to restore cardRatings**

Add `if(saved.cardRatings)cardRatings=saved.cardRatings;` in the load IIFE.
Add `if(d.cardRatings)cardRatings=d.cardRatings;` in both cloud sync merge blocks.

- [ ] **Step 4: Update resetAll and resetSection**

`resetAll`: add `cardRatings={};` after `exercisesDone={};`
`resetSection`: change `if(key==="cards"){stats.cards=0}` to `if(key==="cards"){stats.cards=0;cardRatings={}}`

- [ ] **Step 5: Add fcRate and fcWeightedShuffle functions**

`fcRate(rating)`: records the rating in `cardRatings[card.f]`, increments cards count, auto-advances to next card.

`fcWeightedShuffle()`: builds a weighted array (missed=3x, unsure=1.5x, known/unrated=1x), picks random weighted index.

- [ ] **Step 6: Add rating UI to renderFlashcards**

When card is flipped (answer showing), display three rating buttons below the answer: Missed (red), Unsure (amber), Know it (green). Rating auto-advances. The existing Prev/Shuffle/Next row remains below.

Replace `fcShuffle()` call with `fcWeightedShuffle()` in the Shuffle button.

- [ ] **Step 7: Add Weak Cards pill**

In `renderFlashcards`, calculate weak count (cards rated missed or unsure). Show an amber "Weak (N)" pill alongside the Favorites pill. When clicked, filter to only weak cards.

Update `fcGetCards` to handle `fcCat==="⚠ Weak"` filter.

- [ ] **Step 8: Add empty Weak Cards view**

When "Weak" filter is active and there are no weak cards, show encouraging empty state: "No weak cards! Rate flashcards to see them here."

- [ ] **Step 9: Add Weak Cards stat to dashboard**

Add a new stat card to the dashboard stat-grid: "Weak Cards" with count, clicking navigates to Cards with Weak filter.

- [ ] **Step 10: Verify and commit**

Flip cards, rate them, verify: ratings persist on refresh, Weak pill appears, weighted shuffle works, dashboard shows weak count.

```bash
git add setup/crash-course-TEMPLATE.html
git commit -m "feat(template): add spaced repetition with card ratings and weak cards for v3.1"
```

---

### Task 6: Bump PLATFORM_VERSION and Fix Dashboard

**Files:**
- Modify: `setup/crash-course-TEMPLATE.html`

- [ ] **Step 1: Bump PLATFORM_VERSION**

Change `const PLATFORM_VERSION='v3.0';` to `const PLATFORM_VERSION='v3.1';`

- [ ] **Step 2: Fix hardcoded dashboard**

The template's `renderDashboard` has hardcoded "Google Cloud Next" countdown. Replace with a generic self-paced dashboard: 8 stat cards (Quizzes, Best Quiz, Cards Flipped, Scenarios, Rapid Rounds, Best Rapid, Exercises, Weak Cards) plus a dynamic focus text that changes based on exercise completion progress.

- [ ] **Step 3: Verify and commit**

```bash
git add setup/crash-course-TEMPLATE.html
git commit -m "feat(template): bump to PLATFORM_VERSION v3.1, fix generic dashboard"
```

---

### Task 7: Backport v3.1 to PCA Crash Course

**Files:**
- Modify: `public-courses/pca-crash-course.html`

- [ ] **Step 1: Add all new CSS classes** (font size, code block, code copy, code inline)
- [ ] **Step 2: Bump PLATFORM_VERSION** to v3.1
- [ ] **Step 3: Add constants** (RELATED_COURSES, DIFF_LABELS, DIFF_COLORS)

PCA RELATED_COURSES:
```javascript
const RELATED_COURSES=[{title:"OpenTelemetry (OTCA) Crash Course",url:"otca-study-app.html",icon:"🔭",desc:"OpenTelemetry Certified Associate exam prep"}];
```

- [ ] **Step 4: Add all new JS functions** (setFontSize, copyCode, highlightCode, codeBlock, fcRate, fcWeightedShuffle)
- [ ] **Step 5: Add cardRatings state and fcDiff state**
- [ ] **Step 6: Update persistence** (saveState, cloudSave: add cardRatings, version 8; load IIFE and sync merges: restore cardRatings; font size init)
- [ ] **Step 7: Update resetAll and resetSection**
- [ ] **Step 8: Update renderFlashcards** (difficulty filter, weak pill, rating buttons, weighted shuffle, empty weak view)
- [ ] **Step 9: Update renderDashboard** (add Weak Cards stat card)
- [ ] **Step 10: Update renderManage** (add Font Size section)
- [ ] **Step 11: Update renderResourcesTab** (add Related Courses)
- [ ] **Step 12: Verify in browser and commit**

```bash
git add public-courses/pca-crash-course.html
git commit -m "feat(pca): backport platform v3.1 engine upgrades"
```

---

### Task 8: Backport v3.1 to Remaining Vanilla JS Courses

**Files:**
- Modify: `public-courses/dash0-terraform-crash-course.html`
- Modify: `public-courses/gke-study-app.html`
- Modify: `public-courses/otca-study-app.html`
- Modify: `private-courses/pairwise-testing-crash-course.html`

Apply the exact same changes from Task 7 to each file. Only `RELATED_COURSES` differs:

**Terraform:**
```javascript
const RELATED_COURSES=[{title:"Dash0 CLI Crash Course",url:"dash0-cli-crash-course.html",icon:"🖥️",desc:"CLI interface including dash0 apply for GitOps"},{title:"Dash0 API Crash Course",url:"dash0-api-crash-course.html",icon:"🌐",desc:"REST API and observability-as-code patterns"}];
```

**GKE:**
```javascript
const RELATED_COURSES=[];
```

**OTCA:**
```javascript
const RELATED_COURSES=[{title:"Prometheus (PCA) Crash Course",url:"pca-crash-course.html",icon:"📊",desc:"Prometheus Certified Associate exam prep"}];
```

**Pairwise:**
```javascript
const RELATED_COURSES=[];
```

Note: GKE app has a custom dashboard with Google Cloud Next countdown — preserve it, just add Weak Cards stat card.

- [ ] **Step 1: Backport to Terraform course**
- [ ] **Step 2: Backport to GKE course** (preserve custom dashboard)
- [ ] **Step 3: Backport to OTCA course**
- [ ] **Step 4: Backport to Pairwise course**
- [ ] **Step 5: Verify each in browser**
- [ ] **Step 6: Commit**

```bash
git add public-courses/dash0-terraform-crash-course.html public-courses/gke-study-app.html public-courses/otca-study-app.html private-courses/pairwise-testing-crash-course.html
git commit -m "feat: backport platform v3.1 to terraform, gke, otca, pairwise courses"
```

---

### Task 9: Backport v3.1 to Value Framework Trainer (React/Babel)

**Files:**
- Modify: `private-courses/dash0-value-framework-trainer.html`

This course uses React 18 + Babel CDN. Same 6 features, adapted for React:
- CSS: identical (CSS doesn't change for React)
- JS: `className` instead of `class`, JSX instead of template literals, React state pattern instead of global variables
- `RELATED_COURSES=[]` (internal-only app)

- [ ] **Step 1: Add CSS classes** (identical to vanilla)
- [ ] **Step 2: Bump PLATFORM_VERSION**
- [ ] **Step 3: Add constants** (RELATED_COURSES, DIFF_LABELS, DIFF_COLORS)
- [ ] **Step 4: Adapt JS functions** for React state/render pattern
- [ ] **Step 5: Update persistence** (cardRatings, version 8)
- [ ] **Step 6: Update React components** (font size picker, difficulty pills, rating buttons, weak cards, resources)
- [ ] **Step 7: Verify in browser and commit**

```bash
git add private-courses/dash0-value-framework-trainer.html
git commit -m "feat(value-framework): backport platform v3.1 (React adaptation)"
```

---

### Task 10: Update index.html Fallback Array

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Replace fallback array** (around line 137)

```javascript
const fallback = [
  { name: 'dash0-api-crash-course.html', title: 'Dash0 API', desc: 'REST API, OTLP ingestion, PromQL queries', guide: 'dash0-api-study-guide.md' },
  { name: 'dash0-cli-crash-course.html', title: 'Dash0 CLI', desc: 'Terminal commands, GitOps, asset management', guide: 'dash0-cli-study-guide.md' },
  { name: 'dash0-mcp-crash-course.html', title: 'Dash0 MCP', desc: 'AI-native observability via Model Context Protocol', guide: 'dash0-mcp-study-guide.md' },
  { name: 'dash0-terraform-crash-course.html', title: 'Dash0 Terraform', desc: 'Terraform provider for observability-as-code', guide: 'dash0-terraform-study-guide.md' },
  { name: 'gke-study-app.html', title: 'Google Kubernetes Engine (GKE)', desc: 'Cloud-native Kubernetes on Google Cloud', guide: null },
  { name: 'otca-study-app.html', title: 'OpenTelemetry Certified Associate (OTCA)', desc: 'Observability and distributed tracing certification', guide: null },
  { name: 'pca-crash-course.html', title: 'Prometheus Certified Associate (PCA)', desc: 'Prometheus monitoring and PromQL certification', guide: 'pca-study-guide.md' },
];
```

- [ ] **Step 2: Verify and commit**

```bash
git add index.html
git commit -m "feat(index): add CLI, API, MCP courses to fallback list"
```
