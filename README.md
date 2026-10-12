# Factory Vision

**AI-assisted visual inspection for Intelligent Industry.**

Factory Vision is a mobile-first web application that examines factory photos and sampled video frames for visible defects, machine issues, and safety hazards. It records findings against visual evidence and produces actionable inspection reports.

**Project creator:** Hossam Elshahaby  
**Documentation date:** October 12, 2026  
**Live app:** [https://bright-eye-detect.lovable.app](https://bright-eye-detect.lovable.app)  
**Video walkthrough:** [https://youtu.be/HGintBYyeGU](https://youtu.be/HGintBYyeGU)  
**Detailed specifications:** [PROJECT_SPECIFICATIONS.md](./PROJECT_SPECIFICATIONS.md)

> AI results support—not replace—qualified quality, maintenance, and safety personnel. A clear result means no issue was identified in the submitted frames; it is not a certification that an asset or workplace is safe.

## Contents

- [Purpose and audience](#purpose-and-audience)
- [Implemented features](#implemented-features)
- [Using the app](#using-the-app)
- [Evidence and AI behavior](#evidence-and-ai-behavior)
- [Architecture](#architecture)
- [Technology stack](#technology-stack)
- [Repository map](#repository-map)
- [Local development](#local-development)
- [Configuration and deployment](#configuration-and-deployment)
- [Data and privacy](#data-and-privacy)
- [Testing and validation](#testing-and-validation)
- [Known limitations](#known-limitations)
- [Troubleshooting](#troubleshooting)
- [Contributing and license](#contributing-and-license)

## Purpose and audience

The project aims to help factories spot problems earlier and operate more effectively:

| Goal | How the app supports it |
| --- | --- |
| Reduce downtime | Flag visible wear, leaks, corrosion, and misalignment for investigation. |
| Improve quality | Identify visually supported cracks, contamination, and surface defects. |
| Increase safety | Highlight visible PPE issues, spills, and hazardous conditions. |
| Improve resource efficiency | Recommend actions that may reduce waste, scrap, and rework. |

Primary audiences are floor operators, quality inspectors, maintenance technicians, safety personnel, and plant managers. These are intended audiences, not separate permission roles: the current app stores inspections per individual account.

No measured downtime reduction, detection accuracy, safety improvement, or financial return is established by the current implementation.

## Implemented features

### Capture and inspection

- Mobile camera capture through the device's browser/file picker.
- Upload one or multiple photos, with a maximum of six selected frames per inspection.
- Select a video and extract six evenly spaced frames in the browser.
- Preview evidence and remove unwanted frames before submission.
- Add optional production line, machine, shift, and free-text inspection context.
- See preparation, evidence-upload, and AI-analysis states.

### Findings and results

- Generated inspection title and summary.
- Overall severity: `none`, `low`, `medium`, `high`, or `critical`.
- A model-generated risk score from 0 to 100.
- Frame-based bounding-box annotations for individual findings.
- Finding category, label, confidence, description, and recommended action.
- Qualitative business impact covering downtime, quality, safety, and resources.
- Inspection-level Resolve and Reopen controls.

### Reports and history

- Automatically generate a report when opening a report page without a saved report.
- Save report text with the inspection; Redo replaces that saved text.
- Export an A4 PDF with report text and up to four annotated evidence images.
- Share report text through the device's native sharing interface where supported, or copy it to the clipboard.
- Search history by title, line, machine, and summary.
- Filter by severity and by open/resolved status.
- Signed-in overview with inspection counts, open inspections, average risk, critical-open counts, seven-day activity, and five recent inspections.

### Account access

- Email/password sign-in and account creation.
- Google sign-in action using the configured Lovable authentication integration.
- Protected inspection pages and authenticated AI requests.
- Sign-out through the application shell.

Authentication provider settings are managed separately from the source. This documentation does not claim a fresh live verification of Google configuration or email confirmation settings.

## Using the app

1. Open the [live app](https://bright-eye-detect.lovable.app).
2. Sign in, or create an account. If account confirmation is required, follow the confirmation email before signing in.
3. Select **Inspect** or **New inspection**.
4. Take a clear photo, select photos, or choose a video file. Browser support determines available camera and video formats.
5. Review the prepared evidence. Add the line, machine, shift, and any relevant notes.
6. Select **Run AI inspection** and wait for uploading and analysis to finish.
7. Review the annotations, confidence, severity, risk score, and recommendations. Check every important finding against the original scene.
8. Open the report, review its proposed causes and actions, then select **PDF**, **Share**, or **Redo** as needed.
9. Resolve the inspection only after the appropriate human checks and corrective work. Reopen it if further action is required.
10. Use **History** to revisit and filter previous inspections.

For useful evidence, use good lighting, a stable camera, close-up views of the target, and additional angles where practical. Do not enter unsafe areas or operate machinery unsafely to obtain an image.

The linked video is a walkthrough resource. Voice-command sequences in demonstration material should not be interpreted as an implemented feature: the app currently has no voice control.

## Evidence and AI behavior

### Supported finding categories

`crack`, `corrosion`, `leak`, `misalignment`, `wear`, `contamination`, `missing_part`, `surface_defect`, `overheating`, `safety_ppe`, `safety_hazard`, `spill`, `housekeeping`, `other`.

These are model output categories, not separately trained or experimentally validated industrial detectors. For example, visible imagery cannot establish an actual equipment temperature or diagnose hidden internal faults.

### Media limits

| Item | Current behavior |
| --- | --- |
| Photo selection | Up to 6 frames in the interface. |
| Video sampling | 6 midpoint samples across equal-duration segments. |
| Image preparation | Preserve aspect ratio; resize longest edge to at most 1280 pixels; do not upscale. |
| Compression | JPEG quality setting 0.82. |
| AI input validation | 1–8 frames per server-function request. |
| Evidence storage | Prepared JPEG frames, not original photo/video files. |
| Evidence access | Signed image URLs issued with a 3600-second lifetime. |
| PDF evidence | At most the first 4 stored images. |

Video analysis is sampled still-image analysis, not continuous video monitoring. Events occurring between samples can be missed. Video audio is not analyzed.

### Analysis and reporting

The configured model identifier is `openai/gpt-6-astra`, called through Lovable AI Gateway. The gateway streams output internally, but the interface waits for the completed analysis rather than displaying live generated text.

Analysis requests use a strict JSON output schema. The server parses the response and clamps confidence and individual bounding-box coordinates to 0–1, frame indices to the submitted frame range, and risk score to 0–100. These clamps are not a complete runtime validation of every field or a calibration of model confidence.

Reports are generated from stored inspection metadata and findings, rather than by reanalyzing the images. The prompt requests fewer than 450 words and the following sections:

1. Executive summary.
2. Evidence & findings, referencing F1, F2, and so on.
3. Likely root causes.
4. Business impact.
5. Prioritized actions with proposed owner roles and urgency.
6. Preventive recommendations.

The word limit and evidence-only behavior are prompt instructions, not mechanically enforced guarantees. Proposed root causes must be treated as hypotheses.

## Architecture

```text
Browser: capture / upload + optional context
    |
    +--> Canvas: compressed JPEG frames
    |
    +--> Lovable Cloud: inspection row + private evidence storage
    |
    +--> Authenticated TanStack server function
             |
             +--> Lovable AI Gateway: visual analysis
             |
             +--> Lovable Cloud: findings + inspection summary
                                      |
                                      +--> Results and history
                                      +--> Report generation
                                      +--> Browser PDF / text share
```

- **Frontend:** React with TanStack file-based routing and TanStack Query for reads and cache invalidation.
- **Application server:** TanStack Start server functions with bearer-token authentication and CSRF middleware.
- **AI boundary:** Centralized server-only gateway helper, dynamically imported inside authenticated handlers.
- **Persistence:** Lovable Cloud PostgreSQL tables plus private object storage.
- **Deployment:** Existing Lovable/Vite configuration targets a Cloudflare-compatible server runtime.

## Technology stack

Versions below reflect package declarations, not a separately verified deployed dependency inventory.

| Layer | Technology |
| --- | --- |
| UI | React 19, TypeScript, Radix/shadcn-style primitives, Lucide icons |
| Routing/server | TanStack Start v1, TanStack Router v1 |
| Data state | TanStack Query v5 |
| Styling | Tailwind CSS v4; Space Grotesk headings and DM Sans body |
| Build tooling | Vite 8 as currently declared in `package.json`; Lovable TanStack configuration |
| Auth and persistence | Lovable Cloud and its generated browser/server clients |
| AI | Lovable AI Gateway streamed Responses API |
| PDF | jsPDF v4, loaded on demand |
| Testing | Vitest v4, Testing Library, JSDOM |
| Schema history | SQL migrations under `drizzle/migrations/` |

The visual direction is cinematic industrial: dark graphite surfaces, amber primary signals, cyan accents, evidence overlays, and restrained scanning/transitional effects. Mobile uses bottom navigation; desktop uses a side rail.

## Repository map

```text
AGENTS.md                         Architecture rules for contributors
README.md                         Project overview and development guide
PROJECT_SPECIFICATIONS.md         Detailed implementation specification
package.json                      Dependencies and development scripts
vite.config.ts                    Lovable/TanStack build configuration
drizzle/
  schema.ts                       Currently an empty schema entry point
  migrations/                     Existing application SQL and storage policies
src/
  router.tsx                      TanStack Router bootstrap
  start.ts                        Authentication attachment and request middleware
  server.ts                       Server rendering error wrapper
  styles.css                      Theme tokens and shared visual styles
  routes/
    __root.tsx                    Document and shared shell
    index.tsx                     Public landing / signed-in overview
    auth.tsx                      Account access
    _authenticated/
      route.tsx                   Protected layout
      inspect.tsx                 Capture and submission
      history.tsx                 Search and filters
      inspection.$id.index.tsx    Findings and inspection status
      inspection.$id.report.tsx   Report, PDF, share, regeneration
  components/                     Shell, annotated evidence, rows, UI primitives
  hooks/                          Session and viewport helpers
  lib/
    ai-gateway.server.ts          Central server-only AI transport
    ai.functions.ts               Authenticated analysis/report functions
    media.ts                      Browser JPEG preparation and video sampling
    inspection.ts                 Reads, types, severity helpers, signed URLs
  integrations/                   Managed authentication and Cloud clients
  test/                           Test setup and route regression test
```

## Local development

### Prerequisites

- A modern Node.js runtime compatible with the declared Vite/tooling versions; Node.js 22.12+ is a reasonable baseline, not a project-enforced engine requirement.
- Bun or npm.
- Authorized project access and the managed Lovable Cloud configuration.
- Server-side AI configuration for real analysis/report requests.

### Install and run

From your cloned repository directory:

```sh
bun install
bun run dev
```

Or use npm consistently:

```sh
npm install
npm run dev
```

Use the local URL printed by the development server. The hosted Lovable sandbox manages its own already-running preview server.

### Available scripts

| Command | Purpose |
| --- | --- |
| `bun run dev` | Start local development. |
| `bun run build` | Create the production build in a local/CI environment. |
| `bun run build:dev` | Build using development mode. |
| `bun run preview` | Preview a local production build. |
| `bun run lint` | Run ESLint. |
| `bun run format` | Format the repository; this command changes files. |
| `bun run test` | Run Vitest once. |
| `bun run test:watch` | Run tests in watch mode. |

The equivalent `npm run <script>` commands are also defined. Do not assume that installing packages alone provisions authentication, tables, storage, or AI access.

## Configuration and deployment

| Configuration name | Scope | Purpose |
| --- | --- | --- |
| `VITE_SUPABASE_URL` | Managed browser configuration | Cloud connection endpoint. |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | Managed browser configuration | Public client configuration; not a privileged key. |
| `LOVABLE_API_KEY` | Server secret | Authorize Lovable AI Gateway requests. |
| `LOVABLE_DB_MIGRATION_URL` | Migration tooling only | Read by `drizzle.config.ts`; not needed for ordinary browser operation. |

No credential values are included here. Do not expose private keys through `VITE_*`, commit secrets, or manually alter generated Cloud client files. Lovable Cloud privileged credentials are not available for manual retrieval; manage the backend through Lovable.

For another environment, verify:

1. Both application tables, their grants, row-level policies, and indexes exist.
2. The `inspection-media` bucket exists and is private, with owner-folder policies.
3. Required sign-in providers and valid same-origin redirects are configured.
4. The server-side AI secret and sufficient AI credits are available.
5. Authenticated requests carry the session bearer token.
6. The target runtime supports the project's Web API-based server code.

Existing SQL migrations document schema and policies. They do not create the media bucket, configure Google sign-in, or define all hosted authentication settings. Because `drizzle/schema.ts` is empty, do not regenerate migrations from it without first establishing a deliberate schema workflow.

The live app is [bright-eye-detect.lovable.app](https://bright-eye-detect.lovable.app). This documentation change does not republish or reconfigure it.

## Data and privacy

- Application tables are `public.inspections` and `public.findings`.
- Both have row-level security and authenticated-owner policies using `user_id`.
- Findings reference inspections and cascade on inspection deletion.
- Evidence is stored as `inspection-media/<user_id>/<inspection_id>/<frame_index>.jpg`.
- Storage read, insert, and delete policies check the owner folder.
- Signed evidence URLs are temporary bearer links; avoid distributing them casually.
- Images and optional context are transmitted to the AI service for analysis; stored findings and context are sent for report generation.
- The gateway request sets `store: false`. This is a request setting, not a documented guarantee about every provider's retention or jurisdiction.

Before production use, establish factory policies for proprietary imagery, employee privacy, consent, retention, access reviews, and permitted AI processing. No comprehensive retention/deletion workflow or compliance certification is implemented.

## Testing and validation

The checked-in route test confirms that `/` matches a page rather than falling through to not-found. It does not render the app, run loaders, sign in, upload evidence, call AI, validate policies, or inspect PDFs.

For an operational acceptance check, use an authorized account and representative factory evidence:

1. Verify email/password and configured Google sign-in.
2. Upload a photo and then a supported video in separate inspections.
3. Confirm stored evidence matches annotations and frame numbers.
4. Verify a second account cannot read the first account's records or media.
5. Generate, reopen, and regenerate a report; check that text persists.
6. Download and inspect every PDF page, including mixed portrait/landscape images.
7. Exercise native sharing and clipboard fallback on suitable browsers.
8. Resolve/reopen an inspection and verify history/overview updates.
9. Simulate upload/AI failures and inspect evidence cleanup.

No fresh authenticated end-to-end run, model accuracy benchmark, load test, or compliance audit was performed for this documentation delivery.

## Known limitations

- No implemented voice commands, speech input, live camera inference, continuous monitoring, or machine-control integration.
- No offline queue, installed-app guarantee, organization/team management, admin roles, or work-order assignment.
- No original-video playback/archive, audio analysis, temporal tracking, or exhaustive frame coverage.
- History and overview are computed from at most 200 newest inspections, not the entire lifetime dataset.
- Risk/confidence are model judgments, not calibrated probabilities or measured business outcomes.
- Resolve/Reopen is an inspection-level status change, not evidence of corrective work or a signed audit trail.
- Inspection notes are supplied to analysis but are not stored in a dedicated notes column.
- Redo overwrites the current report; report revisions are not archived.
- Share sends report text, not a public authenticated inspection link or automatically attached PDF.
- Upload errors are not comprehensively surfaced; failed uploads can leave missing or misaligned evidence. Deleting a failed inspection does not remove uploaded objects.
- Database policies isolate rows by `user_id`, but do not additionally prove that a finding's parent inspection belongs to that same user. Repeated analysis also lacks a dedicated idempotency safeguard.
- Basic PDF formatting can need improvement for long metadata, unusual image ratios, and richer Markdown; unavailable images are silently skipped.
- There is no committed factory-specific evaluation dataset, calibrated threshold, latency guarantee, or measurable improvement baseline.

See the specifications for detailed operational gaps and suggested acceptance criteria.

## Troubleshooting

| Symptom | Check or next action |
| --- | --- |
| Account created but not signed in | Follow the confirmation email if required; verify hosted account settings. |
| Google sign-in fails | Verify the configured provider and permitted redirect origin through Lovable. |
| Camera opens a picker | Camera capture is browser/device-dependent; choose an existing photo if needed. |
| Video cannot be prepared | Try a format the current browser can decode, or submit still photos. |
| AI unavailable or busy | Confirm server AI configuration/credits; retry a temporary capacity failure. |
| Evidence image missing | Inspect upload failures and signed-link expiry; reopen the inspection to request fresh links. |
| Report remains in a waiting state | Check the displayed error and request logs; the current UI has limited first-generation failure recovery. |
| Share is unavailable | Use clipboard fallback on a supported secure browser, or download the PDF. |
| Older inspections absent | The current fetch retrieves only the newest 200; pagination is not implemented. |

## Contributing and license

Read `AGENTS.md` before changing the project. Keep AI credentials server-side, authenticated pages within the established protected layout, and evidence within owner-prefixed paths. Preserve generated integration files and the TanStack routing bootstrap.

Submit focused changes with relevant tests and explicitly document any changed media limits, schema, or security behavior. Avoid rewriting published repository history connected to Lovable.

No license grant is declared in this README. Confirm ownership and add an explicit license before redistributing the project or third-party assets. Dependency and asset licenses remain applicable.
