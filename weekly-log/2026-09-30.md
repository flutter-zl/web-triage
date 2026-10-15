# Flutter Web Triage - September 30, 2026

This was a read-only pass. Nothing was changed on GitHub. Everything below is a recommendation for you to apply by hand.

---

## Weekly Health Checks

### 1. Unassigned P0/P1: CLEAR

There are no unassigned P0/P1 issues on `team-web`. #190661, carried over for 3 weeks, is off the list, so last week's review ask on PR #192965 went through.

### 2. Open PR count: FLAGGED

There are **26 untriaged web PRs** on flutter/flutter. The threshold is 15. The count was 26 last week and 25 the week before, so it hasn't moved. Four PRs left the list (#192965, #191003, #192664, #193090) and four new ones came in.

### 3. P0 issues missing a weekly update: N/A

There are no open P0 issues on `team-web`.

### 4. Stale "waiting for customer response": CLEAR (but see note)

The guide's query returns nothing. **Note:** the label on current issues is `waiting for response`, not `waiting for customer response`. The label looks renamed, so the guide's queries no longer match it:
- The stale check matches nothing.
- The untriaged query's `-label:"waiting for customer response"` filter doesn't exclude anything, which is why #193439 shows up below.

Running the check again with `waiting for response` turns up only #193439, which was updated 2 days ago and isn't stale. **Recommend updating both queries in `web-triage.md` to use `waiting for response`.**

---

## Issue Triage

There are 11 untriaged issues. Most of them only need a `triaged-web` confirmation because another team already owns them.

### #192618 - Semantics Accessibility issues with aria-label
- **Action**: re-route + request info (lean toward closing as WAI)
- **Priority**: P3
- **Suggested assignee**: (blank, re-routing off team-web)
- **Labels to add**: triaged-web, team-accessibility, fyi-web, P3, framework, found in release: 3.47, waiting for response
- **Labels to remove**: team-web
- **Title cleanup**: `[web] Buttons expose accessible name via DOM text instead of aria-label; Semantics wrapper adds an extra node`
- **Comment**: "On web, Flutter gives buttons their accessible name through DOM text, not `aria-label`, on purpose. It works better with crawlers, and JAWS ignores `aria-label` on empty elements (#122607). A name taken from content is valid under ARIA and passes WCAG 4.1.2, which you can confirm in Chrome's Accessibility pane. The extra node in the Cancel example happens because `ElevatedButton` already has button semantics, and the outer `Semantics(button: true, ...)` adds a second button around it. Setting `excludeSemantics: true` on the outer `Semantics`, which is commented out in your sample, removes the duplicate. Could you share which audit tool or rule needs an explicit `aria-label`? That would help us decide whether to add a way to opt into `aria-label` for buttons."
- **Reasoning**: suryaKommana2662 said he was routing this to `team-accessibility` 19 days ago, but the labels were never changed. The ask ("client needs aria-label") is a compliance preference rather than an ARIA violation. The Dropdown2 case is also a third-party package.
- **Root cause**: the web engine picks a `LabelRepresentation` per role. Buttons use `domText` on purpose ([tappable.dart:14](https://github.com/flutter/flutter/blob/master/engine/src/flutter/lib/web_ui/lib/src/engine/semantics/tappable.dart#L14)), so the name comes from content and there's no `aria-label`. The extra node comes from app code: `ElevatedButton` already creates its own button semantics node, so wrapping it in `Semantics(container: true, button: true, label:)` nests one button inside another. A node with children switches to `aria-label` ([label_and_value.dart:478](https://github.com/flutter/flutter/blob/master/engine/src/flutter/lib/web_ui/lib/src/engine/semantics/label_and_value.dart#L478)), which is why only the wrapper gets the attribute.
- **Potential solutions**: (1) Close as WAI with an explanation. (2) A P3 opt-in to force `aria-label` for buttons, only if a mainstream audit tool turns out to flag this.

#### Follow-up investigation

**Demo**: https://flutter-demo-69-before.web.app. Source: `~/dev/flutter-apps/working on/issue_192618_before`. Test steps: `~/dev/flutter-demo-apps/working on/issues logs/issue_192618/test_instructions.md`. It uses the built-in `DropdownButtonFormField` in place of the third-party `dropdown_button2`.

**Verified DOM**, from a headless Chrome dump of the demo. All four claims reproduce:

| Widget | DOM | `aria-label` |
|---|---|---|
| TextFormField | `<input aria-label="First name">` | yes, the control case |
| Plain `ElevatedButton` | `role="button"`, text "Submit All" | no, the name comes from content |
| `ElevatedButton` in `Semantics(label: 'Cancel', button: true)` | outer `role="button" aria-label="Cancel"` with **no tabindex**, containing inner `role="button" tabindex="0"` with text "Cancel" | outer only. Focus lands on the inner, unlabeled button, and a button inside a button is invalid ARIA |
| `DropdownButtonFormField` in `Semantics(container: true)` | `role="button" aria-expanded="false"`, text "Department / Select a department sample hint" | no, the name comes from content |

**Design history**: buttons used to render `aria-label`. Yegor moved them to DOM text in flutter/engine#50794 (2024, commit `826074672d7`, fixing #122607) for two reasons:
- JAWS [ignores `aria-label` on empty elements](https://github.com/FreedomScientific/standards-support/issues/759).
- Crawlers ignore `aria-label`.

Going back to `aria-label` alone would bring the JAWS bug back. Adding `aria-label` on top of the DOM text avoids that, but it has problems of its own:
- Google Translate rewrites DOM text but not `aria-label`, so on a translated page the name read aloud would stay untranslated while the visible text changes.
- It adds nothing for users, since the name is identical.
- The same argument would then apply to links and headings.

**Reporter's use case**: this is inferred, because the only stated reason is one sentence: "It does have a computed name but to meet accessibility requirements, client needs aria-label." The reporter looks like an agency or contractor with a client accessibility checklist that tests whether the `aria-label` attribute is present, not whether a name exists. The sample reads like an audit test page: one of every control type, a radio group that sets `role`, `label`, `tooltip` and `hint` together, and a commented-out `excludeSemantics: true` workaround. It looks like a bug to them for three reasons:
- Text fields get `aria-label` and buttons don't, which looks inconsistent.
- Their check wants the attribute itself, not the computed name.
- The `Semantics` workaround produced an extra node.

No screen reader problem is reported, the audit tool isn't named, and the reporter hasn't replied since 2026-09-11.

**Decision**: don't add `aria-label` to buttons by default. The deciding question is which tool requires it:
- axe `button-name`, Lighthouse and WAVE all accept a name taken from content.
- If the answer is a mainstream checker, look again. If it's a contractual checklist, it's at most a P3 opt-in feature request.

### #193243 - [web] CanvasKit: intermittent "Null check operator used on a null value" at startup with CPU-only rendering (no WebGL) under CPU contention
- **Action**: triage
- **Priority**: P2
- **Suggested assignee**: harryterkelsen
- **Labels to add**: triaged-web, P2, has reproducible steps, found in release: 3.44, found in release: 3.47
- **Labels to remove**: none
- **Reasoning**: The symbolized stack is clear and there's a public stress harness, but it needs a niche config (no WebGL plus CPU starvation), fails in only 6-11% of loads, and the app recovers. Valid, but not urgent.
- **Root cause**: `CanvasKitRenderer.initialize` creates the picture-to-image surface via `MultiSurfaceRasterizer.createPictureToImageSurface`. Code awaiting `CkSurface.initialized` then force-unwraps state that the CPU-only (software surface, no `GrContext`) path hasn't set yet when the first frame races ahead under starvation.
- **Potential solutions**: (1) Null-guard the CPU-only branch after `initialized` completes. (2) Create the picture-to-image surface lazily on first `toImage` instead of during `initialize`. (3) Gate the first frame on surface initialization.

### #193221 - [Web] Disposing a shader before `Picture.toImage` changes sampled pixels in Chrome
- **Action**: triage
- **Priority**: P2
- **Suggested assignee**: harryterkelsen
- **Labels to add**: triaged-web, P2, e: web_canvaskit, has reproducible steps, found in release: 3.44
- **Labels to remove**: none
- **Title cleanup**: `[web] FragmentShader.dispose() before Picture.toImage renders black; native keeps shader alive`
- **Reasoning**: The repro is minimal (release build, default renderer, so CanvasKit). The web behavior doesn't match native ref-counted lifetime semantics or the spirit of the `Shader.dispose` docs.
- **Root cause**: on web, the recorded `Picture` holds only the Dart-side shader and resolves the Skia shader when it rasterizes. `FragmentShader.dispose()` deletes the underlying `SkShader` right away, so replaying in `toImage` draws with no shader (black). Native pictures hold an `sk_sp` ref that keeps the shader alive.
- **Potential solutions**: (1) Have the picture/paint take a counted ref (`CountedRef`) on the Skia shader at record time, so `dispose()` only drops the Dart ref. (2) Snapshot the shader into the display list at `drawRect` time. (3) Otherwise, document `FragmentShader` lifetime explicitly.

### #193287 - [web] SelectionArea in MaterialApp.builder throws "RenderBox was not laid out" on startup (refile of #191508)
- **Action**: triage (confirm routing only)
- **Priority**: (leave to team-text-input; P2 suggested)
- **Suggested assignee**: (blank, owned by team-text-input)
- **Labels to add**: triaged-web
- **Labels to remove**: none
- **Reasoning**: suryaKommana2662 already moved it to `team-text-input` + `fyi-web`, so per the rule we only close out the web review. It isn't a duplicate: #191508 was auto-closed while waiting for response. A contributor (BartSimpson001) says they're investigating.
- **Root cause**: web-specific trigger. The web embedder sends a view-focus change at startup before the first layout. `_ViewState.didChangeViewFocus` then runs `findFirstFocus`, which reads `FocusNode.rect` and calls `RenderBox.size` on the `SelectionArea`'s node inside the not-yet-laid-out `Overlay` from `builder`.
- **Potential solutions**: (1) In `didChangeViewFocus`, defer `findFirstFocus` to a post-frame callback when the tree isn't laid out yet. (2) Skip nodes whose render box lacks `hasSize` in focus traversal sorting.

### #193235 - web: `EngineSemanticsOwner.updateSemantics` throws `Null check operator used on a null value` when a sent parent lists a child id with no `SemanticsObject`, and never recovers the tree
- **Action**: triage (confirm routing only)
- **Priority**: (leave to team-accessibility; P1 suggested after confirming the repro, see follow-up below)
- **Suggested assignee**: (blank, owned by team-accessibility; squarely in flutter-zl's area if web picks it up)
- **Labels to add**: triaged-web
- **Labels to remove**: none
- **Title cleanup**: `[web] Semantics: updateSemantics null-check throw on unknown child id freezes a11y DOM tree`
- **Reasoning**: already on `team-accessibility` + `fyi-web`. There's a minimal repro, production data, and a symbolized stack on 3.47.5. It's related to #175180, whose fix (#177069) is already shipped in the affected builds.
- **Root cause**: `SemanticsObject.updateChildren`, `recomputeChildrenAdjustment`, and `_visitDepthFirstInTraversalOrder` force-unwrap `owner._semanticsTree[childId]!`. The throw escapes before `_finalizeTree()`, and the framework keeps re-sending the inconsistent parent, so every later frame throws too. `reset()` walks the same stale lists before clearing them.
- **Potential solutions**: (1) Skip missing child ids with a debug-only `assert` so the producer bug still shows up. (2) Run `_finalizeTree()` in a `finally`. (3) Clear `_semanticsTree`/`_detachments` before finalizing in `reset()`. Separately, find the framework producer that emits a child id it never sent.

#### Follow-up investigation

**Demo**: https://flutter-demo-70-before.web.app. Source: `~/dev/flutter-apps/working on/issue_193235_before`. Test steps: `~/dev/flutter-demo-apps/working on/issues logs/issue_193235/test_instructions.md`. It's the reporter's `dart:ui` repro, which pushes raw `SemanticsUpdateBuilder` updates into `FlutterView.updateSemantics` 2 seconds after startup. Two changes from the reporter's code:
- The results show on the page inside `ExcludeSemantics`, so the framework tree stays at the root node and doesn't mix with the injected updates.
- It passes `textDirection: null`, which `updateNode` on master now requires.

**Confirmed on master**, dart2js + CanvasKit release build. Headless Chrome and a manual Chrome run gave the same output, which matches the reporter's 3.47.5 run:

```
step c,  0 -> [1, 2], node 2 never sent: THROW: Null check operator used on a null value
step d,  0 -> [1]:                        no throw
step d2, resend 0 -> [1, 2]:              THROW: Null check operator used on a null value
step e,  dispose handle and re-ensure:    ok
step e resend, 0 -> [1]:                  no throw
```

**Observations:**
- Every update whose parent names a missing child id throws a raw `TypeError` from the engine. The semantics DOM afterwards is `node-0 > node-1`, with no `node-2`.
- The owner isn't permanently broken: a corrected child list (step d) goes through. The "never recovers" in production is caused by the sender, because the framework re-sends the bad parent every frame, which is d2 over and over.
- Step e doesn't really test `reset()`. On web, the engine turns semantics on by itself and the framework holds a handle driven by the platform, so disposing the app's handle never reaches `setSemanticsTreeEnabled(false)`. The reporter corrected this themselves. From reading the source, `reset()` still walks the stale child lists before clearing them, and that part hasn't been reproduced.
- The engine fix is simple and doesn't depend on the framework bug. The framework code that sends the bad parent is still unknown, and the reporter hasn't narrowed it down to a widget.

**Recommendation**: raise the suggested priority to **P1** for `team-accessibility`. It's a confirmed engine crash with a minimal repro. In the field it silently freezes the accessibility tree for the rest of the page while pixels keep working, so only screen reader users are affected. The reporter's builds already include the #177069 fix. The engine fix is small and in flutter-zl's area, so the web team could offer to take it even though the issue is routed to `team-accessibility`.

### #193439 - [web] Multi-threaded skwasm hangs or crashes in FreeType: SkMutex is a no-op under -sWASM_WORKERS
- **Action**: no action (waiting on the reporter)
- **Priority**: (hold)
- **Suggested assignee**: harryterkelsen (if it reproduces on master)
- **Labels to add**: none yet
- **Labels to remove**: none
- **Reasoning**: it only shows up because of the label-rename gap above. mbcorona asked the reporter 2 days ago to retest on master, since #190048 (thread-local strike caches, fix for #190039) might cover it.
- **Next step**: if it still reproduces on master, triage as P1 (`c: crash`, permanent hang in production, and the stated root cause of no-op `SkMutex` affects every mutex, not just the strike cache). If it doesn't, close as a duplicate of #190039.

### #193406 - [wasm] Set flutter_runtime_mode in tools/gn to_gn_wasm_args
- **Action**: re-route (label cleanup)
- **Priority**: P2 (already set)
- **Suggested assignee**: (blank, owned by team-engine)
- **Labels to add**: triaged-web, fyi-web
- **Labels to remove**: team-web
- **Reasoning**: it has both `team-engine` and `team-web`. gaaclarke explicitly took it ("Moving to us") and `triaged-engine` is already set.
- **PR review**: #193407 (kevmoo) is a one-line fix: it adds `gn_args['flutter_runtime_mode'] = args.runtime_mode` to `to_gn_wasm_args`, with `gn_test.py` coverage for debug/profile/release and `--target-os wasm`. It fixes the actual root cause: wasm release builds defaulted to `debug`, so `IMPELLER_DEBUG=1` and `CheckFramebufferStatusDebug` fired 50+ times per frame. It's approved by eyebrowsoffire and ready to land. One risk to watch: anything that relied on the debug-mode defines in wasm "release" artifacts, e.g. asserts or tracing, now disappears. That's the intent, but benchmark deltas will shift.

### #192704 - [url_launcher_web] Expose newly created Window instance
- **Action**: triage (confirm routing only)
- **Priority**: P3 (already set)
- **Suggested assignee**: (blank, owned by team-ecosystem)
- **Labels to add**: triaged-web
- **Labels to remove**: none
- **Reasoning**: last week's recommended re-route to `team-ecosystem` was applied, and there's now a single `team-*` label. The web team just needs to close out its review.

### #193452 - [google_maps_flutter_web] projection_test.dart flakily fails with DriverError on web
- **Action**: triage
- **Priority**: P2 (already set)
- **Suggested assignee**: mdebbar
- **Labels to add**: triaged-web, c: flake
- **Labels to remove**: none
- **Reasoning**: ecosystem triaged it and routed it to `team-web` with `fyi-ecosystem`, so keep it. It's a web test-infra flake, which is mdebbar's area. Piinks has disabled the test, so it isn't blocking CI.
- **Root cause**: `projection_test` and `overlays_test` both await `onMapCreated` with no timeout, and `marker_clustering_test` waits on a map ID. When the Maps JS API doesn't finish initializing in headless Chrome, the test never reports and `request_data` returns null after the driver timeout.
- **Potential solutions**: (1) Add a bounded timeout with a clear failure message around `onMapCreated`. (2) Wait for the Maps JS `idle`/`tilesloaded` event, or pre-load the API in the test harness. (3) Retry map creation once before failing.

### #193488 - Triage process self-test
- **Action**: triage
- **Priority**: P2 (already set)
- **Suggested assignee**: (blank)
- **Labels to add**: triaged-web
- **Labels to remove**: none
- **Reasoning**: this is a flutter-triage-bot self-test ("handle as a normal valid low-priority issue"). Other teams already added their `triaged-*` labels.

### #193506 - [web] Render soft hyphens (`Hyphens` API) on CanvasKit, Skwasm, and WebParagraph
- **Action**: triage
- **Priority**: P2
- **Suggested assignee**: harryterkelsen
- **Labels to add**: triaged-web, P2
- **Labels to remove**: none
- **Reasoning**: this is a feature request split out of #18443, a long-standing and heavily upvoted soft-hyphen issue. That demand justifies P2 over the P3 feature default. The scope is clear: three web backends.
- **PR review**: #185152 (dbebawy) adds the cross-platform `Hyphens` API and wires it natively through `setRenderSoftHyphens`. On web it only accepts and stores the value, which is why this issue exists. It's large (52 reviews), `CHANGES_REQUESTED`, and has had no review since 2026-07-22 even though the author pushed today. The web gap is correctly carved out, and CanvasKit is blocked on Skia CL 1378836 exposing `renderSoftHyphens` in `SimpleParagraphStyle`.
- **Potential solutions**: (1) Roll CanvasKit after the Skia CL and pass `hyphens` through `CkParagraphStyle`. (2) Add the matching field to the Skwasm `paragraph_style.cc` / `raw_paragraph_style.dart` bindings. (3) Emit a hyphen glyph in the WebParagraph line breaker when breaking at U+00AD.

---

## PR Triage

### flutter/flutter: 26 untriaged, above the 15 threshold

Add `triaged-web` to all of these. Suggested extra labels and flags:

| PR | Author | Review state | Last review | Add labels | Flag |
|---|---|---|---|---|---|
| #184281 | koji-1009 | REVIEW_REQUIRED | 08-31 | f: gestures | **Stale (30 days). Fourth week flagged.** Assign an owner or close. |
| #185152 | dbebawy | CHANGES_REQUESTED | 07-22 | a: typography | **Stale on the reviewer side (70 days)**. The author pushed today. Needs a re-review. Linked to #193506. |
| #185622 | mdebbar | CHANGES_REQUESTED | 08-18 | | **Stale (43 days).** Fixes #185034, which now has two independent reports. |
| #188628 | MarlonJD | REVIEW_REQUIRED | 09-15 | e: web_skwasm | Needs harryterkelsen. Second week flagged. |
| #189835 | apinilabs-pascal | APPROVED | 08-13 | f: routes | **Approved, not landed, fourth week flagged.** Merge it. |
| #189953 | gaaclarke | CHANGES_REQUESTED | 09-17 | e: impeller, a: tests | Waiting on the author. |
| #190126 | sero583 | CHANGES_REQUESTED | 09-01 | | `waiting for response`. Cross-link #192544. |
| #190959 | gmackall | REVIEW_REQUIRED | 09-09 | | Mostly Android. Web reviewer: flutter-zl for the web side. |
| #191107 | schultek | REVIEW_REQUIRED | 09-10 | | 20 days idle. Needs mdebbar (embedding/multi-view). |
| #191529 | rkishan516 | REVIEW_REQUIRED | 08-22 | a: accessibility | **Stale (39 days). Third week flagged.** Semantics hit-test path: flutter-zl / mdebbar. |
| #192224 | ValentinVignal | REVIEW_REQUIRED | 09-03 | a: typography | 27 days since last review. |
| #192260 | sm-sayedi | REVIEW_REQUIRED | 09-04 | | Cross-platform. The web team only needs to confirm there's no web regression. |
| #192963 | kevmoo | CHANGES_REQUESTED | 09-29 | | Active. Fixes #192792 and #191484. flutter-zl is reviewing. |
| #192964 | kevmoo | APPROVED | 09-25 | | Ready to land. |
| #192971 | kevmoo | REVIEW_REQUIRED | 09-18 | | Needs flutter-zl (listbox/option roles). |
| #193149 | holzgeist | REVIEW_REQUIRED | 09-30 | c: new feature | Document PiP. Active. |
| #193271 | kevmoo | REVIEW_REQUIRED | 09-28 | e: wasm | Active. |
| #193275 | Hellomik2002 | CHANGES_REQUESTED | 09-28 | | Owned by team-engine (Impeller). `triaged-web` only. |
| #193293 | kevmoo | REVIEW_REQUIRED | 09-25 | e: web_skwasm | Needs harryterkelsen. |
| #193362 | diegolopezrm | REVIEW_REQUIRED | 09-25 | f: gestures | Fixes #129933 (Ctrl+wheel zoom). |
| #193385 | rkishan516 | REVIEW_REQUIRED | 09-26 | | Active. |
| #193402 | zzzjim | REVIEW_REQUIRED | 09-26 | f: scrolling | Needs flutter-zl (scroll chaining to the host page). |
| #193426 | kevmoo | APPROVED | 09-29 | a: typography | Ready to land. |
| #193497 | andywolff | REVIEW_REQUIRED | 09-30 | | New. |
| #193510 | mdebbar | APPROVED | 09-30 | a: tests | Ready to land. |
| #193562 | dbebawy | REVIEW_REQUIRED | 09-30 | | New. It's barely web-related (JDK cleanup), so `triaged-web` only. |

**Stale (30+ days with no review):** #184281, #185152, #185622, #189835, #191529.

**Approved and ready to land:** #189835, #192964, #193426, #193510, plus #193407 from the issue section, which isn't `platform-web` labeled.

### flutter/packages: 4 untriaged

- **#7950**: [camera_web] Support for camera stream on web. `CHANGES_REQUESTED`, last review 08-16, **stale (45 days)**, open since October 2024. Assign a reviewer or close.
- **#12357**: [google_maps_flutter] Background color for unloaded tiles. **Approved 08-30 and still not landed.** Second week flagged.
- **#12186**: [google_maps_flutter_web] Implement my location. `CHANGES_REQUESTED` 09-02. The author updated on 09-29, so it needs a re-review.
- **#11872**: [google_maps_flutter] Add `onPointOfInterestTap`. Active today and waiting on the author. No action.
- **Resolved since last week:** #11966 is off the list (landed).

---

## Triage Summary - September 30, 2026

- Triaged: 11 issues
- Close: 0 issues
- Request info: 1 issue (#192618, asking which audit tool needs `aria-label`; #193439 is already waiting on the reporter)
- Re-route: 2 issues (#192618 to team-accessibility; #193406 remove team-web, keep team-engine)
- Confirm-only, routing already set by others: 5 issues (#193287, #193235, #192704, #193488, plus #193406 after cleanup)
- Keep on team-web: 4 issues (#193243, #193221, #193452, #193506)
- Remaining untriaged: 0 issues (1 on hold: #193439)

### Carry-overs

1. **#184281**: 5+ months and no owner. Fourth week flagged.
2. **#189835**: approved, not landed. Fourth week flagged.
3. **#191529**: third week flagged, now 39 days since its only review.
4. **Packages #12357**: approved, not landed. Second week flagged.

### Recommended actions, in order

1. **Land #193407.** It's a one-line fix that removes `IMPELLER_DEBUG` from every wasm release build. It's approved and has a real perf impact. Then drop `team-web` from #193406.
2. **Review #192971 and #193402.** Both are in flutter-zl's area and have no review activity yet.
3. **Land the approved PRs:** #189835, #192964, #193426, #193510, and packages #12357.
4. **Re-route #192618** to `team-accessibility` and post the drafted comment. The previous triager meant to re-route it and never changed the labels. Don't add `aria-label` to buttons by default: DOM text is a deliberate JAWS and crawler fix from flutter/engine#50794. Close as WAI unless the reporter names a mainstream audit tool that flags it.
5. **Confirm #193235 on the issue and suggest P1.** Reproduced on master with the reporter's repro (demo: https://flutter-demo-70-before.web.app). Offer to take the engine side: skip missing child ids with a debug `assert`, and run `_finalizeTree()` in a `finally`.
6. **Assign #193243, #193221, and #193506 to harryterkelsen.**
7. **Assign #193452 to mdebbar** and add `c: flake`.
8. **Update `web-triage.md` queries** to use the `waiting for response` label.
9. **Make a call on #184281 and packages #7950.** Re-flagging them every week isn't getting them resolved.
