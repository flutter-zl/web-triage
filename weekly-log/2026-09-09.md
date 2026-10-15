# Flutter Web Triage - September 9, 2026

## Weekly Health Checks

### 1. Unassigned P0/P1 (1)

**#190661** - [web] VoiceOver announces menu items with an off-by-one position/count (N-item menu reads as "2 of N+1")
- P1, `team-web`, `triaged-web`, unassigned. Last touched 2026-08-26, so no movement in 14 days.
- **Carry-over from September 2.** Same recommendation was made last week and was not applied. Not a zombie, it is a real open bug.
- **Recommend assigning to `flutter-zl`.** The `role="menu"` node emits the `SingleChildScrollView` as a `role="group"` child while the real items are hoisted with `aria-owns`, so macOS VoiceOver counts N+1 members. NVDA and JAWS honor `aria-owns` and get it right.
- **Label cleanup still pending.** `p: material_ui`, `package`, and `platform-macos` are all wrong.
  - Remove: `p: material_ui`, `package`, `platform-macos`
  - Add: `platform-web`, `framework`, `engine`

No open P0 issues on `team-web`, so the P0 weekly-update check is a no-op.

### 2. Open PR count - FLAGGED

25 untriaged web PRs on flutter/flutter, above the 15 threshold. Unchanged from last week, the backlog did not move.

5 untriaged web PRs on flutter/packages.

### 3. Stale "waiting for customer response"

None. Clean.

### 4. Stale PRs (30+ days without activity)

- **#188010** `refactor(web): use singular crossOriginStorage.requestFileHandle()` (tomayac) - approved 2026-08-05, idle 35 days. Only stale PR this week. It was on last week's "land these" list and still has not landed.

---

## Issue Triage

### #192227 - "Null check operator used on a null value" during hot reload
- **Action**: re-route
- **Priority**: P2
- **Suggested assignee**: (blank - re-routing off team-web)
- **Labels to add**: triaged-web, team-tool, fyi-web, P2
- **Labels to remove**: team-web
- **Reasoning**: The failure is in the recompile path, not the web engine. `-d web-server` is only the device that surfaces it. goderbauer already cc'd bkonyi and traced it to `pkg/frontend_server/lib/frontend_server.dart` catching the exception and forwarding it without a stack trace. P2 rather than P1 because the reporter has no reliable repro yet, so nobody can act on it until the stack trace lands.
- **Comment**: none needed, goderbauer is already driving it and has a Dart CL open at `https://dart-review.googlesource.com/c/sdk/+/545260` to recover the stack trace.

### #192054 - Flutter web serving stale app on reload of tab
- **Action**: re-route
- **Priority**: P2
- **Suggested assignee**: (blank - re-routing off team-web)
- **Labels to add**: triaged-web, team-tool, fyi-web, tool, P2
- **Labels to remove**: team-web
- **Reasoning**: **Carry-over from September 2**, the re-route was recommended then and never applied. Since then bkonyi opened a fix, which confirms the routing: the whole thing lives in `flutter_tools`.
- **Root cause**: the dev server keys DDC library-bundle modules by their strongly-connected-component name. Breaking an import cycle changes the SCC, so `main.dart` recompiles under a new key while the old bundled module stays in `WebMemoryFS`. DWDS then lists both in `main_module.bootstrap.js`, and on a tab reload the stale module executes first. Hot restart re-resolves the entrypoint, which is why only the reload path shows it.
- **PR review**: #192428 (bkonyi, still draft) makes `WebMemoryFS.write` read library URIs out of the incoming module metadata and evict any prior module that defined those libraries, then rebuilds `_mergedMetadata` from the surviving modules and prunes `_modules`/`_digests` in `WebAssetServer` to match. That addresses the root cause directly rather than papering over the bootstrap. Two things worth a look before it leaves draft: `_parseLibraries` swallows all parse failures and returns an empty set, which silently degrades to today's stale-module behavior instead of surfacing a problem, and the new `_modules.removeWhere` reconstructs the `.js` filename by string concatenation after `path` was produced by `replaceAll('.js', '')`, which is not round-trip safe for a module whose name contains `.js` elsewhere. It has unit and regression tests, and the approach is sound.

### #192347 - [Web][CanvasKit] WebGL: INVALID_VALUE: texImage2D: no image when rendering Slivers with RepaintBoundary
- **Action**: triage (keep team-web)
- **Priority**: P2
- **Suggested assignee**: `harryterkelsen`
- **Labels to add**: triaged-web, P2, engine, browser: chrome, found in release: 3.47
- **Labels to remove**: none
- **Reasoning**: CanvasKit texture upload path, which is harryterkelsen's area, and it is already reproduced by a team member on 3.47.2. P2: real and reproducible, but the repro needs a fairly specific setup and the visible damage is console spam plus dropped texture layers rather than a hard break. I did not add `c: regression` because nobody has confirmed a last-known-good version. The "3.27+" in the title is the reporter's claim, not a bisect.
- **Root cause**: very likely the same image-ownership bug as #191800. `makeTexture` throwing `texImage2D: no image` means the upload read an `<img>` element with no pixels, and #191800 is exactly that: disposing the last Dart image handle clears the backing element while a recorded `SkPicture` still holds the lazy `SkImage`. The repro uses `cached_network_image` with a periodic `setState`, which churns `ImageCache` eviction on every tick, which is #191800's documented trigger.
- **Comment**: suggest asking the reporter to retest against PR #192354, which fixes #191800 by leaving the element's `src` intact on close. If it clears, close #192347 as `r: duplicate` of #191800. Do not close it as a duplicate yet, the "since 3.27" claim conflicts with #191800 being scoped to a 3.47.0 regression, so they may not be the same thing.

### #192155 - [google_maps_flutter_web] Add double-tap and context-menu callbacks for interactive map objects
- **Action**: re-route
- **Priority**: P3
- **Suggested assignee**: (blank - re-routing off team-web)
- **Labels to add**: triaged-web, team-ecosystem, fyi-web, P3
- **Labels to remove**: team-web
- **Reasoning**: **Carry-over from September 2**, unchanged and not applied. First-party plugin in flutter/packages. The capability only exists on web, but the proposal adds public API to the app-facing package and the platform interface, so ecosystem owns the design call, including the `consumeDoubleTapEvents` / `consumeContextMenuEvents` question the reporter explicitly asks for feedback on. P3 per the guide: single reporter, no demonstrated multi-user demand.
- **Note**: the reporter also authored packages PR #11952, so they will likely implement once the API shape is settled.

### #192433 - Hot reload requests should have a common path
- **Action**: re-route
- **Priority**: P3
- **Suggested assignee**: (blank - re-routing off team-web)
- **Labels to add**: triaged-web, team-tool, fyi-web, P3, c: proposal
- **Labels to remove**: team-web
- **Reasoning**: The 404s are on `web_entrypoint.dart.lib.js` and `web_plugin_registrant.dart.lib.js`, which `WebAssetServer` in `flutter_tools` serves and the DDC module loader requests. No web engine code is involved. This is an enhancement request from one reporter with a working, if fragile, workaround, so P3.
- **Root cause**: the generated bootstrap requests DDC modules by server-root-relative path and ignores the `<base href>` in `index.html`, so every request lands outside the reverse proxy's prefix. The reporter's real complaint is that the path layout has no stable prefix and changes between releases, which is a dev-server API-stability question for tool team.

### #192464 - Shaka Player integration for video_player web
- **Action**: re-route
- **Priority**: P3
- **Suggested assignee**: (blank - re-routing off team-web)
- **Labels to add**: triaged-web, team-ecosystem, fyi-web, P3
- **Labels to remove**: team-web
- **Reasoning**: Same call as #192155. `video_player` is an ecosystem-owned first-party package, and the proposal is a new federated implementation package plus platform-interface surface for audio-track and quality selection, so ecosystem owns the architecture decision. P3: a design doc, not a bug, with no demonstrated multi-user or top-tier-customer demand attached to the doc itself.
- **Comment**: keep `fyi-web` on it and ask `mdebbar` to review the web-facing parts specifically, namely how shaka-player is loaded (CDN vs bundled, and the CSP implications for apps that lock down `script-src`) and the JS interop boundary. That is the part of the design where web team has context ecosystem does not.

### #192466 - [web][iOS Safari] Long Semantics label overflows its button and intercepts taps on a later sliver
- **Action**: triage (routing already set by another team member, confirm only)
- **Priority**: P2
- **Suggested assignee**: `flutter-zl`
- **Labels to add**: triaged-web, P2, engine
- **Labels to remove**: none
- **Reasoning**: Already carries `team-accessibility` plus `fyi-web` from a prior re-route, so per the guide the routing stays and web team just closes out its review. Excellent report: measured element rects, four controlled comparisons, and `elementsFromPoint` output. P2 rather than P1 because it needs semantics enabled plus a long label plus a sliver layout, and it does not reproduce in `Column`. Bump to P1 if a second reporter hits it, because a tap silently firing the wrong widget's callback is a correctness break, not a cosmetic one.
- **Root cause**: two engine behaviors combine. First, every `flt-semantics` element is created with `overflow: visible` (`engine/src/flutter/lib/web_ui/lib/src/engine/semantics/semantics.dart:733`), and a button's label is appended as a raw DOM text node directly to that element by `DomTextRepresentation` (`engine/src/flutter/lib/web_ui/lib/src/engine/semantics/label_and_value.dart:147-162`). With no width constraint the long label wraps and paints far below the node's 52x52 box, which matches the reporter's 459px-tall text range against a 52px element. Second, sibling z-index is assigned in reverse child order (`.../semantics/semantics.dart:2089-2096`, `zIndex = childCount - i`), so the earlier sliver child (UPPER, z-index 5) stacks above the later one (LOWER, z-index 3). Safari's hit test then returns UPPER for a point inside LOWER and the engine dispatches the tap there. In the `Column` control the z-index order comes out 4/6, LOWER wins, and the bug hides. That is why the reporter's `?layout=column` case passes.
- **Potential solutions**:
  1. Clip the label text to the node box. Setting `overflow: hidden` on nodes that use `DomTextRepresentation` keeps the text available to screen readers via the accessibility tree while removing it from the hit-test area. Needs care: `overflow: visible` is deliberate for nodes whose children are transformed outside the parent's box, so this should be scoped to leaf label-bearing roles, not applied globally.
  2. Constrain the text node itself instead of the host. Wrap the label in a span sized to the node's rect the way `SizedSpanRepresentation` already does, so wrapping happens inside the box regardless of label length.
  3. Do not rely on the browser's hit test for semantic taps at all. Resolve the target from the semantics tree using the framework's own hit-test result, which would also fix the general class of stacking mismatches rather than this one instance. Largest change and the riskiest.

---

## PR Triage

### flutter/flutter - 25 untriaged (FLAGGED, over the 15 threshold)

All 25 already carry `platform-web` and `team-web` and need `triaged-web` added. Notes on the ones that need more than a label:

**No reviewer assigned at all (REVIEW_REQUIRED, zero requested reviewers) - 8 PRs.** These are the real backlog, nobody is on the hook for any of them:
- **#184281** `[web] Enable address bar collapse on mobile by switching touch input` (koji-1009) - open since 2026-03-28, over 5 months with no reviewer ever assigned. Worst offender on the list. Touches mobile touch input, so `mdebbar`.
- **#190651** `fix(web): guard parseBrowserLanguages against invalid locale tags` (sankalpsthakur) - small defensive fix, needs any web reviewer.
- **#191342** `chore: correct copy paste error in test name` (holzgeist) - trivial test-name typo, has sat 21 days. Should be a one-minute merge.
- **#191529** `fix: prefer pointer events over semantics tap for mouse clicks on web` (rkishan516) - **needs `flutter-zl`**, this is semantics tap handling and it interacts directly with the #192466 hit-test area triaged above. Also a dual-action risk: it adds a pointer path alongside the existing semantics tap path.
- **#191647** `[web] Report env(safe-area-inset-*) as FlutterView.viewPadding` (diegolopezrm) - labeled `f: scrolling`, so `flutter-zl` or `mdebbar`.
- **#192354** `[web] Preserve image element pixels for retained pictures` (yhisazumi) - fixes #191800, which is assigned to `harryterkelsen`. **Assign him as reviewer**, and note it may also resolve #192347 triaged above. Flag: the author states the full web test suite and a locally built engine were not run, and that the change is AI-assisted. Worth a careful look at the ownership change before landing.
- **#191003** `Make web test subshards dynamically determined by .ci.yaml` (mdebbar) - author is web team, needs a reviewer picked. Add `a: tests`.
- **#185622** `[web] Fix multi-view sizing race condition (Lock approach)` (mdebbar) - CHANGES_REQUESTED with no reviewer currently on it, 22 days idle. Author needs to re-request.

**Approved and idle, just needs someone to land them:**
- **#188010** (tomayac) - approved, 35 days idle. **Stale, see health checks.**
- **#189835** (apinilabs-pascal) - approved, 19 days idle.
- **#189506** (Gustl22) - approved, `flutter-zl` is a requested reviewer, updated today. **In your review queue.**

**Label fixes:**
- **#190856** (flutter-zl) - **still carries `r: duplicate`.** Flagged September 2 and not removed. It is not a duplicate.
- **#191003** - add `a: tests`.
- **#192354** - add `a: images`.

**Blocked on the author, no action needed from web team:** #185152, #189953, #190126 (all CHANGES_REQUESTED with a named reviewer).

**Healthy, active review in progress:** #184029, #189858, #190942, #191107, #191736, #192072, #192128, #192224, #192260, #192328.

### flutter/packages - 5 untriaged

All need `triage-web` cleared once reviewed.

- **#11966** `[google_maps_flutter_web] Fix AdvancedMarker anchors on web` (3ph) - approved, 27 days idle. Was on last week's land list and still has not landed. Closest to stale.
- **#7950** `[camera_web] Re: Support for camera stream on web` (TecHaxter) - open since 2024-10-28, CHANGES_REQUESTED, no reviewer. Nearly 11 months. Recommend either finding it an owner or closing it as abandoned and asking the author to reopen if they pick it back up.
- **#12186** `[google_maps_flutter_web] Issue 64073 implement my location` (Zubii12) - CHANGES_REQUESTED, no reviewer. Blocked on author.
- **#12357** `[google_maps_flutter] Add background color for unloaded tiles` (RyanHolanda) - approved, three reviewers requested, active. No action.
- **#11872** `[google_maps_flutter] Add onPointOfInterestTap callback` (tenninebt) - CHANGES_REQUESTED, reviewers on it. Blocked on author.

---

## Triage Summary - September 9, 2026

- Triaged: 7 issues
- Close: 0 issues
- Request info: 0 issues
- Re-route: 5 issues (#192227, #192054, #192433 to team-tool; #192155, #192464 to team-ecosystem)
- Confirm-only, routing already set by others: 1 issue (#192466)
- Keep on team-web: 1 issue (#192347)
- Remaining untriaged: 0 issues
- Priority spread: P2 x4 (#192227, #192054, #192347, #192466), P3 x3 (#192155, #192433, #192464)

### Carry-overs not acted on since September 2

Three items were recommended last week and are still in the same state:
1. #190661 - unassigned P1, wrong labels, now 14 days without movement.
2. #192054 and #192155 - re-routes never applied, so both resurfaced in this week's untriaged queue.
3. #190856 - `r: duplicate` label still attached.
4. #188010 and packages #11966 - approved, still not landed.

### Recommended actions, in order

1. **Assign #190661 to `flutter-zl` and fix its three wrong labels.** Unowned P1 for 34 days now, and it is the only unassigned P1 on the board.
2. **Apply the #192054 and #192155 re-routes.** Both were triaged on September 2 and both came back this week because the labels were never changed. Applying them removes two issues from next week's queue.
3. **Assign #192347 to `harryterkelsen` and cross-link it to #191800.** Ask the reporter to retest against #192354.
4. **Take #192466 yourself at P2.** Root cause is confirmed in the source, three candidate fixes are written up above, and it overlaps with PR #191529 which is also unreviewed.
5. **Review #189506.** Approved, you are a requested reviewer, updated today.
6. **Review #191529 or hand it to someone.** Semantics tap handling with a dual-action risk, 18 days with no reviewer, and it touches the same hit-test path as #192466.
7. **Land the idle approved PRs**: #188010 (35 days stale), #189835, packages #11966.
8. **Remove `r: duplicate` from PR #190856.** Second week asking.
9. **Merge #191342.** A test-name typo fix should not sit for 21 days.
10. **Decide on packages #7950.** Open since October 2024 with no reviewer. Either give it an owner or close it as abandoned.
11. **Find reviewers for the 8 orphaned PRs.** The 25-PR backlog did not move at all this week, and #184281 has now gone 5 months without a reviewer ever being assigned.
