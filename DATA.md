# PRK WhatBuffs — raw data

`prk-data.json.gz` is the exact dataset behind PRK-Items.html (WhatBuffs), extracted
straight from the PRK 18.4 client's resource database. Gunzip it and you get one JSON
object — no need to dissect the minified tool. Great for loading into SQL, a spreadsheet,
or your own scripts.

```
gunzip -k prk-data.json.gz        # -> prk-data.json (~66 MB)
```

## Top-level structure

| key         | what it is |
|-------------|------------|
| `items`     | array of 119,540 items |
| `nanos`     | array of 11,160 nano programs |
| `stats`     | dict: stat id (string) → stat name (e.g. `"148": "Burst"`) |
| `itemclass` | dict: item class id → name (`0` None, `1` Weapon, `2` Armor, `3` Implant, `4` Monster, `5` Spirit) |

## Item fields

| field | meaning |
|-------|---------|
| `i`   | item id (links to PRK.GG: `https://www.prk.gg/items/<i>`) |
| `n`   | name |
| `q`   | quality level (QL) of this record |
| `c`   | item class id (see `itemclass` — this is the field to filter categories on) |
| `ic`  | icon id |
| `d`   | in-game description text |
| `s`   | stats dict: stat id → value (raw client values; includes flags fields like 0, 30, 298) |
| `a` / `df` | attack / defense skill percentages (weapons) |
| `b`   | buff entries this item grants while equipped — see **Buff entries** |
| `r`   | wear/use requirements — see **Requirements** |

## Nano fields

| field | meaning |
|-------|---------|
| `i`, `n`, `d`, `s`, `b` | same as items |
| `q`   | QL of the teaching nano crystal (not the NCU) |
| `ncu` | NCU cost (= stat 54) |
| `sf`  | 1 = self-only cast, 0 = castable on others |
| `np`  | 1 = no player-obtainable crystal exists (NPC/system nano — the tool hides these) |
| `r`   | cast requirements, with the crystal's requirements merged in |

## Buff entries (`b`)

Each entry is `[event, stat, amount, spell, target]`:

- `event` — 0 = on wear/on cast (the ones that matter)
- `stat` — stat id being modified (look up in `stats`)
- `amount` — modifier value (negative = debuff)
- `spell` — internal spell function id (53045 = modify stat, etc.)
- `target` — `2` = self, `3` = the cast target (i.e. usable on others)

## Requirements (`r`)

A flattened postfix (RPN) expression; each node is `[kind, stat, value, op]`.
Evaluate left to right with a stack:

- comparison ops: `0` = equal, `1` = less than, `2` = greater than, `24` = not equal
  → push `char[stat] <op> value`
- logic ops (stat/value ignored): `3` = OR, `4` = AND, `42` = NOT

Useful stat ids in requirements: `54` Level, `60` Profession, `368` VisualProfession
(profession locks live on either 60 or 368; 1=Soldier, 2=Martial Artist, 3=Engineer,
4=Fixer, 5=Agent, 6=Adventurer, 7=Trader, 8=Bureaucrat, 9=Enforcer, 10=Doctor,
11=Nano-Technician, 12=Meta-Physicist, 14=Keeper, 15=Shade).

## Why your row count won't match live helpbots

The raw db has 119,540 item records, but most are QL-step variants of the same item —
one record per QL breakpoint (e.g. Carbonum plate exists at many QLs, each its own id).
There are only **48,934 unique item names**. Live-server helpbot databases (~35k) look
smaller because they merge each item family's QL breakpoints into a single low-id/high-id
row with interpolated stats, and drop monster/NPC records.

To get a helpbot-style table from this data:

1. drop `c` = 4 (Monster) and 5 (Spirit)
2. group records by `n` (name), sort each group by `q`
3. within a group, consecutive records are the QL breakpoints of one item — keep the
   group as one logical item with a QL range, or keep all breakpoints if you want exact
   stats per QL (stats between breakpoints interpolate linearly by QL)
4. drop dev leftovers: names starting `TESTITEM`, `NODROP TEST`, etc.

## Filtering out the junk

The dataset is the raw client db, so it includes test items, NPC-only gear and monster
records. The tool's own cleanup rules, if you want to reproduce them:

- drop `c` = 4 (Monster) and 5 (Spirit) for player-facing lists
- drop nanos with `np` = 1 (no obtainable crystal)
- names starting with `TESTITEM`, `NODROP TEST`, etc. are dev leftovers

Data extracted from the PRK 18.4 client. Updated when PRK patches. Questions → ping
`.everkill` on Discord.
