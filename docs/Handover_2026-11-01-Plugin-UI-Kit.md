# Handover 2026-11-01 — Plugin UI kit (plugin API v1, compatible additions)

## Why

Plugins can't look like SlideMage today. The plugin API exposes only `Page`, `Field`, `caption`, `Surface`, `Button`, `Chip`, `IconButton`, `IconLabel`, `style_chat_widget` and `theme_color("info")`. A plugin with its own window therefore falls back to stock Qt (`QTabWidget`, `QListWidget`, plain `QLabel`s). That window looks foreign, especially in dark mode.

Exam Forge (`JuergenFleiss/slidemage-plugin-exam-forge`) prompted this. Its redesign prototype is at `docs/redesign/prototype.dc.html` on branch `docs/ui-redesign-handoff`. The prototype uses cards, segmented switches, pill chips, count badges, list rows with two lines of text, modal sheets and a toast. A first PySide6 build of it looked plain and clipped text. Nearly everything it needs already exists in `src/slidemage/ui/kit` (`Card`, `SegmentedControl`, `Chip` with `icon`/`elide`, `ElidedLabel`, `RowButton`, `Toast`, `ModalSheet`, `Tokens`, `apply_shadow`). It just isn't reachable from `plugin_api`.

**Goal:** expose the kit to plugins through **compatible additions to API v1**, so that a plugin window built only from host widgets looks like SlideMage in light and dark, and follows a live theme switch.

## Base branch and versions — read first

- Exam Forge needs `SlideRef`, `Host.deck_slides`, `Host.slide_source` and `Host.ai_chat`. These come from **PR SlideMage/SlideMage#63** (`JuergenFleiss:feature/ai-chat` → `develop`), which is **still open**.
- This work **stacks on #63**: the branch `docs/plugin-ui-kit-handoff` starts from `feature/ai-chat`.
  - While #63 is open, keep the UI kit in its own commits on top of it.
  - Open its PR only after #63 is merged, against `develop`. Alternatively, open it as a stacked PR into `feature/ai-chat`, if that is preferred.
  - Don't mix the UI-kit changes into #63.
- **Version conflict:** #63 calls itself v0.13.13, but upstream `develop` is already at v0.13.16, and its v0.13.13 is a different release without these APIs.
  - Before #63 merges, renumber it to the next free version (≥ v0.13.17) and move its CHANGELOG entry to the end.
  - This work then takes the version after that.
- **Consequence for Exam Forge:** its `requires_slidemage = ">=0.13.13"` is wrong upstream, because 0.13.13–0.13.16 lack `ai_chat` and `deck_slides`. Raise it to #63's final version now, and to this work's version once the restyle uses the new calls.

## Rules

- `plugin_api.py` stays standard library only. Widgets stay typed `Any`, and the docstrings in the `Host` protocol are the documentation, in the existing style.
- Additions only: `API_VERSION` stays 1. New keyword arguments get defaults, and new dataclass fields get defaults. Regenerate the contract snapshot with `python tests/test_plugin_api_contract.py --update` and review the diff: it must contain only additions.
- Implement in `plugins/host.py` by delegating to `ui.kit`, the way `Button` and `Chip` do now, and through `WindowHostUi` (`ui/plugin_ui.py`) where the window is needed. Return the **kit objects themselves**, so the app-wide QSS (`ThemeManager.apply` → `QApplication.setStyleSheet`) styles them.
- A plugin callback that raises is logged and reported like other plugin callbacks, and never reaches the event loop.
- Follow CHANGELOG.md's rules: bump `VERSION_NUMBER` and add an entry in the same commit. Mention the additions in the Plugins section of `docs/SlideMage.md`.

## A. Theme

1. **`ThemeTokens`**, a new frozen dataclass in `plugin_api`, standard library only.
   - It is a curated subset of `ui.kit.tokens.Tokens` and does not expose `Tokens` itself, so internal tokens can still change.
   - Fields: `dark: bool`, `bg`, `surface`, `text`, `text_2`, `text_3`, `text_4`, `line`, `hover`, `inverse`, `on_inverse`, `focus`, `on_solid`, `ok`/`ok_bg`/`ok_fg`, `warning`/`warning_bg`/`warning_fg`, `error`/`error_bg`/`error_fg`, `info`/`info_bg`/`info_fg` (all hex strings), `radius_sm`, `radius_md` (ints), `font_sans`, `font_mono` (family names).
   - Docstring: later versions may add fields, so read the fields you need by name.
2. **`Host.theme() -> ThemeTokens`**: the current theme, built from `window.theme.tokens`. Before a window exists, fall back to `tokens_for(<the saved dark_mode>)`.
3. **`Host.on_theme_changed(callback: Callable[[ThemeTokens], None]) -> Callable[[], None]`**:
   - Called on the UI thread **after** `ThemeManager.apply()` has applied the new QSS.
   - Returns an unsubscribe function, and listeners are dropped when the plugin unloads.
   - `ThemeManager` needs a listener hook, a small callback list or a Qt signal, fired from `set_dark` after `apply`.
4. `theme_color("info")` stays, implemented as `theme().info`.

## B. Widgets

All of these are thin wrappers over existing kit classes unless marked *new*.

| Host call | Backed by | Notes |
|---|---|---|
| `Window(title: str) -> Widget` | *new*, small | A top-level, non-modal plugin window: the page background, SlideMage's font, a kit object name and the app's window icon, with `window.body` (QVBoxLayout) to fill. **Check the font:** the base QSS rule `QMainWindow, QWidget {…}` sets only background and colour, and a parentless Exam Forge window rendered in the system font (Ubuntu) instead of Roboto. Set the font on the window explicitly, or add a base `font-family` rule. |
| `Card(raised: bool = False) -> Widget` | `kit.Card` | Fill it through `card.layout()`; `raised` adds the kit shadow. |
| `SegmentedControl(items: list[tuple[str, str]], current: str \| None = None) -> Widget` | `kit.SegmentedControl` (`roving=True`) | Signal `changed(str)` and `set_current(key)`. Keep `tones`/`mono`/`icons` internal for now. |
| `Chip(text, tone="neutral", *, icon=None, elide=False, pressable=False)` | `kit.Chip` | Keyword-only additions, which the kit already supports. Document the tones: `neutral`, `muted`, `info`, `warning`, `error`, `ok`. |
| `Badge(text: str, tone: str = "neutral") -> Widget` | *new* kit widget, unless `SEGMENT_BADGE` styling can be reused | A small round count, 20 px high with a minimum width of 20. `neutral` uses `inverse`/`on_inverse`; also `info` and `error`. Exam Forge uses it for its "needs attention" counts. |
| `ElidedLabel(text: str) -> Widget` | `kit.ElidedLabel` | Elides with "…" and shows the full text as its tooltip. |
| `RowButton(title, sub="", *, icon=None, chip=("", ""), selectable=False)` | `kit.RowButton` | A clickable two-line row. Add a `selected` property styled by kit QSS for list selection. Row height must come from font metrics, never a fixed height: Exam Forge's first build cut off descenders ("No questions vet"). |
| `Label(text: str, role: str = "body") -> Widget` | kit object names | Text roles `title`, `heading`, `body`, `secondary`, `caption`, `mono`, mapped to existing kit text styles (sheet title, panel caption, …), so plugins get SlideMage's type scale without hard-coding sizes. |
| `ModalSheet(title: str, parent) -> Widget` | `kit.ModalSheet` | `add_tab` and `add_button(text, kind, role)` as in the kit. Exam Forge's AI drafting, "Review changes" and topic dialogs would use it. |
| `toast(parent, text: str) -> None` | `kit.Toast` | Shows a transient message pinned to the bottom of `parent` (a plugin window). Unlike `show_message`, it doesn't go to the main window's status line. |

Each widget keeps its kit `objectName`, so the global QSS reaches it. Nothing gets a local stylesheet.

## C. Slide data (`SlideRef`: new fields with defaults)

1. **`title_text: str = ""`**: the frame title as plain display text.
   - LaTeX markup removed: `\\` → space, `\&` → `&`, ``` ``…'' ``` → “…”, `\ ` → space, `{}` groups unwrapped, `\emph{x}` → `x`.
   - `title` stays the raw frame title.
   - Exam Forge shows `VO Grundlagen \\ Digitalisierung` and ``` ``Big Data'' vs.\ ``AI'' ``` today.
   - Reuse whatever the outline or sidebar already uses to display titles. If there is no such helper, add one next to `deck_context.frame_title` and test it with these cases.
2. **`section: str = ""`**: the title of the `\section` the slide call stands under (as in `deck_outline.SectionRow`), or `""` before the first section. Exam Forge will offer "create topics from the deck's sections".

## Tests

- In `test_plugin_host.py` / `test_plugin_ui.py` (offscreen):
  - Every new host widget returns the kit class with its object name.
  - The `Chip` keyword arguments work.
  - `SegmentedControl` emits `changed` on a click but not on `set_current`.
- `theme()` matches `tokens_for(dark)` for both themes.
- `on_theme_changed`:
  - Fires once per `set_dark`/`toggle`, with the new tokens.
  - After unsubscribing, it doesn't fire.
  - A raising listener is logged, and the other listeners still run.
- `SlideRef.title_text`: a table of the escape cases above. `section` is filled correctly before, within and after sections.
- The contract snapshot is updated and contains only additions. `test_plugin_boundary` still passes.
- Optional: add the newly exposed widgets to `ui/kit/gallery.py`, so light and dark can be checked by eye.

## Acceptance

A throwaway test plugin (not committed) builds a window **only** from host widgets: a header with a `SegmentedControl` and two `Button`s, a list of `RowButton`s with `Badge`s, `Card`s with `Chip`s, and a `ModalSheet`. It looks like SlideMage's own panels, has no clipped text at 100% and 125% scaling, and re-themes live when the theme is toggled.

## Out of scope

- The Exam Forge restyle itself. It follows in its own repo once this ships, using these calls.
- `API_VERSION` 2, and exposing raw `Tokens`, `PopoverMenu`, `SplitButton` or `ThumbnailGrid`. Add those later if a plugin needs them.

## Open questions

1. Which version #63 ships under after its renumbering, and whether this work comes as a stacked PR or after #63 merges.
2. Whether `Window` should be owned by the host, so it closes on a project switch and is restored at startup, or stay a plain QWidget the plugin manages.
3. The `Badge` tones: is neutral/info/error enough?
