**English** | [简体中文](README.md)

# play-console-assets

A Claude Code skill: after a brand icon / feature graphic / screenshot
redesign, cascade the change to every place that consumes it — the app's
own launcher icon, in-app embedded logos, static-page favicons, and
Google Play Console's Default store listing graphics. Generic skill, not
tied to any specific project directory. Body text below is Chinese-first
in the shipped SKILL.md content (per this author's default), but this
top-level README is English for discoverability; code, CLI commands,
file paths, and technical terms are kept verbatim either way.

## Install

```
/plugin marketplace add mosjin/play-console-assets-skill
/plugin install assetpub@play-console-assets
```

Invoke with the short name `assetpub` (e.g. `/assetpub:version`,
`/assetpub:upgrade`), following the same convention as
`mosjin/android-voice-test-kit`: a descriptive marketplace/repo name
paired with a short plugin command name.

## What this covers

- A checklist of every place a brand icon redesign actually needs to
  reach: platform launcher icon (`flutter_launcher_icons`), in-app
  embedded logos (QR codes, watermarks), static legal-page favicons
  (inline base64), and Play Console store listing graphics — don't stop
  at the launcher icon and call it done.
- A fully local SVG → PNG rasterization pipeline
  (`gi.repository.Rsvg` + `cairo`) for designing a new icon without
  depending on any external image-generation API.
- **Six real, hard-won gotchas in Play Console's Assets editing page**,
  documented in full inside the `assetpub` skill body, including:
  - The page has several links with the *exact same text* ("Add
    assets") — one per field (App icon / Feature graphic / Phone
    screenshots). Text-matching without scoping will silently operate
    on the wrong field.
  - The file picker (`<input type=file>`) for multi-image fields (Phone
    screenshots) is single-file only, despite the field accepting 2-8
    images — you must upload one at a time.
  - "Uploaded / auto-selected" is not the same as "added to the field"
    — you still have to click "Add" afterward, and for multi-image
    fields you must repeat the full "upload → Add" cycle per image
    rather than batch-uploading several and multi-selecting at the end.
  - The "needs cropping" unselectable state is very likely a scoping
    bug (operating on the wrong field), not an actual aspect-ratio
    violation — the author burned real time re-cropping/re-padding
    screenshots (2:1 → 16:9 → 1080×1920 → 1350×2400) before discovering
    the real bug was field mis-targeting all along.
  - Nothing persists until you click "Save as draft" — reloading the
    page silently discards all unsaved edits.
  - After "Save as draft", Play Console's own global chrome (the app
    switcher icon, top-right) does *not* update immediately — that only
    happens once the change is submitted for review and approved.
- A real-device re-screenshotting gotcha: `adb shell input text` fails
  to inject CJK text on many devices (`NullPointerException`) —
  recommends seeding demo data directly via the backend instead of UI
  text input.
- The `upgrade` skill runs `git pull` directly against the marketplace
  git checkout (via Bash, no manual command needed from you). Refreshing
  Claude Code's cached snapshot still requires picking "Update" from the
  `/plugin` menu, **immediately followed by `/reload-plugins`** — Update
  refreshes the on-disk registry but does not automatically propagate
  into an already-running session; skipping that step looks like a
  failed upgrade when it isn't.

## Related

- [`play-console-app-publish`](https://github.com/mosjin/play-console-app-publish-skill)
  (playpub) — the Chrome CDP connection method, app creation, Internal
  testing uploads, subscription product creation. This skill fills the
  "store listing graphics" gap playpub doesn't cover; use both together.
- [`flutter-release-aab-build`](https://github.com/mosjin/flutter-release-aab-build-skill)
  (aabbuild) — packaging the release AAB, another stage of the same
  release pipeline.
- [`android-real-device-driver`](https://github.com/mosjin/android-real-device-driver-skill)
  (adbdrive) — general real-device adb screenshot/interaction gotchas,
  referenced by the re-screenshotting step.
