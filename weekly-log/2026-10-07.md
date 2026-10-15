# Flutter Web Triage - October 7, 2026

This was a read-only pass. Nothing was changed on GitHub. Everything below is a recommendation for you to apply by hand.

The last pass was only two days ago, on October 5. None of its recommendations have been applied yet, so five of the six issues below carry over. What changed since then: #192618 got a reply from the reporter, #193695 turned out to be assigned to harryterkelsen, and #193847 is new.

---

## Weekly Health Checks

### 1. Unassigned P0/P1: CLEAR

No unassigned P0/P1 issues on `team-web`.

### 2. Open PR count: FLAGGED

**23 untriaged web PRs** on flutter/flutter. The threshold is 15. Up from 22 on October 5. Three left the list: #193402, #193497, #193681. Four came in: #193188, #193850, #193920, #193953.

### 3. P0 issues missing a weekly update: CLEAR

No open P0 on `team-web`. #193727, which was open on October 5, is gone from the list.

### 4. Stale "waiting for customer response": CLEAR

The guide's query still returns nothing because the label is now `waiting for response`. With the new label the query returns #193439, updated 5 days ago, and #95010, updated yesterday. Neither is stale. #192618 lost the label after the reporter replied today. **The `web-triage.md` queries still need updating to `waiting for response`.**

---

## Issue Triage

6 untriaged issues.

### #192618 - Semantics Accessibility issues with aria-label
- **Action**: close as working as intended. Triage as P3 if you prefer to leave it open
- **Priority**: P3
- **Suggested assignee**: flutter-zl
- **Labels to add**: triaged-web, P3, r: invalid or working as intended if closing
- **Labels to remove**: none
- **Title cleanup**: `[web][a11y] Buttons expose their accessible name through DOM text instead of aria-label`
- **Reasoning**: the reporter replied today. VoiceOver and NVDA announce everything correctly, and no assistive tool needs `aria-label`. The only driver is a client checklist that requires `aria-label`, `aria-labelledby`, or `title`. WCAG does not require that: SC 4.1.2 asks for an accessible name, and for `role=button` that name can come from content. Dropdown2 is the third-party `dropdown_button2` package, not a Flutter widget.
- **Root cause**: by design. Buttons use DOM text for their name since flutter/engine#50794, which fixed a JAWS bug and made button text visible to crawlers.
- **Potential solutions**: (1) Close and explain the WCAG point. (2) If the team ever wants it, add an opt-in that also emits `aria-label` when it matches the DOM text. That brings back the Google Translate mismatch you described.
- **Comment**:
  > Thanks for confirming that VoiceOver and NVDA announce these correctly. WCAG 2.2 SC 4.1.2 requires every control to have an accessible name, but it does not say which attribute supplies it. For `role="button"`, the accessible name computation takes the name from the element's content, so `<flt-semantics role="button">Submit All</flt-semantics>` meets the criterion with no `aria-label`. The W3C ACT rule "Button has non-empty accessible name" (97a4e1) passes on this markup, and axe-core reports no violation for it. A rule that demands `aria-label`, `aria-labelledby`, or `title` is stricter than WCAG, so it may help to share this with your client's auditor. Since this behavior is intentional, I'm closing the issue. Feel free to reopen if you find a case where assistive technology announces a name incorrectly.
- **Explanation**: this is a compliance-checklist complaint, not a screen reader bug.

  | Element | Role | Accessible name | `aria-label`? |
  |---|---|---|---|
  | Text fields | textbox | "First name Enter your first name", etc. | yes |
  | `ElevatedButton` "Submit All" | button | "Submit All" | no, name from DOM text |
  | Dropdown2 in a `Semantics` wrapper | button | "Department Select a department sample hint" | no, name from DOM text |

  Buttons, links, and headings render their label as DOM text on purpose. `LabelRepresentation.domText` in `engine/src/flutter/lib/web_ui/lib/src/engine/semantics/label_and_value.dart` is documented as the best option for buttons, links, and headings, and `tappable.dart`, `link.dart`, and `heading.dart` all pick it. yjbanov moved buttons off `aria-label` in flutter/engine#50794, fixing #122607, because JAWS ignores `aria-label` for some roles and crawlers ignore it entirely.

  The client rule is stricter than WCAG. SC 4.1.2 only requires an accessible name, and the Accessible Name Computation spec lets `role=button` take its name from content. A plain `<button>Submit</button>` passes, and axe-core's `button-name` rule passes on text content. Adding `aria-label` on top of the DOM text would make the spoken name go stale under Google Translate, since Translate rewrites DOM text but not attributes, and the same request would then extend to links and headings. There is no public API today to force `aria-label` on a button.
- **Alternate comment**: shorter, and asks which audit tool flags it instead of closing right away.
  > Thanks for clarifying. WCAG 4.1.2 requires that each control has an accessible name, but it does not require that the name come from `aria-label`, `aria-labelledby`, or `title`. Per the W3C Accessible Name Computation spec, the `button` role supports naming from content, so a button whose text is inside the element has a valid accessible name. This is the same as a plain HTML `<button>Submit</button>`, which passes automated checkers like axe-core and Lighthouse. Chrome DevTools shows the computed name "Submit All", and VoiceOver and NVDA announce it correctly, which matches what you observed.
  >
  > Flutter renders button labels as DOM text on purpose, for JAWS compatibility and crawler support. Adding a duplicate `aria-label` would make the name go stale under Google Translate. If your client's audit tool flags this, please share which tool and rule it is, and we can check whether the rule is stricter than WCAG.

### #193439 - [web] Multi-threaded skwasm hangs or crashes in FreeType: SkMutex is a no-op under -sWASM_WORKERS
- **Action**: triage, and handle the duplicate #193695
- **Priority**: P1
- **Suggested assignee**: harryterkelsen
- **Labels to add**: triaged-web, P1, e: wasm, has reproducible steps
- **Labels to remove**: waiting for response
- **Reasoning**: carry-over. The retest on 3.49.0-0.2.pre, which includes #190048 and #191014, still fails 5 of 5 times with isolation and 0 of 5 without, so the strike cache fix does not cover it. **New since October 5**: #193695, "SkSemaphore is a no-op under -sWASM_WORKERS=1, racing FT_Face", is already assigned to harryterkelsen. Keep #193439 because it is older, has the full repro and retest data, and is the issue PR #193801 targets. Close #193695 as `r: duplicate` of #193439 and move harryterkelsen's assignment over. Or do the reverse if he would rather keep his own issue. Either way, link the two.
- **PR review**: #193801 by rambah replaces Skia's `SkSemaphore` contended path with Emscripten Wasm Worker semaphores in a new `skia/SkSemaphore_wasm_workers.cpp`. Workers block, and the main thread spins on a nonblocking acquire. It goes after the actual no-op lock, it adds a skwasm regression test in `concurrent_text_layout_raster_test.dart`, and the author says the new test hangs without the fix. Risks: the main thread spins under contention, so it can delay input, and it changes `skia/BUILD.gn`. No reviewer is requested, so harryterkelsen needs to pick it up.

### #193705 - [web][a11y] Material Slider leaves an empty viewport-sized semantics node with pointer-events: auto
- **Action**: triage
- **Priority**: P2
- **Suggested assignee**: flutter-zl
- **Labels to add**: triaged-web, P2, a: accessibility, engine, c: regression
- **Labels to remove**: none
- **Title cleanup**: `[web][a11y] Slider's OverlayPortal leaves an empty viewport-sized semantics node that intercepts DOM hit tests`
- **Reasoning**: reproduced locally today on stable. Regression from 3.35.4 to 3.47.5, confirmed on master. The pointer-events tiering in #183077 is your code and is the best place to fix it, so it stays on `team-web`, though a framework change also contributes, see root cause. No linked PR.
- **Root cause**: two changes after 3.35.4 combine. Neither is wrong alone.
  1. **Framework creates the node.** `Slider` keeps its value-indicator `OverlayPortal` always shown, `slider.dart:681`. The portal's `_RenderDeferredLayoutBox` is sized to the whole overlay, `overlay.dart:2668`. Since the OverlayPortal semantics refactor, #173005 relanded as #178095 in November 2025, that box sets `traversalChildIdentifier`, `overlay.dart:2702`, which marks it annotated and gives it its own semantics node. The indicator has no semantics, so the node is a viewport-sized leaf with no label, role, or actions.
  2. **Engine makes it catch hits.** #183077, March 2026, gives non-interactive `defer` leaves `pointer-events: auto`, `semantics.dart:2044-2059`. That tier was meant for real overlay content, but it cannot tell an empty node apart.
  3. **Result.** The overlay node comes later in traversal, so it gets a higher z-index and wins every DOM hit test. Real clicks still work because the event bubbles to `flutter-view` and Flutter hit-tests itself.
- **Potential solutions**: (1) **Recommended, engine**: in the tier 3 branch, use `pointer-events: none` for leaves with no label, value, role, actions, or text. Covers any always-mounted empty overlay, such as `Tooltip` or `MenuAnchor`, and keeps #183077 working for overlays with content. (2) Framework: mark `_RenderDeferredLayoutBox` `SemanticsHitTestBehavior.transparent`. Broader, and risky if overlay content relies on inheriting `defer`. (3) Framework: show the Slider portal only while the indicator is visible. Narrowest, but needs rework of the reason given at `slider.dart:680`.
- **Unverified**: that the z-index comes from traversal order, and that #178095 is the first point where the node appears.
- **Demo**: https://flutter-demo-74-before.web.app, source `flutter-demo-apps/working on/issue_193705_before/`. Full steps in `working on/issues logs/issue_193705/test-instructions.md`.
  - **Steps**: open the demo in desktop Chrome and paste this in the DevTools console. Semantics are on at startup, no placeholder click needed. Or run `node repro.mjs https://flutter-demo-74-before.web.app/` from the issue log folder.
    ```js
    {
      const btn = [...document.querySelectorAll('flt-semantics[role=button]')].find(e => /taps/.test(e.textContent));
      const r = btn.getBoundingClientRect();
      const hit = document.elementFromPoint(r.x + r.width / 2, r.y + r.height / 2);
      console.log('button:', btn.id, '| on top:', hit.id, hit.getAttribute('role'), hit.offsetWidth + 'x' + hit.offsetHeight);
    }
    ```
  - **Bug**: `on top:` is a different, empty node with role `null` and the size of the viewport, and Playwright `click()` times out with "intercepts pointer events". A real mouse click still works. Fixed: `on top:` matches `button:` with role `button`, and `click() OK`.
  - **Reproduced today** on the before demo, desktop Chrome: `button: flt-semantic-node-4 | on top: flt-semantic-node-7 null 1083x960`. The empty node covers the whole 1083x960 window and sits above the button. Same node id as the reporter's `flt-semantic-node-7`, so the issue reproduces on current stable.
  - **Control**: rebuild with `showSlider = false` and the empty leaf goes away.

### #193639 - [web][Android] View keeps a keyboard-reduced height and reports negative viewInsets after focus moves from an iframe input to a TextField
- **Action**: triage
- **Priority**: P2
- **Suggested assignee**: flutter-zl
- **Labels to add**: triaged-web, P2, has reproducible steps, found in release: 3.47, platform-android, browser: chrome
- **Labels to remove**: none
- **Reasoning**: carry-over, no change. Reproduced on stable and master, and the reporter offered to send a PR. No PR has been opened yet. The code is the size preservation in `window.dart`. Your #193744 fixes a related iOS bug, #193743, in the neighboring `full_page_dimensions_provider.dart`, and does not touch this path. Both bugs come from trusting a stale height when the viewport grows.
- **Root cause**: `_handleBrowserResize` keeps the old `_physicalSize` while a Flutter text field is being edited on mobile. When an iframe input already has the keyboard open, Flutter is not editing, so `_physicalSize` takes the keyboard-reduced height. The next `TextField` focus keeps that wrong height, and every inset after it comes out negative.
- **Potential solutions**: (1) Reporter's fix: while the size is being kept, a viewport taller than `_physicalSize` updates `_physicalSize`. (2) Snapshot the full height before the keyboard opens. (3) Clamp the inset at zero, keeping the assert.
- **Comment**: use the draft from the October 5 log. If its last sentence mentions #193744, reword it so it does not suggest #193744 fixes this issue. #193744 only changes the iOS sizing path.
- **Demo**: https://flutter-demo-73-before.web.app, source `flutter-demo-apps/working on/issue_193639_before/`. Full steps in `working on/issues logs/issue_193639/test-instructions.md`.
  - **Steps**: on Chrome for Android, tap **iframe input**, then **Flutter TextField**, then close the keyboard with Back.
  - **Bug**: the view stays at the keyboard-open height, the red page background shows below the green bar, and the readout shows a negative `insets.bottom`, about -374. Fixed: green bar at the bottom and `insets.bottom=0`.
  - **Controls**: TextField only, or iframe input only, followed by Back, should not trigger it.

### #183265 - FlutterLoader could not find a build compatible with configuration and environment (wasm + canvaskit)
- **Action**: triage
- **Priority**: P3
- **Suggested assignee**: mdebbar
- **Labels to add**: triaged-web, P3, d: docs/
- **Labels to remove**: none
- **Reasoning**: carry-over, no change. The runtime works as designed and the docs set the wrong expectation: wasm builds only ship skwasm.
- **Root cause**: a `--wasm` build lists only skwasm in `buildConfig`, so a request for `renderer: "canvaskit"` matches no build and the loader throws. `flutter build web --wasm` also emits a JS build, which is why the built app falls back without an error.
- **Potential solutions**: (1) Fix the renderer docs. (2) Make the loader error list the renderers the build supports. (3) Make `flutter run --wasm` warn when the config asks for a renderer the build cannot provide.

### #193847 - [web] After the browser drops a composition without compositionend, every edit keeps reporting the old composing range
- **Action**: triage, keep the current routing
- **Priority**: leave to team-text-input. P2 if asked
- **Suggested assignee**: none, it is owned by another team
- **Labels to add**: triaged-web
- **Labels to remove**: none
- **Reasoning**: new. mbcorona already routed it to `team-text-input` with `fyi-web`, so per the guide only `triaged-web` is needed. The reporter verified it with CDP and pointed to the exact lines, and it reproduced by hand today with macOS Pinyin. It may also be the cause of #138136. No linked PR.
- **Root cause**: `CompositionAwareMixin` in `lib/web_ui/lib/src/engine/text_editing/composition_aware_mixin.dart` clears `composingText` and `composingBase` only on `compositionstart` and `compositionend`. When the framework rewrites composed text, Chrome drops the composition without firing `compositionend`, so `determineCompositionState` keeps adding the stale range to every later edit.
- **Potential solutions**: (1) In `DefaultTextEditingStrategy.handleChange`, clear the composition state when an `input` event has `isComposing == false`. (2) Also clear it in `setEditingState` when it writes a changed value during a composition. (3) Check whether the same fix covers #183078, which has the same symptom after a window blur.
- **Demo**: https://flutter-demo-75-before.web.app, source `flutter-demo-apps/working on/issue_193847_before/`. Full steps in `working on/issues logs/issue_193847/test-instructions.md`.
  - **Automated**: `node cdp_repro.mjs https://flutter-demo-75-before.web.app/` from the issue log folder. It sends `imeSetComposition('a')`, then `insertText('b')` and `insertText('c')`. Bug: no `compositionend`, and `VALUE` lines keep flipping back to `composing: TextRange(start: 0, end: 1)` after plain typing. Fixed: the range stays `-1, -1` after the rewrite.
  - **Manual**: open DevTools Console, type `n` with macOS Chinese Pinyin so the formatter uppercases the preedit, switch to English, type `b` and `c`. Same expected and actual as above.
  - **Reproduced today by hand** with macOS Chinese Pinyin on the before demo, desktop Chrome. The reporter had only shown it through CDP, so this confirms it happens with a real input method. The text ends up correct, but later edits keep reporting the stale composing range on the first letter. Worth adding to the issue as a confirmation comment for team-text-input.

---

## PR Triage

### flutter/flutter: 23 untriaged, above the 15 threshold

Add `triaged-web` to all of these. Suggested extra labels and flags:

| PR | Author | Review state | Updated | Add labels | Flag |
|---|---|---|---|---|---|
| #184281 | koji-1009 | REVIEW_REQUIRED | 09-24 | f: gestures | **Stale. Sixth week flagged.** Review it once #193744 lands. |
| #185622 | mdebbar | CHANGES_REQUESTED | 08-18 | | **Stale, 50 days.** Ask mdebbar whether he still plans to land it. |
| #188628 | MarlonJD | CHANGES_REQUESTED | 10-03 | e: web_skwasm | Needs harryterkelsen. |
| #189953 | gaaclarke | CHANGES_REQUESTED | 09-17 | e: impeller, a: tests | Waiting on the author. |
| #190126 | sero583 | CHANGES_REQUESTED | 10-03 | | The final revision promised for last weekend has not arrived. Fixes your #174773. |
| #190959 | gmackall | REVIEW_REQUIRED | 09-23 | | Mostly Android. |
| #191107 | schultek | REVIEW_REQUIRED | 09-10 | | **27 days idle.** Needs mdebbar or eyebrowsoffire. |
| #191529 | rkishan516 | REVIEW_REQUIRED | 09-09 | a: accessibility | **Stale. Fifth week flagged.** |
| #192224 | ValentinVignal | REVIEW_REQUIRED | 10-06 | a: typography | Review requested from LongCatIsLooong. |
| #192260 | sm-sayedi | REVIEW_REQUIRED | 10-05 | | Cross-platform. Web only needs a check that nothing regresses. |
| #192971 | kevmoo | REVIEW_REQUIRED | 10-05 | a: accessibility | Needs flutter-zl. **Third week.** |
| #193149 | holzgeist | REVIEW_REQUIRED | 10-06 | c: new feature | Document PiP. No reviewer. |
| #193188 | mdebbar | APPROVED | 10-07 | | New to the list. Typo fixes, ready to land. |
| #193269 | simonpham | REVIEW_REQUIRED | 10-06 | a: text input | Needs your re-review. Clears #192544. |
| #193275 | Hellomik2002 | CHANGES_REQUESTED | 09-29 | | Owned by team-engine. Add `triaged-web` only. |
| #193293 | kevmoo | REVIEW_REQUIRED | 10-06 | e: web_skwasm | Needs harryterkelsen. Third week. |
| #193362 | diegolopezrm | REVIEW_REQUIRED | 09-25 | f: gestures | No reviewer. Third week. |
| #193385 | rkishan516 | REVIEW_REQUIRED | 09-28 | | No reviewer. |
| #193744 | flutter-zl | APPROVED | 10-07 | | Yours. Ready to land. |
| #193801 | rambah | REVIEW_REQUIRED | 10-05 | e: web_skwasm, e: wasm | Fixes #193439. Needs harryterkelsen. |
| #193850 | dbebawy | REVIEW_REQUIRED | 10-05 | e: web_skwasm, a: typography | New. Remove `a: text input`, which is the wrong label. Soft hyphens on Skwasm, part of #193506. Needs harryterkelsen. |
| #193920 | mdebbar | REVIEW_REQUIRED | 10-06 | a: tests, c: flake | New. Logs which golden step stalls. No reviewer. |
| #193953 | tomayac | REVIEW_REQUIRED | 10-07 | e: wasm | New. Renames `requestFileHandle` to `getFileHandle` in `instantiate_wasm.js`. Small. Needs mdebbar or harryterkelsen. |

**Stale, 30+ days with no review:** #184281, #185622, #191529. #191107 crosses the line this weekend.

**Approved and ready to land:** #193188, #193744.

### flutter/packages: 4 untriaged

- **#7950**: [camera_web] camera stream. Open since October 2024. **Seventh week flagged.** Close it or assign it.
- **#11872**: [google_maps_flutter] `onPointOfInterestTap`. No action.
- **#12186**: [google_maps_flutter_web] my location. Needs a re-review.
- **#12357**: [google_maps_flutter] background color for unloaded tiles. **Approved, not landed. Fourth week flagged.**

---

## Triage Summary - October 7, 2026

- Triaged: 5 issues
- Close: 1 issue, #192618, plus #193695 as a duplicate
- Request info: 0 issues
- Re-route: 0 issues. #193847 keeps its current `team-text-input` routing
- Remaining untriaged: 0 issues

### Recommended actions, in order

1. **Promote #193439 to P1 and assign harryterkelsen.** Then close #193695 as its duplicate, and point him at #193801.
2. **Close #192618** with the drafted comment.
3. **Land #193744.** It is approved, fixes #193743, and unblocks #184281.
4. **Triage #193705 as a P2 regression and take it.**
5. **Add `triaged-web` to #193847.**
6. **Re-review #193269 and review #192971.** Both are in your area.
7. **Bulk-apply `triaged-web` to the 23 PRs.** That clears the PR-count flag.
8. **Update the `web-triage.md` queries** to use `waiting for response`.
