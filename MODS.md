# Wingram Mods — the `<modname>.strc` format

Mods are plain-text `.strc` files (INI-style sections, `key = value` pairs,
`#` comments) that live in the `mods/` folder next to `wingram.exe`.
Wingram scans that folder at startup and whenever you press **reload mods**
in Settings. Toggle any mod with the switch — enabled mods apply immediately
and the choice persists in `config.ini`.

## File anatomy

```ini
# comments start with # and blank lines are ignored

[m[m[m[m[m[m[m[mod]                       # required: identity
id = darker-dark            # unique id (used for save state) — required
name = Darker Dark          # shown in the Mods list
version = 1.0.0
author = you
description = Deepens the dark theme and adds a violet accent.

[theme]                     # optional: palette overrides (0xRRGGBB ints)
bg = 0x050507
surface = 0x0B0B0F
surface2 = 0x14141B
border = 0x1C1C24
text = 0xEDEDF2
text_dim = 0x9A9AB0
text_faint = 0x5A5A6B
accent = 0x8B5CF6
accent_soft = 0x1C1633
like = 0xFF3040
badge = 0xFF3040
online = 0x22C55E
note_self = 0x241B3D
note_other = 0x262626
sticker_a = 0x7C3AED        # music sticker gradient start
sticker_b = 0xDB2777        # music sticker gradient end

[features]                  # optional: feature flags (strings)
feed_density = compact      # any value; wingram reads known ones
note_songs = on
round_avatars = on
```

## Rules

- **One mod per file.** The file name becomes the chip shown in Settings
  (`darker_dark.strc` → `darker_dark.strc`), but the identity is the
  `id` under `[m[m[m[m[m[m[m[mod]` — that is what the enable list persists.
- **A file without a valid `[m[m[m[m[m[m[m[mod]` `id` is skipped** (shown as
  `(unnamed mod)` is never stored).
- **Last enabled wins.** If two enabled mods set the same key, the one later
  in the scan order (alphabetical by file name) takes effect.
- **Unknown sections and keys are preserved.** Store anything under any
  section — `my_cool_mod.panel_color = 0xFF00FF` — and read it with the same
  engine API, so mods can carry custom data for future wingram versions or
  companion tools.
- **Colors** accept `0xRRGGBB` or `#RRGGBB`. Invalid colors are ignored
  (the previous value stays).
- **Hot reload**: the *reload mods* button re-scans the folder; toggles apply
  the theme instantly. No restart needed.

## Feature flags Wingram reads today

| Key | Values | Effect |
| --- | --- | --- |
| `feed_density` | `default` / `compact` / `comfortable` | reserved for v0.2 spacing |
| `note_songs` | `on` / `off` | reserved for v0.2 (disabling the song picker per-mod) |

These are read generically through the engine, so adding new flags is a
Wingram-side change only — mods keep working across versions.

## Engine API (for hacking on Wingram itself)

```
engine.scan()                          # re-scan the mods folder
engine.value(section, key, fallback)   # merged lookup across enabled mods
engine.has_feature(key, expected)      # features.<key> == expected
engine.apply_theme(make_theme())       # returns a Theme with overrides applied
engine.set_enabled(id, on) / toggle(id)
save_config("config.ini", engine, settings) / load_config("config.ini")
```

`parse_mod(path, text)` is the parser — used by the selftest, so the format
is exercised on every CI run.
