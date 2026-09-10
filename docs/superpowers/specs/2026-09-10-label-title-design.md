# Label Title Overlay — Design Spec

Date: 2026-09-10
Status: Approved for planning

## Summary

Add a "label title" decoration mode to swayFx: a small floating title
label that overlaps window content instead of reserving space like a
normal titlebar. Position (edge + alignment), max width, corner
radius, auto-hide, and cursor-avoidance are all configurable per
container. It works independently of `border` mode (none/pixel/normal/csd).

## Motivation

Sway's current titlebar (`border normal`) always reserves a horizontal
strip across the full window width, pushing content down and wasting
space when only a small title indicator is wanted. The label overlay
gives users a compact alternative: a title-width (not window-width)
tag that floats over the window, positioned at any edge/alignment, and
can get out of the way (fade on unfocus, or slide away from the
cursor) when not needed.

## Non-goals (v1)

- IPC exposure of label state/geometry.
- Global default config separate from per-container (can follow the
  `corner_radius` global-then-per-container evolution later).
- New theming beyond corner radius — label reuses existing titlebar
  colors/fonts.
- Multi-line or wrapped label text.

## Config commands

All per-container, registered in the runtime `command_handlers[]`
table (same as `border`/`title_format`, not the config-only
`handlers[]` table `corner_radius` currently uses):

- `label enable|disable|toggle`
  Enables/disables the overlay. When enabled, suppresses the reserved
  titlebar strip even if `border normal` is also set for that
  container — label and full titlebar never both reserve/show space
  at once; label wins.
- `label position <top|bottom> <left|center|right>`
  Vertical edge + horizontal alignment along that edge. Horizontal
  values reuse the existing `title_align` enum; vertical edge is a new
  two-value enum.
- `label max_width <px>|<percent>%`
  Caps label width. Accepts either an absolute pixel value or a
  percentage of the window's width (parser distinguishes by trailing
  `%`, similar to mixed-unit parsing elsewhere in sway config). Label
  is never wider than the window regardless of measured title text
  width — text is ellipsized if it would exceed the cap.
- `label corner_radius <px>|match`
  Fixed pixel radius for the label's own corners, or `match` to
  inherit the container's `corner_radius` value live (tracks changes).
- `label autohide <off|<ms>>`
  `off` (or `0`) keeps the label always visible. A positive value is
  the delay in milliseconds after the container loses focus before the
  label fades out; regaining focus or a cursor-avoidance hover event
  cancels the timer and restores full opacity.
- `label avoid_cursor <on|off>`
  When `on`, the label slides out of the way when the pointer enters
  its hit region (e.g. hovering a button or text it would otherwise
  cover), sliding back when the pointer leaves.

## Data model

New fields on `struct sway_container` (mirrors the existing
`corner_radius` per-container field pattern in
`include/sway/tree/container.h`):

```c
bool label_enabled;
enum sway_label_edge label_edge;       // TOP, BOTTOM
enum sway_title_align label_align;     // reuse existing enum (LEFT/CENTER/RIGHT)
int label_max_width;                   // px, or...
float label_max_width_percent;         // ...percent of window width; unit flag distinguishes
bool label_max_width_is_percent;
int label_corner_radius;
bool label_corner_radius_match_window;
int label_autohide_ms;                 // 0 = always visible
bool label_avoid_cursor;

// runtime (not config), state.c or container runtime struct:
bool label_visible;                    // current faded/visible state
double label_cursor_offset_x, label_cursor_offset_y; // avoid-cursor slide offset
uint32_t label_autohide_timer;         // wl_event_source handle
```

## Geometry and rendering

The label reuses the existing `title_bar` scene subtree
(`con->title_bar.tree`, created in `container_create()`,
`sway/tree/container.c`) and the existing text-node infrastructure
(`sway_text_node_create`, `container_update_title_bar()`,
`container_update_marks()`) — no new rendering backend needed.

Two things change relative to a normal titlebar:

1. **Content box is not shrunk.** When `label_enabled` is true,
   `view_autoconfigure()`'s `B_NORMAL` case
   (`sway/tree/view.c:443-458`) and the tabbed/stacked title-strip
   offset logic in `sway/tree/arrange.c` (`apply_tabbed_layout`/
   `apply_stacked_layout`) must skip the
   `container_titlebar_height()` subtraction/offset for this
   container — the content box is the full container box (minus
   normal border thickness only, not titlebar height).

2. **Positioning is overlay math, not reserved-strip math.** In
   `sway/desktop/transaction.c`, where `arrange_title_bar()`
   (`transaction.c:372-386`) is called from `_arrange_container()`,
   the x/y/width passed in are computed from `label_edge`/`label_align`
   instead of the full-width strip. Width = `min(measured title text
   width, max_width)` (resolving percent against container width).
   The label rect overlaps the top or bottom rows of the content area
   rather than sitting above it.

Corner radius: `get_titlebar_corners()` (`sway/tree/container.c:322-352`)
is extended to accept the label's own radius (fixed or
match-window-live) instead of always deriving from
`con->corner_radius`.

## Interaction

The label's scene node keeps the titlebar's existing pointer
handling — drag-to-move, double-click-to-maximize, right-click menu —
via the same `seatop` hit-testing path used for normal titlebars, just
scoped to the label's (smaller) box instead of the full-width strip.
Pointer events outside the label's bounds fall through to window
content underneath, same as clicking anywhere else on the surface.

## Move-out-of-the-way behavior

Two independently configurable triggers:

- **Autohide timer.** On container focus-lost, if `label_autohide_ms >
  0`, arm a `wl_event_loop` timer for that duration. On fire, animate
  label opacity to 0 (reusing the animation/transition infrastructure
  already driving corner-radius/blur/shadow transitions). Refocus, or
  an avoid-cursor hover event, cancels the timer and restores opacity.
- **Avoid-cursor.** On pointer motion, if the cursor position
  intersects the label's current scene-node box, compute a slide
  offset (along the label's edge, or perpendicular to it) sufficient
  to clear the cursor, and animate the label's position by that offset
  using the same transition infrastructure. Reverses when the cursor
  moves away.

Both triggers are independent and can be combined; either, both, or
neither may be enabled per container.

## Testing

- Manual: verify label renders at each edge/alignment combination,
  respects max_width cap (px and percent) with ellipsis on overflow,
  corner radius fixed vs. match-window (including live-follow when
  `corner_radius` changes at runtime).
- Manual: verify content box is full-size (no reserved strip) with
  label enabled, across `border none/pixel/normal/csd`.
- Manual: verify label suppresses full titlebar strip when
  `border normal` + `label enable` both set.
- Manual: verify drag/double-click/right-click on label match normal
  titlebar behavior, and pass-through outside label bounds.
- Manual: verify autohide timer fades/restores correctly on
  focus/unfocus and cancels on hover; verify avoid-cursor slide
  engages/disengages smoothly and doesn't fight with autohide.
- Existing sway/swayfx test suite (if any covers titlebar/geometry)
  should still pass unchanged for containers with `label` disabled
  (default), confirming no regression to existing titlebar behavior.
