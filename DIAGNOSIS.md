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
