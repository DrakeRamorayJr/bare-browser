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

# Follow-up: the url text still doubled and the favicon copy was square (patch `0000458`)

A test of `0000457` on an emulator with 40x slower animations showed the doubled icons were fixed, and two
things were left in the pill during the close cross-fade. The url text still read as doubled toward the end
of the fade ("wiki//Android_(("), although its first glyph and baseline lined up. The favicon copy was a
square where the real favicon is a circle.

## Root cause

- Url text. `0000457` still drew the text from the pill's own `TextView`s and moved them so the first glyph
  and the baseline met the real ones. Nothing else about the two layouts was shared. The pill's text view is
  a `match_parent` box in a stack inside the pill, with its own end padding, no horizontal scroll, no fading
  edge and an ellipsis on its own width, while the real `UrlBar` has its own width and end, its scroll
  offset, its fading edge and its own ellipsis. Two layouts of one string that start at the same pixel end
  at different ones, so the drift grows along the line and the two copies read as one word printed twice.
  Matching more properties (spacing, scale, ellipsis) narrows it and cannot close it.
- Favicon. The real favicon is not square because of its drawable. `StatusView` gives the view an oval
  outline and `setClipToOutline(true)`, so whatever bitmap arrives is cut to a circle. A clip to an outline
  is applied only when the compositor draws the view. `0000457` copies an icon by drawing the real view into
  a software bitmap with `View.draw()`, which does not apply that clip, so the bitmap keeps its square
  corners. In a Robolectric run with native graphics, an `ImageView` with an oval outline and a clip drawn
  this way has an opaque pixel in its corner.

## Fix (patch `0000458`)

Only the hand-off copies change. The bar band, the `+` highlight, the page and top toolbar restore, the
open path and the 130 ms fade are as they were.

- The real url view is copied as one bitmap, `drawViewSnapshot()`, shown by a new floating view
  (`morph_url_snapshot`) at the real view's own size and place, with the hand-off's alpha. The text,
  its end, the ellipsis, the fading edge, the spacing and the colour spans are the real view's because the
  real view draws them. The canvas is shifted by `-scrollX, -scrollY` first, the shift a parent applies
  before it draws a child, so a url bar that is scrolled to its host is copied from where it is scrolled.
  The icon copies use the same function, `drawIconSnapshot()` now calls it.
- It runs only on a hand-off that read the live bar (`mHandoffLive`), so the existing recapture tolerance
  applies as it did to the icons. While the bitmap is up the pill's own text views are drawn at alpha 0
  and are otherwise untouched, so `0000457`'s alignment stays as the fallback for a bar that was not read,
  a view too big to copy or one that cannot be drawn into a software bitmap. When the fade ends or the
  sheet opens again, `clearMorphFloats()` hides the bitmap and frees it, and the pill text is back.
- An icon copy now clips like its real view. Every ghost gets a `GhostClip` outline provider. Each frame
  `matchGhostClip()` asks the real view for its outline and, if it clips to one the platform can clip to,
  gives the copy `clipToOutline` and the real view's provider, asked on the copy so the shape follows the
  copy's own size (a circle stays a circle). A copy that moves on to an icon without a clip loses it. The
  outline carries alpha 0, because the copy sits at the morph's elevation and would otherwise cast a
  shadow in that shape. A provider that gives an empty outline on the copy (a clip by background) gets the
  copy's full rect, as an empty outline would clip the copy away. It applies to the copies while the pill
  travels as well, so the favicon is round before the hand-off too.
- A failed copy now falls back on any `RuntimeException`, not only `IllegalStateException`, so a view that
  cannot be drawn into a software canvas can never fail a close.
- The url bitmap now records that it has cleared the pill's text. Taking it down in a layout frame needs
  nothing else, since `applyMorphText()` runs before it in every frame and gives the text its alphas back,
  so a frame that finds the live bar gone or the copy failed shows the text again. A teardown with no
  frame of its own is `clearMorphFloats()`, which the settle, the reset paths and `destroy()` all end in:
  it now gives the label alpha 1 back when the bitmap had cleared it, as a settled pill has. The bitmap
  code never changes the label's or the copy's visibility, so there is none to restore.

Not done: the optional change to draw the icon copies at their final size before the hand-off. Before the
hand-off a copy is the real drawable scaled to the whole padded box (the first ~1.4x jump at the start of
the hand-off). Fixing it means reading the real view's padding, scale type and tint into the copy, or
taking the bitmap for every frame of the pull, from stale rest rects and at the changing scale of the
pull. That touches the whole open and close path and an existing test pins the copy's scale type, so it is
left for a separate patch.

## Risks and things not confirmed

- The drawer and its test class still cannot be built here. They compile with the diagnostics they had
  before (the unresolved Chromium types, plus three more of the same `org.chromium.chrome.R` kind in the
  new tests). The platform behaviour the fix rests on was run in a standalone Robolectric project, on copies
  of the new helpers, not on the drawer: a snapshot with the scroll shift is pixel-identical to the real
  parent-driven draw of a scrolled, spaced, bold, span-coloured, fading-edge text (0 differing pixels at
  scroll 0, 37 and 120, about 2500 without the shift), and the clip provider gives a circle at the copy's
  size, alpha 0, and none for an icon without a clip.
- Not confirmed on a device: that the compositor clips the copy to the circle, and that the alpha-0 outline
  casts no shadow at the morph's elevation (hwui skips the shadow for an outline of alpha 0). A faint ring
  round the favicon copy would mean it does.
- The url bitmap is redrawn from the real view every frame of the fade, up to 1200dp by 160dp. It is a
  software draw of a one-line text view. If the real url changes during the 130 ms the copy follows a
  frame later, and at the first frame of the hand-off the pill's text (from the string the bar showed
  before the sheet opened) is replaced by the real view's current text.
- Reopening in the middle of the fade does not leave the label hidden, although the first version of the
  test for it reported that it did. `show()` drops the hand-off flags and lays out once at progress 0, which resets the morph's
  anchor. The next frame is then the first morph frame of any open: the pill is the bar itself, the url
  is in the from-view and the label is at alpha 0 by design. A test that jumps from 0 to the half detent
  in one step ends on that frame, so the label is at 0 there with or without the url bitmap, which is
  already down by then (a frame after `show()` is never a live hand-off frame). The test now checks that
  the text is up on that frame, in the label or the from-view, and that the label is whole on the next
  frame, the settled pill. A test for `destroy()` during the fade covers the teardown path.
- Two unit tests need a canvas that keeps its translation, which Robolectric's native graphics mode does
  and its legacy mode does not. The scroll test skips itself where the canvas does not.

# Follow-up: the icon copies dropped in size and the url text jumped when the hand-off started (patch `0000459`)

A test of `0000458` on an emulator showed one url text and a round favicon copy. Two steps were left in the
close, both on the frame the hand-off starts (the frame where the page shifts). Before it, the icon copies in
the pill were about 1.4 times larger than the real toolbar icons and dropped to the real size in one frame.
The url text jumped 4 to 10 px on that frame and then settled.

## Root cause

- Icon size. Before the hand-off an icon copy was the real view's drawable, set on an `ImageView` the size
  of the whole padded box of the real view with `FIT_CENTER`. The real view draws the same drawable inside
  its padding, so the glyph in the copy filled a box the real one only fills to its padding. From the
  hand-off frame on, the copy is a bitmap of the real view (`0000457`), which has the padding, scale type
  and badge, so the glyph is drawn at the real size. The drop in size is the change from one to the other.
- Text position. Two things differed between the last frames of the travel and the hand-off.
  - The pill's text inset. `applyMorphText()` used the stored resting glyph inset (`mRestTextLead`, the
    real url view's left plus its own padding, less the bar's left) only on the first frame of an open, and
    otherwise the inset of the url view itself (its left less the bar's left, without its padding). The
    glyphs sit further in by the url view's padding, so the pill text and the point it ellipsizes at were
    off by that padding for the whole travel and moved to the resting inset on the hand-off frame.
  - The alignment. The hand-off frames move the pill's text views so the first glyph and the first
    baseline are the real ones (`0000457`). The frames before it did not, so a difference in font padding
    and baseline between the two views showed up as a step on the hand-off frame as well.

## Fix (patch `0000459`)

Only the frames of a close before the hand-off change. The bar band, the `+` highlight, the page and top
toolbar restore, the open path, the 130 ms fade, the url bitmap and the circular favicon copy are as they were.

- The drawer now knows which way the sheet is travelling. `applyProgress()` sets `mMorphClosing` when a frame
  moves progress down by more than 0.0002 and clears it when one moves it up by that much, so a close, a drag
  down and a pull back down all count and a drag that turns round flips it. `closeAtVelocity()` sets it and
  `show()` clears it. The way up is untouched by everything below.
- Icons. While closing, a copy of an icon whose view is laid out is the same bitmap of the real view the
  hand-off uses (`drawIconSnapshot()`), at the size and place it already had. A view that is not laid out, or
  that cannot be drawn into a bitmap, keeps the old drawable copy. The size and place of the copy are
  unchanged, so the only change is what is drawn inside it.
- Text inset. While closing, the resting inset is used the whole way down, not only on the first frame.
- Text position. While closing and before the hand-off frames, `alignMorphTextTowardsReal()` shifts the pill's
  two text views by `(1 - eased) * (real - own)`. `real` is the real first glyph and baseline relative to
  the real bar, `own` the view's own relative to the pill, both measured in that frame (the pill is laid out
  synchronously by `placePillOnRoot()`, so no value is predicted or cached). At the bar rect the factor is 1
  and the shift is the whole alignment the hand-off makes, so its first frame moves nothing. At the small
  pill the factor is 0 and the text is where it always was. A hand-off frame still uses `alignMorphText()`,
  and a frame that is not closing, a reopen, and the teardown of the morph clear the shift.

Tests (all in `BareVerticalTabsDrawerUnitTest`): the icon copy is a bitmap of the real icon's box in the frames
before the hand-off; its size and place do not change across the hand-off frame; on the way up it is still
the drawable copy; the first glyph and baseline of the text are on the real ones in the last frame before the
hand-off and across it; the shift grows as the pill comes down and is nearly gone near the top; with a real
`url_bar` the resting inset gives the real width and the real first glyph with no shift to make up for it;
the way up is not shifted; a drag that turns back up and a reopen clear the shift.

## Risks and things not confirmed

- Not run on the drawer, as before. The drawer and its test class still cannot be built here. They compile
  with the diagnostics they had before (unresolved Chromium types, plus more of the same
  `org.chromium.chrome.R` kind in the new tests). What was run is the platform behaviour the fix rests on,
  in a standalone Robolectric project on a model of the pill (a `LinearLayout` with the text stack, the pill
  placed by translation, the same shift formula), in both graphics modes: the pill's first glyph and
  baseline land on the real ones with a bar whose text sits 30 px inside the view (x, y exact at the bar
  rect, with and without a different font padding), the shift is 14, 12.6, 7 and 1.4 px at the ease 0, 0.1,
  0.5 and 0.9, and a `url_bar` inset and end padding give a text width of 198 px for both the copy and
  the real view and need no shift. A view drawn with `View.draw()` ignores its own alpha, so the icon copy of
  a source the pull has faded to 0 is not blank.
- The first glyph and baseline are continuous. The look (letter spacing, typeface, ellipsis and the colour
  spans) is matched by the hand-off frames only, as in `0000458`. The pill's text before the hand-off is
  the string in one colour, so the colour of the host and the path still changes on the hand-off frame. The
  real `UrlBar` has no letter spacing, and a bar that had would also move the glyphs by half of it.
- The shift is measured from the real views each frame, the same views the hand-off reads. If the real bar
  lays out again between the last travel frame and the hand-off frame (a status icon that appears, a url
  view that changes its inset), what is left of the jump is that change.
- The direction comes from progress, so a drag that reverses flips the geometry on that frame: the icons go
  from bitmaps to drawables and the shift goes. The way up is meant to look as it did.
- A close that does not read the live bar (a bar that is mid-slide at the hand-off) keeps the shift, measured
  from the real views relative to their bar, so the pill text is where the real text will be on the next frame.
- While a close travels, up to 16 small icon views are drawn into software bitmaps each frame, as the
  hand-off already does for its 130 ms. The bitmaps are reused, and a view that cannot be drawn falls back.
