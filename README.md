# Wingram

**A PC Instagram-style client written entirely in [Still](https://github.com/StillAdum/Still)** — no Electron, no Tauri, no web views. The whole app is Still source (`.strc`) compiled to native Windows code through Still's C emitter.

![Wingram](assets/banner.png)

## What it is

Wingram looks and feels like instagram.com (dark web layout: top bar, left
nav rail, centered feed column) and implements the mobile-exclusive features
the web client never got — **notes with songs** front and center — plus a
full **Mods** system driven by plain-text `.strc` files.

### Features

| Area | What you get |
| --- | --- |
| **Feed** | Post cards with likes/save/caption, gradient artwork (or real JPEGs dropped into `assets/`), stories tray, notes tray |
| **Notes + songs** | Compose 60-char notes like mobile, attach a song from a searchable picker, hear a **synthesized audio preview**, see the music sticker on the feed tray; notes persist in `wingram_notes.json` |
| **Reels** | Vertical reel viewer with side actions and song stickers |
| **Explore** | Search box + photo grid with hover stats |
| **Messages** | Inbox + thread view with send box |
| **Profile** | Header with stats, bio, post grid |
| **Notifications** | Activity list (likes/comments/follows/mentions) |
| **Settings → Mods** | List `<modname>.strc` files, toggle them live, hot-reload, persisted in `config.ini` |
| **Backends** | local demo data (offline) • **fediverse live timelines** (real data via Mastodon/Pixelfed public API) • **instagram (experimental)** |

### Songs in notes — how it works on PC

The mobile app lets you attach a 30-second song sticker to a note. Wingram
does the same on desktop: the composer has a **pick a song** button that
opens a searchable catalog (title, artist, duration), every song carries an
original melody spec (`"freq:ms,..."`), and previews are **synthesized at
runtime** into a WAV in memory and played through winmm — no audio assets,
works on every machine. The sticker renders on the feed's notes tray exactly
like mobile, and clicking a sticker plays/stops that song.

## Backends (read this)

Instagram shut down the Basic Display API in December 2024 and offers no
official API for third-party clients, so Wingram ships three data sources:

1. **local demo** (default) — a bundled dataset so every screen works offline.
2. **fediverse** — REAL live data: public timelines from open Mastodon /
   Pixelfed instances via the documented `/api/v1/timelines/public`
   endpoint. Configure the instance in Settings → Account. No login needed
   on open instances; paste an access token for locked-down ones.
3. **instagram (experimental, opt-in)** — talks to Instagram's private
   mobile API (`i.instagram.com/api/v1`) the same way
   [instagrapi](https://github.com/subzeroid/instagrapi) does: paste the
   `sessionid` cookie from a logged-in browser into `config.ini`
   (`account.instagram_sessionid = ...`) and set
   `account.ig_target = <username>`; the client derives `ds_user_id` from
   the sessionid's leading digits and presents the cookie + Android app
   headers on each request.

   **This violates Instagram's Terms of Service.** It can rate-limit,
   challenge, or ban your account, and endpoints drift without notice.
   It is read-mostly by design (profile + posts). Use an account you can
   afford to lose.

## Build

Wingram needs the **Still 0.6.4+** compiler (its `net`, `gui`, `audio`,
`media`, `json` stdlib modules and `read_dir`/`file_exists`/`dict_key_at`
builtins are what Wingram is built on; 0.6.4 fixes three code-generation
bugs that crashed GUI programs — always build Wingram with 0.6.4+):

```powershell
# get the compiler (Windows x64)
# download still-vX.Y.Z-windows-x64.zip from https://github.com/StillAdum/Still/releases
powershell -ExecutionPolicy Bypass -File .\install.ps1
still setup-cc          # portable TinyCC, one time

still run wingram.strc              # dev run
still build wingram.strc -o wingram.exe --release
.\wingram.exe                       # the client
.\wingram.exe --selftest            # headless logic checks (CI uses this)
```

GitHub Actions does all of this automatically on every push — see
`.github/workflows/build.yml` — and attaches `wingram.exe` to every `v*` tag.

## Mods

Drop `.strc` files into the `mods/` folder next to the exe, then toggle them
in **Settings → Mods** (hot-reload button included). Format and a full
reference in [MODS.md](MODS.md). Quick taste:

```ini
# mods/my_theme.strc
[mod]
id = my-theme
name = My Theme
version = 1.0.0
author = you
description = Violet accent, deeper black.

[theme]
accent = 0x8B5CF6
bg = 0x050507

[features]
feed_density = compact
```

## Project layout

```
wingram.strc              entry point: window, frame loop, router, selftest
src/
  theme.strc              palette + color parsing
  types.strc              data models (User/Post/Story/Note/Reel/Chat/...) + demo data
  app.strc                the App state struct every screen renders from
  ui_kit.strc             widgets: avatars, buttons, icons, nav rail, toggles
  songs.strc              song catalog + melody specs + preview playback
  mods_engine.strc        .strc mod scanning/parsing/applying + config.ini
  storage.strc            notes persistence (JSON) + HTML stripping
  adapters.strc           local / fediverse / instagram backends
  screens/                feed, explore, reels, notes, dm, profile, notifications, settings
.github/workflows/        CI: typecheck → build → selftest → artifact/release
```

## Changelog

- **v0.1.2** — fix the crash-on-launch white-window bug (root cause was in
  the Still 0.6.3 compiler: `d.get(key, x.f)` with a field fallback could
  free the fallback before storing it; rebuilt against Still 0.6.4). CI now
  also opens a real window for 90 frames (`--smoke`) so GUI regressions
  fail the build. Fixes corrupted `[mod]` markers in docs and sample mods.
- **v0.1.1** — release packaging fixes (source zip, release notes).
- **v0.1.0** — first public version: feed, stories, reels, explore, DMs,
  profile, notifications, settings, Mods system, songs in notes.

## Screenshots

Run it — or check the releases. The UI is fully drawn each frame with GDI
(rects, rounded rects, circles, text) through Still's `gui` module; there is
not a single line of HTML or JavaScript in this repository.

## License

Same as the Still repository's licensing (ask before redistributing).
