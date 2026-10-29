# Pokemon card prices

Pulls the price history of every Pokemon card in every set from [pricecharting.com](https://www.pricecharting.com) into pandas, ready for analysis.

## Project layout

| Path | What it is |
| --- | --- |
| `all_sets.ipynb` | The main notebook. Lists every set, pulls every card's price history and combines it into `pokemon_all`. |
| `archive/card_num.ipynb` | The first version: Base Set only, with card numbers and names taken from pokellector.com. Kept for reference. |
| `data/sets/<set>.csv` | One file per set, written as soon as that set finishes. |
| `data/set_release_dates.csv` | Release date for each set, cached so it's only looked up once. |

`data/` is in `.gitignore`, so the pulled data stays local.

## Setup

```bash
python -m venv .venv
```

```bash
.venv\Scripts\activate
```

```bash
pip install pandas numpy matplotlib requests beautifulsoup4 jupyter
```

## Running it

Open `all_sets.ipynb` and run the cells in order.

1. **Settings.** Choose what to pull:
   - `LANGUAGES` takes any of `"English"`, `"Japanese"`, `"Chinese"`, `"Korean"`.
   - `SETS_TO_PULL` is `None` for every set, or a list of set slugs such as `["pokemon-base-set", "pokemon-jungle"]`.
2. **Pull all set names.** Builds `sets_df` from pricecharting's Pokemon category page.
3. **Release dates.** Puts the sets in order: Base Set through to the newest release, with promo sets last. When a scraped date is wrong, add the correct one to `RELEASE_DATE_OVERRIDES`.
4. **Main loop.** Pulls each set and saves it to `data/sets/`. Sets that are already saved are skipped, so you can stop the loop and re-run it to carry on where it left off. Cards that failed are collected in `failed_cards`.
5. **Combine.** Reads every saved set into `pokemon_all` and joins on the set details from `sets_df`.

A full run takes a long time. Narrow it with `LANGUAGES` or `SETS_TO_PULL` while testing.

## Output: `pokemon_all`

One row per card, per grade, per month.

| Column | Type | Notes |
| --- | --- | --- |
| `grade` | str | `Ungraded`, `Grade 7`, `Grade 8`, `Grade 9`, `Grade 9.5` or `PSA 10` (mapped from pricecharting's chart keys in `GRADES`) |
| `date` | datetime | Month of the price point |
| `price` | float | Price in dollars |
| `set_slug` | str | pricecharting's set id, e.g. `pokemon-base-set` |
| `product_id` | str | pricecharting's card id |
| `name` | str | Card name, lower case |
| `variant` | str | e.g. `1st Edition`, blank for the standard card |
| `number` | str | Card number in the set. Kept as text because some numbers aren't plain integers. |
| `set_name`, `language`, `release_date`, `promo` | | Set details from `sets_df` |

## Rate limiting

pricecharting will start refusing requests if they come too fast. All requests go through `polite_request`, which:

- waits `delay` seconds (±20%) between requests, starting at `START_DELAY`.
- speeds up gradually after each good response, but never below `MIN_DELAY`.
- treats a 403, 429 or 503, a Cloudflare or captcha page, or a dropped connection as being rate limited. It then doubles the delay, raises the minimum so it doesn't creep back to the speed that got it limited, and cools off before retrying (honouring `Retry-After` if the site sends one).
- stops the run with `RateLimited` after `MAX_RETRIES` limited attempts in a row on one request, rather than risk a ban.

If a run stops for rate limiting, wait a while (ideally hours) and re-run the main loop. The set that was in progress wasn't saved, so it will be pulled again from the start.

pricecharting also offers an official API for paying subscribers. Check their terms of use before pulling large amounts of data.

## Notes

- **Card lists come from pricecharting.** Card URLs are read from each set's page rather than built from names, so they're always correct. Sealed products such as booster packs and blisters are dropped, since they don't have a card number.
- **Combine with one `pd.concat`.** Saved sets are read into a list and concatenated once. Starting from an empty `pd.DataFrame(columns=[...])` and concatenating inside a loop turns every column into `object` and gets slower with each set.
- **dtypes when reading saved data.** When reading the CSVs, pass `dtype={"number": str, "product_id": str, "variant": str}` and `parse_dates=["date"]` so the columns come back with the same types they were saved with.
