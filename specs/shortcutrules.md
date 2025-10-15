# Shortcut Parsing Rules

macLLM supports two user-facing constructs:

- Shortcuts: simple text expansions that start with the `/` symbol.
- Tags: plugin-driven `@...` expressions that can add context (files, clipboard, images, URLs, etc.).

Processing order: shortcuts expand first, then tags are processed by plugins.

## Shortcut Syntax (`/`)

- Shortcuts are declared in TOML as two-element arrays: `["/trigger", "expansion text"]`.
- They are read from:
  1. The application-supplied TOML file `config/default_shortcuts.toml`
  2. Any `*.toml` file found in the user directory `~/.config/macllm/`
- At runtime, occurrences of the exact `/trigger` are replaced with the configured expansion before tag processing.
- Example:
  - In TOML: `["/blog", "Expand the following into paragraphs...\n---\n"]`
  - In input: `Please /blog this:` → the `/blog` token is replaced with the configured text.

Configuration tags in TOML:

- Any entry whose trigger starts with `@` is treated as a configuration tag for plugins (not a user shortcut).
- Example: `[@IndexFiles, "/some/path"]` is consumed by the file plugin to build an index for path tags.

## Tag Syntax (`@`)

Tags are parsed in the user’s input after shortcuts expand. The parsing rules are:

1. A tag runs until the first whitespace character: `@clipboard some text` → tag is `@clipboard`.
2. Backslash-escaped spaces are included: `@/path/with\ spaces/file.txt` → tag is `@/path/with spaces/file.txt`.
3. Quoted tags include everything until the closing quote or newline (quotes are stripped):
   - `@"~/My Home/foo"` → tag is `@~/My Home/foo`

Tags are handled by plugins that implement `get_prefixes()` and `expand(...)`, potentially adding context to the conversation.

## Autocomplete and Highlighting (shared)

- The editor provides the same autocomplete popup and inline “pill” highlighting for both `/` shortcuts and `@` tags.
- Typing starts with `/` lists configured shortcuts; typing `@` lists tag prefixes and dynamic suggestions from plugins.
- Enter inserts the selected suggestion as a pill; Tab inserts the raw text and keeps it editable (quoted forms `@"..."` and `/"..."` are supported).
- Pills for shortcuts show plain text (no icon). Tag pills may show an icon as provided by the plugin’s `display_string`.
- The UI applies a short minimum-length filter so suggestions appear after a few characters.

## Plugins and Tags

Tags live in the `macllm/tags/` directory and inherit from the base class `TagPlugin` (`macllm.tags.base.TagPlugin`).

- Each plugin should implement:
  - `get_prefixes() -> list[str]` — return all `@` prefixes the plugin should react to
  - `expand(tag: str, conversation, request) -> str` — return the replacement string to insert into the prompt
- Plugins may optionally implement dynamic autocomplete and display mapping.
- Some plugins can also expose configuration tags for use in TOML via `get_config_prefixes()` and `on_config_tag(...)`.

## Current Plugins (examples)

- ClipboardTag (`@clipboard`) — Inserts clipboard text as context.
- FileTag (path-like tags: `@/`, `@~`, `@"/`, `@"~`) — Reads file contents (up to 10 KB) as context; config tag `@IndexFiles` builds an index.
- URLTag (`@http://`, `@https://`) — Downloads and strips web page content as context.
- ImageTag (`@selection`, `@window`) — Captures screenshots for image analysis.
- SpeedTag (e.g. `@fast`, `@normal`, `@slow`) — Adjusts processing speed/sticky mode.

## Processing Order Summary

1. Expand all `/...` shortcuts using the configured TOML mappings.
2. Process all `@...` tags using the loaded tag plugins, which may add context and replace tags in-place. 