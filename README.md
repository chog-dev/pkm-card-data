# Pokémon TCG card data

Local snapshot of English Pokémon Trading Card Game data: set lists, full card objects, weekly market prices, and (optionally) card images.

Sourced from the [TCGdex API](https://tcgdex.net). **TCG Pocket sets are excluded.** There are no fetch scripts in this repo; regenerate or update files from the sibling `tcgdex-data` project.

As of the files in this tree:

| Dataset | Count |
| --- | ---: |
| Sets (`data/sets.json`) | 206 |
| Cards (`data/cards.json`) | 21,812 |
| Pricing snapshot (`data/pricing/latest.json`, 2026-10-10) | 21,355 |
| Indexed images (`pokemon-cards/index.json`) | 20,051 |

Card and pricing counts differ because they are produced by different jobs at different times. Pricing walks the live API on snapshot day; `cards.json` is a catalog dump. Some cards have no image URL, so the image index is smaller still.

## Layout

```
just-card-data/
├── data/
│   ├── sets.json                 # Brief set list
│   ├── cards.json                # Full cards, keyed set_id → card_id
│   └── pricing/
│       ├── {YYYY-MM-DD}.json     # Dated snapshots, keyed by card_id
│       ├── latest.json           # Most recent complete snapshot
│       └── status.json           # Per-day completion + run history
└── pokemon-cards/                # gitignored (images are ~1.4 GB)
    ├── index.json                # key → {path, name, set, cached_at}
    └── images/{set_id}/en/{localId}.webp
```

`pokemon-cards/` is gitignored. Keep images locally (or unpack them from a sidecar archive); they are not part of a normal clone.

## `data/sets.json`

Array of brief set objects:

```json
{
  "id": "base1",
  "name": "Base Set",
  "logo": "https://assets.tcgdex.net/en/base/base1/logo",
  "symbol": "https://assets.tcgdex.net/univ/base/base1/symbol",
  "cardCount": { "total": 102, "official": 102 }
}
```

`logo` and `symbol` are omitted when TCGdex has none. `cardCount.total` includes extras (secret rares, etc.); `official` is the printed set size.

## `data/cards.json`

Nested object `{ set_id: { card_id: <card> } }`. Look up by TCGdex ids:

```python
import json

cards = json.load(open("data/cards.json"))
alakazam = cards["base1"]["base1-1"]
```

Card objects are stored as the API returned them. Typical fields:

- Identity: `id`, `localId`, `name`, `category` (`Pokemon` / `Trainer` / `Energy`), `rarity`, `illustrator`, `image`
- Pokémon: `hp`, `types`, `stage`, `evolveFrom`, `dexId`, `attacks`, `abilities`, `weaknesses`, `retreat`
- Printings: `variants`, `variants_detailed`
- Set brief: `set` (`id`, `name`, `cardCount`, optional `logo`)

`image` is a TCGdex asset URL **without** a file extension. High-quality webp is `{image}/high.webp`.

## Pricing

Snapshots live under `data/pricing/`. Each file is keyed by card id:

```json
{
  "base1-1": {
    "name": "Alakazam",
    "set": "base1",
    "localId": "1",
    "dexId": [65],
    "fetched_at": "2026-10-10T22:02:17+00:00",
    "pricing": {
      "cardmarket": { "unit": "EUR", "avg": 51.3, "low": 9.99, "trend": 43.41 },
      "tcgplayer": { "unit": "USD", "holofoil": { "marketPrice": 70.76 } }
    }
  }
}
```

The `pricing` block is stored **verbatim** from TCGdex (Cardmarket EUR, TCGplayer USD, including variant shapes such as `-holo` suffixes and `holofoil` / `reverse` / `normal` sub-objects). Missing markets are `null`. `dexId` is omitted for trainers and energy.

- **`latest.json`** — copy of the newest *complete* snapshot. Prefer this for “current prices.”
- **`{YYYY-MM-DD}.json`** — history. Files dated **2026-09-05 and earlier** are daily; from **2026-09-12** they are **Saturdays only**. The format did not change.
- **`status.json`** — per-day run log, expected vs fetched counts, coverage (`full` / `cardmarket_only` / `tcgplayer_only` / `none`), and upstream-date flags.

Gzipped snapshots (if present) are the same JSON:

```python
import gzip, json
snap = json.load(gzip.open("data/pricing/2026-07-06.json.gz", "rt"))
```

Prices cannot be backfilled. TCGdex only serves the current market numbers.

Card text, names, and images are © Pokémon / Nintendo / Creatures / GAME FREAK / The Pokémon Company. Data is assembled via [TCGdex](https://tcgdex.net). This repo is a personal local cache, not an official Pokémon product.
