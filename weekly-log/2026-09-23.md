# Flutter Web Triage - September 23, 2026

Read-only pass. Nothing was modified on GitHub. All items below are recommendations to apply manually.

---

## Weekly Health Checks

### 1. Unassigned P0/P1 - STATUS CHANGED

**#190661** - [web] VoiceOver announces menu items with an off-by-one position/count

Still the only unassigned P1 on `team-web`, now 41 days unowned. **But the recommendation changes this week:** a fix PR now exists.

- **PR #192965** (kevmoo, opened 5 days ago, `REVIEW_REQUIRED`, no reviewer assigned) declares `Fixes #190661`.
- Its diagnosis matches the root cause recorded in the September 9 log exactly: `SemanticScrollable` sets `role="group"` in its constructor, `SemanticMenu` / `SemanticMenuBar` hoist the real items with `aria-owns`, and WebKit counts the intermediate scrollable as member 1. The fix sets `role="none"` on unlabeled intermediate `SemanticScrollable` containers inside `SemanticMenu` and `SemanticMenuBar`.
- **Do not assign this to `flutter-zl` as an implementation task anymore.** Assign the *PR review* instead.

Label cleanup is still pending from the last two weeks:
- Remove: `p: material_ui`, `package`, `platform-macos`
- Add: `platform-web`, `framework`, `engine`

No open P0 issues on `team-web`, so the P0 weekly-update check is a no-op.

### 2. Open PR count - FLAGGED

**26 untriaged web PRs** on flutter/flutter, against a threshold of 15. Last week was 25. The backlog grew by one and no PR was triaged out of it.

### 3. P0 issues missing a weekly update - N/A

No open P0 `team-web` issues.

### 4. Stale "waiting for customer response" - CLEAR

Zero issues in the bucket. Nothing to close.

---

## Issue Triage

8 untriaged issues. Two are carry-overs whose re-routes were never applied.

### #192054 - Flutter web serving stale app on reload of tab
- **Action**: re-route
- **Priority**: P2
- **Suggested assignee**: (blank - re-routing off team-web)
- **Labels to add**: triaged-web, team-tool, fyi-web, tool, P2
- **Labels to remove**: team-web
- **Reasoning**: **Third appearance.** Recommended for re-route on September 2 and again on September 9; the labels were never changed, so it keeps resurfacing in the untriaged queue. Nothing about the diagnosis has changed. The entire failure lives in `flutter_tools`.
- **Root cause**: the dev server keys DDC library-bundle modules by their strongly-connected-component name. Breaking an import cycle changes the SCC, so `main.dart` recompiles under a new key while the old bundled module stays in `WebMemoryFS`. DWDS lists both in `main_module.bootstrap.js`, and on a tab reload the stale module executes first. Hot restart re-resolves the entrypoint, which is why only the reload path shows it.
- **PR review**: #192428 (bkonyi) is **still draft and still has no reviewer, 10 days after it was last touched**. Unchanged since the September 9 review. The two concerns raised then still stand: `_parseLibraries` swallows all parse failures and returns an empty set, silently degrading to today's stale-module behavior rather than surfacing a problem, and the `_modules.removeWhere` reconstructs the `.js` filename by string concatenation after `path` was produced by `replaceAll('.js', '')`, which is not round-trip safe for a module whose name contains `.js` elsewhere. Approach is still sound and it carries unit and regression tests.

### #192544 - Text field permanently stops accepting input after `TextInput.updateConfig` on a live connection
- **Action**: triage (confirm routing only)
- **Priority**: P1
- **Suggested assignee**: (blank - owned by team-text-input)
- **Labels to add**: triaged-web, P1
- **Labels to remove**: none
- **Reasoning**: already carries `team-text-input` + `fyi-web` from mbcorona's re-route, so per the routing rule the web team does not change ownership and only needs to close out its review with `triaged-web`. Flagging the priority as **P1 rather than the P2 default**: the failure is permanent, silent, and unrecoverable within the session, and the most common trigger is a password-visibility toggle or `TextField(enabled: ...)` on a focused field, which is ordinary application code. Final priority call belongs to `team-text-input`.
- **Root cause**: web-only, so recording it here even though it is routed away. On `updateConfig` for a live connection, the cleanup branch takes `safeRemove(activeDomElement)` instead of `goDormant()`, so the field's autofill `<form>` stays in the DOM and stays registered in `dormantForms` with `elements[uniqueIdentifier]` pointing at a now-detached element. Every later session adopts that stale form and reaches the `replaceWith` swap path, but `ChildNode.replaceWith` on a parentless node is a spec no-op, so the new editing element never enters the document and nothing throws. It never recovers because `formIdentifier` is derived from the `EditableTextState`'s `autofillId`, so only a brand-new `EditableTextState` produces a key that misses `dormantForms`.

### #192433 - Hot reload requests should have a common path
- **Action**: re-route
- **Priority**: P3
- **Suggested assignee**: (blank - re-routing off team-web)
- **Labels to add**: triaged-web, team-tool, fyi-web, tool, c: proposal, P3
- **Labels to remove**: team-web
- **Reasoning**: **Carry-over from September 9**, re-route never applied. This is a request that the dev server honor `<base href>` from `index.html` when emitting hot-reload asset paths. The request paths are generated by `flutter_tools`, not the engine. P3 because it is a single-reporter developer-environment request with a working, if fragile, proxy workaround.

### #192704 - [url_launcher_web] Expose newly created Window instance
- **Action**: re-route
- **Priority**: P3
- **Suggested assignee**: (blank - re-routing off team-web)
- **Labels to add**: triaged-web, team-ecosystem, fyi-web, c: proposal, P3
- **Labels to remove**: team-web
- **Reasoning**: routed to the web team as a proposal, but the routing order puts package issues on `team-ecosystem`, and the issue already carries `fyi-ecosystem` and `p: url_launcher`. The ask cannot be satisfied inside `url_launcher_web` alone: returning the `Window` handle means changing the `launchUrl` signature across the platform interface, and the reporter openly concedes he does not know what non-web platforms would return. That is a plugin-API design decision for ecosystem, not a web-engine question. P3 as a single-reporter proposal with no cross-platform answer yet.

### #192792 - [web][a11y] Focused TextField inside a SingleChildScrollView loses DOM focus when an ancestor Theme rebuilds
- **Action**: triage (confirm routing only)
- **Priority**: P2
- **Suggested assignee**: (blank - owned by team-accessibility)
- **Labels to add**: triaged-web, P2
- **Labels to remove**: none
- **Reasoning**: already carries `team-accessibility` + `fyi-web` from a prior re-route and was confirmed on master by mbcorona, so the web team only adds `triaged-web`. A fix PR is already open, which is the main thing to act on.
- **Root cause**: web-only. A `ThemeData` that never compares equal forces a rebuild on every frame; the `SingleChildScrollView`'s `hasImplicitScrolling` flips, which flips `isScrollContainer`, which drives a semantics role swap between scrollable and generic. `_updateRole()` creates a replacement `<flt-semantics>` element and calls `parent.replaceChild(...)`, and replacing the element that currently holds `document.activeElement` drops focus to `<body>`. Either half alone is harmless, which is why both the scroll view and the theme churn are needed to reproduce.
- **PR review**: **#192963** (kevmoo, 5 days old, `REVIEW_REQUIRED`, **no reviewer assigned**) declares `Fixes #192792` and also `Fixes #191484`. It addresses the root cause directly: `_updateRole()` now checks whether the outgoing element held `document.activeElement` before disposal and calls `focusWithoutScroll()` on the replacement when the node is still focusable. That is the right seam. Two things worth attention in review: the restore is conditional on "the node remains focusable", so it is worth confirming the focusable check is evaluated against the *new* role rather than the old one, and the PR bundles an unrelated second fix (`isAccessibilityFocusBlocked` / `aria-hidden` for modal barriers) that touches `LabelAndValue.update()` — the two changes have independent risk profiles and would be easier to review, and to revert, apart.

### #192784 - [web][skwasm] Different-sized multi-view windows race on the shared OffscreenCanvas surface
- **Action**: close as duplicate
- **Priority**: (inherits P1 from the original)
- **Suggested assignee**: (blank - original is already assigned)
- **Labels to add**: triaged-web, r: duplicate
- **Labels to remove**: none
- **Reasoning**: duplicate of **#185034** ("[web] Multi-view sizing regression: canvases fail to match host container dimensions based on last view added"), which is open, P1, `triaged-web`, and assigned to `mdebbar`. Both describe the identical mechanism: `OffscreenCanvasRasterizer` owns one `offscreenSurface` shared across all `OffscreenCanvasViewRasterizer` instances, a view calls `setSize()` in `prepareToDraw`, and another view resizes the shared surface before the first view's `rasterizeToImageBitmaps` runs. The reporter even proposes serializing the resize→raster→copy transaction, which is precisely the lock that PR **#185622** already implements against #185034.
- **Comment**: suggested text — "Thanks for the unusually thorough writeup, including the 120-sample headless probe and the `forceSingleThreadedSkwasm: true` control. This is the same shared-`OffscreenSurface` race tracked in #185034, and the serialization you propose is what PR #185622 implements. Closing as a duplicate so the discussion stays in one place. Your popup-window repro exercises a case the existing `rasterizer_order_test.dart` does not — two concurrent view render queues rather than two sequential frames on one view — so it would be very welcome as a test case on #185622."
- **Root cause**: recorded on #185034. Worth carrying over one detail this report adds that the original does not: it reproduces under `forceSingleThreadedSkwasm: true`, so the race is in the render-queue interleaving rather than in cross-thread access, and a threading-level fix alone would not close it.

### #192687 - Web debug (DDC) start-up regressed ~4x between 3.24.5 and 3.44.9 on a large app
- **Action**: re-route
- **Priority**: P1
- **Suggested assignee**: (blank - re-routing off team-web)
- **Labels to add**: triaged-web, team-tool, fyi-web, tool, P1, dependency: dart
- **Labels to remove**: team-web
- **Reasoning**: the two moving parts are the DDC module loader (`dart-sdk/lib/dev_compiler/ddc/ddc_module_loader.js`, a Dart SDK artifact) and the `flutter_tools` dev server that serves the modules. Neither is web-engine code, and the routing order sends dev-server and CLI issues to `team-tool`. Adding `dependency: dart` because the loader constants (`maxRequestPoolSize = 1000`, `maxAttempts = 6`) live in the SDK and a real fix likely needs an SDK change alongside the tool change. **P1 rather than the P2 regression floor**: debug web start-up is about as widely used as a web feature gets, the app never reaches first frame at all on 3.44.9 and 3.47.4, and the only workaround the reporter found is `--profile`, which costs them hot reload and nine minutes per cycle.
- **Root cause**: web-only. The require.js loader was replaced by a DDC loader that fires all module requests into a pool of 1000 and gives up after 6 attempts. At ~1936 modules the dev server goes CPU-bound and serves a 1.4 KB module in ~0.9-1.1 s against ~0.44 s on 3.24.5, so the pool saturates, retries pile onto an already-saturated server, and the loader exhausts `maxAttempts` at roughly 1600 of 1933 modules and stops. The reporter's measurement that a hand-issued request still returns in 0.7 s at that moment is the key detail: the server is not deadlocked, the client-side loader has simply given up. Module count is dominated by framework and package code rather than app code — a 3x smaller app still declares 1731 modules — so splitting the app is not a workaround and any large app is exposed.

### #193046 - Buttons become unresponsive on Flutter web when opened in an iframe, zoom level <100%
- **Action**: triage (keep team-web)
- **Priority**: P2
- **Suggested assignee**: `mdebbar`
- **Labels to add**: triaged-web, P2, engine, browser: chrome, found in release: 3.47
- **Labels to remove**: none
- **Reasoning**: genuinely web-engine territory — pointer coordinate mapping in an embedded, fractionally-scaled view — so it stays on `team-web`. Independently reproduced by puneetkukreja98 on stable 3.47.5 at 75% zoom, so `has reproducible steps` is already correctly applied. Suggesting `mdebbar` as the closest area owner: this is iframe embedding and view geometry rather than semantics or rendering. P2 because the trigger combination (iframe **and** `border-radius` on the iframe **and** sub-100% zoom **and** device emulation) is narrow, and dropping the `border-radius` is an immediate workaround.
- **Root cause**: not confirmed, so treat this as the lead rather than an answer. The `border-radius` dependency is the informative part — a border radius on the iframe element forces the browser to promote it to its own compositing layer with a clip, and combined with a fractional zoom factor the rounding between CSS pixel coordinates and the engine's device-pixel hit-test grid appears to leave pointer coordinates offset enough to miss their targets. Worth checking `pointer_binding.dart`'s client-coordinate conversion and the `devicePixelRatio` rounding in view geometry against a fractional zoom, and confirming whether the offset scales with the radius value.
- **Cleanup**: the code sample is the unmodified counter template and does not include the `index.html` that actually triggers the bug — that only appears in puneetkukreja98's reproduction comment. Worth editing the issue body to inline the iframe `index.html` so the repro is self-contained.

---

## PR Triage

### flutter/flutter - 26 untriaged, above the 15 threshold

**The kevmoo a11y cluster is the highest-value thing on the board this week.** Four PRs opened 5 days ago, all `REVIEW_REQUIRED`, three of the four with no reviewer assigned at all:

| PR | Title | Reviewer | Note |
|---|---|---|---|
| #192963 | Preserve DOM focus on role update and honor `isAccessibilityFocusBlocked` | none | **Fixes #192792 and #191484** |
| #192964 | Propagate `aria-label` to inner slider and text field inputs | none | |
| #192965 | Omit group role on menu scrollables, region role on named routes | none | **Fixes #190661**, the standing unassigned P1 |
| #192971 | Add `SemanticsRole.listBox` and `SemanticsRole.option` | mboetger | |

Two of these close issues that have been sitting in this triage queue — one of them for three weeks as the only unassigned P1. They are squarely in `flutter-zl`'s area. Add `triaged-web` to all four and get reviewers onto them.

**Stale and orphaned** (no reviewer ever assigned, or no review activity in 30+ days):

- **#184281** - [web] Enable address bar collapse on mobile by switching touch input. Created 5 months ago, still zero reviewers. **Third week flagged.** Either give it an owner or close it.
- **#185622** - [web] Fix multi-view sizing race condition (Lock approach). 4 months old, `CHANGES_REQUESTED`, last touched a month ago, no reviewer listed. Newly important: #192784 arrived this week as a duplicate of the issue this PR fixes, which is a second independent report of the same race. Worth unblocking.
- **#188628** - Add opt-in Skwasm multi-surface rasterizer. 2 months, no reviewer, `REVIEW_REQUIRED`, but active 2 days ago and carrying substantial paired benchmark data (68-78% raster reduction on Safari, 79-92% on Firefox). Needs `harryterkelsen`.
- **#189835** - [web] Stop percent-decoding URLs written to browser history. **APPROVED, not landed, untouched for a month.** Third week flagged. This just needs someone to merge it.
- **#190126** - [web] Fix password-manager autofill not reaching text fields. `CHANGES_REQUESTED`, 21 days idle. Worth cross-linking to #192544, which is a different autofill-form failure in the same `dormantForms` machinery.
- **#191529** - fix: prefer pointer events over semantics tap for mouse clicks on web. 13 days idle, `mdebbar` requested. **Second week flagged.** Touches the same hit-test path as #193046 above.
- **#190959**, **#191107** - 13 days idle each, reviewers assigned but silent.

**Approved and moving** (no action beyond landing): #191003, #192664, #193090.

**Resolved since last week**: #188010 is active again (updated 2 hours ago, now `REVIEW_REQUIRED` with yjbanov / kevmoo / mdebbar). #190856 has **merged**, so the standing `r: duplicate` request is now cosmetic on a closed PR — drop it from the follow-up list unless label hygiene on merged PRs matters to you.

### flutter/packages - 5 untriaged

- **#11966** - [google_maps_flutter_web] Fix AdvancedMarker anchors on web. **APPROVED**, active minutes ago. Third week on the list. Land it.
- **#12357** - [google_maps_flutter] Add background color for unloaded tiles. **APPROVED**, 7 days idle. Land it.
- **#7950** - [camera_web] Re: Support for camera stream on web. Open since October 2024, `CHANGES_REQUESTED`, but active 20 hours ago after last week's flag. Someone is moving on it. Give it a reviewer or make the call to close.
- **#12186**, **#11872** - `CHANGES_REQUESTED`, waiting on their authors. No action.

---

## Triage Summary - September 23, 2026

- Triaged: 8 issues
- Close: 1 issue (#192784, duplicate of #185034)
- Request info: 0 issues
- Re-route: 4 issues (#192054, #192433, #192687 to team-tool; #192704 to team-ecosystem)
- Confirm-only, routing already set by others: 2 issues (#192544, #192792)
- Keep on team-web: 1 issue (#193046)
- Remaining untriaged: 0 issues
- Priority spread: P1 x3 (#192544, #192687, #190661 carried), P2 x2 (#192054, #192792, #193046), P3 x2 (#192433, #192704)

### Carry-overs not acted on

1. **#192054** - re-route recommended September 2 and September 9, never applied. Third appearance.
2. **#192433** - re-route recommended September 9, never applied. Second appearance.
3. **#190661** - unassigned P1 for 41 days, wrong labels for 3 weeks. Now has a fix PR, so the shape of the ask changed.
4. **#184281** - 5 months, no reviewer ever. Third week flagged.
5. **#189835** and packages **#11966** - approved, still not landed. Third week flagged.
6. **#191529** - 13 days idle. Second week flagged.

### Recommended actions, in order

1. **Review PR #192965.** It closes #190661, the only unassigned P1 on the board and a 3-week carry-over. No reviewer is assigned. Reviewing this is strictly better than assigning the issue, which is what the last two logs recommended.
2. **Review PR #192963.** Closes #192792 from this week plus #191484. No reviewer assigned. Ask for the bundled `isAccessibilityFocusBlocked` change to be split out, or at least reviewed as a separate commit.
3. **Apply the #192054 and #192433 re-routes to `team-tool`.** Both have now bounced through triage twice with no label change. Applying them is the single cheapest way to shrink next week's queue.
4. **Close #192784 as a duplicate of #185034**, and ask the reporter to contribute the popup-window repro as a test on PR #185622. It covers concurrent view render queues, which the existing `rasterizer_order_test.dart` does not.
5. **Re-route #192687 to `team-tool` at P1.** A 4x debug start-up regression that never reaches first frame is the most severe new issue this week, and it is not a web-engine bug.
6. **Get reviewers onto #192964 and #192971**, the other two PRs in the kevmoo a11y cluster, while the context is loaded.
7. **Land the approved-and-idle PRs**: #189835, packages #11966, packages #12357.
8. **Unblock #185622.** Two independent reports of the same multi-view race now exist, and the fix has been sitting at `CHANGES_REQUESTED` for a month with no reviewer.
9. **Assign #193046 to `mdebbar`** and cross-link it to #191529, which touches the same hit-test path.
10. **Decide on #184281 and packages #7950.** Both have gone months or years without an owner. Either assign or close; re-flagging them weekly is not triage.
11. **Cross-link #190126 and #192544.** Different symptoms, same autofill `dormantForms` machinery. Whoever picks up one should see the other.
