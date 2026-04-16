---
Flutter Web triage workflow for processing untriaged web issues and PRs
---

# Flutter Web Triage

When asked to triage web issues, follow this workflow. **Read-only: DO NOT run any `gh` commands that modify issues, add labels, close issues, or post comments. Only use `gh` to fetch and read issue data. Output recommendations for the user to act on manually.**

Triage is sorting and routing, not fixing. Spend 15-30 seconds per issue. You are reading the issue, deciding "is this clear, who owns it, how important is it," recommending labels, and moving on. Do NOT propose full fixes. For issues owned by team-web with no linked PR, add a brief root cause note (1-2 sentences) to help whoever picks it up next.

## Key URLs

- [Untriaged web issues](https://github.com/flutter/flutter/issues?q=is%3Aissue+is%3Aopen+label%3Ateam-web%2Cfyi-web+-label%3Atriaged-web+no%3Aassignee+-label%3A%22will+need+additional+triage%22+sort%3Aupdated-asc+-label%3A%22waiting+for+customer+response%22)
- [Unassigned P0/P1 issues](https://github.com/flutter/flutter/issues?q=is%3Aopen+is%3Aissue+label%3Ateam-web+label%3AP1%2CP0+no%3Aassignee)
- [Web PRs on flutter/flutter](https://github.com/flutter/flutter/pulls?q=is%3Aopen+is%3Apr+label%3Aplatform-web+sort%3Acreated-asc+draft%3Afalse+-label%3Atriaged-web)
- [Web PRs on flutter/packages](https://github.com/flutter/packages/pulls?q=is%3Aopen+is%3Apr+label%3Atriage-web+sort%3Aupdated-asc+-is%3Adraft)

## Fetching Issues

Use `gh` only to read issue data:

```bash
# List untriaged web issues, oldest first (fetch metadata only, not body)
gh issue list -R flutter/flutter \
  --search "is:issue is:open label:team-web,fyi-web -label:triaged-web no:assignee -label:\"will need additional triage\" -label:\"waiting for customer response\" sort:updated-asc" \
  --limit 30 --json number,title,labels,createdAt,updatedAt

# Read a specific issue with comments
gh issue view <NUMBER> -R flutter/flutter --comments
```

Note: the `no:assignee` filter means issues that have already been re-routed by a team member (e.g. had their `team-*` label changed but no one assigned) will still appear in the list. Always check whether the `team-*` label has already changed before recommending a re-route.

## Triage Each Issue

For each issue, answer these questions in order:

### 1. Can I understand what the bug is?
If not, recommend requesting more info with `waiting for customer response`.

### 2. Is it actionable?
An actionable issue has clear steps to reproduce, or is a well-argued feature request with a solid use case. If it's vague, a help request, or a third-party package issue, recommend closing it.

### 3. Is it a duplicate?
If it looks familiar, search for similar issues. Recommend closing with `r: duplicate` and link the original.

### 4. Does it belong to team-web?
Use this routing order, stop at the first match:
- Accessibility issue: `team-accessibility`, also add `fyi-web`
- Text input / text selection: `team-text-input`, also add `fyi-web`
- Material / Cupertino widget: `team-design`
- Flutter tool / CLI / dev server: `team-tool`
- Package issue: `team-ecosystem` or `team-framework` depending on the package
- Engine issue on web: keep `team-web`
- Framework issue on web: keep `team-web` unless it's clearly not web-specific

If the issue has `fyi-web` but already carries a different `team-*` label from a prior re-route, do not change the routing. Just add `triaged-web` to close out web team's review.

### 5. What priority?
- **P0**: Build break, regression blocking shipping, blocking top-tier customer
- **P1**: High priority, top of work list, actively being worked on
- **P2**: Important, clear, valid issue, but not urgent. This is the default
- **P3**: Lower importance, unlikely to be worked on soon

**Regressions**: start at P2 minimum. Bump to P1 if the regression is in a widely-used feature or was reported by a customer. A regression in a niche feature with a workaround can stay P2.

**Feature requests/proposals**: P3 unless there's demonstrated demand from multiple users or a top-tier customer. Add `c: new feature` or `c: proposal`.

### 6. Suggested assignee?

Use the table below to recommend an assignee based on the issue's area. If an issue spans multiple areas, pick the strongest match. Do not recommend an assignee for issues being re-routed to a different team.

| Area | Assignee | Examples |
|---|---|---|
| Accessibility, semantics, screen readers, ARIA | `flutter-zl` | a11y tree, VoiceOver, focus order, `SemanticsAction`, `ensureSemantics` |
| Scrolling on web, nested scroll, iframe scroll | `flutter-zl` | `f: scrolling`, browser-driven scroll, scroll over platform views |
| Framework-level web bugs with a11y angle | `flutter-zl` | `MenuAnchor` dismiss with semantics, dialog a11y, `OverlayPortal` click-through |
| CanvasKit rendering, image decoding, Skwasm | `harryterkelsen` | `e: web_canvaskit`, `e: web_skwasm`, image downscaling, GPU crashes, `CkSurface` |
| Engine-level rendering, shaders, surfaces | `harryterkelsen` | `c: rendering`, `c: crash` in engine, WebGL context loss, blur, gradients |
| Wasm runtime, renderer unification | `harryterkelsen` | `e: wasm`, renderer unification, garbage collection |
| Web packages, web_benchmarks, pointer_interceptor | `mdebbar` | `p: web_benchmarks`, `p: pointer_interceptor`, `p: camera` web |
| Test infrastructure, flakes, CI | `mdebbar` | `a: tests`, `c: flake`, test harness, Chrome test failures |
| Platform views, multi-view | `mdebbar` | `a: platform-views`, view factories, multi-view layout |
| Text input, autofill, keyboard on web | `mdebbar` | `a: text input` non-a11y, autofill, IME, keyboard events |

When no area clearly matches, leave the assignee recommendation blank.

### 7. What labels?
Add all applicable labels from the checklist below.

## Label Checklist

- **`triaged-web`** to mark as triaged
- **One `team-*` label** to assign ownership
- **Priority**: P0, P1, P2, or P3
- **Category**: `engine`, `framework`, `tool`
- **Subcategory**: `e: *`, `f: *` labels for specific areas
- **`platform-web`** for web-specific issues
- **`browser: *`** only if browser-specific: `browser: chrome`, `browser: firefox`, `browser: safari`
- **`c: *`** labels: `c: regression`, `c: crash`, `c: new feature`, `c: performance` (use `c: performance` for hangs and infinite loops too, not just slowness)
- **`a: *`** area labels: `a: accessibility`, `a: text input`, `a: images`, `a: platform-views`
- **`has reproducible steps`** if the issue includes clear repro steps OR a confirmed production stack trace with a root cause analysis (e.g. Sentry crash with deobfuscated frames)
- **`found in release: x.yy`** if a version is mentioned

## Issue Cleanup

While triaging, also recommend cleanup if needed:
- Fix the title to be a clear, searchable summary
- Format unformatted code blocks, stack traces, and `flutter doctor` output
- Note any low-quality or off-topic comments to hide

## Triage Output Format

For each issue, output a recommendation:

```
### #NNNNN - <title>
- **Action**: triage / close / request info / re-route
- **Priority**: P2
- **Suggested assignee**: mdebbar / harryterkelsen / flutter-zl / (blank if unclear)
- **Labels to add**: triaged-web, platform-web, engine
- **Labels to remove**: (if re-routing)
- **Title cleanup**: (suggest improved title, if needed)
- **Comment**: (suggested comment text, if needed)
- **Reasoning**: (one sentence explaining why)
- **Root cause**: (1-2 sentences, only when no linked PR exists)
- **PR review**: (brief summary, only when a linked PR exists)
```

Add the **Root cause** field when ALL of the following are true:

- The cause can be reasonably inferred from the issue
- At least one of:
  - The issue is owned by `team-web`, OR
  - The issue is confirmed web-only (only reproduces on web), even if routed to another team like `team-framework`

Add the **PR review** field when a linked PR exists. Fetch the diff with:

```bash
gh pr diff <PR_NUMBER> -R flutter/flutter
gh pr view <PR_NUMBER> -R flutter/flutter --comments
```

Keep the review to 2-4 sentences covering: does the fix address the root cause, are there obvious edge cases or risks, and is the approach reasonable. This is a quick sanity check, not a full code review.

After all issues, add a summary:

```
## Triage Summary - <date>
- Triaged: X issues
- Close: X issues
- Request info: X issues
- Re-route: X issues
- Remaining untriaged: X issues
```

Save triage output to `web-triage/weekly-log/<Month Day, Year>.md`.

## PR Triage

After triaging issues, review untriaged web PRs. For each PR:

1. **Read the PR title and description.** Does it reference an issue? Is the scope clear?
2. **Check staleness.** Flag PRs with no review activity for 30+ days.
3. **Label it.** Add `triaged-web` once reviewed. Add other relevant labels like `engine`, `framework`, `a: *` if missing.
4. **Flag if needed.** If the PR needs a specific reviewer or is blocked, note that in the triage output.

You do not need to code-review PRs during triage. Just confirm they are labeled, not stale, and have a clear owner.

## Weekly Health Checks

Before triaging new issues, run these checks and report findings.

### Commands

```bash
# Unassigned P0/P1
gh issue list -R flutter/flutter \
  --search "is:open is:issue label:team-web label:P1,P0 no:assignee" \
  --limit 20 --json number,title,labels

# Untriaged web PRs (flutter/flutter)
gh pr list -R flutter/flutter \
  --search "is:open is:pr label:platform-web sort:created-asc draft:false -label:triaged-web" \
  --limit 50 --json number,title,createdAt

# Untriaged web PRs (flutter/packages)
gh pr list -R flutter/packages \
  --search "is:open is:pr label:triage-web sort:updated-asc -is:draft" \
  --limit 20 --json number,title,createdAt

# Stale "waiting for customer response" (no activity 30+ days)
gh issue list -R flutter/flutter \
  --search "is:open is:issue label:team-web label:\"waiting for customer response\" sort:updated-asc" \
  --limit 20 --json number,title,updatedAt
```

### Checklist

1. **Unassigned P0/P1 issues.** If an unassigned P0/P1 is already marked `r: duplicate` or `triaged-web` but was never closed, recommend closing it. Do not keep reporting the same zombie issue weekly.
2. **Open PR count.** Flag if 15 or more untriaged web PRs on flutter/flutter.
3. **P0 issues missing a weekly update.** Check that any open P0 has had a status comment in the last 7 days.
4. **Stale "waiting for customer response" issues.** If no activity for 30+ days, recommend closing with a standard "closing due to no response, feel free to reopen with more details" comment.
