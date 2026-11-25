# In-Brief backend (AppProject)

GitHub Actions runs this every 3 hours and on demand. It collects science news, adds AI summaries, and saves JSON files that the Flutter app downloads. No server, no hosting bill.

```
sources.json --> fetch_news.py --> data/news.json ------> summarize_news.py --> data/summarized_news.json --> app
                                                              |        ^
                                                              v        |
                                                      Gemini API   data/ai_summaries.json (cache)
```

`AppProject` is the private backend. After a successful **Fetch News** run, its **Sync Public JSON** workflow publishes only `data/summarized_news.json` and this README to the public `AppProject-Data` repository.

## Files

| File | Purpose |
|---|---|
| `sources.json` | Source list and settings. Edit this to add, remove or fix a source or category |
| `fetch_news.py` | Fetch, clean, tag, remove near-duplicate stories, keep the last 21 days |
| `summarize_news.py` | Read article text, ask the AI, validate, cache, build the app feed |
| `requirements.txt` | Python libraries (`requests`, `feedparser`, `trafilatura`) |
| `.github/workflows/*.yml` | Schedule (`0 */3 * * *`) and the manual **Run workflow** button |
| `data/news.json` | Original news. Keep it as the untouched source feed |
| `data/ai_summaries.json` | Cache of accepted AI results. Do not delete it |
| `data/summarized_news.json` | **What the app reads first** |
| `data/run_log.json` | Each source `ok` or `failed`, with the error |
| `.github/workflows/sync-json.yml` | Publishes the app feed and this README to `AppProject-Data` |
| `merge_sources.py`, `sources_additions.json` | Optional: add a prepared batch of sources safely |

## Sources and categories

Each entry in `sources.json` has `name`, `category`, `type`, `url`, `licence` (`open` or `check`) and `full_summary`. Optional: `enabled`, `max_age_days`, `article_text`, `og_image`, `note`.

- **Categories are free text.** The fetcher copies the `category` to each article. The app builds its chips from the categories in the downloaded news, so a new category appears after the next refresh. Keep the spelling consistent, because `Health` and `health` count as two topics.
- **Types the fetcher knows:** `feed`, `usgs`, `eonet`, `apod`, `europepmc`, `biorxiv`, `guardian`, `sportsdb`. A new kind of API needs a new function in `fetch_news.py`.
- Published stories must have at least 40 complete, useful words; newly generated descriptions target 40-50 words. Short, incomplete, or visibly truncated source text is excluded. Similar stories in the same category are conservatively deduplicated; separate Disaster & Weather events are preserved.
- **`open` sources** get a 40-50 word summary when enough readable article text is available. **`check` sources** are excluded from the app feed until their terms are confirmed.
- A broken source never stops the others. Its address is the first thing to check in `run_log.json`.

### Add or fix a source
1. Edit `sources.json`.
2. **Actions → Fetch News → Run workflow**.
3. Open `data/run_log.json`. Fix any `failed` source (usually the `url`) and run again.

## AI summaries

| Setting | Name |
|---|---|
| GitHub secret | `GEMINI_API_KEY` |
| GitHub variable | `GEMINI_MODEL` (overrides the script's default: keep them consistent) |

Per run: up to 10 new articles, in batches of 5. Accepted results are saved in `data/ai_summaries.json`; cached summaries are refreshed when the source headline or description changes. If the AI fails or the answer is rejected, a complete 40-50 word source description may be used (`summary_by: extract`); otherwise the article is omitted. A publisher block (HTTP 403 or 429) is not a Gemini failure: the script falls back to the feed text only when it passes the same quality checks.

To check what you got, look at `summary_by` on each article, not at `provider` at the top of the file.

### Using a local model (Ollama) instead
Free, and it does not use the Gemini quota. Run `summarize_news.py` on your laptop with an Ollama model. Run `git pull` first, and do not run it at the same time as the Actions job: both write `ai_summaries.json`, and the uploads can clash.

## Article fields

`id, title, summary, category, source, author, url, published, image`, plus after AI processing (some may be missing):
`original_title`, `original_summary`, `summary_by` (`gemini` or `extract`), `content_source` (`article` or `rss`), `flow` (3 to 5 short steps).

## Rules to keep

1. Keep API keys in GitHub secrets, never in the code.
2. Do not delete `ai_summaries.json` without accepting the cost of re-summarising everything.
3. Show a summary and a link, never a full article. Show pictures only from open sources.
4. After a big change, read the Actions log and `run_log.json`.

## Public feed address used by the app

`https://raw.githubusercontent.com/sandeeprana502/AppProject-Data/main/data/summarized_news.json`

The app falls back to the copy saved on the phone if it cannot reach this public feed. GitHub caches the file for a few minutes.

## Next checks

- Run the fetch workflow and open `data/run_log.json`.
- Confirm new categories appear in `summarized_news.json` or `news.json`.
- `flutter analyze`, test on a phone, then `flutter build apk --release`.
