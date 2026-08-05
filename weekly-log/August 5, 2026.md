# Flutter Web Triage, August 5, 2026

## Weekly Health Checks

- **Unassigned P0/P1 issues**: One. [#190367](https://github.com/flutter/flutter/issues/190367) is a P1 test failure with no owner. It is not a zombie, it is a live tree problem. Recommend assigning mdebbar and landing the skip gaaclarke proposed.
- **P0 issues missing a weekly update**: None. Both open P0s are flaky-test trackers and both are assigned and active. [#190199](https://github.com/flutter/flutter/issues/190199) has a July 30 analysis from Piinks plus a fresh flakiness report, and [#190606](https://github.com/flutter/flutter/issues/190606) was filed today.
- **Stale waiting-for-customer-response issues**: None. The query returns an empty list.
- **Untriaged web PRs**: 23 in `flutter/flutter`, above the 15-PR alert threshold, plus 2 awaiting triage in `flutter/packages`. Down from 26 last week.
- **Stale PRs**: `flutter/flutter` [#183835](https://github.com/flutter/flutter/pull/183835), [#185080](https://github.com/flutter/flutter/pull/185080), [#187181](https://github.com/flutter/flutter/pull/187181), [#188010](https://github.com/flutter/flutter/pull/188010), and [#188032](https://github.com/flutter/flutter/pull/188032) have no human review activity in the last 30 days. [#183835](https://github.com/flutter/flutter/pull/183835) is the worst case: the author has now pinged for a second approval three times, most recently today, and no reviewer has acted since May 18. `flutter/packages` [#7950](https://github.com/flutter/packages/pull/7950) has had no substantive review since May 20.
- **Approaching stale**: [#186540](https://github.com/flutter/flutter/pull/186540) at 27 days and [#189086](https://github.com/flutter/flutter/pull/189086) at 29 days with no human review at all.

## Issue Triage

### [#190192](https://github.com/flutter/flutter/issues/190192) - Web: every real WebGL context loss permanently kills the app on 3.44 stable
- **Action**: triage
- **Priority**: P1
- **Suggested assignee**: harryterkelsen
- **Labels to add**: triaged-web, P1, c: crash, has reproducible steps, found in release: 3.44
- **Labels to remove**: none
- **Title cleanup**: [web] Real WebGL context loss permanently kills the app on 3.44 stable
- **Comment**: The fix is already on master via [#185116](https://github.com/flutter/flutter/pull/185116) and re-landed as `855588f86`. No cherry-pick PR exists against the 3.44 release branch yet, so the next step is to open that PR and label it `cp: review`. The cherry-pick labels belong on the PR, not on this issue.
- **Reasoning**: A production-verified, deterministic failure on the current stable branch whose fix is a one-line change already validated on master, so the cherry-pick is low risk and high value.
- **Root cause**: `_handledContextLostEvent` is declared `late` and is only ever assigned by the test-only `triggerContextLoss` path, so the real `webglcontextlost` listener reads an uninitialized late field and throws before `acquireCanvas` and `recreateContextForCanvas` can run. Release tree shaking removes the only writer, which folds the getter to an unconditional throw, so every real context loss is fatal and the surface is never recreated.
- **Potential solutions**:
  1. Cherry-pick [#185116](https://github.com/flutter/flutter/pull/185116) to the 3.44 release branch as requested. The change drops `late` from the field in `lib/web_ui/lib/src/engine/canvaskit/surface.dart` and `lib/web_ui/lib/src/engine/skwasm/skwasm_impl/surface.dart`, so the cherry-pick is about as small as one can get.
  2. If the release team prefers a patch authored against the branch rather than a cherry-pick, initializing the field to null at its declaration is equivalent and equally minimal.
  3. On master, add a regression test that dispatches a real `webglcontextlost` event on the surface canvas instead of calling `triggerContextLoss`, so the test-only writer stops masking this class of defect.

### [#190288](https://github.com/flutter/flutter/issues/190288) - Proposal: Support --web-define-from-file
- **Action**: re-route
- **Priority**: P3
- **Suggested assignee**:
- **Labels to add**: triaged-web, fyi-web, P3, team-tool
- **Labels to remove**: team-web
- **Title cleanup**: Proposal: Support `--web-define-from-file` for `flutter build web`
- **Comment**: `--web-define` and `--dart-define-from-file` are both Flutter tool flags, and the substitution happens during the tool's web build, so this belongs to team-tool. Keeping `fyi-web` because the template substitution touches `web/index.html` and `web/flutter_bootstrap.js`.
- **Reasoning**: A well-argued proposal with a concrete use case and a stated workaround, but it is a tool flag addition with a single reporter, so P3 under the feature-request rule.
- **Potential solutions**:
  1. Add `--web-define-from-file` to the web build options, reusing the existing `--dart-define-from-file` parser for .env and .json, and feed the resulting map into the same substitution path `--web-define` already uses. This is what the reporter asked for and it keeps the two flags symmetrical.
  2. Skip the new flag and let `--dart-define-from-file` values participate in `{{NAME}}` substitution, but only for keys that already appear as placeholders in `web/index.html` or `web/flutter_bootstrap.js`. Smaller surface, though it changes existing behavior and needs care so unrelated keys in a shared secrets file never reach a public artifact.
  3. Decline the flag and document the shell workaround plus its `launch.json` limitation, which is the honest outcome if team-tool does not want to grow the flag set.

### [#190282](https://github.com/flutter/flutter/issues/190282) - Flutter web reports pressure as 0.5 on mouse
- **Action**: triage
- **Priority**: P3
- **Suggested assignee**: mdebbar
- **Labels to add**: triaged-web, P3, engine, c: parity
- **Labels to remove**: none
- **Title cleanup**: [web] Mouse pointer events report pressure 0.5 instead of 0.0
- **Comment**: The reporter has volunteered to fix this and also asked that their related stylus issue [#187532](https://github.com/flutter/flutter/issues/187532) be looked at first, which is already assigned to mdebbar. Worth confirming the intended cross-platform contract for mouse pressure before any code lands, since the same normalization path also carries stylus and touch values.
- **Reasoning**: The browser value matches the Pointer Events spec, so this is a cross-platform parity gap rather than a defect, and there is a trivial app-side check on `PointerDeviceKind`.
- **Root cause**: `pointer_binding.dart` forwards `event.pressure` unchanged and only maps null to 0.0. Browsers report 0.5 for a mouse while a button is held, per the Pointer Events spec, whereas other platforms report 0.0 for `PointerDeviceKind.mouse`, so web is the only platform that reports a non-zero value.
- **Potential solutions**:
  1. In `pointer_binding.dart`, force pressure to 0.0 when the DOM `pointerType` is mouse, leaving stylus and touch untouched. Smallest change and it matches every other platform.
  2. Normalize in the framework instead, wherever `PointerDeviceKind.mouse` pressure is produced, so one rule covers all platforms rather than web patching itself into line. Larger blast radius and it needs a look at whether any embedder relies on the current value.
  3. Close as working as intended and document that web forwards the spec-defined value, since apps can branch on `kind` today. Cheapest, but it leaves the cross-platform inconsistency in place.

### [#190350](https://github.com/flutter/flutter/issues/190350) - [web] Android WebView textZoom makes lineHeightScaleFactorOverride report the ~624.9375 sentinel
- **Action**: triage
- **Priority**: P1
- **Suggested assignee**: harryterkelsen
- **Labels to add**: triaged-web, P1, c: regression, browser: chrome-android, found in release: 3.44
- **Labels to remove**: none
- **Title cleanup**: none
- **Comment**: none
- **Reasoning**: Every plain `Text` is unrenderable from first paint in any Android WebView whose system font size is not exactly 100 percent, there is no practical app-side workaround, and the earlier fix [#178862](https://github.com/flutter/flutter/pull/178862) for [#178856](https://github.com/flutter/flutter/issues/178856) did not close this path, so it is a regression in a widely-reachable configuration.
- **Root cause**: Inferred from the reported data rather than confirmed in source. The engine appears to detect "no user line-height preference" by comparing a measured probe ratio against a fixed sentinel. Android WebView's `textZoom` scales the probe measurement slightly, so the equality check misses and the sentinel value near 624.9375 is surfaced as a real `lineHeightScaleFactorOverride`. `Text` then multiplies its line height by that value and lays glyphs out thousands of pixels below the viewport, while `RichText` is unaffected because it never reads the override. The near-constant value across every zoom level is what points at the sentinel rather than at a genuine measurement.
- **Potential solutions**:
  1. Stop comparing the probe result against a sentinel by exact equality and use a plausibility band instead, treating any ratio outside a sane range as "no override". The dartdoc describes a multiple of the font size, so anything above roughly 4 is not a real preference.
  2. Cancel the zoom out of the measurement before the comparison, by dividing the probe result by the ratio the same probe reports for a known baseline, so `textZoom` no longer perturbs the equality check.
  3. Add a framework-side guard so `Text` ignores `lineHeightScaleFactorOverride` values outside a sane range. Worth doing regardless as a floor against total text invisibility, but it treats the symptom and is not a substitute for the engine fix.

### [#190367](https://github.com/flutter/flutter/issues/190367) - `Linux linux_web_engine_tests` fails in `platform_dispatcher/view_focus_binding_test.dart`
- **Action**: triage
- **Priority**: P1
- **Suggested assignee**: mdebbar
- **Labels to add**: triaged-web, engine, platform-web, a: tests, browser: safari-macos
- **Labels to remove**: none
- **Title cleanup**: none
- **Comment**: Agreed with the proposal to skip this single test to unblock the suite, with the skip referencing this issue so it is not forgotten. The failing target is the Safari dart2js CanvasKit engine suite, and the suspected trigger is an out-of-band browser update on the bot, so the `browser: safari-macos` label is a signal rather than a confirmed scope.
- **Reasoning**: This blocks a whole engine test suite and has no owner, which is the one unassigned P1 in the queue this week.
- **Root cause**: The test asserts exactly one `ViewFocusEvent` but the run emits a `focused` event followed by an `unfocused` event. A browser change in blur and focus ordering most likely makes `ViewFocusBinding` emit the extra unfocused event when focus is changed in the middle of a blur, which is precisely the case the test names.
- **Potential solutions**:
  1. Skip this single test with a reference back to this issue, which unblocks the suite today and is what gaaclarke proposed. Do this first regardless of which of the next two turns out to be right.
  2. If the extra `unfocused` event is correct under the updated browser, update the expectation to accept the focused-then-unfocused pair and note why the sequence changed.
  3. If the extra event is wrong, coalesce redundant transitions in `ViewFocusBinding` so a focus change during a blur produces one event rather than a spurious unfocus. This is the real fix if the binding is now double-reporting to the framework in production too.

### [#190355](https://github.com/flutter/flutter/issues/190355) - Packages roller blocked on JS interop errors
- **Action**: triage
- **Priority**: P1
- **Suggested assignee**: mdebbar
- **Labels to add**: triaged-web, P1, platform-web, p: camera, c: regression
- **Labels to remove**: none
- **Title cleanup**: none
- **Comment**: none
- **Reasoning**: The flutter/flutter to flutter/packages roller has been blocked since July 31, stuartmorgan-g has already pinged this week's gardener, and nobody on web has picked it up, so it needs a named owner now.
- **Root cause**: Inferred from the error text, not verified against the file. The Dart roll in [#190158](https://github.com/flutter/flutter/pull/190158) promoted "the `@JS` annotation on an extension type constructor has no effect" from a warning to a compile error, and `packages/camera/camera_web/lib/src/pkg_web_tweaks.dart` carries `@JS` on two `external factory` object-literal constructors. Dropping the annotation from those two constructors, while leaving the extension types themselves annotated, is the smallest change and should compile on both stable and master, which is the compatibility constraint stuartmorgan-g called out.
- **Potential solutions**:
  1. Remove `@JS` from the two `external factory` constructors in `packages/camera/camera_web/lib/src/pkg_web_tweaks.dart` and keep it on the extension types. The annotation is what the compiler now calls out as having no effect, so removing it should satisfy both stable and master.
  2. If those constructors must keep an explicit interop shape, replace them with `JSObject` literal construction helpers that build the same objects without an annotated constructor.
  3. To unblock the roller while either fix is reviewed, pin or temporarily revert the offending roll in flutter/packages. This buys time but leaves the incompatibility to be paid down later, so it is a fallback rather than a plan.

### [#190500](https://github.com/flutter/flutter/issues/190500) - [web] iOS 26 draws the file picker menu's glass backdrop from <flutter-view>
- **Action**: triage
- **Priority**: P2
- **Suggested assignee**: mdebbar
- **Labels to add**: triaged-web, P2
- **Labels to remove**: none
- **Title cleanup**: none
- **Comment**: Linking [#110638](https://github.com/flutter/flutter/issues/110638), which reports the same picker being anchored to the wrong element on iOS web and is still open at P2. Both look like the same wrong anchor, so whoever takes one should take both. Keeping them separate for now because the iOS 26 symptom and the fix verification differ.
- **Reasoning**: Reproduced on a bare app with a video, a plugin-free sample, and a narrowed cause, but the impact is a one-second visual artifact with a working platform-view workaround, so it does not clear the P1 bar.
- **Root cause**: iOS 26 builds the picker menu's glass backdrop from the DOM element that received the tap. When Flutter handles the tap and calls `click()` on a synthesized input from Dart, the element holding focus is `<flutter-view>`, which spans the viewport, so the backdrop is rendered at viewport size over the page. The reporter confirmed this by patching `HTMLInputElement.prototype.click` and observing `document.activeElement` as `FLUTTER-VIEW` with a full-viewport rect. Making the real input the click target, sized and positioned at the tap, is what the working workaround demonstrates.
- **Potential solutions**:
  1. Let the real `<input type="file">` take the gesture: mount it in the DOM at the tap location, sized to the tapped widget, so iOS builds its backdrop from an element of the right size. This is what the reporter's OVERLAY case already proves works, and it is the same anchoring problem [#110638](https://github.com/flutter/flutter/issues/110638) describes.
  2. Cheaper experiment first: focus the input element immediately before calling `click()`, so `document.activeElement` is the input rather than `<flutter-view>` when the menu is presented. Unverified, but it is a few lines and would tell us whether iOS reads focus or the true event target.
  3. Fix `image_picker_for_web` first, since that is where most users meet this, while the engine-level anchoring is settled together with [#110638](https://github.com/flutter/flutter/issues/110638).

### [#190510](https://github.com/flutter/flutter/issues/190510) - [web] App fails to bootstrap when navigator.languages contains a tag Intl.Locale rejects
- **Action**: triage
- **Priority**: P1
- **Suggested assignee**:
- **Labels to add**: triaged-web, P1, engine
- **Labels to remove**: none
- **Title cleanup**: none
- **Comment**: none
- **Reasoning**: This is a total bootstrap failure with a permanently blank page and no interception point, since it throws before `main()` runs, and the trigger is the default Chromium locale on Linux with `LANG` unset, which is what standard test container images ship. No assignee suggested because locale parsing does not map to an owner in the routing table, but this is a small, self-contained web engine fix with a reporter-supplied patch and a reproduction, so it suits a new contributor well.
- **Root cause**: `EnginePlatformDispatcher.parseBrowserLanguages` in `lib/web_ui/lib/src/engine/platform_dispatcher.dart` constructs a `DomLocale` for every entry of `navigator.languages` with no validation and no guard, and it is called from a field initializer. A tag that `Intl.Locale` rejects, such as `en-US@posix`, therefore throws while the dispatcher itself is being constructed, so the engine never mounts a `flutter-view`. Skipping rejected tags and falling back to the default locale only when none survive preserves the existing non-empty contract.
- **Potential solutions**:
  1. Wrap each `DomLocale` construction in a try/catch inside `parseBrowserLanguages`, skip the tags the runtime rejects, and fall back to the default locale only if none survive. This is the reporter's patch and it keeps the documented non-empty contract.
  2. Validate the tag before constructing it, rejecting shapes such as an `@` suffix or an empty subtag. Avoids exception-driven control flow, but it hardcodes knowledge of what `Intl.Locale` refuses and will miss forms nobody predicted, so option 1 is the safer default.
  3. Independently of the locale fix, move `parseBrowserLanguages` out of the field initializer so that any throw during locale parsing cannot stop the engine from mounting a view. That turns this whole class of bug from a blank page into a degraded start.

### [#190512](https://github.com/flutter/flutter/issues/190512) - [web] DefaultTextEditingStrategy.moveFocusToActiveDomElement still throws null-check on 3.44.2
- **Action**: triage
- **Priority**: P1
- **Suggested assignee**: mdebbar
- **Labels to add**: triaged-web, P1, engine, has reproducible steps
- **Labels to remove**: none
- **Title cleanup**: [web] Null-check crash in `DefaultTextEditingStrategy.moveFocusToActiveDomElement`
- **Comment**: Keeping this open as the live tracker for the crash, since [#187461](https://github.com/flutter/flutter/issues/187461) was auto-closed for lack of a repro and is locked. The second report on this thread adds a simpler trigger worth covering in the regression test: a `Form` whose validation surfaces an error, after which fields silently stop accepting input, and a variant with no `Form` at all where clicking into the middle of existing text stops editing until a blur and refocus cycle.
- **Reasoning**: A source-mapped production trace pinpoints an unguarded read that [#180795](https://github.com/flutter/flutter/pull/180795) did not cover, a second user independently confirms it on 3.44.8 with a plain single-line `TextField`, and the crash is silently swallowed in release, so it presents as dropped characters and dead text fields rather than as a visible error. Keeping `team-web` rather than re-routing to text input, because the defect and the fix are entirely inside the web engine's text editing strategy.
- **Root cause**: `moveFocusToActiveDomElement` reads `activeDomElement`, whose `return domElement!` is protected only by an assert that release and profile builds strip. When a scheduled `setEditableSizeAndTransform` handler runs after the editing element has been torn down, the null check throws. [#180795](https://github.com/flutter/flutter/pull/180795) guarded only the `focusedFormElement` call inside `placeElement` and the `GloballyPositionedTextEditingStrategy` override, leaving the base-class path on the very next line unguarded.
- **Potential solutions**:
  1. Change `moveFocusToActiveDomElement` to `domElement?.focusWithoutScroll()`, which is the reporter's one-line fix and mirrors what [#180795](https://github.com/flutter/flutter/pull/180795) already did on the adjacent line.
  2. Guard one level up by returning early from `updateElementPlacement` and `placeElement` when `domElement` is null. Slightly larger, but it covers the whole family of stale-handler paths instead of the one line we happen to have a trace for, which is how this bug came back twice.
  3. Fix the lifecycle rather than the symptom: drop or cancel pending `setEditableSizeAndTransform` messages once a strategy is disabled, so no handler ever runs against a torn-down element. Most correct and the largest change, and it would also explain the dropped characters the second reporter describes.

## Pull Request Triage

### flutter/flutter [#183835](https://github.com/flutter/flutter/pull/183835) - Use dart2wasm app.support.js expression with `js-string` builtin requirement
- **Scope**: Clear WebAssembly loader compatibility change. Links a Dart SDK change but no Flutter issue.
- **Owner/reviewer**: mdebbar was pinged directly today. eyebrowsoffire was pinged on July 28. Both prior approvals are dismissed.
- **Staleness**: Stale, and getting worse. The author has rebased and asked for a second approval three times with no reviewer response since May 18.
- **Labels to add**: triaged-web, e: wasm
- **Blocker**: Needs a fresh approval. Dashboard Checks require action.

### flutter/flutter [#184961](https://github.com/flutter/flutter/pull/184961) - Fix rendering custom element size
- **Scope**: Understandable tactical engine fix. References closed issue [#182940](https://github.com/flutter/flutter/issues/182940).
- **Owner/reviewer**: harryterkelsen approved on July 15. mdebbar is still the requested reviewer and should route the failing internal test.
- **Staleness**: Not stale.
- **Labels to add**: triaged-web, c: crash, c: rendering
- **Blocker**: Google testing fails.

### flutter/flutter [#185080](https://github.com/flutter/flutter/pull/185080) - Fix: activateSystemCursor should set cursor on all views
- **Scope**: Clear multi-view cursor fix. Fixes [#140226](https://github.com/flutter/flutter/issues/140226).
- **Owner/reviewer**: Approved by mdebbar and yjbanov. mdebbar is the right owner for the internal-test follow-up.
- **Staleness**: Stale. Last human activity was June 24 and the branch state is now unknown.
- **Labels to add**: triaged-web, a: mouse
- **Blocker**: Google testing fails. Approvals are in place, so this only needs the internal breakage resolved.

### flutter/flutter [#185152](https://github.com/flutter/flutter/pull/185152) - Support soft hyphen (U+00AD) rendering with a Hyphens API
- **Scope**: Clear but broad engine and framework API change. Fixes [#18443](https://github.com/flutter/flutter/issues/18443) and references [#185154](https://github.com/flutter/flutter/issues/185154).
- **Owner/reviewer**: justinmc has requested changes. LongCatIsLooong and stuartmorgan-g are engaged on the API design.
- **Staleness**: Not stale. Discussion ran through July 22.
- **Labels to add**: triaged-web, a: typography
- **Labels to remove**: a: text input, f: material design
- **Blocker**: Requested changes are open and Check Code Freeze fails.

### flutter/flutter [#185622](https://github.com/flutter/flutter/pull/185622) - [web] Fix multi-view sizing race condition
- **Scope**: Clear engine synchronization fix. References regression [#185034](https://github.com/flutter/flutter/issues/185034).
- **Owner/reviewer**: yjbanov commented on July 15 and should resolve the batching question. harryterkelsen is a strong second reviewer.
- **Staleness**: Not stale.
- **Labels to add**: triaged-web, e: web_skwasm, c: regression, c: rendering
- **Blocker**: The architectural alternative is unresolved and Google testing fails.

### flutter/flutter [#186540](https://github.com/flutter/flutter/pull/186540) - Add visible focus ring for keyboard-focused semantics nodes
- **Scope**: Clear accessibility goal. Fixes [#186044](https://github.com/flutter/flutter/issues/186044), but the implementation does not yet solve the reported case.
- **Owner/reviewer**: flutter-zl, chunhtai, and mdebbar are all requested. mdebbar should settle the engine-versus-framework question.
- **Staleness**: Approaching. 27 days since the last comment and the only substantive review was May 20.
- **Labels to add**: triaged-web, team-accessibility, fyi-web
- **Blocker**: Duplicate rings appeared on existing widgets while custom semantics still lacked one, so the approach needs a redesign before more review time goes in.

### flutter/flutter [#187181](https://github.com/flutter/flutter/pull/187181) - Drawing decorations directly on the target Canvas
- **Scope**: Partially clear. Fixes [#188322](https://github.com/flutter/flutter/issues/188322), but the body does not state expected behavior or risk.
- **Owner/reviewer**: Rusino is the author. mdebbar reviewed on June 10 and is the right reviewer to re-engage.
- **Staleness**: Stale. 56 days since the last human review.
- **Labels to add**: triaged-web, a: typography
- **Blocker**: Google testing, `Linux linux_fuchsia`, and `Linux linux_web_engine` all fail, and the branch state is unknown.

### flutter/flutter [#188010](https://github.com/flutter/flutter/pull/188010) - refactor(web): use singular crossOriginStorage.requestFileHandle()
- **Scope**: Clear web API migration with compatibility detail. Links an external specification issue but no Flutter issue.
- **Owner/reviewer**: kevmoo approved on July 6. yjbanov is the outstanding requested reviewer.
- **Staleness**: Stale as of this week, at exactly 30 days since the last review.
- **Labels to add**: triaged-web, e: wasm
- **Blocker**: Google testing fails and a second current approval is needed.

### flutter/flutter [#188032](https://github.com/flutter/flutter/pull/188032) - Fixed couple of bugs
- **Scope**: Unclear. The title and body name RTL APIs with no motivating issue, expected behavior, or validation.
- **Owner/reviewer**: Rusino is the author. mdebbar is the right reviewer for web paragraph and RTL selection.
- **Staleness**: Stale. 51 days old with no human review at all.
- **Labels to add**: triaged-web, a: typography
- **Blocker**: `Mac mac_unopt` fails, the branch state is unknown, and the PR still needs a real description before review is worth requesting.

### flutter/flutter [#189086](https://github.com/flutter/flutter/pull/189086) - refactor(web): SkwasmPath and SkwasmPathBuilder separation
- **Scope**: Clear Skwasm path refactor. Closes [#184840](https://github.com/flutter/flutter/issues/184840).
- **Owner/reviewer**: mdebbar is requested. harryterkelsen is the strongest Skwasm reviewer and should be added.
- **Staleness**: Approaching. 29 days with no human review.
- **Labels to add**: triaged-web, e: web_skwasm
- **Blocker**: Awaiting a first human review. No failing checks.

### flutter/flutter [#189500](https://github.com/flutter/flutter/pull/189500) - Wait for web rendering before first-frame event
- **Scope**: Clear first-frame rendering fix. Fixes [#189499](https://github.com/flutter/flutter/issues/189499).
- **Owner/reviewer**: harryterkelsen and flutter-zl both approved on July 31, after two rounds of requested changes.
- **Staleness**: Not stale. This moved from changes-requested to approved this week.
- **Labels to add**: triaged-web, c: rendering
- **Blocker**: Google testing fails. That is the only thing left before it can land.

### flutter/flutter [#189835](https://github.com/flutter/flutter/pull/189835) - [web] Stop percent-decoding URLs written to browser history
- **Scope**: Clear routing fix. Fixes [#180373](https://github.com/flutter/flutter/issues/180373), [#171757](https://github.com/flutter/flutter/issues/171757), [#147857](https://github.com/flutter/flutter/issues/147857), and [#155992](https://github.com/flutter/flutter/issues/155992).
- **Owner/reviewer**: No reviewer is requested. mdebbar should be added for web routing.
- **Staleness**: Not stale by age at 14 days, but no human review has occurred and no reviewer is assigned.
- **Labels to add**: triaged-web, f: routes
- **Blocker**: Needs a reviewer requested. Four linked issues make this a high-value review.

### flutter/flutter [#189858](https://github.com/flutter/flutter/pull/189858) - Fixing edge cases for wrapping text with newlines
- **Scope**: Understandable but underexplained. No linked issue and no acceptance criteria.
- **Owner/reviewer**: Rusino is the author and mdebbar is requested.
- **Staleness**: Not stale by age, but no human review has occurred.
- **Labels to add**: triaged-web, c: rendering
- **Blocker**: `Tree_analyze`, `Mac_arm64_verify_binaries`, and Google testing all fail, and Dashboard Checks require action. CI needs to be green before review time is worth spending.

### flutter/flutter [#189945](https://github.com/flutter/flutter/pull/189945) - Adds error about wimp_heavy not being implemented
- **Scope**: Clear diagnostic for unsupported `wimp_heavy`. References [#187212](https://github.com/flutter/flutter/issues/187212).
- **Owner/reviewer**: walley892 approved on July 23.
- **Staleness**: Not stale.
- **Labels to add**: triaged-web, e: web_skwasm
- **Blocker**: None evident. This one is ready and should just land.

### flutter/flutter [#189953](https://github.com/flutter/flutter/pull/189953) - [wimp] Adds wimp-heavy variant for non-Chrome browsers
- **Scope**: Understandable. Review found missing fallback behavior and browser coverage. References [#187212](https://github.com/flutter/flutter/issues/187212).
- **Owner/reviewer**: eyebrowsoffire requested changes and walley892 is also requested.
- **Staleness**: Not stale.
- **Labels to add**: triaged-web, e: web_skwasm
- **Blocker**: Requested changes on image-decoding fallback plus Firefox and Safari test coverage.

### flutter/flutter [#190014](https://github.com/flutter/flutter/pull/190014) - [web] Keep the keyboard up during an iOS caret drag
- **Scope**: Clear iOS text-input lifecycle fix. Fixes [#189744](https://github.com/flutter/flutter/issues/189744).
- **Owner/reviewer**: Renzo-Olivares commented on August 3, which is the first human engagement.
- **Staleness**: Not stale.
- **Labels to add**: triaged-web
- **Blocker**: No approval yet and Dashboard Checks require action.

### flutter/flutter [#190071](https://github.com/flutter/flutter/pull/190071) - [web] Fix null-check crash when a platform view is disposed mid-frame
- **Scope**: Clear platform-view lifecycle fix with a regression test. Fixes [#190017](https://github.com/flutter/flutter/issues/190017).
- **Owner/reviewer**: harryterkelsen approved on August 4.
- **Staleness**: Not stale.
- **Labels to add**: triaged-web, a: platform-views
- **Blocker**: Dashboard Checks require action. Otherwise ready.

### flutter/flutter [#190126](https://github.com/flutter/flutter/pull/190126) - [web] Fix password-manager autofill not reaching text fields
- **Scope**: Clear goal but broad engine and framework change. Fixes [#174773](https://github.com/flutter/flutter/issues/174773).
- **Owner/reviewer**: flutter-zl requested changes on July 31. Renzo-Olivares is requested for the framework text-input side.
- **Staleness**: Not stale.
- **Labels to add**: triaged-web, f: focus
- **Blocker**: Requested changes are open, including the unresolved connection-close edge case.

### flutter/flutter [#190227](https://github.com/flutter/flutter/pull/190227) - Glitch
- **Scope**: Unclear. The title says nothing, the body is one sentence about preserving the width-to-height ratio under zoom and scaling, and there is no linked issue, no expected behavior, and no stated validation.
- **Owner/reviewer**: Rusino is the author and no reviewer is requested. mdebbar is the right reviewer for web paragraph work.
- **Staleness**: Not stale by age at 7 days, but no human review has occurred.
- **Labels to add**: triaged-web, c: rendering
- **Blocker**: Needs a real title, a description, and a motivating issue before a reviewer is requested. Dashboard Checks require action. This is the third open PR from the same author with a title that does not describe the change, so it is worth asking for PR hygiene once rather than per PR.

### flutter/flutter [#190314](https://github.com/flutter/flutter/pull/190314) - [web] Unify MaskFilter and ColorFilter primitives across CanvasKit and Skwasm
- **Scope**: Clear renderer unification step. References [#175630](https://github.com/flutter/flutter/issues/175630).
- **Owner/reviewer**: mdebbar is requested. navaronbracke commented on July 31.
- **Staleness**: Not stale.
- **Labels to add**: triaged-web, e: web_canvaskit, e: web_skwasm
- **Blocker**: Google testing fails and no approval yet.

### flutter/flutter [#190486](https://github.com/flutter/flutter/pull/190486) - [web] Anchor flt-semantics-host at 0,0 to fix WebKit semantics offset
- **Scope**: Clear one-property CSS fix with before and after measurements. Fixes [#190483](https://github.com/flutter/flutter/issues/190483), which is P1 and assigned.
- **Owner/reviewer**: No reviewer requested yet. mdebbar for the view and DOM structure, harryterkelsen as a second.
- **Staleness**: New, opened August 4.
- **Labels to add**: triaged-web, a: accessibility, browser: safari-ios
- **Blocker**: Needs reviewers requested. Dashboard Checks require action.

### flutter/flutter [#190487](https://github.com/flutter/flutter/pull/190487) - [web] Ensure a device-width viewport meta in the built index.html
- **Scope**: Clear, but it spans the tool and the engine: the `flutter create` template, an injection step in `flutter build web`, and an engine warning change. Addresses [#129324](https://github.com/flutter/flutter/issues/129324).
- **Owner/reviewer**: No reviewer requested yet. This needs both a web reviewer and a team-tool reviewer for the build-time injection, since silently rewriting a user's `index.html` is a tool behavior change.
- **Staleness**: New, opened August 4.
- **Labels to add**: triaged-web, a: accessibility
- **Blocker**: Needs reviewers requested. Dashboard Checks require action.

### flutter/flutter [#190563](https://github.com/flutter/flutter/pull/190563) - [web] Unify ui.Vertices
- **Scope**: Clear renderer unification step, same pattern as [#190314](https://github.com/flutter/flutter/pull/190314). References [#175630](https://github.com/flutter/flutter/issues/175630).
- **Owner/reviewer**: mdebbar is requested.
- **Staleness**: New, opened August 4.
- **Labels to add**: triaged-web, e: web_canvaskit, e: web_skwasm
- **Blocker**: Awaiting first human review. No failing checks.

### flutter/packages [#7950](https://github.com/flutter/packages/pull/7950) - [camera_web] Re: Support for camera stream on web
- **Scope**: Clear camera_web feature. Implements flutter/flutter [#92460](https://github.com/flutter/flutter/issues/92460).
- **Owner/reviewer**: mdebbar is requested and should close out the final web review.
- **Staleness**: Stale. No substantive review since May 20, on a PR opened in October 2024.
- **Labels to add**: none
- **Blocker**: Waiting on final human web review. Worth noting this PR touches the same package as [#190355](https://github.com/flutter/flutter/issues/190355), so the roller fix should land first.

### flutter/packages [#12186](https://github.com/flutter/packages/pull/12186) - [google_maps_flutter_web] Issue 64073 implement my location
- **Scope**: Mostly clear. Adds the My Location control and the location indicator, but the body has little implementation detail. Fixes flutter/flutter [#64073](https://github.com/flutter/flutter/issues/64073).
- **Owner/reviewer**: mdebbar is requested. No Flutter-team human review yet.
- **Staleness**: Not stale by age at 24 days, but no human review has occurred.
- **Labels to add**: none
- **Blocker**: Automated review raised runtime, geolocation safety, cleanup, and null-safety concerns that the author should answer before human review time goes in.

## Triage Summary - August 5, 2026
- Triaged: 8 issues
- Close: 0 issues
- Request info: 0 issues
- Re-route: 1 issue
- Remaining untriaged: 0 issues from this week's queue
- PRs reviewed: 25
