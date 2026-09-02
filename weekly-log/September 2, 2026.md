# Flutter Web Triage - September 2, 2026

## Weekly Health Checks

### 1. Unassigned P0/P1 (1)

**#190661** - [web] VoiceOver announces menu items with an off-by-one position/count (N-item menu reads as "2 of N+1")
- P1, `team-web`, `triaged-web`, unassigned, filed 27 days ago, last touched 6 days ago.
- Not a zombie. Real bug, well-analyzed, still open and unowned.
- **Recommend assigning to `flutter-zl`.** It is a web semantics tree shape problem: the `role="menu"` node emits the `SingleChildScrollView` as a `role="group"` child while the real items are hoisted with `aria-owns`, so macOS VoiceOver counts 11 members instead of 10. NVDA and JAWS honor `aria-owns` and get it right.
- **Label cleanup needed.** `p: material_ui`, `package`, and `platform-macos` are all wrong. `packages/flutter/lib/src/material/popup_menu.dart:770` is still in flutter/flutter, not in a `material_ui` package, and the platform is web, not macOS. VoiceOver merely runs on macOS.
  - Remove: `p: material_ui`, `package`, `platform-macos`
  - Add: `platform-web`, `framework`, `engine`

### 2. Open PR count - FLAGGED

- **flutter/flutter: 25 untriaged web PRs** (threshold is 15). Well over.
- flutter/packages: 7 untriaged (`triage-web`).
- Related gap: **PR #190153** (kevmoo, `--web-content-hash` for entrypoint files) carries only `tool` and `CICD`. It has no `platform-web` or `team-web` label, so it never surfaces in web PR triage even though issues #191914-#191917 are labeled `team-web`. Recommend adding `platform-web` + `team-web` to it.

### 3. P0 weekly update

- Only open `team-web` P0 is **#190199** (Linux web_benchmarks_skwasm 2.04% flaky), assigned to `eyebrowsoffire`, updated 8 hours ago. Has recent activity. No action.

### 4. Stale "waiting for customer response"

- None. Clean.

---

## Issue Triage

### #191800 - [web][canvaskit] Images loaded via createImageCodecFromUrl turn black after their ui.Image is disposed (e.g. by ImageCache eviction) - regression in 3.47.0
- **Action**: triage
- **Priority**: P1
- **Suggested assignee**: harryterkelsen
- **Labels to add**: triaged-web, P1, engine, e: web_canvaskit, a: images
- **Reasoning**: Regression in 3.47.0 that breaks `cached_network_image` on web, one of the most-depended-on packages on pub. Two independent reporters plus a matching third-party report. Guide says bump a regression to P1 when the feature is widely used.
- **Root cause**: `ImageElementImageSource._doClose()` at `engine/src/flutter/lib/web_ui/lib/src/engine/primitives/image_source.dart:106-110` clears `imageElement.src = ''` as soon as the refcount hits zero. The CanvasKit `SkImage` for an `<img>`-backed source samples that element lazily at raster time, so a `ui.Picture` recorded earlier rasterizes black once the handle is disposed. The sibling `ImageBitmapImageSource` and `VideoFrameImageSource` paths close a resource that genuinely owns its pixels, so only the `<img>` path breaks. `ImageCache` eviction hits this without any user `dispose()` call, which is why long image lists go black on scroll-back.
- **Potential solutions**:
  1. Drop the `src = ''` clear entirely and let GC reclaim the `<img>`. Simplest, gives up eager memory release.
  2. Retain the `ImageSource` for the lifetime of any `ui.Picture` that references it, releasing on picture disposal rather than on image-handle disposal.
  3. Snapshot the `<img>` into an `ImageBitmap` at decode time so the `SkImage` owns pixels the element cannot invalidate.

### #191869 - [two_dimensional_scrollables] [web] merged cells and span decorations has severe debug-mode scrolling jank
- **Action**: re-route
- **Priority**: P2
- **Suggested assignee**: (blank - re-routing off team-web)
- **Labels to add**: triaged-web, team-framework, fyi-web, P2
- **Labels to remove**: team-web
- **Reasoning**: The hot path is a framework assertion, not web engine code. It is not web-specific, it is just far more visible in DDC-compiled debug builds than on the VM. Guide says keep `team-web` for framework issues on web "unless it's clearly not web-specific", and this one clearly is not.
- **Root cause**: `RenderTwoDimensionalViewport.parentDataOf` asserts with three O(n) scans (`_children.containsValue`, `_keepAliveBucket.containsValue`, `_debugOrphans!.contains`) at `packages/flutter/lib/src/widgets/two_dimensional_viewport.dart:883-889`. `TableViewport` calls it repeatedly while splitting decorations around merged cells, so debug frames go quadratic in child count. The reporter's suggested `O(1)` fix via `_children[child.parentData.vicinity] == child` is sound, since `ChildVicinity` is already on the parent data.

### #191914 - [web] Support content-hashed entrypoint filenames (--web-content-hash)
- **Action**: triage
- **Priority**: P2 (already set)
- **Suggested assignee**: kevmoo (filed it, and PR #190153 is in flight)
- **Labels to add**: triaged-web
- **Reasoning**: Scoped, well-specified work item under umbrella #149031 with an open PR. Nothing to re-route; kevmoo deliberately filed these against team-web as part of the web build story.
- **Note**: PR #190153 needs `platform-web` + `team-web` so it stops falling out of web PR triage.

### #191915 - [web] Support content-hashed static assets in AssetManifest (--web-content-hash)
- **Action**: triage
- **Priority**: P2 (already set)
- **Suggested assignee**: kevmoo
- **Labels to add**: triaged-web
- **Reasoning**: Same umbrella (#149031), clearly specified, no PR yet. Confirm and move on.

### #191916 - [web] Emit build precache manifest for custom service workers and PWAs (--web-content-hash)
- **Action**: triage
- **Priority**: P3 (already set)
- **Suggested assignee**: kevmoo
- **Labels to add**: triaged-web, c: new feature
- **Reasoning**: Follow-on to the service worker removal in #156910. Lowest of the four, correctly P3.

### #191917 - [web] End-to-end content-hashing for deferred loading parts across Dart SDK, Engine, and Tools
- **Action**: triage
- **Priority**: P2 (already set)
- **Suggested assignee**: kevmoo
- **Labels to add**: triaged-web
- **Reasoning**: Spans dart2js, dart2wasm, `flutter.js`, and `flutter_tools`. Confirm ownership stays with the person driving the umbrella; the Dart SDK half will need its own tracking on the SDK side.

### #191920 - [flutter_tools] Web test run reports "did not complete" for every remaining test with no diagnostic when the browser connection drops
- **Action**: triage
- **Priority**: P2
- **Suggested assignee**: mdebbar
- **Labels to add**: triaged-web, P2
- **Reasoning**: `flutter_web_platform.dart` is web team code and this is test infrastructure, which the assignee table puts with mdebbar. Diagnostics gap, not a correctness bug, so P2.
- **Root cause**: `BrowserManager` wires `_channel.stream.listen(_onMessage, onDone: close)`, and `close()` sets `_closed = true` before the browser process exits. Both existing "browser went away" diagnostics are guarded on flags that `close()` has already flipped: `BrowserManager`'s `_browser.onExit` handler checks `if (!_closed)`, and `Chromium`'s exit handler checks `if (!_didClose && code != 0)`. So the one failure mode where the browser stops serving without dying is the only one that prints nothing.
- **Blocked contributor worth unblocking**: gabrimatic has a verified fix plus a regression test on branch `web-test-browser-disconnect`, but cannot open the PR because non-write-access contributors are capped at two open PRs and they are at the cap. Either someone lands it on their behalf or the cap gets waived.

### #192013 - `flutter test --platform chrome` hangs forever when Chromium cannot bind its remote debugging port
- **Action**: triage
- **Priority**: P2
- **Suggested assignee**: mdebbar
- **Labels to add**: triaged-web, P2, tool, a: tests, c: performance
- **Reasoning**: `chrome.dart` is web team code. Exceptional report: deterministic repro, verified fix, and the reporter distinguished the IPv4-only case (recovers) from the dual-stack case (hangs). P2 rather than P1 because it needs a port collision to trigger.
- **Root cause**: `_spawnChromiumProcess` in `packages/flutter_tools/lib/src/web/chrome.dart` awaits the `DevTools listening` stderr line with no timeout, and its `orElse` fires only when stderr closes. A Chromium that starts, fails to bind its debug port, and stays alive satisfies neither condition, so the `throwToolExit` inside `orElse` and the 3-try retry are both unreachable. The port comes from `OperatingSystemUtils.findFreePort`, whose own doc comment states the TOCTOU race that produces this state.
- **Potential solutions**:
  1. `.timeout(const Duration(seconds: 30))` on the marker wait, setting `shouldRetry` so the existing retry loop engages. Reporter verified this locally: infinite hang became `Failed to launch browser after 3 tries.` in 41 seconds.
  2. Additionally re-pick the port inside the retry loop rather than once before it, so a retry can actually recover instead of re-colliding.
  3. Detect the `Cannot start http server for devtools.` stderr line directly and fail fast with a message naming the port.
- **Note**: sibling report #192014 from the same reporter, a different unbounded wait (CanvasKit init promise) on the same path. Worth triaging alongside; it did not appear in this week's untriaged list.

### #192054 - Flutter web serving stale app on reload of tab
- **Action**: re-route
- **Priority**: P2
- **Suggested assignee**: (blank - re-routing off team-web)
- **Labels to add**: triaged-web, team-tool, fyi-web, tool, P2
- **Labels to remove**: team-web
- **Reasoning**: The dev server module cache lives in `flutter_tools/lib/src/isolated/devfs_web.dart`, which is tool-team territory (recent history is Ben Konyi and Kevin Moore, not web team). Silently serving stale code is a nasty dev-loop bug and it bit a public Observable Flutter stream, so it is at the top of P2; make it P1 if tool team sees more reports.
- **Root cause**: the web dev server keys DDC library-bundle modules by their strongly-connected-component name. Breaking an import cycle changes the SCC, so `main.dart` recompiles under a new key while the old entry stays in the cache. A full page reload can then still resolve the stale module. Hot restart works because it re-resolves the entrypoint; only the reload path reads the stale key.

### #192091 - [flutter_tools][web] flutter drive -d chrome hangs before WebDriver starts
- **Action**: re-route
- **Priority**: P1
- **Suggested assignee**: (blank - re-routing off team-web; PR #185481 author is temoorx)
- **Labels to add**: triaged-web, team-tool, fyi-web, c: regression, P1, found in release: 3.47
- **Labels to remove**: team-web
- **Reasoning**: Regression between 3.44.8 and 3.47.0 that blocks entire web integration suites, reproduced by a triager on both stable and master, with the causing commit already bisected. `drive.dart`, `web_device.dart`, and `resident_web_runner.dart` are all tool-team files. P1 because a customer's full suite is blocked and the workaround (`-d web-server`) changes what is under test.
- **Root cause**: startup deadlock introduced by #185481 (`4d5e75f`). `DriveCommand` awaits `driverService.start()` before calling `startTest()`, and passes `no-launch-chrome: true` for web devices, so `ChromiumDevice.startApp` never calls `chromeLauncher.launch()`. But `ResidentWebRunner.attach` still awaits `ChromiumLauncher.connectedInstance` because the device is a `ChromiumDevice`, and the only thing that would satisfy that wait is the browser `WebDriverService.startTest()` opens, which `start()` has not returned to allow. Still present on `main`.
- **Potential solutions**:
  1. Skip the `connectedInstance` wait in `ResidentWebRunner.attach` when `no-launch-chrome` is set, as the reporter suggests.
  2. Have `DriveCommand` run `start()` and `startTest()` concurrently for web devices so the WebDriver session can satisfy the connection wait.
  3. Add a command-lifecycle test covering `flutter drive -d chrome`, which is the ordering the regression test in #185481 did not exercise.

### #192102 - [web] Keyboard input stops working after toggling obscureText via suffixIcon and blurring the field
- **Action**: triage only (routing already changed)
- **Priority**: P2
- **Suggested assignee**: (blank - already routed to team-text-input)
- **Labels to add**: triaged-web, P2
- **Reasoning**: Already carries `team-text-input` + `fyi-web` from a prior re-route, so per the guide the routing stands and web team just closes out its review. A dead text field is bad, but it needs a specific toggle-then-blur sequence and clears on reload, so P2.
- **Root cause**: web-only. `DefaultTextEditingStrategy.applyConfiguration` at `engine/src/flutter/lib/web_ui/lib/src/engine/text_editing/text_editing.dart:1623-1625` only ever *sets* `type="password"` when `config.obscureText` is true, with no `else` to clear it. Compare the `readOnly` branch six lines above, which correctly pairs `setAttribute` with `removeAttribute`. So once obscureText has been true, the DOM input keeps `type=password` forever, leaving Chrome's password manager attached to an element the engine now treats as plain text. That asymmetry is a real defect regardless; whether it fully explains the dead-keyboard symptom still needs a repro against a local engine build.

### #191484 - AccessibilityFocusBlockType.blockNode not working in Flutter Web
- **Action**: triage only (routing already changed)
- **Priority**: P3 (already set)
- **Suggested assignee**: (blank - already routed to team-accessibility; `flutter-zl` could take it, since the fix is entirely in web_ui)
- **Labels to add**: triaged-web
- **Reasoning**: Already `team-accessibility` + `triaged-accessibility` + `fyi-web`, so the routing stands. Web team just marks its review done.
- **Root cause**: web-only, confirmed by grep. `AccessibilityFocusBlockType` and `blockNode` appear nowhere under `engine/src/flutter/lib/web_ui/lib/src/engine/`. The framework computes the flag (`packages/flutter/lib/src/semantics/semantics.dart:126-152`, including merge semantics) and ships it in the semantics update, but the web engine never reads it, so the node keeps a focusable ARIA representation and VoiceOver stops on it. Android honors it, which is why the platforms diverge. The fix belongs in web_ui's semantics role and focus handling: when a node is `blockNode`, keep it as a live-region container but drop it from the focusable tree. Related closed PR: #182789.

### #192155 - [google_maps_flutter_web] Add double-tap and context-menu callbacks for interactive map objects
- **Action**: re-route
- **Priority**: P3
- **Suggested assignee**: (blank - re-routing off team-web)
- **Labels to add**: triaged-web, team-ecosystem, fyi-web, P3
- **Labels to remove**: team-web, fyi-ecosystem
- **Reasoning**: First-party plugin in flutter/packages. Even though the capability only exists on web, the proposal adds public API to the app-facing package and the platform interface, so ecosystem owns the design call, including the `consumeDoubleTapEvents` / `consumeContextMenuEvents` question the reporter explicitly asks for feedback on. P3 per the guide: a proposal from a single reporter with no demonstrated multi-user demand.
- **Note**: the reporter also authored packages PR #11952, so they are likely willing to implement once the API shape is settled.

---

## Triage Summary - September 2, 2026

- Triaged: 13 issues
- Close: 0 issues
- Request info: 0 issues
- Re-route: 4 issues (#191869 to team-framework, #192054 and #192091 to team-tool, #192155 to team-ecosystem)
- Confirm-only, routing already set by others: 2 issues (#192102, #191484)
- Remaining untriaged: 0 issues
- Priority spread: P1 x2 (#191800, #192091), P2 x7, P3 x4

---

## PR Triage

### flutter/flutter - 25 untriaged, OVER THE 15 THRESHOLD

Recommend `triaged-web` on all of the below unless noted.

**Stale, needs attention (approved or sitting >2 weeks with no movement):**

| PR | Title | Author | Age | State | Note |
|---|---|---|---|---|---|
| #188010 | refactor(web): use singular `crossOriginStorage.requestFileHandle()` | tomayac | 2mo | APPROVED | Approved 27 days ago and never landed. Closest thing to a stale PR this week. Needs someone to land it or say why not. |
| #189835 | [web] Stop percent-decoding URLs written to browser history | apinilabs-pascal | 1mo | APPROVED | Approved, untouched 11 days. Land it. |
| #185622 | [web] Fix multi-view sizing race condition (Lock approach) | mdebbar | 4mo | CHANGES_REQUESTED | Oldest engine PR still open. mdebbar's own; needs a decision on the Lock approach. |
| #184029 | [a11y] Add a semantics role for slider | hannah-hyj | 5mo | REVIEW_REQUIRED | Oldest web PR in the queue. Cross-team (`team-android` + `team-web`). Needs a web reviewer; `flutter-zl` is the natural fit. |
| #184281 | [web] Enable address bar collapse on mobile by switching touch input | koji-1009 | 5mo | REVIEW_REQUIRED | 5 months with no reviewer. Behavior change to touch handling, needs an owner assigned or an explicit decline. |

**Conflicting, author action needed:**

| PR | Title | Author | Note |
|---|---|---|---|
| #189858 | [WebParagraph] Fixing edge cases for wrapping text (with newlines) | Rusino | CONFLICTING. Needs rebase. |
| #190126 | [web] Fix password-manager autofill not reaching text fields | sero583 | CONFLICTING + CHANGES_REQUESTED. Overlaps #185327 (iOS Chrome autofill). Worth checking against that root cause before more review effort goes in. |

**Needs a reviewer assigned:**

| PR | Title | Author | Age |
|---|---|---|---|
| #190651 | fix(web): guard `parseBrowserLanguages` against invalid locale tags | sankalpsthakur | 27d |
| #191003 | Make web test subshards dynamically determined by .ci.yaml | mdebbar | 21d |
| #191107 | [web] expose engine initialization and view handling | schultek | 19d |
| #191318 | [web] Support local screenshot and golden testing in felt test | harryterkelsen | 14d |
| #191341 | [stable] fix: limit render size of custom elements | holzgeist | 14d |
| #191342 | chore: correct copy paste error in test name | holzgeist | 14d - trivial, just land it |
| #191529 | fix: prefer pointer events over semantics tap for mouse clicks on web | rkishan516 | 11d - a11y-adjacent, route to `flutter-zl` |
| #191647 | [web] Report `env(safe-area-inset-*)` as `FlutterView.viewPadding` | diegolopezrm | 8d |
| #191736 | [WebParagraph] Fix text pixelation, glyph clipping, dual-mode subpixel raster caching | Rusino | 8d - `will affect goldens` |
| #192072 | [web] Defer disposal of stale cached paths to FrameArena | mdebbar | 1d |
| #192128 | Reland: Only render views that need to be rendered | knopp | 21h |
| #189506 | fix: Apply reversed axis flip by macOS on Web when scroll with modifier keys | Gustl22 | 1mo - cross-team with `team-macos` |
| #189953 | [wimp] Adds wimp-heavy variant for non-chrome browsers | gaaclarke | 1mo - CHANGES_REQUESTED |
| #185152 | Support soft hyphen (U+00AD) rendering with a Hyphens API | dbebawy | 4mo - CHANGES_REQUESTED, new public API |
| #192006 | [CP-stable] Fix Actions.handler to forward the intent type to maybeFind (#191052) | amake | 4d - APPROVED, `autosubmit`, `cp: review`. Cherry-pick, will land on its own. |

**Own PRs (flutter-zl), for tracking:**

| PR | State | Note |
|---|---|---|
| #188820 | [web] Fix VoiceOver child focus direction | APPROVED + `autosubmit`. Landing. |
| #190856 | [web] Scope the mobile semantics placeholder to its view | REVIEW_REQUIRED. **Carries an `r: duplicate` label, which is wrong on a PR.** Recommend removing it. |
| #190942 | Reland "[web] Keep the keyboard up during an iOS caret drag" | REVIEW_REQUIRED. Needs a reviewer. |

### flutter/packages - 7 untriaged (`triage-web`)

| PR | Title | Author | Age | State | Recommendation |
|---|---|---|---|---|---|
| #7950 | [camera_web] Re: Support for camera stream on web | TecHaxter | **1 year** | CHANGES_REQUESTED + CONFLICTING | Stale by any measure. Either ping the author with a deadline or close it. |
| #11872 | [google_maps_flutter] Add onPointOfInterestTap callback | tenninebt | 2mo | CHANGES_REQUESTED + CONFLICTING | Federated, needs rebase and author follow-up. |
| #11952 | [google_maps_flutter_web] Avoid replacing advanced marker content on move | Gibbo97 | 2mo | REVIEW_REQUIRED | Needs a reviewer. Same author as issue #192155. |
| #11966 | [google_maps_flutter_web] Fix AdvancedMarker anchors on web | 3ph | 2mo | APPROVED | Approved 19 days ago, not landed. Land it. |
| #12186 | [google_maps_flutter_web] Issue 64073 implement my location | Zubii12 | 1mo | REVIEW_REQUIRED | Needs a reviewer. |
| #12357 | [google_maps_flutter] Add background color for unloaded tiles | RyanHolanda | 29d | APPROVED | Land it. |
| #12647 | [camera_web] Fix TypeError when reading the torch capability | 0xharkirat | 6d | APPROVED | Land it. |

Recommend `triaged-web` on all seven.

---

## Action Items

1. **Assign #191800 to harryterkelsen at P1.** Widely-used-package regression in 3.47.0, root cause identified, one-line-ish fix in `image_source.dart`.
2. **Assign #192091 P1 and re-route to team-tool.** Regression from #185481 blocking web integration suites; loop temoorx in.
3. **Clean up labels on #190661 and assign it to `flutter-zl`.** Three wrong labels and unowned P1 for 27 days.
4. **Unblock gabrimatic on #191920.** Verified fix with a test, blocked only by the two-open-PR cap.
5. **Land the four approved-but-idle PRs**: #188010, #189835 (flutter/flutter), #11966, #12357, #12647 (flutter/packages).
6. **Add `platform-web` + `team-web` to PR #190153** so the content-hash work stops falling out of web triage.
7. **Remove the `r: duplicate` label from PR #190856.**
8. **Address the 25-PR backlog.** Five PRs have gone 1-5 months with no reviewer at all: #184029, #184281, #185152, #185622, #189506.
