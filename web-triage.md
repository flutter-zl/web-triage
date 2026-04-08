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
# List untriaged web issues, oldest first
gh issue list -R flutter/flutter \
  --search "is:issue is:open label:team-web,fyi-web -label:triaged-web no:assignee -label:\"will need additional triage\" -label:\"waiting for customer response\" sort:updated-asc" \
  --limit 30 --json number,title,body,labels,createdAt,updatedAt

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

### 6. What labels?
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
- **Labels to add**: triaged-web, platform-web, engine
- **Labels to remove**: (if re-routing)
- **Title cleanup**: (suggest improved title, if needed)
- **Comment**: (suggested comment text, if needed)
- **Reasoning**: (one sentence explaining why)
- **Root cause**: (1-2 sentences, only for issues owned by team-web with no linked PR)
```

Add the **Root cause** field when ALL of the following are true:

- No linked PR exists yet
- The cause can be reasonably inferred from the issue
- At least one of:
  - The issue is owned by `team-web`, OR
  - The issue is confirmed web-only (only reproduces on web), even if routed to another team like `team-framework`

Skip it when: a PR is already linked, the cause is unknown, or the bug reproduces on all platforms (not web-specific). Keep it to 1-2 sentences: what triggers it and where in the code the fix should go. Do NOT design a full fix.

After all issues, add a summary:

```
## Triage Summary - <date>
- Triaged: X issues
- Close: X issues
- Request info: X issues
- Re-route: X issues
- Remaining untriaged: X issues
```

## Weekly Health Checks

Before triaging new issues, check and report on:
1. Any unassigned P0/P1 issues
2. Any P3 issues in the backlog project
3. Open PR count, flag if 15 or more
4. Any P0 issues missing a weekly update
