---
name: rom-curation
description: >
  Curate, clean, dedupe, validate, rename, convert, and organize retro-game ROM
  collections for an emulation frontend (esp. Batocera). Use when the task involves
  ROM/disc-image folders, GoodTools/No-Intro/TOSEC/Redump/GoodSMS/GoodN64/GoodGen naming,
  deduping regional/dump variants, stripping dump codes, removing non-game junk
  (sound rips, dev tools, demos, prototypes), applying fan translations, converting
  CD images to .chd, MAME/FBNeo Neo Geo romsets, splitting multi-system folders,
  consolidating BIOS, fixing Batocera gamelist.xml / scraped art, controller-to-keyboard
  (pad2key) mapping, or deploying/cleaning ROMs on a Batocera box over SSH.
  Triggers: "clean/dedupe/sort/validate/move <system> roms", "neogeo cd", "msx", "32x",
  "showing up twice", "prototypes", "translations", ".chd", "bios", "padto.keys".
---

# ROM Library Curation

A playbook for turning messy ROM dumps into a clean, deduped, correctly-named,
ready-to-play library. Built from curating a large multi-system collection in
a library root (e.g. `~/roms`) and deploying to a Batocera box over SSH.

## Golden rules

1. **Always dry-run first.** Compute the full plan (delete list + rename map), print a
   sample + collision check, and verify counts reconcile (`deleted + kept == original`)
   BEFORE executing. Many ops are irreversible (or 25 GB+ moves).
2. **Write manifests.** Save the original listing and the delete/rename maps to `~/`
   (e.g. `~/<system>_original_listing.txt`, `~/<system>_delete.txt`). Renames on the
   same volume are instant (`mv`/`os.rename`); deletes are not recoverable.
3. **Match the user's established preferences** (below). Confirm only genuinely new
   judgment calls (e.g. "keep best vs keep all variants", which naming standard,
   whether to delete a whole bundled set).
4. **One ROM per game** unless the user says otherwise.

## The user's standing decisions (apply by default)

- **Naming = No-Intro style** `Title (Region).ext`. Convert GoodTools/GoodGen/GoodSMS/GoodN64
  codes to it; strip dump codes.
- **Keep only the best** variant per game; drop betas/protos/demos/alts/bad dumps/hacks
  *when a better version exists*. Keep a proto/beta only when it's the **sole** version.
- **Region: prefer USA**, but **keep sole others** (a Europe/Japan-only game stays).
- **Prefer the `.zip`** (or cleaner container) over a loose ROM when both exist.
- **Translations:** keep English fan-translations as distinct items (best per language);
  **remove non-English** translations; an English translation **supersedes** the Japanese
  original (delete the original). Pre-patched translated ROMs are ready to use.
- **Prototypes:** keep complete/playable/beatable cancelled games; remove incomplete /
  demo / tech-demo / non-beatable builds (vet them — see below).
- **Non-games:** remove sound/music rips, dev tools, slideshows, tech demos, diagnostics.
- **CD images → `.chd`** (single-file). **Cartridge ≠ CD** — never mix them in one system.

## Standard workflow

1. **Survey:** `ls`/`find` the folder — extension counts, subfolders, sample names,
   naming convention (GoodTools `(U) [!]`, No-Intro `(USA)`, TOSEC `(year)(publisher)`,
   Redump, MAME short codes). Identify metadata/junk (`.DS_Store`, `_info.txt`,
   `gamelist.xml`, `images/`, `videos/`, emulator `.exe`).
2. **Validate:** confirm each archive actually contains a ROM/disc image of the right type
   (`unzip -l` → look for `.sfc/.smc/.md/.gba/.sms/.z64/.cue/.bin` etc.). Flag missing/empty.
3. **Clean:** drop junk + non-games + (per user) unreleased/incomplete protos.
4. **Dedupe:** group by normalized title → keep best variant.
5. **Rename:** GoodTools/etc. → No-Intro.
6. **Move** to the destination system folder (use the Batocera system name).

## Normalization (for grouping/dedupe)

```python
import re
ROMAN={'ii':'2','iii':'3','iv':'4','v':'5','vi':'6','vii':'7','viii':'8','ix':'9'}
def norm(f):
    b=os.path.splitext(f)[0].replace('&',' and ').replace('+',' and ')
    b=re.sub(r'\([^)]*\)','',b); b=re.sub(r'\[[^\]]*\]','',b); b=re.sub(r'\{[^}]*\}','',b)
    b=re.sub(r',\s*(the|a|an)\b','',b,flags=re.I); b=re.sub(r'^(the|a|an)\s+','',b,flags=re.I)
    b=b.lower()
    return ''.join(ROMAN.get(t,t) for t in re.split(r'[^a-z0-9]+',b) if t)
```
Group files by `norm()`. Groups with >1 file are variants to collapse.

## Best-variant scoring

Region dominates, then verified, then penalties (penalties must be able to beat region
so a USA bad-dump loses to a clean Japan dump). Recognize USA *anywhere* — `(USA, Europe)`,
`(World)`, `(JUE)`, `(Japan, USA)` all count as USA-inclusive.

```python
def region_rank(n):
    if re.search(r'\b(USA|World)\b',n): return 100
    for c in re.findall(r'\(([A-Za-z0-9]{1,4})\)',n):
        if c in {'U','UE','EU','JU','UJ','JUE','UJE','EJU','UEJ','JEU','4','W'}: return 100
    if re.search(r'\bEurope\b',n) or re.search(r'\((E|F|G|S|I|UK)\)',n): return 50
    if re.search(r'\bJapan\b',n) or re.search(r'\((J|JE|1)\)',n): return 40
    if re.search(r'\b(Brazil|Australia|Korea|Taiwan)\b',n): return 25
    return 55  # unknown/homebrew
# penalties: [b] bad -20000, [o] over -15000, [h]/hack -18000, [t] trainer -12000,
#   [f] fix -2000, [a] alt -300/-600, (Beta)/(Proto)/(Demo)/(Sample)/(Kiosk) -15000.
# bonus: [!] +200 ; prefer higher (Rev/V1.x) ; prefer .zip ; tiebreak shorter then alpha.
```
`best = max(group, key=lambda f:(region_rank(f)*1000 - penalty(f) + bonus(f), is_zip(f), -len(f), f))`.
Note: `[!]` (verified-good) generally beats a higher unverified rev — but flag this when it
conflicts so the user can override.

## Naming: GoodTools/GoodGen/etc. → No-Intro

Strip ALL `[...]` dump codes (`[!] [b1] [h1] [a1] [o1] [c] [t1] [hI]`) and the parenthetical
`(!)` form. Map region codes (recognize letter-set permutations of J/U/E):
`(U)→(USA)  (E)→(Europe)  (J)→(Japan)  (W)→(World)  (UE)→(USA, Europe)  (JU)→(Japan, USA)
(JUE)/permutations→(World)  (4)→(USA)  (B)/(Brazil)→(Brazil)  (K)→(Korea)`.
For an already-clean No-Intro/Redump/TOSEC set, **don't rename** — just dedupe.
Keep meaningful tags: `(SGX)`, `(SG-1000)`, `(Proto)` (for sole protos), `(Unl)`, `(T-Eng)`.

## Non-game junk to remove

- **Sound/music rips:** `*Sounds*`, SNES `.spc`, PCE `*_sound_*`/PCM/MML players, `.srm` saves,
  Jaguar `JSSDemo`/sound demos, savestates (`.000`/`.spc` inside odd zips).
- **Dev tools / diagnostics:** PCEmon/RS232 monitors, BRAM/flash-cart tools, `chdman`-style
  utilities, Jaguar `BadCode`/`Jaguar Server` examples, Atari `(Demo)` test stubs,
  CMD/SDK/"Check Program"/"Sample Program" carts, MSX firmware word-processors/printer ROMs.
- **Slideshows / tech demos:** raster/fractal/scroll/palette/"Wavy Pics"/Artshow/SlideShow.
- **Mislabeled stray files** (e.g. a `delete.zip` holding `delete.me`, lowercase `.bin` dupes).
- **Redundant legacy BIOS** (e.g. flat `neo-geo.rom` when the split `neogeo.zip` exists).

## Fuzzy / cross-name dedupe (the hard cases)

`norm()` misses same-game pairs that differ in title. Catch these with care:
- **Abbreviations / spelling:** `AD&D` ↔ `Advanced Dungeons & Dragons`, `Mega Man Soccer` ↔
  `Mega Man's Soccer`, `Aliens 3` ↔ `Alien 3`, `Bubble and Squeek` ↔ `Squeak`.
- **& vs and vs +; roman vs arabic** — normalize these.
- **Subtitle added:** `Phalanx` ↔ `Phalanx - The Enforce Fighter`, `Chakan` ↔
  `Chakan - The Forever Man`. (Match a short title to another's pre-`" - "` segment.)
- **Alternate regional titles (same game):** `lastsold`(Korean) = `lastblad`,
  `Wonder Boy in Monster World` = `Wonder Boy V - Monster World III`(JP),
  `Romance of the Three Kingdoms IV` = `Sangokushi IV`, `2020 Super Baseball` =
  `Super Baseball 2020`, Korean Neo Geo releases (`aof3k`,`rbff2k`,`kof97k`,`fswords`).
- **GUARD against sequel false-positives:** require the *number set* (roman-aware, ignoring
  region) to match — `Chuck Rock` ≠ `Chuck Rock II`, `Might and Magic` ≠ `…2`, and watch
  franchise prefixes (`Spider-Man` ≠ `Spider-Man - Mysterio's Menace`,
  `The Incredibles` ≠ `…Rise of the Underminer`). When unsure, KEEP both.
- Always print fuzzy matches and **hand-review** before deleting.

## Format-specific handling

- **GoodGBA "merged" zips:** each zip can hold ALL variants (regions/hacks/translations) —
  a 16 MB zip may contain 5 ROMs, a 1.2 GB one hundreds. Look *inside* (`zipfile.namelist()`),
  pick the single best ROM (USA→Europe→World→Japan, `[!]`, skip `[b]/[h]/[f]/[t]/beta/kiosk`;
  for Japan-only games prefer an English `[T+Eng]` patch), re-zip just that ROM. Some "games"
  are PD-intro/GameCube-multiboot stubs — note them.
- **Neo Geo CD:** zipped multi-track `.bin`+`.cue`. Convert to `.chd` for Batocera
  (zipped multi-track usually won't load): `chdman createcd -f -i game.cue -o game.chd`,
  then `chdman verify`. chdman comes from Homebrew `rom-tools` (the `mame` bottle does NOT
  include it on macOS). ~half size. Cartridge Neo Geo (`Cylum's…`, chip files `030-c1.c1`)
  is a DIFFERENT system — don't put it in `neogeocd`.
- **`.neo` files:** modern single-file Neo Geo format → needs the **Geolith** core, not FBNeo/MAME.
- **Neo Geo cartridge for FBNeo/MAME** needs exact short romset names (`mslug3.zip`,`kof98.zip`).
  Map descriptive→short via the libretro FBNeo gamelist:
  `raw.githubusercontent.com/libretro/fbalpha2012_neogeo/master/gamelist.txt`.
  Validate by basename, abort on collisions. (If the zips actually hold `.neo`, naming is moot — use Geolith.)
- **N64 byte order:** zips may hold `.z64` (native), `.v64` (swapped), `.n64` (little-endian).
  Modern Mupen64Plus-Next/ParaLLEl auto-detect; if a picky emulator chokes, byte-swap to `.z64`.

## Translations

Translated ROMs are usually **pre-patched** (a zip with a translated `.rom`) — ready to use.
Dedupe by `(game, language)`; prefer `[T+]` (complete) over `[T-]`, higher version/%, and
`Fixes`/`New Font`; drop `[b]/[h]/[t]` variants. English translation supersedes the Japanese
original (delete original). For a translated set, name as `Title (Region) (T-Eng).ext`
or `Title [T-En].ext`. Keep distinct fan-translation languages only if the user wants them.

## Prototype vetting (keep playable, drop incomplete)

"Proto-only" (sole version of a game) ≠ "unreleased". Many cancelled games shipped as
**complete, beatable** late protos (keep) — but many protos are early/demo/broken (drop).
This needs per-title research. For a large batch, run a **multi-agent workflow**: research
each (web: AtariProtos, AtariAge, Hidden Palace, Sega Retro) → structured verdict
(keep complete-&-beatable / remove incomplete-demo-broken) → an **adversarial verify pass**
on every proposed removal (default to keep on doubt). Then delete confirmed removals.
Be conservative — a wrong removal loses a game.

## Multi-system & cross-folder issues

- A game **showing up twice** in the frontend usually means it's in two system folders
  (e.g. 32X titles bundled in a No-Intro Genesis set AND in `sega32x`), or a stale/duplicate
  `gamelist.xml` entry, or the same game under two regional titles. Diagnose before deleting.
- **Split mixed folders:** 32X out of Genesis; **MSX1 + translations → `msx1`**, **MSX2 → `msx2`**
  (MSX2+ usually empty); **SG-1000 out of `mastersystem` → `sg1000`** (strip the `(SG-1000)` tag).
- For cross-system dupes of the same game, prefer the enhanced version (MSX2 over MSX1).

## BIOS

Consolidate all BIOS into one `<library>/_bios/` with a `README.txt` mapping each to its
Batocera target. Game folders should be **games-only** (strip BIOS out of rom subfolders;
Batocera reads BIOS only from `/userdata/bios`, not `roms/<sys>/BIOS/`).
Batocera names: Neo Geo `neogeo.zip`; Atari 5200 `5200.rom`; Atari 7800 `7800 BIOS (U)/(E).rom`;
Sega CD `bios_CD_U.bin`; GBA `gba_bios.bin` (optional); 32X `32X_M/S/G_BIOS.BIN`;
MSX `MSX.ROM` / `MSX2.ROM` + `MSX2EXT.ROM` / `MSX2P.ROM`+`MSX2PEXT.ROM`, plus `DISK.ROM`,
`FMPAC.ROM`, `KANJI.ROM`. Dreamcast `dc_boot.bin`/`dc_flash.bin`; Intellivision `exec.bin`/`grom.bin`.
TOSEC firmware sets store each ROM inside a per-entry folder; the canonical name is the
`[NAME.ROM]` tag — copy the inner `.rom` out and rename. Don't fetch copyrighted BIOS; the
user supplies them. MSX/SMS-CD note: Batocera ships **C-BIOS** for blueMSX (cartridge `.rom`
games may run with no real BIOS). Verify with **Batocera → Game Settings → Missing BIOS Check** (md5).

## Batocera specifics

- System folder names (use exactly): `snes genesis megadrive sega32x mastersystem sg1000
  gba pcengine neogeo neogeocd dreamcast atari2600 atari5200 atari7800 c64 intellivision
  jaguar n64 msx1 msx2 msx2+`.
- Emulators load by **content**, so filenames are free-form EXCEPT **arcade** (Neo Geo cart →
  FBNeo/MAME needs exact short names). Disc/console = any name.
- **System not showing on the frontend at all** (folder has games but the system is absent):
  the ROM extension may not be in that system's scan list — use the canonical one
  (Jaguar: rename `.jag`→**`.j64`**; also `.lnx`,`.a26`,`.sms`,`.j64`,`.z64`…). If it's STILL
  missing, the image's `/usr/share/emulationstation/es_systems.cfg` may simply **lack that
  system** (it's a static read-only file; a reboot does NOT regenerate it, and it lists ALL
  supported systems incl. empty ones, so a missing entry = genuinely unsupported in this build).
  Fix by ADDING it via the merge file **`/userdata/system/configs/emulationstation/es_systems_custom.cfg`**
  (a `<systemList><system>…</system></systemList>` — copy `<command>`/`<emulators>` from a similar
  existing block; the core comes from `configgen-defaults.yml`). The `_custom` file MERGES (never
  replaces). Takes effect on an ES **game-list refresh / restart**. Verify the system name exists in
  `configgen-defaults.yml` first (e.g. `jaguar:` → core `virtualjaguar`).
- `images/`, `videos/`, `gamelist.xml`, `*_meta.sqlite/.xml` are scraper media/metadata — leave
  them when just deduping ROMs (orphaned entries are harmless). After deletions: the device may
  show **ghost entries** until **Menu → Game Settings → Update Gamelists**.
- **gamelist.xml fixes:** it's XML with `<game ...>` (attributes) blocks. Mis-scrapes:
  a wrong `<name>` (one game showing as another) → edit the name; two real games sharing a
  display name → rename both to distinguish (DON'T delete — they're different games);
  identical art on two games → the scraper saved the same `-image.png`/`-video.mp4` under both
  (compare md5); if no distinct art exists, blanking or accepting shared art are the only fixes.
  Remove a whole game block with a tempered regex:
  `re.sub(r'[ \t]*<game (?:(?!</game>).)*?<PATH>(?:(?!</game>).)*?</game>\n?','',xml,flags=re.S)`.
- **pad2key (controller → keyboard)** for keyboard computers (MSX, Amstrad, C64…): JSON `.keys`
  file at `/userdata/system/configs/evmapy/<system>.keys`. **Only map the spare buttons**
  (start, select, x, y, pageup=L1, pagedown=R1) — leave d-pad + a/b UNmapped so they stay the
  joystick (model on shipped `/usr/share/evmapy/amstradcpc.keys`). Targets are `KEY_*`
  (KEY_SPACE/ENTER/F1/ESC/UP). Example MSX: start→KEY_SPACE, x→KEY_F1, y→KEY_F2,
  pageup→KEY_ENTER, pagedown→KEY_ESC. evmapy reads it on game launch (no reboot).

## Device (SSH) operations

`ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new root@batocera` (key auth). ROMs in
`/userdata/roms/<system>/`, BIOS in `/userdata/bios/`. Always **list/confirm** on the device
before deleting (its set may differ from local — e.g. GoodGen on the box vs No-Intro locally).
Back up gamelists (`cp gamelist.xml gamelist.xml.bak`) before editing. The Pi has Python3.
Use `find -maxdepth 1 -iname '*32X*' -delete` etc. carefully. Deploy with `scp`/`rsync`;
`/userdata` is best on a USB3 SSD (random I/O), OS on the SD.

## Validation checklist

- Zip not corrupt / contains a ROM of the right type for the system.
- CD image has a `.cue` + `.bin` track(s); `.chd` passes `chdman verify`.
- No name collisions after rename (`Counter(new_names)`); totals reconcile.
- Right system: cartridge vs CD; SG-1000 vs SMS; 32X vs Genesis; Neo Geo cart vs CD.

## macOS / zsh gotchas

- Globs that don't match error under zsh (`no matches found`) — guard or use Python/`find`.
- Paths with spaces/apostrophes (`Cylum's…`, `neogeo CD`) — quote carefully; prefer Python
  for path handling.
- Python `glob.glob` treats `[...]` in a path as a character class — use `os.listdir` for
  folders/files whose names contain brackets (TOSEC `[MSX.ROM]`).
- `mv`/`os.rename`/`shutil.move` within the same volume is instant (metadata only).
- Long conversions (chdman, GBA reduction): run in the background, log to `/tmp`, poll.
