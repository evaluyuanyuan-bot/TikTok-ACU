# FastMoss Weekly ACU Scrape — Handover Runbook

Paste this whole file as your first message to Claude (along with that week's template `.xlsx` attached), or ask Claude to save it as memory before starting. It replaces context that would otherwise take a full session to rediscover.

## The task

Weekly recurring job: a template `.xlsx` lists shop names, FastMoss shop-marketing URLs, and a date range. Fill in ACU ("online users") data for each shop at 6 fixed daily timeslots (8:00, 10:00, 12:00, 15:00, 18:00, 20:00), scraped from FastMoss (TikTok shop analytics) via browser automation. Return a completed copy of the same template.

**Prerequisites before starting:**
- FastMoss login in the browser tab Claude will use (Claude cannot log in itself — a human must do this).
- Claude in Chrome (or equivalent browser tool) connected.

## Template layout

- Row 1: merged header. Row 2: header row — col A = Shop Name, B = Fastmoss URL, C = Time, D onward = one column per date.
- Per shop: 6 consecutive rows, one per timeslot (8:00, 10:00, 12:00, 15:00, 18:00, 20:00). Shop Name/URL populated only on the first of the 6 rows.
- Read shop list, URLs, and date range directly from the template — it's the source of truth, not the prompt text.

## Per-shop scraping procedure

1. Open `shop-marketing/detail/{id}` > 直播分析 (Live Stream Analysis) > 直播列表 (Live Stream List). Filter 直播类型 = 店铺播 (store broadcast only — not 达人播/influencer broadcasts). Set the date range to match the template. Sort by 开播时间 (broadcast time) descending by clicking its sort caret — the default order is unstable/flaky, sorted by a different column; don't trust it un-sorted.
   - Click filter buttons, date fields, and sort carets via a `find`-located element ref, **not** a blind screenshot coordinate — the layout shifts with scroll position and this causes silent mis-clicks (e.g. landing on 达人播 instead of 店铺播, or a date off-by-one).
   - Paginate via numbered page links (re-`find` the ref each page, they go stale) — not the jump-to-page textbox, which is unreliable.

2. Build the session list fast, without screenshots. On each page, run:
   - Room IDs, in DOM order: `Array.from(document.querySelectorAll('a[href*="/zh/live/detail/"]')).map(a=>a.getAttribute('href').match(/\d+/)[0]).join(',')`
   - Readable start-time/duration/viewer text: `Array.from(document.querySelectorAll('main')).find(m=>m.innerText.includes('直播列表')).innerText.split('直播列表')[1].split('视频分析')[0]`

   Combine both into a table of {room_id, start_time, duration}. Compute each session's end time, and mark which of the 6 target slots it covers (±60 min tolerance). **Skip any session covering zero target slots.** Long broadcasts (8h+) sometimes cover all 6 slots in one room — a whole day can be one room visit, not four.

3. For each room actually needed, navigate straight to `/zh/live/detail/{room_id}#trend_chart` (skips finding/clicking the 直播流量趋势 tab), then scroll the "Live streaming traffic trends" chart into view.

4. The chart's x-axis is a genuine time-scaled axis (not evenly-spaced categories) — pixel position is proportional to elapsed time. The plot's interactive area runs roughly **x≈277 to x≈1298** (in a standard ~1568px-wide screenshot), spanning the session's full start→end. Compute the target hover position instead of guessing:
   `target_x = 277 + (target_time − session_start) / (session_end − session_start) × 1021`
   This normally lands within tolerance on the first hover.

5. For each covered slot, hover the computed x to find the nearest actual data point. Keep the value only if within ±60 minutes of the target slot; otherwise record 0.

6. **Mandatory verification (this is the single biggest source of past errors):**
   - Never trust a single hover. In one batched call: hover an "away" point first (clearly on the line, away from the target), then hover the target pixel, then screenshot/read.
   - Check the tooltip's timestamp is plausible for that x-position. If it matches the away-hover's timestamp, or is way off from what that x-position should show, the hover didn't register — the chart's real interactive area is sometimes narrower than its visible drawn area, especially near edges. Nudge ±10-150px and retry.
   - Only record the value once the timestamp is consistent with the hovered x-position.
   - First axis label ≈ first real data point almost always, but a data-logging lag of 10-90 min after the session's stated start time is normal and not a bug — confirm by checking nearby x-values converge, not by accepting the first read.

## Output

Write verified values into columns D→(D+n) of the user's own template, preserving structure — don't rebuild from scratch. If the filename is locked/open elsewhere, save as `<original name> COMPLETED.xlsx` in the same folder and say why.

## Practical notes

- Track progress in a scratch CSV per shop as you go (columns: date, 08:00, 10:00, 12:00, 15:00, 18:00, 20:00) so nothing is lost if the session is interrupted or the user asks to pause between shops.
- FastMoss may kick the session ("logged in on another device") mid-task — this is unrelated to chat history, just tell the user and wait for them to log back in.
- Confirm the exact date range up front if it's not unambiguous in the prompt — don't guess between a prompt placeholder and the template's own pre-filled dates.
