PROJECT RULES: AI-Powered Resume Screening and Intelligent Job Matching System

WORKSPACE
The repository root is AI-Resume-Screening, and you are opened in it. Every path in these rules and in every prompt is relative to that root. Never create a nested AI-Resume-Screening folder or any new project folder inside it, and never write outside it. Do not rename or move the top-level folders below without asking me.

REPO LAYOUT
AI-Resume-Screening/
  .agent/rules/project-rules.md
  docs/                      SPEC.md, api-contract.md, Decisions.md, Flow.md, PROGRESS.md, EVAL_*.md, VECTOR_DECISION.md, DEPLOYMENT.md
  backend/
    app/{api,core,models,schemas,services,ai/{parsing,jd,matching,explain,llm,search}}
    alembic/  tests/  data/  storage/ (local resume files)  .venv/ (virtualenv)
    requirements.txt  Dockerfile  .env.example
  frontend/
    src/{api,hooks,pages,components,context}
    public/  package.json  vite.config.ts  .env.example
  scripts/                   eval, seed and smoke-test scripts (Python, so they run on Windows too)
  docker-compose.yml  README.md  .gitignore
Git-ignored: .env files, backend/.venv, backend/storage, frontend/node_modules, build output.

SOURCES OF TRUTH
docs/SPEC.md is the product specification. docs/api-contract.md is the contract between backend and frontend. docs/Decisions.md, docs/Flow.md and docs/PROGRESS.md are the project record. If a phase prompt conflicts with SPEC.md, the phase prompt wins; point out the conflict in your plan.

WORKING PROTOCOL (every phase)
1. Read the files named at the top of the prompt before anything else.
2. Do ONLY the phase named. Do not start later phases, add unrequested features, or refactor unrelated code.
3. Show your plan first: files to create or change, endpoints or screens, tests, risks, open questions. Wait for me to reply 'go'. Write no code before that.
4. Build, then verify. Verification means commands you actually ran, with their output pasted. Never claim something works without evidence.
5. If you find a mismatch between the code and docs/api-contract.md, or between SPEC and reality, STOP and report it. Do not work around it.
6. Before finishing, update Decisions.md, Flow.md and PROGRESS.md (formats below). Then stop and wait.

RECORD FILES
Decisions.md: numbered entries D-001, D-002... each with Date, Phase, Context, Decision, Alternatives considered, Consequences. Record every choice SPEC.md does not already fix (libraries, thresholds, schemas, trade-offs). Never edit an old decision; add a new one that supersedes it.
Flow.md: an Architecture section (Mermaid), a Data model section (Mermaid ERD), an AI pipeline section, and one short section per finished phase describing what flows where. Update diagrams when behavior changes.
PROGRESS.md: a phase table (Phase, Status: Not started / In progress / Done, Date) and one log entry per phase: summary, files touched, evidence pasted, deviations from the prompt, known issues, next step.

FIXED STACK
Backend: Python 3.11+, FastAPI, Pydantic v2, SQLAlchemy 2, Alembic, pydantic-settings, PostgreSQL (JSONB, pgvector), pytest, ruff.
AI and parsing: sentence-transformers (all-MiniLM-L6-v2), spaCy, scikit-learn, pandas, numpy, PyMuPDF, pdfplumber, python-docx.
Frontend: React 18, TypeScript, Vite, React Router, TanStack Query, Axios, Tailwind CSS, Recharts, vitest. No UI component library.

BACKEND CONVENTIONS
Routes only call services. All routes under /api/v1. Errors are RFC 7807 ProblemDetails (application/problem+json) with type, title, status, detail, instance. Port 8000. Scoring functions are pure and deterministic. Run Python inside backend/.venv.

FRONTEND CONVENTIONS
api/types.ts is copied exactly from docs/api-contract.md. api/client.ts holds typed Axios functions, reads VITE_API_BASE_URL, and parses ProblemDetails. Server state lives in TanStack Query hooks under hooks/. Dev server on port 5173 with a Vite proxy for /api to http://localhost:8000. Every screen has explicit loading (skeleton), empty, error (banner with Retry) and not-found states. No 'any'. Labels on every input, keyboard-navigable interactive components, responsive to tablet width. Clarity over design.

PRODUCT RULES
- Never invent data. If extraction fails, store null and a reason.
- A skill not mentioned in a resume is NOT DEMONSTRATED, never confirmed absent.
- The score is a decision-support signal, not a hiring prediction. The system never accepts or rejects a candidate on its own.
- Never use or infer protected characteristics (gender, age, ethnicity, religion, marital status, nationality, photo, anything inferred from a name).
- Every score and explanation traces to evidence (resume section and snippet).
- An LLM only receives verified structured data, and its output is validated against that data.

ENGINEERING RULES
Secrets only in environment variables; keep .env.example current. Validate all input and uploaded files. Log without leaking personal data. Prefer free, open-source tools and ask before adding anything paid. Strong types, small modules, tests for new logic. Commands in README and docs must work on Windows PowerShell as well as macOS and Linux, or show both.