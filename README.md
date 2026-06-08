# rom-curation — a Claude Code skill

A reusable [Claude Code](https://claude.com/claude-code) skill (and field-tested playbook)
for **curating, cleaning, deduping, validating, renaming, and converting retro-game ROM
collections** for an emulation frontend — especially **Batocera**.

It encodes the methodology and the hard-won gotchas from cleaning a large multi-system
library and deploying it to a Batocera box.

## What it covers

- **Workflow:** survey → validate → clean → dedupe → rename → move/deploy, dry-run first, manifests.
- **Naming:** GoodTools / GoodGen / GoodSMS / GoodN64 / No-Intro / TOSEC / Redump → No-Intro,
  with region-code and dump-code maps.
- **Dedupe:** title normalization, best-variant scoring (prefer USA, keep sole others),
  fuzzy cross-name matching with sequel-false-positive guards.
- **Cleanup:** non-game junk categories, fan-translation handling, multi-agent prototype vetting.
- **Formats:** Neo Geo CD → `.chd` (`chdman`), GoodGBA merged sets, `.neo`/Geolith vs FBNeo
  short names, N64 byte order (`.v64`/`.n64` → `.z64`).
- **Multi-system:** splitting 32X / MSX1·2 / SG-1000, BIOS consolidation.
- **Batocera:** system folder names, `gamelist.xml` surgery, scraped-art dupes, `pad2key`
  controller→keyboard mapping, and adding a missing system via `es_systems_custom.cfg`.
- **Ops:** SSH device workflow, macOS/zsh gotchas.

## Install

Clone (or copy) into your Claude Code skills directory:

```sh
git clone <this-repo-url> ~/.claude/skills/rom-curation
```

Claude Code auto-triggers it on ROM-curation tasks, or invoke it explicitly with `/rom-curation`.

## Note

The playbook contains setup-specific references (local paths, a LAN device hostname) as
concrete examples — adjust them to your environment.
