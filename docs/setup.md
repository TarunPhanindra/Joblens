# JobLens setup guide

This guide is based on the exported workflow and the author's local configuration. A clean installation following this guide has not yet been tested. The workflow export alone does not install services, credentials, or data tables.

## 1. Prepare the services

The inspected local configuration uses:

- n8n image: `docker.n8n.io/n8nio/n8n:2.41.4`
- Gotenberg image: `gotenberg/gotenberg:8` (major-version tag, not an immutable image pin)
- Ollama model: `qwen3:8b`, selected in both model nodes
- Timezone: `Europe/Berlin`

Install and start Ollama, and download the model using `ollama pull qwen3:8b`. Run n8n and Gotenberg in a shared Docker network if retaining the workflow's Gotenberg URL. The existing environment exposes n8n at `http://localhost:5678`.

The workflow requests job listings from `https://www.arbeitnow.com/api/job-board-api`; internet access is required for discovery. It sends HTML to `http://gotenberg:3000/forms/chromium/convert/html` for PDF conversion.

## 2. Configure résumé storage

The inspected n8n configuration maps the host folder `./storage/resumes` to `/files/resumes` inside the container and sets `N8N_RESTRICT_FILE_ACCESS_TO=/files/resumes`.

Create the host subfolder `storage/resumes/pending-review` and ensure the n8n container user can write to it. `Save Resume for Review` writes timestamp-prefixed PDFs into `/files/resumes/pending-review/`.

## 3. Import the workflow

Download `workflows/joblens-workflow.json` from this repository. In n8n, import the workflow from that file. Keep it inactive while configuring it.

The published export deliberately omits installation-specific credential references and data-table IDs. Importing it does not create those resources.

## 4. Connect Ollama

Create an Ollama credential in n8n using the Ollama service address reachable from the n8n runtime. Select it in both `Ollama Chat Model` and `Ollama Chat Model1`, and confirm both use `qwen3:8b`.

When n8n runs in a container, `localhost` refers to that container, not the host machine. The appropriate host address depends on your Docker and Ollama deployment. Verify connectivity before starting a batch; this export does not include the original credential's endpoint.

## 5. Create the data tables

Create the following tables in your n8n project. The types below reproduce the schema metadata in the exported nodes. Let n8n manage its built-in row ID and timestamp fields.

### Jobs

| Column | Type |
|---|---|
| `external_id` | string |
| `source` | string |
| `company` | string |
| `title` | string |
| `location` | string |
| `remote` | string |
| `url` | string |
| `posted_at` | string |
| `status` | string |
| `fit_score` | string |
| `decision` | string |
| `matched_skills` | string |
| `missing_skills` | string |
| `fit_reason` | string |

### Failed Evaluations

| Column | Type |
|---|---|
| `external_id` | string |
| `source` | string |
| `company` | string |
| `title` | string |
| `url` | string |
| `fit_score` | number |
| `decision` | string |
| `evaluation_status` | string |
| `evaluation_errors` | string |
| `requires_manual_review` | boolean |
| `processed_at` | string |

The Jobs export declares `fit_score`, `remote`, and `posted_at` as strings, while its expressions originate from numeric/boolean job values. This guide preserves the exported schema rather than silently changing it. Confirm value conversion during a controlled write test; if you change types, update the node mappings consistently.

Select your tables in these nodes:

- `If row does not exist` → Jobs
- `Upsert Job Record` → Jobs
- `Save Failed Evaluations` → Failed Evaluations

Review mappings after selecting tables. Both upsert nodes match `external_id` and `source`; the initial existing-row check matches only `external_id`. These are the exported settings, not a guarantee of duplicate handling across multiple sources.

Skill arrays and evaluation-error arrays are serialized as JSON strings for storage.

## 6. Review the candidate profile

The workflow contains the author's real professional profile as a personal case study. To evaluate another candidate, update the profile, scoring prompt, validator skill vocabulary and candidate-specific requirement checks, eligibility logic, and résumé header consistently. Review `Attach Candidate Profile`, `Basic LLM Chain`, `Validation AI Evaluation`, `Check Job Eligibility`, and `Build Tailored Resume` in particular.

Changing only the résumé name does not change the candidate used for matching. Review generated content against the actual profile before using it.

## 7. Run a controlled test

Start with a small batch in isolated test tables/output storage. The exported limit is 20 items; reduce it for a smoke test if appropriate.

Use **Execute workflow** for a full-workflow test. **Execute step** on a node tests a targeted execution scope and should not be reported as complete downstream coverage.

Record the execution ID, workflow revision, exact model and runtime versions, validator version, input identifiers, and duration. Verify:

- Every input has an accounted-for outcome.
- Valid plus invalid counts equal evaluated inputs.
- Valid and failed records actually appear in the intended tables.
- Repeated inputs behave as expected without unintended duplicates or status changes.
- Eligible, needs-review, ineligible, and fallback paths are tested with controlled examples.
- When résumé generation is reached, the PDF exists and its contents are reviewed.

A real batch with no qualifying scores will not exercise résumé generation. Test that path separately without changing real scores just to force advancement.

## Test evidence boundary

Execution #371 processed 20 jobs with 7 valid and 13 flagged evaluations. It was targeted at `If1`, so it did not establish full-workflow persistence or résumé coverage. Earlier PDFs in pending review provide separate artifact evidence. See the README for the observed results and limitations.
