---
name: kivo-studio-production
description: Use when making films or short dramas with Kivo Studio MCP, including multi-agent handoffs, scene consistency, draft sampling, production upgrades, audiovisual QC and delivery.
---

# Kivo Studio multi-agent film production

Use **Kivo Studio** ([kivst.com](https://kivst.com)) through its MCP connector. This is independent guidance derived from recent team production, not an official service or quality guarantee. Read [MCP_CONFIG.md](MCP_CONFIG.md) before tool use.

## 1. Assign roles and artifact ownership

| Role | Handoff |
|---|---|
| Director | Shot list, continuity rules, candidate MAIN/backup selections, lock ledger and acceptance gates |
| Writer | Versioned script, dialogue master, subtitle text; no silent edits after timing lock |
| Asset/scene artist | Approved character, costume, location, prop and voice references with inspection notes |
| Storyboard operator | Saved prompt/version, draft candidate receipts and selected job/asset mapping |
| Editor | EDL, timeline, sound, screen-text repairs, subtitles and QC |
| Technical coordinator | Quotes, authorized submissions, task ledger and canonical delivery manifest |

One owner per mutable artifact. Parallel agents work on separate shots or copies. Handoffs must record revision, owner, approval state and unresolved flags. Creative approval does not authorize spending: obtain user-approved budget/scope from the payer before generation. Stop paused shots. One coordinator owns final uploads; sharing requires explicit authorization for the destination/content.

## 2. Lock story and build consistent scenes

Lock script, dialogue, shot purpose, duration target and aspect ratio. Maintain a character/scene bible with appearance views, costume, voice, props, geography, lighting direction, camera axis and continuity state. Inspect and approve default references before prompt binding. References improve control but cannot guarantee identity or continuity.

Upload only authorized material: `kivo_assets_prepare` → client signed S3 POST of real bytes → `kivo_assets_complete` → `kivo_assets_list` until ready. Never send base64 via MCP. Set `rightsConfirmed: true` only with user authorization; do not assume real-person consent. `kivo_assets_addToLibrary` is for ready generated results, not a required step for uploaded references.

Save VIDEO prompts with stable `@[name](asset:ID)` notation and `referenceIds` using `kivo_prompts_save`; updates use `promptId` and `expectedVersion`. Keep precise numbers/logos/UI blank or blurred where possible and composite them later. Review adjacent shots for eyelines, screen direction, light, costume and props.

## 3. Sample before scaling

Read current catalog per model. Use saved `promptId`/`promptVersion` for `stage=draft`, initially with a small approved candidate count. As an additional recommended gate, test a representative mini-sequence covering dialogue close-up, wide geography, a continuity transition and difficult action/UI. This sampling plan is a recommendation, not a measured API feature.

Compare candidates for identity, composition, performance, timing and lip sync; lock MAIN and optional sibling backup. Assemble a continuous sample to reveal cut/sound problems. A sample with disclosed soft flags is not whole-film acceptance.

Each shot handoff includes stable shot label, prompt ID/version, selected draft `sourceJobId`, output `assetId`, model, duration, ratio, audio setting, local file/checksum, approval and soft flags. Exclude stopped and post-only shots. Hash immutable parents with SHA-256; edits get new filenames.

## 4. Quote, authorize, create and track

1. Discover live tools; read `kivo_catalog_get` and authorized `kivo_projects_get` context. The project creator pays, not necessarily the connected account.
2. `kivo_jobs_quote` the exact selection. Present credit cost and scope for user review; quote does not reserve credits.
3. Only within spending authorization, call `kivo_jobs_create` with quoted `acceptedCredits`, `billingVersion`, `modelVersion` unchanged and a unique logical-request `idempotencyKey`.
4. Poll persisted tasks with `kivo_jobs_get`/`kivo_jobs_list`, bounded and roughly 10 seconds apart. queued/running/saving are not failures. Use a deadline/backoff; report unfinished work instead of looping forever.
5. Never duplicate pending or `submission_unknown` tasks. On uncertain submission or timeout, reconcile existing jobs before retrying; do not blindly change the key. Retain safe diagnostics. Only a genuinely new authorized attempt uses a new key and optional `retryOf` under current semantics.
6. On confirmed success, request its output asset through `kivo_assets_url`; download privately. Signed URLs expire and are not permanent references.

Current catalog ceilings are 10 jobs/request and 50 active/account; schema count is 1–10. Re-read at runtime. The create schema takes one request object, not a jobs array. Team batches of up to ten separate calls are orchestration practice, not an API batch payload. Count actual tasks including candidates/collaborators; avoid race conditions and do not treat capacity as a throughput target.

## 5. Upgrade locked drafts only

Production uses `stage=production` and a successful unexpired same-model **Kivo** `sourceJobId`, not a raw supplier ID. Match the director's selected source asset before quoting. Preserve prompt ID/version for traceability and pass when appropriate; quote does not universally require them in its input schema.

Production inherits duration and references. Select final resolution from catalog, handling singular/plural draft fields and preserving `4K` spelling. Verify a small upgrade selection first, then scale within budget. Replace plates in the EDL and recheck cuts/audio. Higher resolution does not fix acting, continuity or lip-sync defects. Never upgrade post-only shots. TTL and prices are runtime values.

## 6. Audiovisual and subtitle QC

Team practice used a 24 fps timeline, H.264/yuv420p and AAC stereo 48 kHz: project choices, not Kivo requirements. Select a common timebase deliberately, normalize source clips/audio before concatenation and verify compatibility before stream-copy. PCM intermediates and one final AAC encode can reduce repeated lossy encoding.

Local metadata inspection (requires ffprobe):

```bash
ffprobe -v error -show_streams -show_format -of json "clip.mp4"
```

Check stream starts/durations, time bases, nominal/average frame rates, sample rate and codec against EDL and decoded frame counts. Metadata alone cannot prove sync. Review the assembled film continuously with headphones; inspect cuts and beginning/middle/end for progressive drift. Agree project-specific tolerances before sign-off.

- Container alignment is not lip sync. Inspect visible speech frame-by-frame against audible consonants. Prefer another candidate, bounded regeneration or ADR for source performance errors; global offset is not a universal fix.
- ASR can assist dialogue checks but cannot prove alignment. Inspect missing ambience, clipped speech, noise, abrupt cuts and loudness continuity.
- Record owner and disposition for soft flags; unresolved hard gates block final acceptance.
- Separate untimed subtitle master from aligned SRT/ASS/TSV. Preview soft subtitles first; burn a separate approved version. Mark inaudible/skipped dialogue instead of phantom subtitles.
- Check bilingual safe margins, UI overlap and screen-text repairs. Preserve an immutable clean parent for masking/compositing.

## 7. Recovery and delivery

Retry confirmed failed jobs only, with bounded serial attempts and no repeats of successes. Cancellation only works while queued before provider submission. Diagnose quote errors using stage/model/source/prompt version/resolution. A remembered missing-prompt-ID INVALID_FORMAT incident remains unconfirmed; do not turn it into an API rule.

Deliver clean master, optional burned-subtitle version, timed subtitle package, manifest (filename, duration, SHA-256, subtitle state) and QC notes/unresolved items. Verify current host attachment limits; there is no universal 25 MB Kivo limit. Use approved private storage; public links require explicit approval and expiry disclosure.

Never publish real project/job/prompt/asset IDs, private scripts/character names, receipts, emails, credentials, signed URLs or checkout sessions. Keep runtime artifacts outside the skill package. See [README.md](README.md).
