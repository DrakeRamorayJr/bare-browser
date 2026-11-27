# Diagnosis: one-frame snap at the end of the bottom tab sheet close

This was worked out by reading the patched sources (patches up to `0000455`). Chromium cannot be built
in the place this was written, so nothing below was seen running on a device.

## Symptom

The bottom-bar tab sheet slides down smoothly. Then, in one frame, the sheet pill goes away, the pill
text changes from the full "Search DuckDuckGo (No AI) or type URL" to a cut-off "Search DuckDuckGo (No
AI) e...", the uBlock Origin and other extension icons appear, and the button bar looks complete.

## Root cause

All of it is in `chrome/android/java/src/org/chromium/chrome/browser/bare/BareVerticalTabsDrawer.java`.

The end of a close is `closeAtVelocity()` -> spring `SheetSpring.end()` -> `scheduleCloseHandoff()` ->
`commitCloseHandoff()` twice -> `finishCloseChrome()`:

1. Park. When the spring settles, `applyProgress(0)` runs with `shouldParkClose()` true, so
   `layoutBottomBarMorph()` sets `mForceBarRect`. The morph pill, its icon copies (`syncMorphIcons()`)
   and the toolbar-colour cover (`layoutBottomBarCover()`) sit opaque on the resting bar rect. The real
   address views are hidden (`setAddressSurfaceHidden(true)`), the toolbar texture is suppressed
   (`suppressToolbarSceneLayer(true)`), and every real icon outside the bar is set to alpha 0 by
   `fadeSourceIcon()`.
2. Reveal. The first `commitCloseHandoff()` sets `mCloseReveal`. The real views and the texture come
   back, but the morph, the copies and the cover stay opaque on top of them, and `fadeSourceIcon()` still
   hides the real trailing icons on every `syncMorphIcons()` pass. Nothing real can be seen yet.
3. Commit. One frame later the second `commitCloseHandoff()` does everything at once: the `!show` branch
   of `layoutBottomBarMorph()` hides the pill, clears the copies and hides the cover; `restoreFadedBarIcons()`
   puts the real icons back; `applySheetInsets()` stops covering the bar slot (`coverBar` is false); and
   `finishCloseChrome()` hides the root and dismisses the glass window.

So the only thing between "opaque morph" and "real bar" is a single reveal frame, and everything that
differs between the two is swapped in the same vsync: the opaque morph is removed, the real icons go from
alpha 0 to 1, and the real url text replaces the morph text. The real text is laid out differently from
the morph text. ToolbarPhone pads the end of the url bar for the extensions
(`keepUrlTextClearOfExtensions`, patch `0000208`), which is why it is cut off with "...". That one frame
is the snap.

The button bar is less clear. Its Android row is driven only by progress (`buttonBarGone()`,
`fadeBottomButtonBar()`), so by reading the code its fade and slide are already finished near progress
0.175, well before the hand-off. The hand-off does not change the row. The one thing that happens late is
`BottomControlsMediator.setBareSheetTravel()` swapping the row from the Android view back to the
composited texture once the travel is 0.001 or less, which is the last few frames of the spring and which
drops the cached bitmap on every travel frame (`BottomControlsCoordinator.setBareSheetTravel()`). If the
row pops, that swap is the likely place. That code is not in the Bare files this patch edits, and it could
not be checked on a device, so it is not changed here.

## Fix (patch `0000456`)

Keep the same park and reveal, and put a short timed cross-fade between reveal and commit, so the
real bar is already there and complete when the morph is taken away.

- Reveal starts the fade (`startHandoffFade()`), a frame loop on `postOnAnimation` (`stepHandoffFade()`)
  that lasts `CLOSE_HANDOFF_FADE_MS` (130 ms) times the animator duration scale. It is capped by a frame
  count, so a clock that does not move still ends it. With animations off the old two-frame hand-off runs.
- `restoreFadedBarIcons()` runs at reveal and `fadeSourceIcon()` stops hiding icons during the reveal, so
  the real icons are in their final place and visible under the cover and the copies.
- The pill and the icon copies fade out with `morphHandoffAlpha()` over the whole fade. The cover
  clears over the first 60% of it (`coverHandoffAlpha()`), so the real bar and icons show through the pill
  as it fades. The text goes from the morph text to the real text over the fade, not in one frame.
- The commit frame now only removes views that are already clear. The existing `finishCloseChrome()`
  path is unchanged.
- `show()` ends a fade in progress (a reopen during the fade turns the morph back to opaque).
  `destroy()` and `onSheetWindowDismissed()` cancel it. `flushCloseHandoffForTesting()` runs it to the end.

Only `BareVerticalTabsDrawer.java` and `BareVerticalTabsDrawerUnitTest.java` change. Opening, the spring
(`SheetSpring`), the button-bar slide and fade, and the pull from the button bar are not touched.

## Risks and things not confirmed

- Not run. The Java was checked with `javac` against the Android framework with the Chromium symbols
  missing: the new code adds no compile errors to the drawer or to its test. The unit tests, and the
  animation itself, need a real build.
- The cross-fade shows the full and the cut-off text on top of each other for about 130 ms. The first part
  of the text is at the same place, so the difference is the end of the line.
- The glass window now stays up about 130 ms longer, so `restorePageCard()` and
  `deliverPendingOmniboxFocus()` happen that much later. A pill tap in that time hits the close path,
  which does nothing because the sheet is already closed.
- The real bar under the fading morph depends on the toolbar texture or views being drawn by then. If the
  texture is late on a device, the bar area could still be empty for a frame or two. Tune
  `CLOSE_HANDOFF_FADE_MS` and `COVER_HANDOFF_SHARE` first if the hand-off needs more time or less cover.
- The button-bar pop, if there is one, is not fixed by this patch (see above).

# Follow-up: the hand-off copies did not line up with the real toolbar (patch `0000457`)

A device test of `0000456` showed the cross-fade works and the last-frame snap is gone. One problem was
left: during the 130 ms fade the morph's copies did not sit exactly on the real toolbar under them, so the
extension icon, its badge, the puzzle icon, the favicon and the url text were seen twice, at two sizes
and a few pixels apart. As before, this was found by reading the code, not by running it.

## Root cause

Both are in `BareVerticalTabsDrawer.java`, in what the hand-off draws over the real bar.

- Icon copies. `syncMorphIcons()` made each copy an `ImageView` that held only the real view's drawable
  (`getConstantState().newDrawable()`), with `FIT_CENTER`, no padding, no tint and no matrix, sized to the
  real view's whole box. A toolbar button is a padded box that draws a smaller glyph (padding, `CENTER` or
  `CENTER_INSIDE`, tint, a badge in the drawable), so the copy filled the box and was drawn larger than the
  real glyph. The copy's place came from `restIconRect()`, the rect stored before the sheet opened
  (`captureRestContent()`), not from where the view is when the hand-off runs.
- Text. The pill's own `TextView`s laid the string out with their own settings: the default typeface, font
  padding on, no letter spacing, a text box centred in the pill, and the glyph inset (`mRestTextLead`)
  stored before the open. The real url bar lays out the same string with its own typeface, metrics,
  vertical placement, end padding (`keepUrlTextClearOfExtensions`) and colour spans. The 6dp margin of the
  text stack and the in-flow lock and favicon also stay in the row unless a copy replaces them.

## Fix (patch `0000457`)

Everything is measured from the live views at hand-off time and nothing is a constant. It runs on the
parked frame and on every frame of the fade (`mHandoffLive`); every other frame, the open path and the
spring are unchanged.

- `captureHandoffBar()` re-reads the bar rect and the url inset from the live bar (`storeRestBar()`, the
  same store `captureRestBar()` uses). A bar that is mid-slide, or more than `HANDOFF_RECAPTURE_TOLERANCE_DP`
  from where it rested, keeps the stored rect and the old path.
- The reveal shows the real bar and icons before the measuring, not after it.
- Each icon copy is a bitmap of the real view, drawn with `View.draw()`, shown at the view's size and place
  (`drawIconSnapshot()`), so padding, scale type, tint and a badge are the real ones. An icon whose view or
  a parent is `GONE` gets no copy (it keeps an old rect). A view too big, or that cannot be drawn into a
  software bitmap, keeps the old drawable copy.
- The pill text copies the real url text's typeface, font padding, letter spacing, text scale, ellipsis,
  hint colour and colour spans (`matchMorphTextLook()`, `setMorphText()`). The in-flow icons and the stack
  margin are cleared. `alignMorphText()` then moves each text view so its first glyph and its baseline are
  where the real text draws them (from the two layouts, `glyphOrigin()`).
- `clearMorphTextMatch()` and `releaseGhostSnapshots()` undo it when the hand-off ends, and the text is
  put back when the sheet opens again during the fade.

## Risks and things not confirmed

- Not run, as before. The drawer and its test compile with the same diagnostics as before (only the
  unresolved Chromium types); the new tests and the on-screen result need a real build.
- The copies are bitmaps taken on the parked frame and redrawn each fade frame from the real views. If
  the real icon changes during the 130 ms (a badge count), the copy follows it a frame later.
- The text alignment moves the glyphs of the first line. A right-to-left or centred real url text is
  aligned by its first glyph only.
- If a device keeps the real bar more than 16dp from its resting rect at hand-off, the old stored rect and
  the old drawable copies are used, and the mismatch can return. Raise
  `HANDOFF_RECAPTURE_TOLERANCE_DP` if that is the case.

