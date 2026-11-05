# AI Data Analyst Workspace

AI Data Analyst Workspace turns CSV files and business questions into interactive, decision-ready reports. Upload one or more related datasets, ask a question in plain language, and a Gemini Managed Agent profiles the data, runs Python analysis in a remote environment, and returns findings, tables, and charts.

## Project overview

For each new analysis, the server stages the agent instructions and skills from `agent/` into a remote environment, along with any inline CSVs. For `gs://` inputs, the server validates each URI and sends a JSON manifest. `agent/download_gcs.py` then fetches the files into the sandbox.

The agent follows the workflow in `agent/AGENTS.md`: it profiles source tables, saves analysis tables, renders charts, and produces `workspace/data/report.json`. The server retrieves the report and chart images from the environment snapshot and streams progress to the browser over Server-Sent Events (SSE). Follow-up questions reuse the same analysis environment, so the original datasets remain available.

The active client in `server/lib/agentClient.ts` targets the Gemini Interactions API with the `antigravity-preview-09-2026` agent and the `gemini-3.8-flash` model. The checked-in `agent/agent.yaml` also declares the `google_search` and `code_execution` tools. Managed Agent availability and access depend on your Google project and API configuration.

### Tech stack

| Layer | Technologies |
| --- | --- |
| Frontend | React 19, TypeScript, Vite 6, Tailwind CSS 4, Motion, Lucide React |
| Backend | Express 4, TypeScript (`tsx`), in-memory uploads via Multer |
| Agents & analysis | Gemini Managed Agents Interactions API (streaming), Pandas, NumPy, Matplotlib, scikit-learn (installed as needed) |
| Exports & files | Markdown rendering, PDF and dashboard exports, client-side archive handling |
| Artifacts & storage | In-memory TAR extraction, server-proxied Google Cloud Storage (GCS) access for allowlisted buckets |

### Agent structure

| Path | Responsibility |
| --- | --- |
| `agent/AGENTS.md` | Analysis workflow, quality principles, and report contract |
| `agent/skills/data-explorer/` | Profiles tables, schema, missing values, statistics, and potential join keys |
| `agent/skills/python-data/` | Guides Pandas analysis, aggregations, joins, and optional machine learning |
| `agent/skills/visualization/` | `make_chart.py` renders analysis tables as PNG charts |
| `agent/skills/reporting/` | `build_report.py` creates the structured report artifact |

## Features

- **Multi-file analysis:** Analyze several related CSV files in a single run.
- **Live progress:** Streams agent activity and analysis progress to the browser over SSE.
- **Grounded analysis:** Profiles tables and data quality, then runs Pandas analysis. Scikit-learn is used when a question calls for modeling.
- **Structured reports:** Includes an executive summary, ranked insights, tables, charts, methodology, and recommendations.
- **Follow-up questions:** Continue working in the same remote analysis environment.
- **Dashboard and exports:** Displays reports in dashboard tabs and exports them as JSON or PDF.
- **Browser storage:** Saves reports in the browser's `localStorage`.
- **GCS input:** Accepts `gs://` dataset references for data that should not be uploaded inline.

## Installation

### Prerequisites

- Node.js and npm
- A Gemini API key with access to the configured Managed Agents API and agent/model
- For private GCS objects: a server identity with permission to read them, and the bucket names listed in `GCS_ALLOWED_BUCKETS`

Python dependencies are not needed to run the web server. For analyses, they are installed in the remote agent environment from `agent/requirements.txt`.

### Configuration

Set these variables in `.env.local` (or `.env`).

| Variable | Purpose |
| --- | --- |
| `GEMINI_API_KEY` | Authenticates server requests to the Gemini Interactions API. |
| `GCS_ALLOWED_BUCKETS` | Comma-separated bucket allowlist. Required for authenticated GCS proxy downloads. |
| `PUBLIC_SERVER_URL` | Optional public base URL that the sandbox download helper uses to reach the server's GCS proxy. If omitted, the server derives a URL from the incoming request when possible. |
| `DISABLE_HMR` | Set to `true` to disable Vite hot module replacement. |

The server obtains a Google Cloud access token from the GCP metadata server when one is available. Do not put service-account credentials in frontend code or agent sandbox files.

## Quick start

1. Install dependencies:

   ```bash
   npm install
   ```

2. Create `.env.local` in the project root and add your Gemini API key:

   ```dotenv
   GEMINI_API_KEY=your_gemini_api_key
   ```

3. Start the application:

   ```bash
   npm run dev
   ```

The Express server starts on port `3000` and uses Vite middleware to serve the frontend in development. The app loads `.env.local` before `.env`.

## Usage

### Run an analysis

1. Upload one or more CSV files, or add a `gs://bucket/path/file.csv` URI.
2. Enter a business question, such as "What are the strongest sales trends?"
3. Watch the streamed activity as the agent profiles, analyzes, and reports on the data.
4. Review the overview, charts, tables, and recommendations. Ask follow-up questions in the same session as needed.
5. Export the report as JSON or PDF, or save it in the browser.

### File handling

- **Inline uploads:** Limited to 1 MiB per file in both the browser and the server. The 50 MiB Multer request limit does not change this threshold.
- **Larger files:** Use a GCS URI. Private objects fetched through the server proxy require the bucket to be listed in `GCS_ALLOWED_BUCKETS`. Public objects may be fetched anonymously when the sandbox can access them. The proxy rejects objects that report a content length above 200 MiB.
- **Credentials:** GCS URIs are validated before use. The server's access token stays on the server and is not passed to the agent sandbox.

### Report contents

Reports follow the schema defined in `src/types.ts`: `dataset_name`, `question`, `title`, `executive_summary`, `insights`, `charts`, `tables`, `methodology`, `recommendations`, and `generated_at`. In the remote environment, report artifacts are stored under `workspace/data/` and chart images under `workspace/charts/`.

### Development commands

| Command | Description |
| --- | --- |
| `npm run dev` | Runs the Express server with Vite middleware through `tsx`. |
| `npm run build` | Builds the frontend and bundles the server to `dist/server.cjs`. |
| `npm start` | Runs the production server bundle. |
| `npm run preview` | Runs Vite's static preview server. |
| `npm run lint` | Runs the TypeScript compiler without emitting files. |
| `npm test` | Runs the Node test suite for GCS source validation. |
| `npm run clean` | Removes the `dist/` build directory. This shell command is intended for Unix-like environments. |

### Limitations

- Analyses require a valid Gemini API key and access to the configured Managed Agent.
- Saved reports are local to the browser profile and are not synchronized through Firestore.
- Quota reporting is available at `/api/quota`, but analysis currently skips daily quota tracking.
- GCS support is for URI-based ingestion only. Inline CSV contents are passed to the remote environment and are not stored in GCS.

## Contributing

Contributions are welcome. Before opening a pull request, run `npm run lint` to type-check the code and `npm test` to run the test suite.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
