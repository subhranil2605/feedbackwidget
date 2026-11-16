# Scope: UAT Feedback

A plug and play React component you drop into your team's apps so UAT testers can report issues from any page: a title and description, a page screenshot they can annotate, the clicks that led there, and recent errors. Reports go to each app team's own cloud storage (Azure first), and your team reviews them in a small dashboard behind a sign in.

**Build approach:** Tracer Bullet (prove the whole pipe end to end first: widget, then cloud storage, then dashboard; then thicken one segment per slice).
**Workflow:** Beta (after `/develop`: `/check verify`, then `/test`). This is the project default level of rigor. `/architect` is the recommended first stop for a feature with a real decision, but you can skip it when you already know the build. Any feature can carry its own tag (for example `· GA`) to do more or less.

_These are recommendations to keep your build orderly, not requirements. Skip anything that does not fit: if you already know how to build a feature, use `/develop` and skip `/architect`. You decide when a feature is `done`._

## At a glance

| # | Feature | Phase | Status |
|---|---------|-------|--------|
| 1 | Stack & architecture | Foundation | in-progress |
| 2 | Coding standards & tooling | Foundation | planned |
| 3 | Data model | Foundation | planned |
| 4 | Design system & UI foundation | Foundation | planned |
| 5 | Storage adapter & Azure upload | Slice 1 | planned |
| 6 | Dashboard sign in | Slice 1 | planned |
| 7 | Core feedback loop | Slice 1 | planned |
| 8 | Page screenshot | Slice 2 | planned |
| 9 | Screenshot annotation | Slice 3 | planned |
| 10 | Click trail repro steps | Slice 4 | planned |
| 11 | Console & network errors | Slice 5 | planned |
| 12 | Sensitive data masking | Slice 6 | planned |
| 13 | Show only in UAT | Slice 7 | planned |
| 14 | Reporter identity | Slice 7 | planned |

## Foundations

### 1. Stack & architecture · in-progress
Decide how the pieces fit together: the React widget package, the review dashboard, and where the storage adapter runs. Also decide how the browser uploads without exposing cloud keys. Then scaffold a runnable project.
**Done when:** the architecture is recorded in a spec, and the empty widget package and dashboard both build and run locally.
- [x] Decide the stack (spec): `/architect stack & architecture`
- [ ] Scaffold from the decision: `/develop stack & architecture`
- [ ] Smoke check it runs: `/test`
Spec [0001](../specs/0001-stack-architecture/index.md)

### 2. Coding standards & tooling
Capture conventions and tooling choices from the real scaffold, then install lint, format, and pre commit checks.
**Done when:** root `AGENTS.md` reflects the real stack, and lint, format, and pre commit run clean.
- [ ] Capture conventions + tooling choices: `/audit`

### 3. Data model · needs a decision
The shape of one feedback report: text fields, auto context (URL, browser, screen size, time), attachments (screenshot, annotations), click steps, error logs, reporter, status. Also covers how reports and files are laid out in storage.
**Done when:** one report shape supports every later slice (screenshot, annotation, steps, errors, identity) without a breaking change, and the storage layout is fixed.
- [ ] Design it (spec): `/architect data model`

### 4. Design system & UI foundation · needs a decision
Look and base components for the widget and the dashboard. The widget sits on top of someone else's app, so its styles must not leak into the host app or be broken by it.
**Done when:** `design.md` covers type, color, spacing, and components, and the widget renders the same inside a host app with clashing global styles.
- [ ] Design it (spec): `/architect design system & UI foundation`

## Slice 1: Core feedback loop (the walking skeleton)

### 5. Storage adapter & Azure upload · needs a decision · GA
One storage interface every cloud plugs into, with Azure as the first real adapter. The browser must upload safely without holding long lived cloud credentials.
**Done when:** a report and its files land in the app team's own Azure storage through the adapter, and no secret ships in the browser bundle.
- [ ] Design it (spec): `/architect storage adapter & Azure upload`

### 6. Dashboard sign in · needs a decision
Only your team can open the review dashboard, since screenshots can show real UAT data.
**Done when:** signed out visitors cannot see any report or file, and team members can sign in and out.
- [ ] Design it (spec): `/architect dashboard sign in`

### 7. Core feedback loop · needs a decision
The thinnest real thread: add the component to an app with one line, a floating button opens a form (title, description), auto context is attached, the report is stored, and it appears in the dashboard list where it can be marked open or resolved.
**Done when:** a tester submits a text report from a host app and a signed in developer sees it in the dashboard with its context, and can resolve it.
- [ ] Design it (spec): `/architect core feedback loop`

## Slice 2: Page screenshot

### 8. Page screenshot · needs a decision
Capture what the tester sees on the current page and attach it to the report; show it in the dashboard detail view.
**Done when:** a report carries a screenshot that matches the visible page, failures to capture are explained to the tester, and the dashboard shows it.
- [ ] Design it (spec): `/architect page screenshot`

## Slice 3: Screenshot annotation

### 9. Screenshot annotation · needs a decision
Let the tester draw boxes and circles on the screenshot and pin short comments to spots, before submitting.
**Done when:** the tester can add, move, and remove boxes, circles, and comments; the dashboard shows the annotated image with comments readable.
- [ ] Design it (spec): `/architect screenshot annotation`

## Slice 4: Click trail repro steps

### 10. Click trail repro steps · needs a decision
Quietly record the last clicks and page changes so a report shows the steps that led to the problem, with numbered markers where the tester clicked.
**Done when:** a report lists recent steps in order, each with its page and target, and the dashboard shows numbered click markers; the tester can review and trim steps before sending.
- [ ] Design it (spec): `/architect click trail repro steps`

## Slice 5: Console & network errors

### 11. Console & network errors · needs a decision
Attach recent JavaScript errors and failed requests to each report, so "it's broken" comes with evidence.
**Done when:** recent console errors and failed requests (method, URL, status) appear on the report in the dashboard, without slowing the host app noticeably.
- [ ] Design it (spec): `/architect console & network errors`

## Slice 6: Sensitive data masking

### 12. Sensitive data masking · needs a decision · GA
Hide passwords and any element the host app marks as sensitive, in screenshots, click steps, and captured requests.
**Done when:** password fields and marked elements never appear readable in any stored screenshot, step, or log.
- [ ] Design it (spec): `/architect sensitive data masking`

## Slice 7: Rollout controls

### 13. Show only in UAT
The widget appears only in chosen environments (or for chosen users) and renders nothing in production.
**Done when:** with the production setting the widget adds nothing to the page, and in UAT it shows as normal.
- [ ] Build it: `/develop show only in UAT`

### 14. Reporter identity
Tag each report with who sent it: passed in by the host app when the user is known, otherwise typed by the tester.
**Done when:** every report in the dashboard shows a reporter, from the host app when given, else from the form.
- [ ] Build it: `/develop reporter identity`

## Deferred
Out of scope for this build pass, kept so the plan stays honest.
- **AWS storage adapter**: second adapter behind the same interface · needs a decision
- **GCP storage adapter**: third adapter behind the same interface · needs a decision
- **Issue tracker push**: turn a report into a Jira, GitHub, or Azure Boards ticket · needs a decision
- **New feedback notifications**: alert the team when a report arrives · needs a decision
- **Widget accessibility**: keyboard use and screen reader support without trapping focus in the host app · needs a decision
- **Translations**: widget text in more than one language
- **Open source release**: public package, docs, versioning
- **Team setup guide**: Entra app registration, managed identity role, and feed access steps for each team (from spec 0001)

## Legend

**The decision box.** Every feature carries exactly one, the sub task whose label ends with `(spec)`. Its wording varies (`Design it (spec)` normally, `Decide the stack (spec)` on Stack & architecture), so skills locate it by that `(spec)` suffix, never by an exact label. Every other box is an execution box and `/architect` never ticks one.

**Feature lifecycle**: the scope updates as a feature moves; each row is what it shows and who sets it:

| State | Set by | The feature shows |
|---|---|---|
| `planned` · needs a decision | `/scope` | one box: `Design it (spec): /architect <feature>` |
| `in-progress` (designed) | **`/architect` at spec capture** | `Design it` ticked; spec linked; `Build it: /develop <feature>` + **2 to 5 milestones**; the tier's closing boxes (`Verify it` Alpha+, `Test it` Beta+, `Review it` + `Document it` GA); any surfaced follow up enrolled |
| `in-progress` (building) | `/develop` | milestone sub boxes tick one by one; code pointer filled |
| `in-progress` (verified) | `/check verify` | `Build it` + milestones ticked; `Verify it` ticked |
| `done` | **you, when you decide it is** (any skill sets it when you say so); `/sync` reconciles | boxes you ran ticked, skipped ones marked skipped; the tier's last stage (`Prototype` after `/develop`; `Alpha` after `/check verify`; `Beta`/`GA` after `/test`) is the suggested point to call it done; `/sync` captures conventions |

- **Next step** = the first unticked box (always a command or a tracked milestone).
- **needs a decision** = run `/architect` first; otherwise straight to `/develop` (or `/audit` for standards & tooling). The tag drops once the spec is captured.
- **Atomic build tasks live in the spec's `## Build plan`, not here**: the scope carries only the milestone rollup.
- **Status** `planned`, then `in-progress`, then `done`, plus `existing` (built before this workflow) and `dropped` (removed from scope, kept for history).
- **Approach tag** beside a heading (for example `· Facade`) overrides the project default for that feature; no tag means it inherits.
- **Workflow tier tag** beside a heading (for example `· GA`) sets that one feature's rigor above or below the project default; no tag inherits the default. It decides the feature's check boxes and each skill's next suggestion.
- **Workflow** (header line) is the project default, what runs after `/develop`: **Prototype** = nothing (trust develop's own build time self check); **Alpha** = `/check verify`; **Beta** = `/check verify` then `/test`; **GA** = adds a fresh model `/check review` then `/document`. A feature built on an unratified decision (an `Assumed` spec) stays flagged, but that never blocks `done`.
- **Pointer line** (`spec <n> · code in <path>`): the spec link added by `/architect`, the code path by `/develop`.
