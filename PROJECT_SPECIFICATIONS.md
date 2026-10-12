# Factory Vision — Detailed Project Specifications

**Project:** Intelligent Industry / Factory Vision  
**Creator:** Hossam Elshahaby  
**Specification version:** 1.0  
**Date:** October 12, 2026  
**Live application:** [https://bright-eye-detect.lovable.app](https://bright-eye-detect.lovable.app)  
**Video walkthrough:** [https://youtu.be/HGintBYyeGU](https://youtu.be/HGintBYyeGU)  
**Companion README:** [README.md](./README.md)

## 1. Document scope and status

This document describes the current source implementation and its intended industrial use. It distinguishes implemented behavior from desired outcomes, acceptance checks, and future improvements. It is not a claim of live operational certification or a fresh hosted-settings audit.

**Current product scope:** AI-assisted inspection of submitted photos and sampled video frames, individual-account inspection records, evidence-linked findings, and generated reports.

**Not current scope:** Autonomous factory control, continuous surveillance, voice commands, sensor telemetry, calibrated fault prediction, team administration, or certified safety assessment.

## 2. Problem statement

Factories often rely on fragmented visual checks and manually written reports. Visible wear, leaks, surface defects, contamination, and safety hazards can be missed or recorded without clear evidence and prioritized action.

Factory Vision provides a browser-based workflow to capture visual evidence, interpret visible issues with AI assistance, retain findings, and produce readable reports for human review.

## 3. Objectives and outcome measurement

| Objective | Intended contribution | Suggested validation metric—not currently tracked |
| --- | --- | --- |
| Reduce downtime | Identify visible deterioration before escalation. | Unplanned downtime per asset before/after an evaluated pilot. |
| Improve product quality | Flag visually supported defects for review. | Defect recall/precision on labeled factory examples; scrap and rework rates. |
| Improve safety | Highlight visible PPE issues and hazards. | Verified hazards found and corrective-action completion time. |
| Improve resource use | Suggest actions that may reduce waste and rework. | Scrap, material waste, and energy per output unit. |
| Improve documentation | Produce reports tied to stored findings. | Report preparation time and evidence completeness. |

These are proposed pilot measures. The app does not currently measure savings, downtime, actual energy use, or incident reduction.

## 4. Stakeholders and access model

| Audience | Typical task |
| --- | --- |
| Operator | Capture a part, machine, or work area and review flagged issues. |
| Quality inspector | Verify defect evidence and assess product disposition. |
| Maintenance technician | Review wear/leak indications and recommended investigation. |
| Safety officer | Review visible hazards and initiate appropriate safety action. |
| Plant manager | Read inspection summaries and prioritized reports. |

All signed-in accounts use the same application capabilities. These stakeholder descriptions are not implemented authorization roles. There are no organization memberships, shared inspection workspaces, or manager-wide views.

## 5. Functional requirements and current implementation

| ID | Requirement | Current behavior |
| --- | --- | --- |
| AUTH-01 | Authenticate a user before inspection. | Email/password and Google sign-in actions; protected layout redirects unauthenticated users to `/auth`. |
| AUTH-02 | Support account creation. | Email/password signup form; confirmation message shown if no session is returned. Hosted confirmation policy is external configuration. |
| CAP-01 | Acquire visual evidence on mobile. | Image input with rear-camera capture hint, image gallery, and video file picker. |
| CAP-02 | Support multiple views. | Up to 6 photo frames; previews and removal controls. |
| CAP-03 | Prepare video for analysis. | Extract 6 midpoint samples across equal time segments in a decodable video. |
| CAP-04 | Attach industrial context. | Optional line, machine, shift, and analysis notes. |
| AI-01 | Analyze visible issues. | Authenticated server function sends prepared frames/context to Lovable AI Gateway. |
| AI-02 | Structure evidence-linked results. | JSON schema returns title, summary, overall severity, risk, four impact strings, and findings with frame/bbox references. |
| RES-01 | Present evidence and priorities. | Annotated frames, finding list, confidence, recommendations, severity, risk, and impact. |
| RES-02 | Track inspection disposition. | Inspection-level Resolve/Reopen update. |
| REP-01 | Generate an evidence-based report. | Prompt-generated Markdown from stored metadata and findings. |
| REP-02 | Persist and regenerate a report. | Save `report` text; Redo overwrites it. |
| REP-03 | Export portable documentation. | Browser-generated A4 PDF, including up to 4 evidence images and drawn boxes. |
| REP-04 | Share report content. | Native text share or clipboard fallback; no public share URL. |
| HIST-01 | Retrieve earlier inspections. | Up to 200 newest owned inspections. |
| HIST-02 | Search and filter. | Client-side text matching and severity/status filters. |
| HOME-01 | Summarize recent activity. | Signed-in floor-status overview using the same 200-row dataset. |

## 6. Pages and navigation

| Browser path | Access | Contents |
| --- | --- | --- |
| `/` | Public/session-aware | Cinematic introduction when signed out; floor-status overview when signed in. |
| `/auth` | Public | Sign in, create account, Google action. |
| `/inspect` | Authenticated | Evidence preparation, optional context, submit. |
| `/history` | Authenticated | Inspection list, text search, severity/status filters. |
| `/inspection/:id` | Authenticated | Annotated evidence, findings, impact, risk, Resolve/Reopen, report navigation. |
| `/inspection/:id/report` | Authenticated | Saved/generated report, PDF, Share, Redo. |

`_authenticated` is a pathless source layout; it is not part of the browser URL. Its route uses client-side authentication checking and disables SSR for the protected branch. Shared navigation uses a side rail on desktop and bottom tabs with prominent Inspect access on mobile.

## 7. End-to-end inspection flow

1. User signs in and opens the capture page.
2. Browser decodes selected media and produces JPEG frame data URLs.
3. User reviews frames and provides optional context.
4. Browser creates an owned inspection with initial title `Analyzing…`.
5. Browser uploads each frame to the private evidence bucket.
6. Successfully uploaded object paths are written to the inspection.
7. The authenticated analysis server function sends all prepared frames and context to AI.
8. Structured findings are inserted; the inspection summary, risk, severity, impact, and status are updated.
9. The inspection list cache is invalidated and the result page opens.
10. Opening an unsaved report triggers generation from stored inspection/findings.
11. User reviews/export/shares the report and may resolve/reopen the inspection.

### State model

Capture UI stages are `idle`, `prep`, `upload`, and `ai`. A persistent processing/completed/failed state machine is not present.

Stored inspection statuses are text values conventionally used as `open` and `resolved`:

```text
New row: open
    |
    +--> AI returns findings: open
    |
    +--> AI returns no findings: resolved

open <---- user Resolve / Reopen ----> resolved
```

These values are application conventions, not database enum/check constraints. Resolving does not rerun analysis or change risk/findings. An AI failure attempts to delete the new inspection; this is not a transactional rollback of all uploaded evidence.

## 8. Media specification

| Property | Specification |
| --- | --- |
| Accepted picker types | `image/*`, `video/*`; actual decoding depends on browser support. |
| Photo maximum | 6 prepared frames retained by the interface. |
| Video maximum used | 6 frames from one chosen video. |
| Image size | Maximum longest edge 1280 px, aspect ratio preserved, no upscale. |
| Format | JPEG data URL for AI; JPEG Blob for storage. |
| Quality setting | 0.82. |
| Video sample time | `duration × (i + 0.5) / count`, for zero-based sample index `i`. |
| Stored object path | `<user_id>/<inspection_id>/<frame_index>.jpg` in `inspection-media`. |
| Original files | Not uploaded/archived by this workflow. |
| Server frame validation | Nonempty frame array, maximum 8. |
| Evidence URL lifetime | 3600 seconds when issued. |

No explicit file-byte limit, maximum video duration, frame-processing timeout, or robust seek-failure timeout is defined by this application code. Infrastructure/browser constraints can still limit requests. Fine defects may be lost during resizing/compression, and brief video events may occur between samples.

## 9. AI analysis contract

### Transport and execution

- Entry point: `analyzeInspection`, authenticated POST server function.
- Input: `inspectionId`, array of frame strings, optional `context`.
- Gateway transport: raw streamed `/v1/responses` request through the centralized helper.
- Configured model: `openai/gpt-6-astra`.
- Reasoning effort: `low`; image detail: `high`.
- Request settings: `stream: true`, `store: false`.
- UI result: completed structured response; no live token output.

### Output fields

| Field | Meaning |
| --- | --- |
| `title` | Prompt requests a 3–6 word inspection name. |
| `summary` | Short interpretation of the visible scene/issues. |
| `overall_severity` | `none`, `low`, `medium`, `high`, or `critical`. |
| `risk_score` | Integer-like model risk estimate; rounded/clamped to 0–100 before persistence. |
| `impact` | Qualitative strings for downtime, quality, safety, resources. |
| `findings` | Array of evidence-linked issues; empty for no identified issue. |

Each finding requires `frame_index`, `category`, `label`, `severity`, `confidence`, `bbox`, `description`, and `recommendation` in the gateway schema. Individual finding severity excludes `none`.

Categories: `crack`, `corrosion`, `leak`, `misalignment`, `wear`, `contamination`, `missing_part`, `surface_defect`, `overheating`, `safety_ppe`, `safety_hazard`, `spill`, `housekeeping`, `other`.

Bounding boxes use normalized top-left `x`, `y`, and width/height `w`, `h`. Coordinates and confidence are independently clamped to 0–1. A sum such as `x + w` is not explicitly constrained, and there is no additional complete runtime schema validation after JSON parsing.

### Risk presentation

| Score | Gauge color role |
| --- | --- |
| 0–24 | None/clear tone |
| 25–49 | Medium tone |
| 50–74 | High tone |
| 75–100 | Critical tone |

Gauge color thresholds are independent of the model's overall-severity field. Confidence is not a calibrated statistical likelihood; risk is not an engineering safety calculation.

### Interpretation constraints

The prompt instructs the model to report only visible evidence, provide concrete recommended actions, and return no findings when nothing appears wrong. These are model instructions, not proof against missed defects or hallucination. Hidden faults, temperatures, internal material condition, and future failures cannot be established from ordinary photos alone.

## 10. Report specification

### Inputs

Inspection title, context fields, creation date, severity, risk, summary, impact, and finding labels/categories/descriptions/recommendations/confidence are supplied as JSON text. The report call does not send the evidence images again.

### Required-by-prompt structure

1. **Executive summary:** major conclusions and priority.
2. **Evidence & findings:** references F1, F2, etc., with frame, severity, and confidence.
3. **Likely root causes:** hypotheses based on available findings.
4. **Business impact:** downtime, quality, safety, resources.
5. **Prioritized actions:** numbered proposed actions with owner role and urgency.
6. **Preventive recommendations:** follow-up prevention measures.

The prompt requests fewer than 450 words. Word count and section presence are not enforced by a post-generation validator. Proposed owner roles do not create assigned tasks. Finding order in the report query is not explicitly sorted, so persistent evidence numbering should not be treated as a versioned identifier.

### Persistence, export, and share

- Store one Markdown report in `inspections.report`.
- Generate automatically on first report-page visit if absent.
- Regenerate on Redo; no report revision archive.
- Export an A4 PDF via dynamically imported jsPDF.
- Include at most the first four stored evidence images; draw bounding rectangles.
- Apply simple heading/emphasis/list conversion rather than full Markdown layout.
- Save a filename based on the inspection ID prefix.
- Native share sends title and report text; fallback writes text to clipboard.

PDF metadata wrapping, mixed image aspect ratios, rich formatting, skipped-image disclosure, and first-generation error recovery require dedicated testing/hardening.

## 11. Data model

### `public.inspections`

| Field | SQL type | Default / purpose |
| --- | --- | --- |
| `id` | UUID, primary key | Generated UUID. |
| `user_id` | UUID, required | Owner; no foreign key to managed auth tables. |
| `title` | text, required | `Inspection`; updated by analysis. |
| `line`, `machine`, `shift` | nullable text | Optional context. |
| `media_type` | text, required | `image`; workflow also uses `video`. |
| `media_paths` | text array, required | Empty array initially; stored JPEG object paths. |
| `status` | text, required | `open`; workflow uses open/resolved. |
| `overall_severity` | text, required | `none`. |
| `risk_score` | integer, required | 0. |
| `summary` | nullable text | AI summary. |
| `impact` | JSONB, required | Empty object initially; four impact strings after analysis. |
| `report` | nullable text | Current generated Markdown. |
| `created_at` | timestamptz, required | Current time at insertion. |

### `public.findings`

| Field | SQL type | Default / purpose |
| --- | --- | --- |
| `id` | UUID, primary key | Generated UUID. |
| `inspection_id` | UUID, required | Inspection FK with cascade deletion. |
| `user_id` | UUID, required | Finding owner. |
| `frame_index` | integer, required | 0; AI frame reference. |
| `category`, `label`, `severity` | text, required | Classification and priority. |
| `confidence` | real, required | 0; model confidence. |
| `bbox` | nullable JSONB | Normalized rectangle. |
| `description`, `recommendation` | nullable text | Explanation and action. |
| `created_at` | timestamptz, required | Current time. |

Indexes cover `findings(inspection_id)` and `inspections(user_id, created_at DESC)`. There are no report-version, original-note, verified-by, resolved-at, corrective-action, organization, or immutable audit-event columns. Status/category/risk/bbox validity is not enforced with database-level check constraints.

## 12. Security and privacy model

### Implemented controls

- Both application tables enable row-level security.
- Authenticated CRUD grants are paired with owner policies: existing and inserted row `user_id` must match the authenticated user.
- Storage select/insert/delete policies constrain objects to the authenticated user's first path folder in the evidence bucket.
- Protected server functions use the generated auth middleware and the caller's Cloud client.
- Client-side function middleware attaches the session bearer token.
- Request middleware installs CSRF protection for server functions.
- AI credentials are read only inside the server helper and are not returned to the browser.

### Important gaps and boundaries

- The finding owner policy does not additionally validate parent-inspection ownership; the parent FK proves existence, not matching ownership. Tighten this before relying on strong multi-tenant integrity.
- The analysis handler does not explicitly preflight inspection ownership before spending AI resources; row policies constrain subsequent operations.
- Findings and inspection updates are not one atomic transaction; repeated requests can produce duplicate findings.
- Signed media links grant access until expiry to whoever possesses them.
- Row deletion cascades to findings but not to storage objects.
- No dedicated retention scheduler, account-data deletion workflow, audit log, or factory-specific compliance controls are present.

Industrial images may contain intellectual property or identifiable employees. Establish permitted capture/processing practices, consent where applicable, data retention, incident response, and access reviews before operational deployment. Do not infer provider retention or regional processing guarantees solely from `store: false`.

## 13. Architecture and configuration

React 19/TypeScript uses TanStack Start v1 and file-based TanStack Router routes. TanStack Query handles browser data reads. Tailwind CSS v4 and shared UI primitives supply styling; jsPDF generates downloads. The declared Vite dependency is currently 8.1.5; documentation should follow package declarations rather than older template assumptions.

Application-internal AI operations remain authenticated TanStack server functions, not separately deployed edge endpoints. The existing Lovable build configuration targets a Cloudflare-compatible runtime. Server code uses standard `fetch`, streams, and text decoding.

| Configuration name | Required context |
| --- | --- |
| `VITE_SUPABASE_URL` | Managed browser Cloud connection configuration. |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | Managed public client configuration. |
| `LOVABLE_API_KEY` | Server-only AI authorization. |
| `LOVABLE_DB_MIGRATION_URL` | Optional managed migration-tool context. |

No endpoint identifiers or key values are documented. Existing managed integration files should not be edited manually. SQL migration history is the current schema reference; the empty Drizzle schema entry point is not a complete declarative mirror of those tables.

## 14. UX and nonfunctional expectations

### Current design

- Mobile-first capture and bottom navigation; desktop side rail.
- Cinematic factory imagery, dark graphite background, amber primary signals, cyan accent.
- Space Grotesk display typography and DM Sans body typography.
- Severity badges, normalized evidence annotations, risk gauge, impact summaries.
- Preparation/upload/analysis feedback and toast error messages.

### Operational expectations to validate

| Area | Proposed acceptance target / check |
| --- | --- |
| Browser support | Test current mobile Safari, Android Chrome, and desktop Chrome/Edge with real supported media. |
| Accessibility | Audit keyboard access, focus, labels, contrast, screen-reader output, and reduced-motion behavior. No conformance claim yet. |
| Performance | Measure preparation, upload, analysis, and report latency separately; no SLA currently defined. |
| Reliability | Reject incomplete evidence rather than producing misleading frame references; provide explicit retry states. |
| Scalability | Add pagination/server aggregates before claiming full-history analytics. |
| Security | Test cross-account read/write and storage isolation, parent ownership, and repeated analysis. |
| Reporting | Inspect every PDF page with long metadata and varied aspect ratios. |
| Maintainability | Preserve centralized AI/error handling and established authenticated-route boundaries. |

The report screen's “about 20 seconds” message is an interface estimate, not a measured guarantee.

## 15. Error and recovery behavior

| Condition | Current response / limitation |
| --- | --- |
| Image decode failure | Show image-read error; return preparation state to idle. |
| Unsupported video | Show browser-format error where decoder reports it. |
| Inspection insert failure | Show error and return to idle. |
| Individual evidence upload failure | Failed object is omitted from stored paths; comprehensive failure reporting/abort is absent. |
| AI HTTP 429 | Show retry-later busy message. |
| AI HTTP 402 | Show insufficient-credits message. |
| Missing server AI configuration | Report AI configuration error. |
| Empty/invalid AI output | Throw handled result error. |
| Analysis failure | Attempt row deletion; uploaded objects may remain. |
| Report generation failure | Toast error; first-generation page can remain visually waiting without a dedicated retry control. |
| PDF image failure | Silently skip that image. |
| Native share dismissed/fails | Native share rejection is swallowed. |

## 16. Acceptance checklist

These are recommended validation cases, not a record of completed tests.

| Test | Expected result |
| --- | --- |
| A01 Signed-out protection | Inspection URLs redirect to account access. |
| A02 Account confirmation | Signup behavior matches verified hosted policy; existing accounts can sign in. |
| A03 Configured Google flow | Complete sign-in and return to the same-origin app. |
| A04 Photo inspection | Prepared JPEG, stored media, results, and history all agree. |
| A05 Video inspection | Six documented sample frames appear; no continuous-video claim. |
| A06 Frame removal | Removed frames are absent from submission/storage/results. |
| A07 Clear result | Empty findings; severity none; status resolved; appropriate human-review caveat. |
| A08 Findings result | Frame indices and boxes match visual evidence; status open. |
| A09 Persistence | Reload reads stored inspection and findings, not transient demo state. |
| A10 Report lifecycle | First visit saves report; later visits read it; Redo replaces it. |
| A11 PDF | Legible, no overlaps, correct evidence, every page inspected. |
| A12 Share | Native text share or clipboard fallback works; no accidental public access. |
| A13 Disposition | Resolve/Reopen persists and refreshes summary/history. |
| A14 Account isolation | Another account cannot read/write owned rows or media. Parent-ownership edge cases are included. |
| A15 Failure integrity | Simulated upload/AI failures do not produce mismatched evidence; storage cleanup gaps are documented or addressed. |
| A16 Responsive/accessibility | Real mobile/desktop and keyboard/screen-reader checks, including long labels. |

The current committed automated test checks route matching for `/` only. This documentation delivery did not run a fresh authenticated end-to-end inspection, quality benchmark, or security audit.

## 17. Future work—not implemented or committed

**Operational hardening priorities:** complete upload validation and cleanup; parent-ownership enforcement; analysis idempotency/atomicity; report retry/version history; PDF layout improvements; persisted processing states; history pagination; calibrated factory-specific evaluation.

**Potential product extensions:** shared factory workspaces, role-based access, corrective-action assignments, work-order integration, voice input, live-video monitoring, offline capture queues, original-video evidence retention, audit trails, and measured resource/downtime analytics.

Each extension needs separate requirements, security review, and acceptance testing. Listing it here does not mean it is built or scheduled.

## 18. Project references and handover

- [Open Factory Vision](https://bright-eye-detect.lovable.app).
- [Watch the project walkthrough](https://youtu.be/HGintBYyeGU).
- [Read the development and usage guide](./README.md).
- Review `AGENTS.md` for project architecture constraints.
- Review `drizzle/migrations/` for current application schema and policy history.

Before production adoption, assign an operational owner, confirm legal/retention requirements, validate with representative factory evidence, and define human approval for quality and safety actions. No license, performance guarantee, or compliance certification is supplied by this specification.