# Kivo Studio MCP integration and verification

Checked 2026-11-08 (Asia/Shanghai) using live tool discovery in `user-kivst` and one read-only `kivo_catalog_get`. No generation, quote, payment, project mutation or upload was executed. Input schemas establish syntax; descriptions document intended semantics, not independent tests of every backend behavior.

## Connection boundary: no invented configuration

Kivo Studio: [kivst.com](https://kivst.com). Where supported, install/connect the Kivo/kivst connector using the host's supported workflow and let the user authenticate through its trusted UI. Discover current tools before invoking them.

In this host, discovery is `GetDynamicTools(namespace="user-kivst")`; invocation is `CallDynamicTool(namespace="user-kivst", toolName=..., arguments=...)`. These are host-specific wrappers, not portable MCP methods or Kivo endpoints. Other clients use their standard MCP tool discovery/invocation; namespace prefixes may differ.

For another IDE/CLI, obtain the official supported server endpoint, transport, authentication method and setup instructions from Kivo Studio's official product/support channel. This review did not verify a public MCP endpoint, OAuth authorization/token URLs/scopes, API-key format or standalone config. No runnable endpoint/auth JSON is supplied. Do not guess URLs from the website, extract session credentials or substitute undocumented APIs. If official setup is unavailable, stop integration at this boundary.

## Complete current inventory (28 tools)

- Account/catalog: `kivo_account_get`, `kivo_catalog_get`, `kivo_credits_list`, `kivo_orders_list`, `kivo_payments_checkout`.
- Projects: `kivo_projects_create`, `kivo_projects_get`, `kivo_projects_list`, `kivo_projects_update`.
- Assets: `kivo_assets_prepare`, `kivo_assets_complete`, `kivo_assets_list`, `kivo_assets_url`, `kivo_assets_addToLibrary`, `kivo_assets_removeFromLibrary`, `kivo_assets_rename`, `kivo_assets_delete`.
- Prompts: `kivo_prompts_save`, `kivo_prompts_list`, `kivo_prompts_history`, `kivo_prompts_delete`, `kivo_prompts_optimize`.
- Jobs: `kivo_jobs_quote`, `kivo_jobs_create`, `kivo_jobs_get`, `kivo_jobs_list`, `kivo_jobs_cancel`, `kivo_jobs_favorite`.

Inventory is not authorization for purchases, generation, deletions, membership changes or sharing.

## Schema facts and description-level conditions

`kivo_jobs_quote` requires `projectId`, `modelId`. `kivo_jobs_create` requires `projectId`, `modelId`, `name`, `idempotencyKey`, `acceptedCredits`, `billingVersion`, `modelVersion`.

Both allow only those fields plus `stage`, `promptText`, `referenceIds`, `promptId`, `promptVersion`, `sourceJobId`, `parameters`, `retryOf`, `promptOptimization`. Extra properties are forbidden. Stage enum: `asset|draft|production` (no preview stage). Parameters allow only `resolution`, `ratio`, `duration`, `count`, `format`, `generateAudio`; count is integer 1–10. Reference IDs are unique, at most 15 in schema, but model limits may be stricter. Prompt version is integer ≥1, credits integer ≥0. IDs are nonempty strings up to 100 characters, job name up to 80, prompt text up to 12000. Syntactically optional fields can be conditionally required by stage semantics.

Descriptions specify saved prompt ID/version for drafts; successful unexpired same-model Kivo source job for production, inheriting duration/references. Quote first and preserve returned credit/version fields unchanged. Do not infer modelVersion solely from top-level catalog version; resolve missing quote values using current documented semantics before create. `promptOptimization` is only typed as object: internal shape is unverified, so omit unless documented for the use case.

`kivo_prompts_save`: required projectId/name/text; optional referenceIds/promptId/expectedVersion. Updates use promptId and expectedVersion; saved reusable prompts are VIDEO prompts. `kivo_prompts_optimize`: required projectId/requestId/text; optional modelId/type/referenceIds/parameters. Type is image/video/audio. It does not start generation; other costs were not tested.

`kivo_assets_prepare`: required projectId/name/contentType/byteSize/category/rightsConfirmed (const true); optional noRealPeople. Categories: scene/prop/character/sound. Upload real bytes to returned signed S3 POST outside MCP; complete(assetId), then list(projectId, scope=library|all) until ready. assets_url requires assetId, optionally preview/download, and returns expiring URLs. addToLibrary is for ready generated results.

## Async and limits: evidence levels

- Tool description: jobs_get(jobId) reads persisted state without supplier polling/generation. Use bounded polls roughly 10 seconds apart. queued/running/saving are not failures.
- Description: jobs_list(projectId) reads persisted jobs; background scheduler updates supplier state independently. Output includes asset IDs and task diagnostics. Response schemas are not exposed here: do not invent a complete terminal-state enum or unverified response-property spelling.
- Description: do not duplicate pending or submission_unknown tasks. For unknown submission retain IDs/diagnostics, investigate before retry. jobs_cancel only works before supplier submission while queued.
- Live catalog: maxJobsPerRequest=10, maxActiveJobsPerAccount=50, projectMembers=100. Re-read at runtime. Create takes one object, not a job array. Schema count=1–10 is distinct from ten separate orchestration calls. Track actual created/active tasks and collaborator activity.
- Team practice: bounded separate-call batches and serial retries. This is not proof of a batch endpoint or permission to saturate limits.
- Description: project creator pays after delivery. projects_get provides role/payer balance; non-creator permission/accounting edge cases were not exercised.

## Catalog snapshot (not hardcoded defaults)

| Model ID | Current observed capability |
|---|---|
| seedream | Image 1K/2K, up to 4 image references |
| seedance | Video 4–30 whole seconds; draft 480p → final 1080p |
| gemini-omni | Video 3–15 whole seconds; direct/draft 360p or 720p → final 1080p or 4K |
| audio-create | Audio; read current duration/formats/reference/billing rules |
| vda-small | Depth-video output; read current source constraints |

Seedance exposes singular draft.resolution/finalResolution; Omni also exposes plural arrays. Both video models currently report TTL 604800 seconds; refresh before production. Omni supports up to 6 images, up to three reference videos ≤3 seconds each for output ≤10 seconds, only one video for longer output, and no audio references. Current catalog alwaysGenerateAudio=true: do not promise generateAudio=false produces silence. Apply aggregate and per-type limits together; never transfer limits between models.

## Request plan (pseudocode, not client setup)

1. Discover tools; read catalog and authorized project context.
2. Quote projectId/modelId/stage=production/sourceJobId/parameters.resolution from the approved selection and catalog; retain draft prompt ID/version where appropriate.
3. Get spending authorization. Create one object with same selection, required name, quoted credit/version fields and logical-request idempotency key.
4. Reconcile timeouts before retrying. Poll boundedly; on confirmed success request output asset URL and download privately.

## Unverified boundaries

Independent-client endpoint/auth; promptOptimization structure; response schemas and exact expiry/error codes; reproducibility of remembered missing-prompt-ID INVALID_FORMAT; host attachment limits; and cross-member accounting/permissions. These are not advertised as guaranteed behavior. No fake credentials or endpoint snippets are included.
