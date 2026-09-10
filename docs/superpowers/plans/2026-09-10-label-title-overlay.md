# Label Title Overlay Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a per-container "label title" overlay decoration to swayFx — a small floating title chip that overlaps window content (instead of reserving a full-width titlebar strip), configurable by edge/alignment, max width, corner radius, autohide-on-unfocus, and avoid-cursor slide.

**Architecture:** Reuse the existing titlebar scene subtree (`con->title_bar`) and its text-rendering/layout code (`container_arrange_title_bar()`, `sway_text_node`) unchanged for drawing. Only the *positioning* math changes: a new `arrange_label()` path in the transaction/arrange code computes a floating box from edge+alignment+max-width instead of a full-width reserved strip, and `view_autoconfigure()` stops subtracting titlebar height from the content box when a label is active. A new per-container `label` command configures it. Focus-based autohide and cursor-avoidance reuse the existing `animation_manager` (fade/slide) and a `wl_event_loop` timer (cursor-hide-timer style).

**Tech Stack:** C, wlroots/scenefx scene graph, Meson build, sway's config-command parser.

**Spec:** `docs/superpowers/specs/2026-09-10-label-title-design.md`

## Global Constraints

- Default `label` state is disabled (`label_enabled = false`) on every container — zero behavior change for anyone not using the feature.
- v1 scope is **leaf view containers only** (`con->view != NULL`). Tabbed/stacked parent-container tab-strip titlebars (`sway/tree/arrange.c` `apply_tabbed_layout`/`apply_stacked_layout`) are untouched and out of scope — this matches the spec's "window title" framing and avoids an unrelated rewrite of tab-strip layout.
- `label` overlay is independent of `border` (works with `none`/`pixel`/`normal`/`csd`); when `border normal` and `label enable` are both set on the same container, the reserved titlebar strip is suppressed — label wins (per approved spec).
- Label width is always `min(measured title text width, max_width)` — never wider than the window.
- No automated test harness exists in this repo (`meson test` has no compositor unit tests; confirmed empty `test/`). Every task's verification step is: (a) the project builds with `ninja -C build`, and (b) a manual `swaymsg`/live-session check with an exact command and expected outcome. This replaces the classic write-test-first loop; there is no lighter-weight option available in this codebase.
- Follow existing code style: tabs for indentation, `snake_case`, alphabetized command-handler tables.

---

### Task 1: Container & config data model

**Files:**
- Modify: `include/sway/tree/container.h` (add enums + fields near `enum sway_container_border` at line 23 and inside `struct sway_container` near `int corner_radius;` at line 150)
- Modify: `include/sway/config.h` (add global defaults near `enum alignment title_align;` at line 574)
- Modify: `sway/tree/container.c` (`container_create()`, seed per-container fields from config, lines 119-142)
- Modify: `sway/config.c` (set built-in defaults in the config defaults initializer)

**Interfaces:**
- Produces: `enum sway_label_edge { LABEL_EDGE_TOP, LABEL_EDGE_BOTTOM };` and `enum sway_label_align { LABEL_ALIGN_LEFT, LABEL_ALIGN_CENTER, LABEL_ALIGN_RIGHT };` in `include/sway/tree/container.h` — every later task uses these two enums and the `sway_container` fields listed below.
- Produces on `struct sway_container`: `label_enabled` (bool), `label_edge` (`enum sway_label_edge`), `label_align` (`enum sway_label_align`), `label_max_width` (int, px), `label_max_width_percent` (float, 1-100), `label_max_width_is_percent` (bool), `label_corner_radius` (int), `label_corner_radius_match_window` (bool), `label_autohide_ms` (int, 0 = always visible), `label_avoid_cursor` (bool), and a nested `label_state` struct: `struct animation animation; float from_alpha, to_alpha; bool hidden; bool was_focused; struct wl_event_source *autohide_timer; struct animation slide_animation; double slide_from_x, slide_from_y, slide_to_x, slide_to_y, slide_x, slide_y;`
- Produces on `struct sway_config`: `label_enabled`, `label_edge`, `label_align`, `label_max_width`, `label_max_width_percent`, `label_max_width_is_percent`, `label_corner_radius`, `label_corner_radius_match_window`, `label_autohide_ms`, `label_avoid_cursor` — same types as above, used as the seed values in `container_create()` (same role `config->corner_radius` plays for `con->corner_radius`).

- [ ] **Step 1: Add the enums and per-container fields**

In `include/sway/tree/container.h`, add after the `enum sway_container_border` block (line 28):

```c
enum sway_label_edge {
	LABEL_EDGE_TOP,
	LABEL_EDGE_BOTTOM,
};

enum sway_label_align {
	LABEL_ALIGN_LEFT,
	LABEL_ALIGN_CENTER,
	LABEL_ALIGN_RIGHT,
};
```

Add a forward declaration near the other forward declarations (line 41, next to `enum wlr_direction;`):

```c
struct wl_event_source;
```

In `struct sway_container`, immediately after `float dim;` (line 153) and before the existing `animation_state` block, add:

```c
	bool label_enabled;
	enum sway_label_edge label_edge;
	enum sway_label_align label_align;
	int label_max_width;
	float label_max_width_percent;
	bool label_max_width_is_percent;
	int label_corner_radius;
	bool label_corner_radius_match_window;
	int label_autohide_ms;
	bool label_avoid_cursor;

	struct {
		struct animation animation;
		float from_alpha;
		float to_alpha;
		bool hidden;
		bool was_focused;
		struct wl_event_source *autohide_timer;

		struct animation slide_animation;
		double slide_from_x, slide_from_y;
		double slide_to_x, slide_to_y;
		double slide_x, slide_y;
	} label_state;
```

- [ ] **Step 2: Add global config defaults**

In `include/sway/config.h`, add after `enum alignment title_align;` (line 574):

```c
	bool label_enabled;
	enum sway_label_edge label_edge;
	enum sway_label_align label_align;
	int label_max_width;
	float label_max_width_percent;
	bool label_max_width_is_percent;
	int label_corner_radius;
	bool label_corner_radius_match_window;
	int label_autohide_ms;
	bool label_avoid_cursor;
```

Add `#include "sway/tree/container.h"` near the top of `include/sway/config.h` (alongside its other `#include "sway/tree/..."` lines, if any — otherwise add a new include line after the existing `#include` block) so `enum sway_label_edge`/`enum sway_label_align` are visible. (Verified: neither header currently includes the other, so this does not introduce a cycle.)

In `sway/config.c`, find where config defaults are seeded (search for `config->title_align = ALIGN_LEFT` or similar in the defaults-setup function) and add alongside it:

```c
	config->label_enabled = false;
	config->label_edge = LABEL_EDGE_TOP;
	config->label_align = LABEL_ALIGN_CENTER;
	config->label_max_width = 0; // 0 = uncapped by pixels (percent still applies)
	config->label_max_width_percent = 50;
	config->label_max_width_is_percent = true;
	config->label_corner_radius = 0;
	config->label_corner_radius_match_window = true;
	config->label_autohide_ms = 0;
	config->label_avoid_cursor = false;
```

- [ ] **Step 3: Seed per-container fields in `container_create()`**

In `sway/tree/container.c`, in `container_create()` right after the existing `c->corner_radius = config->corner_radius;` block (around line 119-122), add:

```c
	c->label_enabled = config->label_enabled;
	c->label_edge = config->label_edge;
	c->label_align = config->label_align;
	c->label_max_width = config->label_max_width;
	c->label_max_width_percent = config->label_max_width_percent;
	c->label_max_width_is_percent = config->label_max_width_is_percent;
	c->label_corner_radius = config->label_corner_radius;
	c->label_corner_radius_match_window = config->label_corner_radius_match_window;
	c->label_autohide_ms = config->label_autohide_ms;
	c->label_avoid_cursor = config->label_avoid_cursor;
	c->label_state.animation = init_animation(c);
	c->label_state.from_alpha = 1.0f;
	c->label_state.to_alpha = 1.0f;
	c->label_state.slide_animation = init_animation(c);
```

- [ ] **Step 4: Build check**

Run: `ninja -C build`
Expected: builds successfully (new fields are unused so far — no functional change).

- [ ] **Step 5: Commit**

```bash
git add include/sway/tree/container.h include/sway/config.h sway/tree/container.c sway/config.c
git commit -m "feat: add label title data model (container + config fields)"
```

---

### Task 2: Content box no longer reserves space for an enabled label

**Files:**
- Modify: `sway/tree/view.c:443-458` (`view_autoconfigure()`, `B_NORMAL` case)

**Interfaces:**
- Consumes: `con->label_enabled` from Task 1.
- Produces: no new symbols — behavior change only.

- [ ] **Step 1: Skip the titlebar-height content-box reduction when the label is active**

In `sway/tree/view.c`, the `B_NORMAL` case of `view_autoconfigure()` currently reads (lines 443-458):

```c
	case B_NORMAL:
		// Height is: 1px border + 3px pad + title height + 3px pad + 1px border
		x = con->pending.x + con->pending.border_thickness * con->pending.border_left;
		width = con->pending.width
			- con->pending.border_thickness * con->pending.border_left
			- con->pending.border_thickness * con->pending.border_right;
		if (y_offset) {
			y = con->pending.y + y_offset;
			height = con->pending.height - y_offset
				- con->pending.border_thickness * con->pending.border_bottom;
		} else {
			y = con->pending.y + container_titlebar_height();
			height = con->pending.height - container_titlebar_height()
				- con->pending.border_thickness * con->pending.border_bottom;
		}
		break;
```

Change the `else` branch (the non-tabbed/stacked, "this container's own B_NORMAL titlebar" branch — `y_offset` is only nonzero inside a tabbed/stacked parent, which is out of scope per Global Constraints) so it skips the titlebar reservation when `con->label_enabled`:

```c
	case B_NORMAL:
		// Height is: 1px border + 3px pad + title height + 3px pad + 1px border
		x = con->pending.x + con->pending.border_thickness * con->pending.border_left;
		width = con->pending.width
			- con->pending.border_thickness * con->pending.border_left
			- con->pending.border_thickness * con->pending.border_right;
		if (y_offset) {
			y = con->pending.y + y_offset;
			height = con->pending.height - y_offset
				- con->pending.border_thickness * con->pending.border_bottom;
		} else if (con->label_enabled) {
			y = con->pending.y;
			height = con->pending.height
				- con->pending.border_thickness * con->pending.border_bottom;
		} else {
			y = con->pending.y + container_titlebar_height();
			height = con->pending.height - container_titlebar_height()
				- con->pending.border_thickness * con->pending.border_bottom;
		}
		break;
```

- [ ] **Step 2: Build check**

Run: `ninja -C build`
Expected: builds successfully.

- [ ] **Step 3: Manual check (will fully verify once Task 3 lands — note expected result for now)**

This step alone doesn't produce a visible change yet, since nothing sets `label_enabled` to `true` at runtime until Task 5's command exists, and nothing repositions the titlebar scene node yet (Task 3). Skip live verification here; it is covered together with Task 3's manual check.

- [ ] **Step 4: Commit**

```bash
git add sway/tree/view.c
git commit -m "feat: don't reserve titlebar height in content box when label is enabled"
```

---

### Task 3: Scene-graph positioning — `arrange_label()` and z-order

**Files:**
- Modify: `sway/desktop/transaction.c` (`_arrange_container()`, lines 530-696, and add a new static `arrange_label()` near the existing `arrange_title_bar()` at line 372)

**Interfaces:**
- Consumes: `con->label_enabled`, `label_edge`, `label_align`, `label_max_width(_percent/_is_percent)` from Task 1; `container_titlebar_height()`, `container_arrange_title_bar()` (existing, `sway/tree/container.c`).
- Produces: `static void arrange_label(struct sway_container *con, int container_width, int container_height);` — used only within `transaction.c` in this task; Task 8 (avoid-cursor) reads the resulting `con->title_bar.tree` position via `wlr_scene_node_coords()`, and Task 4 relies on `con->label_enabled` being what drives this function so its corner-radius branch stays in sync.

- [ ] **Step 1: Add `arrange_label()`**

In `sway/desktop/transaction.c`, right after the existing `arrange_title_bar()` function (ends at line 386), add:

```c
static int resolve_label_max_width(struct sway_container *con, int container_width) {
	int max_width = con->label_max_width_is_percent
		? (int)(container_width * (con->label_max_width_percent / 100.0f))
		: con->label_max_width;
	if (max_width <= 0) {
		max_width = container_width;
	}
	return MIN(max_width, container_width);
}

static void arrange_label(struct sway_container *con,
		int container_width, int container_height) {
	container_update(con);

	int max_width = resolve_label_max_width(con, container_width);
	int height = container_titlebar_height();
	int width = MIN(con->title_width > 0 ? con->title_width : max_width, max_width);
	width = MAX(width, 0);

	if (width <= 0 || height <= 0) {
		wlr_scene_node_set_enabled(&con->title_bar.tree->node, false);
		return;
	}

	int x;
	switch (con->label_align) {
	case LABEL_ALIGN_LEFT:
		x = 0;
		break;
	case LABEL_ALIGN_RIGHT:
		x = container_width - width;
		break;
	case LABEL_ALIGN_CENTER:
	default:
		x = (container_width - width) / 2;
		break;
	}

	int y = con->label_edge == LABEL_EDGE_BOTTOM
		? container_height - height
		: 0;

	// Keep the label above the content it overlaps.
	wlr_scene_node_raise_to_top(&con->title_bar.tree->node);

	wlr_scene_node_set_enabled(&con->title_bar.tree->node, true);
	wlr_scene_node_set_position(&con->title_bar.tree->node,
			x + (int)con->label_state.slide_x, y + (int)con->label_state.slide_y);

	con->title_width = width;
	container_arrange_title_bar(con);
}
```

`MIN`/`MAX` are already available in this file (used elsewhere, e.g. line 619's `MAX`).

- [ ] **Step 2: Call `arrange_label()` from `_arrange_container()` and suppress the reserved strip**

In `sway/desktop/transaction.c`, in `_arrange_container()`, the current logic (lines 583-598) is:

```c
		if (title_bar && con->current.border != B_NORMAL) {
			wlr_scene_node_set_enabled(&con->title_bar.tree->node, false);
			wlr_scene_node_set_enabled(&con->border.top->node, true);
		} else {
			wlr_scene_node_set_enabled(&con->border.top->node, false);
		}

		if (con->current.border == B_NORMAL) {
			vert_border_offset = 0;
			if (title_bar) {
				arrange_title_bar(con, 0, 0, width, border_top);
			} else {
				border_top = 0;
				// should be handled by the parent container
			}
		} else if (con->current.border == B_PIXEL) {
```

Change it to route to `arrange_label()` whenever the label is enabled, regardless of border mode, and to stop the `B_NORMAL` branch from reserving `border_top` when the label is active:

```c
		bool label_active = con->label_enabled && title_bar;

		if (title_bar && con->current.border != B_NORMAL && !label_active) {
			wlr_scene_node_set_enabled(&con->title_bar.tree->node, false);
			wlr_scene_node_set_enabled(&con->border.top->node, true);
		} else if (!label_active) {
			wlr_scene_node_set_enabled(&con->border.top->node, false);
		}

		if (con->current.border == B_NORMAL) {
			vert_border_offset = 0;
			if (label_active) {
				border_top = 0;
			} else if (title_bar) {
				arrange_title_bar(con, 0, 0, width, border_top);
			} else {
				border_top = 0;
				// should be handled by the parent container
			}
		} else if (con->current.border == B_PIXEL) {
```

Then, immediately after the whole `if (con->current.border == B_NORMAL) { ... } else if (...) { ... } else { sway_assert(false, "unreachable"); }` chain ends (after line 613, before `int border_bottom = ...` at line 615), add:

```c
		if (label_active) {
			arrange_label(con, width, height);
		}
```

This runs after `border_top`/`border_width` are finalized for whichever border mode is active, so the label overlay is positioned using the container's actual (non-reserved) `width`/`height` regardless of `border none|pixel|normal|csd`.

- [ ] **Step 3: Build check**

Run: `ninja -C build`
Expected: builds successfully.

- [ ] **Step 4: Manual check**

Run a nested swayFx session (`WLR_BACKENDS=headless sway` or your usual dev-loop) with a test config:

```
label enable
label position top center
for_window [class=".*"] label enable
```

(This `label` command is created in Task 5 — if this task is executed before Task 5 exists, stub a temporary `container->label_enabled = true;` line in `container_create()` to test the geometry in isolation, then remove the stub before committing.)

Open a terminal window. Expected: the window's content fills the *entire* window (no reserved top strip), and a small pill-shaped title chip renders overlapping the top-center of the window content. Confirm it moves to top-left/top-right/bottom-center when you flip alignment/edge in the test config.

- [ ] **Step 5: Commit**

```bash
git add sway/desktop/transaction.c
git commit -m "feat: position label overlay via arrange_label() instead of reserved titlebar strip"
```

---

### Task 4: Corner radius — round all 4 label corners

**Files:**
- Modify: `sway/tree/container.c:322-352` (`get_titlebar_corners()`)

**Interfaces:**
- Consumes: `con->label_enabled`, `label_corner_radius`, `label_corner_radius_match_window`, `con->corner_radius` (existing).
- Produces: no new symbols — `get_titlebar_corners()` keeps its existing signature and caller (`container_arrange_title_bar()`, line 456).

- [ ] **Step 1: Branch on `label_enabled` before the tabbed/stacked corner-cut logic**

Current code (lines 322-352):

```c
static struct fx_corner_radii get_titlebar_corners(struct sway_container *con) {
	int radius = container_has_corner_radius(con) ? con->corner_radius +
		con->current.border_thickness - config->titlebar_border_thickness : 0;
	struct fx_corner_radii corners = corner_radii_top(radius);

	enum sway_container_layout layout;
	const list_t *siblings;
	if (con->current.parent) {
		layout = con->current.parent->current.layout;
		siblings = con->current.parent->current.children;
	} else if (con->current.workspace) {
		layout = con->current.workspace->layout;
		siblings = con->current.workspace->tiling;
	} else {
		return corners;
	}

	if (layout == L_TABBED && siblings->length > 1) {
		if (siblings->items[0] == con) {
			corners.top_right = 0;
		} else if (siblings->items[siblings->length - 1] == con) {
			corners.top_left = 0;
		} else {
			return corner_radii_none();
		}
	} else if (layout == L_STACKED && siblings->items[0] != con) {
		return corner_radii_none();
	}

	return corners;
}
```

Change to:

```c
static struct fx_corner_radii get_titlebar_corners(struct sway_container *con) {
	if (con->label_enabled) {
		int radius = con->label_corner_radius_match_window
			? con->corner_radius
			: con->label_corner_radius;
		return corner_radii_all(MAX(radius, 0));
	}

	int radius = container_has_corner_radius(con) ? con->corner_radius +
		con->current.border_thickness - config->titlebar_border_thickness : 0;
	struct fx_corner_radii corners = corner_radii_top(radius);

	enum sway_container_layout layout;
	const list_t *siblings;
	if (con->current.parent) {
		layout = con->current.parent->current.layout;
		siblings = con->current.parent->current.children;
	} else if (con->current.workspace) {
		layout = con->current.workspace->layout;
		siblings = con->current.workspace->tiling;
	} else {
		return corners;
	}

	if (layout == L_TABBED && siblings->length > 1) {
		if (siblings->items[0] == con) {
			corners.top_right = 0;
		} else if (siblings->items[siblings->length - 1] == con) {
			corners.top_left = 0;
		} else {
			return corner_radii_none();
		}
	} else if (layout == L_STACKED && siblings->items[0] != con) {
		return corner_radii_none();
	}

	return corners;
}
```

`corner_radii_all()` is already used elsewhere in this codebase (`sway/desktop/transaction.c:557`), so no new include is needed.

- [ ] **Step 2: Build check**

Run: `ninja -C build`
Expected: builds successfully.

- [ ] **Step 3: Manual check**

With the same test setup as Task 3, set `label_corner_radius_match_window` off and a specific pixel value (via the Task 5 command once it exists: `label corner_radius 12`), and separately `label corner_radius match` with `corner_radius 20` set globally. Expected: label chip's corners are rounded on **all four** sides (not just top), and the `match` case visibly tracks the window's own `corner_radius` value when changed at runtime.

- [ ] **Step 4: Commit**

```bash
git add sway/tree/container.c
git commit -m "feat: round all four corners of the label overlay, with match-window option"
```

---

### Task 5: `label` config command — enable/disable, position, max_width

**Files:**
- Create: `sway/commands/label.c`
- Modify: `include/sway/commands.h` (add `sway_cmd cmd_label;` declaration, alphabetized)
- Modify: `sway/commands.c` (register in `command_handlers[]`, alphabetized between `"kill"` and `"layout"`, line 138-167)
- Modify: `sway/meson.build` (add `'commands/label.c',` to `sway_sources`, alphabetized within the `commands/*.c` section)

**Interfaces:**
- Consumes: `config->handler_context.container` (existing per-container command context, same as `sway/commands/border.c:65`); `arrange_container()` (existing, declared in `sway/tree/arrange.h`).
- Produces: `struct cmd_results *cmd_label(int argc, char **argv);` — the full `label` command, subcommands `enable|disable|toggle`, `position <top|bottom> <left|center|right>`, `max_width <NUM>|<NUM>%`.

- [ ] **Step 1: Write `sway/commands/label.c` — enable/disable/toggle and position**

```c
#include <stdlib.h>
#include <string.h>
#include "sway/commands.h"
#include "sway/config.h"
#include "sway/tree/arrange.h"
#include "sway/tree/container.h"

static struct cmd_results *label_cmd_enable(struct sway_container *container,
		int argc, char **argv) {
	if (strcmp(argv[0], "enable") == 0) {
		container->label_enabled = true;
	} else if (strcmp(argv[0], "disable") == 0) {
		container->label_enabled = false;
	} else if (strcmp(argv[0], "toggle") == 0) {
		container->label_enabled = !container->label_enabled;
	} else {
		return cmd_results_new(CMD_INVALID,
				"Expected 'label enable|disable|toggle'");
	}
	return NULL;
}

static struct cmd_results *label_cmd_position(struct sway_container *container,
		int argc, char **argv) {
	if (argc < 3) {
		return cmd_results_new(CMD_INVALID,
				"Expected 'label position <top|bottom> <left|center|right>'");
	}
	if (strcmp(argv[1], "top") == 0) {
		container->label_edge = LABEL_EDGE_TOP;
	} else if (strcmp(argv[1], "bottom") == 0) {
		container->label_edge = LABEL_EDGE_BOTTOM;
	} else {
		return cmd_results_new(CMD_INVALID,
				"Expected 'label position <top|bottom> <left|center|right>'");
	}
	if (strcmp(argv[2], "left") == 0) {
		container->label_align = LABEL_ALIGN_LEFT;
	} else if (strcmp(argv[2], "center") == 0) {
		container->label_align = LABEL_ALIGN_CENTER;
	} else if (strcmp(argv[2], "right") == 0) {
		container->label_align = LABEL_ALIGN_RIGHT;
	} else {
		return cmd_results_new(CMD_INVALID,
				"Expected 'label position <top|bottom> <left|center|right>'");
	}
	return NULL;
}

static bool parse_label_max_width(const char *arg, int *px, float *percent,
		bool *is_percent) {
	char *end;
	float value = strtof(arg, &end);
	if (end == arg || value < 0) {
		return false;
	}
	if (*end == '%' && *(end + 1) == '\0') {
		*is_percent = true;
		*percent = value;
		return true;
	}
	if (*end == '\0') {
		*is_percent = false;
		*px = (int)value;
		return true;
	}
	return false;
}

static struct cmd_results *label_cmd_max_width(struct sway_container *container,
		int argc, char **argv) {
	if (argc < 2) {
		return cmd_results_new(CMD_INVALID,
				"Expected 'label max_width <px>|<percent>%%'");
	}
	int px = 0;
	float percent = 0;
	bool is_percent = false;
	if (!parse_label_max_width(argv[1], &px, &percent, &is_percent)) {
		return cmd_results_new(CMD_INVALID,
				"Invalid max_width value '%s' (expected e.g. '200' or '50%%')",
				argv[1]);
	}
	container->label_max_width_is_percent = is_percent;
	if (is_percent) {
		container->label_max_width_percent = percent;
	} else {
		container->label_max_width = px;
	}
	return NULL;
}

static struct cmd_results *label_cmd_corner_radius(struct sway_container *container,
		int argc, char **argv) {
	if (argc < 2) {
		return cmd_results_new(CMD_INVALID,
				"Expected 'label corner_radius <px>|match'");
	}
	if (strcmp(argv[1], "match") == 0) {
		container->label_corner_radius_match_window = true;
		return NULL;
	}
	char *end;
	int value = strtol(argv[1], &end, 10);
	if (*end != '\0' || value < 0 || value > 99) {
		return cmd_results_new(CMD_INVALID,
				"Invalid corner_radius value '%s' (expected 0-99 or 'match')",
				argv[1]);
	}
	container->label_corner_radius_match_window = false;
	container->label_corner_radius = value;
	return NULL;
}

static struct cmd_results *label_cmd_autohide(struct sway_container *container,
		int argc, char **argv) {
	if (argc < 2) {
		return cmd_results_new(CMD_INVALID,
				"Expected 'label autohide off|<ms>'");
	}
	if (strcmp(argv[1], "off") == 0) {
		container->label_autohide_ms = 0;
		return NULL;
	}
	char *end;
	int value = strtol(argv[1], &end, 10);
	if (*end != '\0' || value < 0) {
		return cmd_results_new(CMD_INVALID,
				"Invalid autohide value '%s' (expected 'off' or a positive number of ms)",
				argv[1]);
	}
	container->label_autohide_ms = value;
	return NULL;
}

static struct cmd_results *label_cmd_avoid_cursor(struct sway_container *container,
		int argc, char **argv) {
	if (argc < 2) {
		return cmd_results_new(CMD_INVALID,
				"Expected 'label avoid_cursor on|off'");
	}
	if (strcmp(argv[1], "on") == 0) {
		container->label_avoid_cursor = true;
	} else if (strcmp(argv[1], "off") == 0) {
		container->label_avoid_cursor = false;
	} else {
		return cmd_results_new(CMD_INVALID,
				"Expected 'label avoid_cursor on|off'");
	}
	return NULL;
}

struct cmd_results *cmd_label(int argc, char **argv) {
	struct cmd_results *error = NULL;
	if ((error = checkarg(argc, "label", EXPECTED_AT_LEAST, 1))) {
		return error;
	}

	struct sway_container *container = config->handler_context.container;
	if (!container || !container->view) {
		return cmd_results_new(CMD_INVALID, "Only views can have a label");
	}

	if (strcmp(argv[0], "enable") == 0 || strcmp(argv[0], "disable") == 0
			|| strcmp(argv[0], "toggle") == 0) {
		error = label_cmd_enable(container, argc, argv);
	} else if (strcmp(argv[0], "position") == 0) {
		error = label_cmd_position(container, argc, argv);
	} else if (strcmp(argv[0], "max_width") == 0) {
		error = label_cmd_max_width(container, argc, argv);
	} else if (strcmp(argv[0], "corner_radius") == 0) {
		error = label_cmd_corner_radius(container, argc, argv);
	} else if (strcmp(argv[0], "autohide") == 0) {
		error = label_cmd_autohide(container, argc, argv);
	} else if (strcmp(argv[0], "avoid_cursor") == 0) {
		error = label_cmd_avoid_cursor(container, argc, argv);
	} else {
		return cmd_results_new(CMD_INVALID,
				"Expected 'label enable|disable|toggle|position|max_width|"
				"corner_radius|autohide|avoid_cursor ...'");
	}
	if (error) {
		return error;
	}

	arrange_container(container);

	return cmd_results_new(CMD_SUCCESS, NULL);
}
```

Note: `label_cmd_position`/`label_cmd_max_width`/etc. take the full `argv`/`argc` (not shifted past `argv[0]`) so their own arg-index checks (`argv[1]`, `argc < 2` etc.) line up with the subcommand name at `argv[0]` — consistent with how they're invoked in `cmd_label`.

- [ ] **Step 2: Register the command**

In `include/sway/commands.h`, add (alphabetized, near `sway_cmd cmd_kill;`):

```c
sway_cmd cmd_label;
```

In `sway/commands.c`, add to `command_handlers[]` (line 138), alphabetized between `"kill"` and `"layout"`:

```c
	{ "label", cmd_label },
```

In `sway/meson.build`, add `'commands/label.c',` to the `sway_sources = files(...)` list, alphabetized among the other `'commands/*.c'` entries (near `'commands/kill.c',`/`'commands/layout.c',` if present, otherwise in correct alphabetical position within that section).

- [ ] **Step 3: Build check**

Run: `ninja -C build`
Expected: builds successfully; `sway -C /path/to/test-config -V` (or equivalent config-check invocation) accepts a config containing `for_window [class=".*"] label enable`.

- [ ] **Step 4: Manual check**

```
swaymsg 'for_window [class=".*"] label enable'
```
then open a terminal. Expected: label appears (default: top-center, 50% max width, matching corner radius). Then:
```
swaymsg '[con_id=<id>] label position bottom right'
swaymsg '[con_id=<id>] label max_width 150'
swaymsg '[con_id=<id>] label corner_radius 4'
```
Expected: label moves to bottom-right, visibly caps at 150px (long window titles ellipsize/truncate via the existing `sway_text_node_set_max_width` clipping already wired through `container_arrange_title_bar()`), corner radius changes to a small fixed value.

- [ ] **Step 5: Commit**

```bash
git add sway/commands/label.c include/sway/commands.h sway/commands.c sway/meson.build
git commit -m "feat: add 'label' command (enable/position/max_width/corner_radius/autohide/avoid_cursor)"
```

---

### Task 6: Suppress full titlebar strip when `border normal` + `label enable` combine

**Files:**
- None — already implemented by Task 3's `label_active` guard (`_arrange_container()` only calls the reserved-strip `arrange_title_bar(con, 0, 0, width, border_top)` when `!label_active`, and Task 2 skips the content-box reservation when `con->label_enabled`).

**Interfaces:**
- Consumes: `label_active` from Task 3, `con->label_enabled` from Task 1/2.

- [ ] **Step 1: Manual verification only (no code — confirms Tasks 2+3's combined effect)**

```
swaymsg '[con_id=<id>] border normal'
swaymsg '[con_id=<id>] label enable'
```
Expected: no full-width titlebar strip is drawn; only the floating label chip appears, content fills the window. Then:
```
swaymsg '[con_id=<id>] label disable'
```
Expected: the normal full-width titlebar strip reappears (border was never changed, so disabling the label reverts to plain `border normal` behavior).

- [ ] **Step 2: If the manual check fails**

If a full-width titlebar strip is still visible with both enabled, re-check Task 3 Step 2 — the `label_active` guard must be evaluated with `con->current.border == B_NORMAL` (post-arrange state), not `con->pending.border`, since `_arrange_container()` operates on `con->current`. Fix in place, re-run `ninja -C build`, re-test, then proceed.

- [ ] **Step 3: Commit** (only if a fix was needed in Step 2)

```bash
git add sway/desktop/transaction.c
git commit -m "fix: ensure label suppresses reserved titlebar strip against con->current.border"
```

---

### Task 7: Focus-based autohide (fade on unfocus)

**Files:**
- Modify: `sway/tree/container.c` (`container_update()`, the focus-handling block around lines 300-311)

**Interfaces:**
- Consumes: `con->label_autohide_ms`, `con->label_state` from Task 1; `add_animation()`, `get_animated_value()`, `finish_animation()` from `include/sway/animation_manager.h` (existing); `container_update()` (existing, same file).
- Produces: `static int label_autohide_timeout(void *data);` (file-local callback) — not consumed elsewhere.

- [ ] **Step 1: Give the title bar its own alpha, decoupled from the container's alpha**

`container_update()` already fades title-bar rects, border rects, and shadow together through one shared `alpha` (lines 241-253, `scene_rect_set_color` calls at lines 275-277 and 280-283). Reusing that directly would fade the window border/shadow too whenever the label hides, which we don't want. Instead, compute a second `label_alpha` that only affects the title-bar-specific nodes.

In `sway/tree/container.c`, in `container_update()`, right after the existing `alpha` computation (ends at line 253, the `if (con->current.workspace) { alpha *= ...; }` block), add:

```c
	float label_alpha = alpha;
	if (con->label_enabled) {
		label_alpha *= MIN(1, MAX(0, get_animated_value(con->label_state.from_alpha,
				con->label_state.to_alpha, &con->label_state.animation)));
	}
```

Then change the three title-bar `scene_rect_set_color` calls (lines 275-277) to use `label_alpha` instead of `alpha`:

```c
	scene_rect_set_color(con->title_bar.background_left, colors->background, label_alpha);
	scene_rect_set_color(con->title_bar.background_right, colors->background, label_alpha);
	scene_rect_set_color(con->title_bar.border, colors->border, label_alpha);
```

(Leave the `con->border.*`/`con->shadow` calls at lines 280-287 using the original `alpha` — those are the window border/shadow, not the label.)

Finally, change the title/marks text blocks (lines 290-298) to premultiply `label_alpha` into their color's alpha channel, since `sway_text_node_set_color`/`set_background` take a raw `color[4]` with no separate opacity argument:

```c
	if (con->title_bar.title_text) {
		float text_color[4] = { colors->text[0], colors->text[1],
			colors->text[2], colors->text[3] * label_alpha };
		float bg_color[4] = { colors->background[0], colors->background[1],
			colors->background[2], colors->background[3] * label_alpha };
		sway_text_node_set_color(con->title_bar.title_text, text_color);
		sway_text_node_set_background(con->title_bar.title_text, bg_color);
	}

	if (con->title_bar.marks_text) {
		float text_color[4] = { colors->text[0], colors->text[1],
			colors->text[2], colors->text[3] * label_alpha };
		float bg_color[4] = { colors->background[0], colors->background[1],
			colors->background[2], colors->background[3] * label_alpha };
		sway_text_node_set_color(con->title_bar.marks_text, text_color);
		sway_text_node_set_background(con->title_bar.marks_text, bg_color);
	}
```

This replaces the original unconditional `sway_text_node_set_color(con->title_bar.title_text, colors->text);` /
`sway_text_node_set_background(con->title_bar.title_text, colors->background);` pair (and the equivalent `marks_text` pair). When `con->label_enabled` is false, `label_alpha == alpha`, so non-label titlebars render exactly as before — no regression.

- [ ] **Step 2: Add the timer callback and fade animation wiring**

In `sway/tree/container.c`, add near the top of the file (after existing static helpers, before `container_update()`):

```c
static void label_fade_update(void *data) {
	struct sway_container *con = data;
	container_update(con);
}

static void label_fade_complete(void *data) {
	struct sway_container *con = data;
	con->label_state.hidden = con->label_state.to_alpha == 0.0f;
	container_update(con);
}

static int label_autohide_timeout(void *data) {
	struct sway_container *con = data;
	con->label_state.from_alpha = 1.0f;
	con->label_state.to_alpha = 0.0f;
	add_animation(&con->label_state.animation, label_fade_update, label_fade_complete);
	return 0;
}
```

- [ ] **Step 3: Arm/cancel the timer on the focus-transition edge**

In `container_update()`, locate the focus block (the code you already have open at lines ~300-311: `bool focused = con->current.focused || container_is_current_parent_focused(con);`). Immediately after that `focused` computation, add:

```c
	if (con->label_enabled && con->label_autohide_ms > 0) {
		if (con->label_state.was_focused && !focused) {
			if (!con->label_state.autohide_timer) {
				con->label_state.autohide_timer = wl_event_loop_add_timer(
						server.wl_event_loop, label_autohide_timeout, con);
			}
			if (con->label_state.autohide_timer) {
				wl_event_source_timer_update(con->label_state.autohide_timer,
						con->label_autohide_ms);
			}
		} else if (focused && con->label_state.autohide_timer) {
			wl_event_source_timer_update(con->label_state.autohide_timer, 0);
			con->label_state.from_alpha = con->label_state.to_alpha;
			con->label_state.to_alpha = 1.0f;
			add_animation(&con->label_state.animation, label_fade_update, label_fade_complete);
		}
	}
	con->label_state.was_focused = focused;
```

This needs `#include "sway/server.h"` (for the `server` global and `wl_event_loop`) in `sway/tree/container.c` if not already present — check the existing include block first; `sway/tree/container.c` likely already pulls in enough transitively via `sway/desktop/transaction.h`, but add the explicit include if the build fails on `server`/`wl_event_loop_add_timer` being undeclared.

- [ ] **Step 4: Clean up the timer on container destroy**

In `container_destroy()` (`sway/tree/container.c`, further down in the file), add before the container is freed:

```c
	if (con->label_state.autohide_timer) {
		wl_event_source_remove(con->label_state.autohide_timer);
	}
```

- [ ] **Step 5: Build check**

Run: `ninja -C build`
Expected: builds successfully. If `server`/`wl_event_loop_add_timer` are undeclared, add `#include "sway/server.h"` to `sway/tree/container.c` and rebuild.

- [ ] **Step 6: Manual check**

```
swaymsg '[con_id=<id>] label enable'
swaymsg '[con_id=<id>] label autohide 1000'
```
Focus the window, then click another window to unfocus it. Expected: label fades out over the animation duration starting ~1 second after losing focus. Refocus the original window before the timeout fires. Expected: label stays visible (timer was cancelled/restored) and does not fade.

- [ ] **Step 7: Commit**

```bash
git add sway/tree/container.c
git commit -m "feat: fade label out after configurable delay on focus loss"
```

---

### Task 8: Avoid-cursor slide

**Files:**
- Modify: `sway/input/seatop_default.c` (`handle_pointer_motion()`, currently lines 602-627)

**Interfaces:**
- Consumes: `con->label_enabled`, `label_avoid_cursor`, `label_state.slide_x/y` (read by `arrange_label()` in Task 3) from Task 1; `root_for_each_container()` (existing, `sway/tree/root.h`) to iterate visible containers; `add_animation()`/`get_animated_value()` (existing).
- Produces: no new public symbols — internal motion-handling behavior only.

- [ ] **Step 1: Add the proximity check and slide animation**

In `sway/input/seatop_default.c`, add near the top of the file (file-local static helpers):

```c
static void label_slide_update(void *data) {
	struct sway_container *con = data;
	con->label_state.slide_x = get_animated_value(con->label_state.slide_from_x,
			con->label_state.slide_to_x, &con->label_state.slide_animation);
	con->label_state.slide_y = get_animated_value(con->label_state.slide_from_y,
			con->label_state.slide_to_y, &con->label_state.slide_animation);
	arrange_container(con);
}

static void label_slide_complete(void *data) {
	// no-op: slide_x/y already hold their final value via label_slide_update
}

static void check_label_avoid_cursor(struct sway_container *con,
		void *data) {
	double *cursor = data;
	double cx = cursor[0], cy = cursor[1];

	if (!con->label_enabled || !con->label_avoid_cursor
			|| !con->title_bar.tree->node.enabled) {
		return;
	}

	double lx, ly;
	wlr_scene_node_coords(&con->title_bar.tree->node, &lx, &ly);
	int lwidth = con->title_width;
	int lheight = container_titlebar_height();

	bool hovering = cx >= lx && cx < lx + lwidth && cy >= ly && cy < ly + lheight;

	double target_x = 0, target_y = 0;
	if (hovering) {
		// Slide the label fully out of its own height, in the direction
		// away from its resting edge, so it clears the cursor.
		target_y = con->label_edge == LABEL_EDGE_BOTTOM ? lheight : -lheight;
	}

	if (con->label_state.slide_to_y != target_y) {
		con->label_state.slide_from_x = con->label_state.slide_x;
		con->label_state.slide_from_y = con->label_state.slide_y;
		con->label_state.slide_to_x = target_x;
		con->label_state.slide_to_y = target_y;
		add_animation(&con->label_state.slide_animation,
				label_slide_update, label_slide_complete);
	}
}
```

This needs `#include "sway/tree/root.h"` and `#include "sway/tree/arrange.h"` in `sway/input/seatop_default.c` if not already present.

- [ ] **Step 2: Call it from `handle_pointer_motion()`**

Current end of `handle_pointer_motion()` (lines 602-627):

```c
static void handle_pointer_motion(struct sway_seat *seat, uint32_t time_msec) {
	struct seatop_default_event *e = seat->seatop_data;
	struct sway_cursor *cursor = seat->cursor;

	struct wlr_surface *surface = NULL;
	double sx, sy;
	struct sway_node *node = node_at_coords(seat,
			cursor->cursor->x, cursor->cursor->y, &surface, &sx, &sy);

	if (config->focus_follows_mouse != FOLLOWS_NO) {
		check_focus_follows_mouse(seat, e, node);
	}

	if (surface) {
		if (seat_is_input_allowed(seat, surface)) {
			wlr_seat_pointer_notify_enter(seat->wlr_seat, surface, sx, sy);
			wlr_seat_pointer_notify_motion(seat->wlr_seat, time_msec, sx, sy);
		}
	} else {
		cursor_update_image(cursor, node);
		wlr_seat_pointer_notify_clear_focus(seat->wlr_seat);
	}

	drag_icons_update_position(seat);
	e->previous_node = node;
}
```

Add the avoid-cursor sweep right after `drag_icons_update_position(seat);`:

```c
	double cursor_pos[2] = { cursor->cursor->x, cursor->cursor->y };
	root_for_each_container(check_label_avoid_cursor, cursor_pos);

	drag_icons_update_position(seat);
	e->previous_node = node;
```

(Reordered slightly so the sweep and `drag_icons_update_position` both run unconditionally each motion event; exact ordering between them doesn't matter since they touch disjoint state.)

- [ ] **Step 3: Build check**

Run: `ninja -C build`
Expected: builds successfully.

- [ ] **Step 4: Manual check**

```
swaymsg '[con_id=<id>] label enable'
swaymsg '[con_id=<id>] label avoid_cursor on'
swaymsg '[con_id=<id>] label position top center'
```
Move the mouse cursor slowly onto the label chip. Expected: label slides away (up, out of the way) as the cursor approaches/enters its bounds, then slides back once the cursor moves off it.

- [ ] **Step 5: Commit**

```bash
git add sway/input/seatop_default.c
git commit -m "feat: slide label away from cursor when avoid_cursor is enabled"
```

---

### Task 9: Full manual regression pass

**Files:** none (verification only).

- [ ] **Step 1: Run the full manual matrix from the spec's Testing section**

Using a nested dev session, walk through every row and confirm against `docs/superpowers/specs/2026-09-10-label-title-design.md`'s Testing section:

1. Label renders at all 6 edge/alignment combinations (`top`/`bottom` × `left`/`center`/`right`).
2. `max_width` caps correctly in both px and percent form; a title longer than the cap is truncated/ellipsized, never wider than the window.
3. `corner_radius <px>` vs `corner_radius match` (including live-follow when the container's own `corner_radius` changes at runtime via `swaymsg`).
4. Content box is full window size (no reserved strip) with label enabled, across `border none`, `border pixel 2`, `border normal`, `border csd` (where the view supports CSD).
5. `border normal` + `label enable` together show only the label, never both.
6. Drag-to-move, double-click-to-maximize/restore, and right-click behave the same on the label chip as they do on a normal titlebar; clicks outside the label's bounds pass through to the window content underneath.
7. `label autohide <ms>` fades out after unfocus and restores on refocus, cancelling a pending fade.
8. `label avoid_cursor on` slides the label away as the cursor approaches and back when it leaves; verify it doesn't fight with an in-flight autohide fade (trigger both near-simultaneously).
9. With `label` left at its default (disabled), open several windows with a mix of `border normal`/`pixel`/`none`/`csd` and confirm titlebar/content geometry is pixel-identical to `master` before this branch (no regression for the unused-feature path).

- [ ] **Step 2: Record results**

If any row fails, return to the corresponding task above, fix in place, rebuild, and re-run only the failing row plus row 9 (the no-regression check) before moving on.

- [ ] **Step 3: Final commit** (only if fixes were made in Step 2)

```bash
git add -A
git commit -m "fix: address label overlay regressions found in manual verification pass"
```
