# StreamAd Lighthouse, Sweep Procedure (manual runbook)

This is the canonical procedure for running a Lighthouse sweep. It replaces the old
`Scheduled/StreamAd Lighthouse Daily Sweep/SKILL.md`, which was deleted on 2026-05-29
when the scheduled task was removed. There is no schedule anymore. Sweeps are run
manually, on demand, when Drew asks in chat (or pastes the command the artifact's
sweep picker builds).

Read this before running a sweep. It references `CFG.*` values from
`lighthouse-config.json` and never hardcodes data that lives in the config.

TWO ENTRY MODES share this engine:
1. SWEEP (the broad scan), STEP 0 through STEP 9 below. Sources a batch from Apollo
   per the ICP and the resolved pick mix. This is "Sweep the waters".
2. GUIDE TO HARBOR (single-target), the section at the bottom of this file. Drew drops
   one or more specific URLs and each is run through the same per-prospect machinery
   (STEP 3 onward), with a deeper audit. This is the "Guide to Harbor" button.
Both modes write the SAME legacy blob (STEP 7), push the SAME way (STEP 8), and log to
the SAME file (STEP 9). The only difference is the front: how the prospect is sourced.

==================================================
STEP 0, LOAD CONFIG
==================================================

Read `/Users/drew/Documents/Claude/StreamAd Lighthouse/lighthouse-config.json`. It is
the source of truth for ICP, doctrine, sender, brand, send safety, HubSpot, Apollo.

Bind:
- CFG = parsed JSON
- VOLUME = CFG.volume_per_sweep
- ALLOC = CFG.sweep_allocation (the default fixed mix)
- ICP = CFG.icp
- DOCTRINE = CFG.messaging_doctrine
- BRAND = CFG.brand
- SENDER = CFG.sender
- HUBSPOT = CFG.hubspot

If the file is missing or unparseable, halt and tell Drew. Do not guess.

VALIDATION (required) before STEP 1, confirm these exist and are non-empty, halt naming
any missing path:
- CFG.volume_per_sweep (int > 0)
- CFG.sweep_allocation.tier_1_primary_mission_driven / tier_2_secondary_home_services / tier_3_financial_institutions / backfill_tier
- CFG.brand.company, CFG.brand.url, CFG.brand.positioning
- CFG.sender.name, .email, .title, .calendar_url
- CFG.send_safety.cc_emails, .hubspot_bcc, .daily_send_cap
- CFG.hubspot.pipeline, .initial_stage, .owner_id
- CFG.apollo.email_gate
- CFG.icp.tier_1_primary_mission_driven, .tier_2_secondary_home_services, .tier_3_financial_institutions
- CFG.messaging_doctrine (string > 1000 chars)
- CFG.doctrine_version (string)

SYNC CHECK (required): run `bash /Users/drew/Documents/Claude/StreamAd Lighthouse/sync-config.sh`.
If it reports anything other than `RESULT: CLEAN. Sync successful.`, HALT and tell Drew
the exact failure. Stale doctrine/sender/HubSpot ids in the artifact would corrupt the run.

==================================================
STEP 0.5, RESOLVE THE SWEEP MIX (picker-aware)
==================================================

Two ways a sweep gets its mix:

1. DEFAULT MIX (no custom request). Use CFG.sweep_allocation as-is:
   tier 1 = 2, tier 2 = 2, tier 3 = 2, total = CFG.volume_per_sweep (6). Each tier
   pulls its full `industries_preferred` list per the ICP. If a tier underfills, backfill
   the shortfall from CFG.sweep_allocation.backfill_tier (tier 1).

2. CUSTOM MIX (Drew pasted a command from the artifact sweep picker, or asked in chat).
   As of 2026-06-01 the picker emits a list of PICKS, where each pick is count + size +
   vertical. The command looks like:
   ```
   Run a StreamAd Lighthouse sweep with these exact picks (total 9 prospects):
     - 6 Whale in Community Banks (Financial Institutions tier)
     - 3 Fish in HVAC (Home Services tier)
   ```
   Other pick shapes the picker can emit:
     - "4 any size in Credit Unions (Financial Institutions tier)"   (no size filter)
     - "2 Dolphin in Nonprofits & Charities (Mission-Driven tier)"
     - "1 Whale in Financial Institutions (whole tier per ICP)"      (whole tier, sized)
     - "3 Fish in any vertical (whole ICP)"                          (any vertical, sized)

   Honor each pick exactly:
   - The per-pick count is a hard quota. Do NOT exceed it. Treat each pick independently,
     two picks can name the same vertical at different sizes (e.g. 2 Whale + 2 Dolphin in
     Nonprofits), run them as separate quotas.
   - The vertical sets the Apollo industry filter:
     - A named chip ("Community Banks (... tier)") restricts that pick to the industries
       mapped by that chip (CFG.icp.sweep_picker.<tier>.chips[].industries).
     - "(whole tier per ICP)" uses that tier's full `industries_preferred`, no narrowing.
     - "any vertical (whole ICP)" searches across all three tiers' `industries_preferred`;
       each returned company's tier is set by its industry.
   - The size word sets the employee band for that pick. See SIZE BANDS below. "any size"
     means no size filter (today's default, take whatever the vertical returns).
   - Total may be any number Drew set, not capped at CFG.volume_per_sweep for a custom run.
     Sanity-check anything above ~15 with Drew before burning Apollo credits.
   - Backfill rule per pick: if a pick underfills, first widen the vertical (chip -> whole
     tier -> across tiers) while KEEPING the size band, then relax the size band only with
     Drew's okay. Never silently swap in a size or vertical Drew did not ask for.

SIZE BANDS (CFG.paid_intensity_rules, employees only, uniform for EVERY industry):
   Size is a SEARCH INPUT as well as the post-pull label in STEP 4g. The bands are flat
   across all tiers (Drew, 2026-06-01), employees only, revenue is NOT a size qualifier:
     - Fish:    1 to 20 employees      (CFG.paid_intensity_rules.size_bands_employees.fish)
     - Dolphin: 21 to 100 employees    (...size_bands_employees.dolphin)
     - Whale:   101 to 1000 employees  (...size_bands_employees.whale)
   Pass the exact Apollo buckets from CFG.paid_intensity_rules.apollo_num_employees_ranges
   to `organization_num_employees_ranges`:
     - Fish:    ["1,10", "11,20"]
     - Dolphin: ["21,50", "51,100"]
     - Whale:   ["101,200", "201,500", "501,1000"]
     - any size: the tier's default company.employee_count_range, unchanged
   When a size is requested, this band OVERRIDES the tier's default 20-500 employee range.
   The same bands apply regardless of the pick's vertical, including "any vertical (whole ICP)".
   Revenue is still pulled and shown on the card as an indicator, it just does not gate size.

Record the resolved picks (count + size band + industries) and use them through STEP 2-3.

==================================================
STEP 1, DUPLICATE SUPPRESSION (SHARED, PORTAL-WIDE)
==================================================

Pull existing HubSpot deals to avoid re-pitching anyone EITHER user has already
touched. This is intentionally portal-wide, NOT scoped to the current user, so a
prospect emailed or declined by one teammate is suppressed for the other and vice
versa. The shared HubSpot portal is the shared memory: emailed prospects are
sent-stage deals, declined or skipped ones are Closed/Lost deals, both appear here.

- `mcp__f1dc7048-6352-4a43-9b06-38672a23c164__search_crm_objects`, objectType=deals,
  filter `lighthouse_id` `HAS_PROPERTY` (every Lighthouse deal, ALL owners, ALL stages).
  Do NOT filter by `hubspot_owner_id` here. Scoping to one owner reintroduces cross-user
  double-pitching. (The current user's `HUBSPOT.owner_id` is still used at STEP 8 to stamp
  the NEW deals this sweep creates, just not for this suppression pull.)
- PAGINATE through every page (follow the `after` paging cursor, or step `offset` by the
  page size) until the results are exhausted. The combined two-user pile will exceed one
  page of 100, and a partial pull silently misses older suppressions.
- Extract every email and domain from deal description blobs (`contact.email` and
  `contact_email`) and associated contacts, across the full paginated set.
- Build SUPPRESSED_EMAILS and SUPPRESSED_DOMAINS from that full set.
- Never touch a domain in the suppression list, even if Apollo surfaces it.

==================================================
STEP 2, APOLLO COMPANY SEARCH, PER TIER
==================================================

Run a SEPARATE `mcp__9f57738b-534f-43ec-9b5d-6ef914c816b7__apollo_mixed_companies_search`
for each PICK in the resolved mix (default mix = one pick per tier at "any size").

Common filters (every pick):
- organization_locations = ICP.global_locations
- exclude any industry in ICP.global_exclude_industries

Per-pick filters:
- INDUSTRIES from the pick's vertical (STEP 0.5): a chip's `industries`, the whole tier's
  `industries_preferred`, or all tiers' `industries_preferred` for "any vertical".
- EMPLOYEE RANGE from the pick's SIZE BAND (STEP 0.5), flat across every industry, NOT the
  tier's default 20-500. Pass CFG.paid_intensity_rules.apollo_num_employees_ranges[size]:
    - Whale:   ["101,200", "201,500", "501,1000"]   (101 to 1000 employees)
    - Dolphin: ["21,50", "51,100"]                  (21 to 100)
    - Fish:    ["1,10", "11,20"]                     (1 to 20)
    - any size: the tier's default company.employee_count_range (20-500).
- TIER-SPECIFIC extras still apply on top of the size band:
    - tier 1 mission driven: q_organization_keyword_tags = company.keywords_company_signals;
      revenue is a proxy only (band is marketing budget), do not hard-drop on revenue alone.
    - tier 2 home services: revenue band $2M to $50M (company.annual_revenue_range_usd).
    - tier 3 financial: NO revenue band by design, qualify on institution type
      (also see company.type_list_from_drew).

For each candidate: drop if domain in SUPPRESSED_DOMAINS, drop if industry in
global_exclude, drop if employees fall outside the pick's size band. Rank within each pick
per ICP.outreach_prioritization. Oversample ~1.3x the pick quota to absorb email-verification
dropout in STEP 3.

==================================================
STEP 3, PEOPLE SEARCH AND EMAIL VERIFICATION
==================================================

For each ranked candidate company, find the best decision maker:
- `mcp__9f57738b-534f-43ec-9b5d-6ef914c816b7__apollo_mixed_people_api_search`
- organization_ids = the company id
- titles_any / seniorities / exclude_titles = that tier's `person` block
- Pick the most senior, most marketing-adjacent person. Capture full name, title,
  email_status, linkedin_url.
- LinkedIn URL is standard for every prospect. If the search omits it for the selected
  DM, enrich once via `apollo_people_match` (name + domain). If still none, set
  contact.linkedin_url to empty string and add a caveat, do not drop the prospect.

SECONDARY CONTACT (v6.1, STANDARD, for the redirect P.S.):
- For every prospect, pull a SECOND plausible contact on the same account from the
  same `apollo_mixed_people_api_search` results (or one more call). Pick the next most
  marketing or paid-advertising-adjacent person who is NOT the primary DM.
- Capture real name + real title verbatim into the blob as
  `secondary_contact{name,title}`. No placeholders, no guesses.
- If the account genuinely has no second contact, leave `secondary_contact` null. The
  drafter then falls back to the source-only P.S. This is the only case where the
  redirect P.S. is skipped (doctrine v6.1 makes the redirect standard otherwise).

EMAIL GATE (CFG.apollo.email_gate):
- 'verified': accept only email_status = 'verified'
- 'verified_or_likely': accept 'verified' or 'likely to engage'
Drop candidates with no email, a failing gate, or an email in SUPPRESSED_EMAILS.

Continue until each pick hits its resolved count, or its pool is exhausted. If a pick
exhausts short, apply the STEP 0.5 backfill rule (widen vertical first, keep the size band).

==================================================
STEP 4, PAID AUDIT (per prospect, before drafting)
==================================================

For each prospect (domain = company.primary_domain, brand = company.name):

4a. `mcp__Claude_in_Chrome__navigate` to https://{domain}, then
    `mcp__Claude_in_Chrome__javascript_tool` to capture `document.documentElement.outerHTML`.
4b. Discover one inner page: try https://{domain}/sitemap.xml, pick the first URL matching
    /contact|about|pricing|book|demo|shop|services|locations/i. If no sitemap, parse
    homepage anchors for the same pattern. If nothing, skip the inner page.
4c. If found, navigate and capture that HTML too.
4d. Detect pixels by regex across combined HTML and script src URLs, using
    CFG.pixel_taxonomy detection_patterns. Build `detected[]` (paid) and `tag_management[]`
    from positive matches only. Never invent. Capture real ids into `ids{}` where found.
4e. Write a 2 to 4 sentence factual HTML summary of only what was found, wrapping platform
    names in ad-library anchors (Google Ads transparency, Meta Ad Library, LinkedIn, TikTok)
    per the URL patterns in the doctrine.
4f. Build the 6-entry references array (Google Ads Transparency, Meta Ad Library, LinkedIn
    Ad Library, TikTok Ad Library, BuiltWith, SimilarWeb), set matched:true where detected.
4g. Compute `paid_intensity` (whale/dolphin/fish) per CFG.paid_intensity_rules, by Apollo
    employee count ONLY, uniform across every industry (per classification_note):
    - 1 to 20 employees  -> fish
    - 21 to 100          -> dolphin
    - 101 and above      -> whale (the 1000 in the band is the top pull bucket, not a
      classification ceiling, an org over 1000 still classifies whale)
    No revenue gate. No paid-activity promotion. A domain in CFG.paid_intensity_rules.
    known_whales_domains is forced to whale regardless of employees. If Apollo has no
    employee count, default per CFG.paid_intensity_rules.data_missing_default (fish) and set
    data_status: missing on the audit.
4h. Build the `ad_audit` object (audited_at, pages_audited, detected, tag_management, ids,
    summary, references, paid_intensity). Close all Chrome tabs opened for this prospect.
    If the audit fails entirely (timeout/block/blank), set ad_audit = null, the card shows
    "Paid audit pending."
4i. Capture website launch year via the Wayback sparkline fetch, store as `website_launched`
    (int year or null).
4j. Extract the site logo (header/nav/home-link img, apple-touch-icon, icon link), store as
    `logo_url` + `logo_source: "site-extraction"`, else null and let the card fall back.

NOTE: the audit only completes in a MANUAL run (Chrome permission). This is one of the main
reasons the sweep is manual now.

==================================================
STEP 5, ANNUAL CONTRACT POTENTIAL
==================================================

Estimate annual contract value by tier and size. Build the `potential` object
{range, estimate, label, rationale, services_fit}.

- TIER 1 mission driven: mid nonprofit ~$18K-$42K, established ~$36K-$84K, enterprise ~$72K-$180K.
- TIER 2 home services: growth ~$24K-$60K, mid-market ~$48K-$120K, enterprise ~$96K-$240K.
- TIER 3 financial: size by employees (no revenue band). Community/regional scale ~$36K-$96K,
  larger institutions ~$96K-$240K. Use judgment, banks skew large, contract follows footprint
  and number of markets, not Apollo revenue.

==================================================
STEP 5.5, ESTIMATED MEDIA BUDGET
==================================================

Rough annual paid media budget. Base rate by tier:
- TIER 1 mission driven: ~3% of revenue/budget
- TIER 2 home services: ~8%
- TIER 3 financial: ~5% (regulated, brand-heavy, but real paid budgets)

Sophistication multiplier from ad_audit: 0-2 pixels 1.0x, 3-4 pixels 1.15x, 5+ or any
independent DSP 1.3x. estimate = base * rate * multiplier, low = est*0.5, high = est*1.5.
If revenue is missing (common for tier 3), size from employees or set null. Build
{min, range, estimate, rationale}. `min` shows on the card, full range in Look Deeper.

==================================================
STEP 6, OUTREACH DRAFTS
==================================================

Apply DOCTRINE exactly (CFG.messaging_doctrine is the single source). Generate THREE drafts
per prospect, drafts[0] best fit, [1] and [2] meaningfully different angles. Honor the
seniority tier (CFG.seniority_messaging_tiers), anti-AI voice rules, personalization sources,
DSP mention rule, subject/close alignment, 80-120 word body, 6-word subject. Never hardcode
voice rules here that contradict the doctrine.

DELIVERY AND SIGNATURES (CFG.delivery): drafts reach the sender via the artifact's Open in
Outlook button, an OWA compose deeplink carrying To, Subject, and Body only (CC and BCC are
clipboard pastes, the sender reviews and sends from Outlook, nothing sends automatically).
NEVER include a signature, sign off, name, calendar link, or contact block in any draft
body, Outlook auto appends the sender signature at send time. This applies equally to
chat-generated follow ups, reply with subject, body, and a ready deeplink, no signature.

==================================================
STEP 7, BUILD THE PROSPECT BLOB (LEGACY SHAPE, CRITICAL)
==================================================

Write the LEGACY blob shape with a nested `contact` object. The artifact adapter blanks
name/title if you write the flat audit shape (contact_email/contact_name/paid_audit). Include:
name, domain, industry, location, employees, revenue, founded, website_launched, logo_url,
logo_source, locations, key_metric, estimated_media_budget, fit_score, lighthouse_id,
why_now, caveats, contact{name,title,email,linkedin_url,why_decision_maker},
secondary_contact{name,title} (v6.1, for the standard redirect P.S., null only if no
second contact exists), potential, drafts[3], signals{tech_gap,intent,growth}, ad_audit.
Add a `tier` field naming which tier this prospect came from. Add a `source` field:
"swept" for a normal sweep prospect, "manual_url" for a Guide to Harbor prospect. The
artifact reads this to stamp the card origin badge ("Swept" vs "Harbored"); a missing
source defaults to "Swept" for pre-existing cards. Do NOT include top-level
diagnosis/service_fit/gbp/paid_audit/contact_email/contact_name/contact_title/contact_why.

==================================================
STEP 8, PUSH TO HUBSPOT
==================================================

For each prospect, create a deal via
`mcp__f1dc7048-6352-4a43-9b06-38672a23c164__manage_crm_objects`,
confirmationStatus CONFIRMED, objectType deals, properties: dealname "{Company}, {Contact}",
dealstage HUBSPOT.initial_stage, pipeline HUBSPOT.pipeline, hubspot_owner_id HUBSPOT.owner_id,
amount = potential.estimate, description = JSON.stringify(STEP 7 blob), closedate = +90 days.

==================================================
STEP 9, LOG AND NOTIFY
==================================================

Append one JSONL line to `/Users/drew/Documents/Claude/StreamAd Lighthouse/sweep-log.jsonl`:
{run_at, mix_source ("default" | "custom"), prospects_pushed, tier_1_count, tier_2_count,
tier_3_count, audits_succeeded, audits_failed, apollo_credits_used_estimate, notes}.

Then a tight summary to Drew (max ~10 lines): how many pushed, per-tier breakdown, audits
succeeded vs failed, a reminder to click "Sync from HubSpot" in the artifact, and the top 3
by fit_score.

==================================================
GUIDE TO HARBOR (SINGLE-TARGET MODE)
==================================================

Drew hand-picks one or more specific URLs and each is guided into the pipeline as a
prospect card, badge "Harbored". This is the inverse of a sweep: no ICP query, no pick
mix, no tier quotas. The vessel is already chosen, you just bring it in. Each URL runs
the SAME per-prospect machinery (STEP 3 onward) plus a DEEPER audit, and produces the
SAME legacy blob, HubSpot deal, and log line as a swept prospect.

TRIGGER: Drew pastes a command from the artifact's "Guide to Harbor" button, or asks in
chat, of the form:
```
Run StreamAd Lighthouse "Guide to Harbor" on these URLs (single-target mode):
  - https://example.com | note: referred by Jim at the chamber lunch
  - https://acme-hvac.com
```
Each line is one URL with an OPTIONAL "| note: ..." context tail. Process every URL.

STEP H0, LOAD CONFIG + SYNC CHECK
- Same as STEP 0 and the STEP 0 SYNC CHECK above. Load CFG, bind the same values,
  run `bash sync-config.sh`, require RESULT: CLEAN before proceeding.

STEP H1, SUPPRESSION (SHARED, PORTAL-WIDE)
- Build SUPPRESSED_EMAILS / SUPPRESSED_DOMAINS exactly as STEP 1 (portal-wide, paginate).
- For each input URL, normalize to a bare domain (strip scheme, www, path, query, utm).
- If the domain is already in SUPPRESSED_DOMAINS (already on the board), DO NOT create a
  duplicate. Instead surface the existing deal: report its company name, HubSpot stage,
  and deal link so Drew can see where it stands, then move to the next URL.

STEP H2, COMPANY RESOLUTION (Apollo org enrich, replaces STEP 2)
- For each surviving domain, call
  `mcp__9f57738b-534f-43ec-9b5d-6ef914c816b7__apollo_organizations_enrich` on the domain.
- If Apollo returns NO org for the domain, report "No data found in Apollo." for that URL
  and move on. No card is built for it. (A hand-picked URL with no Apollo record is the one
  case Guide to Harbor produces nothing.)
- If an org is returned, classify it to the CLOSEST tier by its industry (map the Apollo
  industry to tier 1 / 2 / 3 using the ICP `industries_preferred`). If it maps to none of
  the three, default to TIER 1 for the potential/budget math and add a caveat to the card:
  "Off-ICP, hand-picked URL, tier is best-effort." Never reject a hand-picked URL for being
  off-ICP, the whole point is Drew chose it deliberately.

STEP H3, PEOPLE + EMAIL (per STEP 3, with a loosened fallback)
- Run STEP 3 against the resolved org id: primary decision maker via
  `apollo_mixed_people_api_search` using the classified tier's `person` block, plus the
  STANDARD secondary contact for the redirect P.S.
- NO-DM FALLBACK (single-target only): if no person matches the tier's strict title filter,
  DROP the title filter and take the most senior marketing-adjacent person Apollo has for
  that org. Capture name, title, email_status, linkedin_url as usual. Do not leave contact
  null when a real person exists, loosen instead.
- EMAIL GATE, RELAXED FOR HAND-PICKED URLS: apply CFG.apollo.email_gate as normal, but if
  the selected DM's email fails the gate, DO NOT drop the prospect (a sweep would). Build the
  card anyway, keep the email, and set a caveat "Email unverified, confirm before send" and
  flag it on the contact. Drew decides whether to send. The Send button already disables when
  there is genuinely no email; an unverified-but-present email shows with the caveat.

STEP H4, DEEPER PAID AUDIT (extends STEP 4)
- Run STEP 4 (4a-4j) in full, then go deeper because this is a single hand-picked target,
  not one of six in a batch:
  - MORE PAGES: discover and audit up to 3 inner pages (not just 1), preferring
    /pricing|services|about|contact|locations|book|demo/ so the pixel sweep is thorough.
  - LIVE AD CHECK: do not stop at "is a pixel present". Open the ad-library URLs already in
    the references array (Google Ads Transparency, Meta Ad Library) via Chrome and record
    whether the advertiser is running LIVE ads right now. Store on the audit as
    `live_ads{google: bool|null, meta: bool|null, notes}`. A live ad is a far stronger
    signal than a tracking pixel and feeds the draft's "why now".
  - HIRING SIGNAL: call
    `mcp__9f57738b-534f-43ec-9b5d-6ef914c816b7__apollo_organizations_job_postings` for the
    org. If they are hiring a marketing, growth, demand-gen, or paid-media role, capture it
    as `hiring_signal{titles:[...], note}` on the blob. This is a timely, specific outreach
    hook a broad sweep never surfaces.
- Everything else (paid_intensity by employee count, Wayback launch year, logo) is unchanged.
  The whale/dolphin/fish size tag and the tier tag render on the Harbored card exactly as on
  a swept card.

STEP H5, POTENTIAL + BUDGET (STEP 5 and 5.5, unchanged)
- Use the classified tier from H2 for the contract-potential and media-budget math.

STEP H6, DRAFTS (STEP 6, with the per-URL note as context)
- Generate the three drafts per the doctrine exactly as STEP 6. If the URL carried a
  "| note: ..." tail, treat that note as a high-priority personalization source: weave the
  real reason Drew picked them into draft[0]'s opening where it reads naturally (referral
  name, where he saw them, the event). Never fabricate around the note, use it verbatim in
  spirit. If there is also a hiring_signal or a live_ads finding, those are available as
  additional "why now" hooks. Drew's note wins when both exist.

STEP H7, BLOB (STEP 7) — set `source: "manual_url"`, add `harbor_note` (the verbatim note or
null), `hiring_signal` (or null), and `live_ads` (or null) to the blob. Everything else per
STEP 7. The off-ICP caveat from H2, the unverified-email caveat from H3, and any audit
caveats all go in `caveats`.

STEP H8, PUSH TO HUBSPOT (STEP 8, unchanged) — same deal creation, dealname "{Company}, {Contact}".

STEP H9, LOG + NOTIFY — append one JSONL line as STEP 9 but with
`mix_source: "guide_to_harbor"`, `prospects_pushed` = how many cards were created,
`urls_in` = how many URLs were submitted, `no_apollo` = how many returned no Apollo data,
`suppressed` = how many were already on the board. Then a tight per-URL summary to Drew:
for each URL, Harbored (with tier + size), already-on-board (with stage), or no-Apollo-data.
Remind him to click "Sync from HubSpot" in the artifact.

GUIDE TO HARBOR GUARDRAILS (in addition to the global ones below):
- Never auto-send. Harbored prospects are drafts like any other, Drew sends from the artifact.
- Never duplicate a domain already on the board, surface the existing card instead.
- The relaxed email gate is single-target ONLY. A broad sweep still drops on a failed gate.
- Close every Chrome tab opened during the deeper audit, even on error.

==================================================
GUARDRAILS
==================================================

1. Never send outreach automatically. Drafts go to HubSpot, Drew sends from the artifact.
2. Never skip the email-verification gate. Drop the prospect instead.
3. Never exceed the resolved per-pick counts (default mix or custom picks).
4. Never touch a domain in SUPPRESSED_DOMAINS.
5. Always write the LEGACY blob shape (STEP 7).
6. On Apollo/HubSpot rate limit or auth failure, log it, push the partial batch, notify Drew.
7. Close every Chrome tab you opened, even on errors.
8. The picker only shapes the mix (counts, sizes, verticals). All sweep logic, including
   the size-band resolution, lives here, one source, never duplicated into the artifact.
