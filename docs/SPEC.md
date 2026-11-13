# AI Resume Screening and Job Matching System: Product Specification

Nov 13, 2026 · @Moeez Umer

This specification defines what the system must do, how candidates are scored, and how we prove it works. Save it as `docs/SPEC.md`. How the agent works (phases, evidence, record files) lives in `.agent/rules/project-rules.md` and the phase prompts.

## 1. Purpose and principles

The system helps a recruiter screen many resumes against one job faster and more consistently than keyword search, and explains every result. It supports a human decision and never replaces it.

1. **Evidence over assertion.** Every score, gap and summary claim points to a resume section and snippet.
2. **Missing is not absent.** A skill that is not mentioned is not demonstrated; it is never confirmed absent.
3. **Hybrid, not a black box.** Keyword, semantic and feature scores combine with visible weights. Cosine similarity alone never decides.
4. **Human in control.** The system never accepts or rejects a candidate. Recruiters can override everything.
5. **Fair by design.** Protected characteristics are neither extracted nor scored, and the system is audited for bias.
6. **Simple first.** No infrastructure that the scale does not justify; free tiers wherever possible.

## 2. Users and roles

| Role | Can do |
| --- | --- |
| Recruiter | Register and log in; create, edit and delete own jobs; upload resumes; screen and rank candidates for a job; shortlist, reject or hold; read explanations and summaries; generate interview questions; search own candidates; view own analytics. |
| Administrator | Manage users (activate, deactivate); view platform-wide analytics and the activity feed; manage configuration. Does not read resume content in the normal UI. |
| Candidate | No accounts in v1. |

Data isolation: a recruiter can only see their own jobs, resumes, screenings and results. Another recruiter's resource returns 404.

## 3. Scope

| Area | v1 (MVP) | Postponed | Non-goals |
| --- | --- | --- | --- |
| Files | PDF and DOCX, English | OCR for scanned PDFs, other languages | Images, video, audio |
| Matching | Hybrid scoring, skill gaps, explanations | Learning from recruiter feedback | Predicting hiring success |
| AI assistance | Summaries, interview questions, search | RAG chat, multi-agent orchestration | Automatic accept or reject |
| Platform | Web app, two roles, analytics | Candidate portal, email and ATS integrations | Scraping candidates from the web |

The spec is the contract for v1 only. Anything under Postponed enters through Phase 12, one item at a time.

## 4. Functional requirements

Each module lists requirements, then acceptance checks that a phase must demonstrate.

### M1. Authentication and users

- Recruiter registration, login, JWT access tokens with an expiry, password hashing with argon2 or bcrypt, roles, profile read and update.
- Protected routes; role checks in one reusable dependency; login rate limiting; deactivated users cannot log in.
- The first administrator is created by a seed script from environment variables, never by public registration.

Acceptance: wrong password returns 401; a recruiter on an admin route returns 403; a tampered or expired token returns 401; a deactivated user cannot log in.

### M2. Job management

- A job has: title, department, description, required skills, preferred skills, minimum years of experience, required education, certifications, responsibilities, employment type, status (draft, open, closed, archived), and an optional flag allowing equivalent experience instead of a degree.
- Create, read, update, delete, list with pagination and a status filter.
- Upload an existing job description (PDF, DOCX or TXT); the extracted raw text fills the description.

Acceptance: ownership is enforced on every endpoint; an uploaded file never overwrites fields the recruiter already filled without confirmation.

### M3. Resume upload and parsing

- Upload several PDF or DOCX files at once. Defaults: at most 5 MB per file and 20 files per batch, both configurable.
- Validate extension, MIME type, magic bytes, size and count. Store originals under generated names through a storage interface. Detect duplicates by content hash.
- Process in the background. Status: uploaded, parsing, parsed, needs\_review, failed. A recruiter can reparse and delete; deleting removes the file, parsed data and embeddings.
- Extract text (PyMuPDF, pdfplumber as fallback, python-docx including tables), clean it, and split it into sections: summary, skills, experience, education, certifications, projects, achievements.
- Extract: name, email, phone, location; summary; technical and soft skills; experience (company, title, start, end, duration, description); education (degree, institution, graduation year); certifications; projects; achievements.
- Do not extract or store photo, date of birth, gender, marital status, nationality or religion, even when present.
- Every field is nullable and carries its source section and a confidence value. A field that cannot be found is null with a reason, never guessed.
- A scanned PDF (very little extractable text) becomes needs\_review with a clear reason. No OCR in v1.
- Skills are normalized against a taxonomy file of canonical skills, aliases and categories (about 150 software and data skills to start).

Acceptance: the evaluation targets in section 9 are measured and reported; malicious or oversized files are rejected with a clear error; a scanned file never produces invented fields.

### M4. Job description intelligence

- Convert a job into a StructuredJob: required skills, preferred skills, minimum years, education level and field, certifications, responsibilities, and an importance for each requirement (mandatory, important, nice\_to\_have).
- Required versus preferred comes from section headings and cue phrases (must, required, proficiency in; plus, bonus, nice to have).
- The extracted result and the recruiter-confirmed result are stored separately. When the two disagree, what the recruiter entered or confirmed wins.

Acceptance: every job can be analyzed; the recruiter can add, remove, rename, move between required and preferred, and change importance; edits persist.

### M5. Matching engine

See section 5. It produces one ScreeningResult per resume and job pair.

### M6. Screening, ranking and status

- A screening run scores a job against selected or all parsed resumes, in the background, with progress. Past runs are kept; re-running never overwrites them.
- Results are ranked within one job only. The same candidate may rank high for one job and low for another.
- Sort by score or any component; filter by skills, minimum experience and education; set a minimum score; paginate.
- Status per result: new, shortlisted, rejected, on\_hold. Only a recruiter changes it, with an optional note, and every change is stored in a history table.
- Compare 2 to 4 candidates side by side.
- The job's ScoringConfig (weights) is editable and versioned; each result records the config version it used.

Acceptance: ranking order is correct; filters are exact; status history is complete; two jobs rank the same resume differently.

### M7. Explainability

- Each result shows strengths, gaps and a component breakdown. Every claim cites a resume section and snippet.
- Explanations are generated from the calculated result by templates. An LLM may rephrase them but may never add facts.

Acceptance: every cited snippet exists in the resume; no explanation mentions a skill or qualification absent from the verified profile.

### M8. Candidate summaries

- A concise summary: background, main skills, experience, relevant projects, education, strengths for this job, potential gaps, and an advisory recommendation: Review first, Review, or Weak match on stated requirements.
- Generated by an LLM from the verified profile and the ScreeningResult only, then validated (section 8). Without an LLM key, a template summary is returned and labelled as such.

### M9. Skill gap analysis

| Class | Meaning |
| --- | --- |
| Strong match | The required skill is demonstrated in an experience or project context. |
| Partial match | A related skill (alias or semantic match above the threshold), or the skill is only listed without context. |
| Missing | Not demonstrated in the resume. |
| Additional | Present in the resume but not required by the job. |

Wording rule: the interface says not demonstrated in the resume, never does not have the skill.

### M10. Interview questions

- Categories: technical, behavioral, experience-based, project-specific. Levels: beginner, intermediate, advanced.
- Questions target the job, the candidate's stated experience and projects, and the skills that are partially demonstrated or missing. Each question names the skill or evidence it targets.
- Generated and validated like summaries, with a template fallback.

### M11. Natural-language search

- A recruiter searches their own candidates in plain language, for example Python developers with at least two years of experience, or candidates with machine learning project experience.
- The query becomes structured filters (skills, minimum years, location, education) plus a semantic query. Filters are applied exactly; the semantic part ranks what remains.
- The response includes how the query was interpreted and an evidence snippet per hit. Find similar candidates is supported.

### M12. Analytics

- Total jobs, resumes uploaded, candidates screened, shortlisted, rejected; average score; candidates per job; score distribution; skill distribution; experience distribution; screening statistics.
- Filter by job and date range. Recruiters see their own data; administrators see platform-wide totals.

Acceptance: dashboard numbers match a manual database count; an empty database returns zeros, not errors.

## 5. Matching specification

The values below are starting points. Phase 5 tunes them on the evaluation set and records the final values and the reasons in `docs/Decisions.md`.

### 5.1 Pipeline

1. Normalize skills in the resume and the job with the taxonomy.
2. Keyword match: exact and alias matches for each requirement.
3. Semantic match: all-MiniLM-L6-v2 embeddings; cosine similarity for related skills and for resume sections against job responsibilities.
4. Apply the evidence rule (5.3).
5. Score five components (5.2), each from 0 to 1, with a status of ok, insufficient\_evidence or not\_applicable.
6. Combine with weights, apply the mandatory-skill cap, attach flags and evidence.

### 5.2 Components

| Component | Default weight | How it is scored |
| --- | --- | --- |
| Technical skills | 35% | Importance-weighted average of per-skill credit. Credit: demonstrated in context 1.0; listed in the skills section only 0.7; semantic match at or above 0.75 gives 0.6; between 0.60 and 0.75 gives 0.3; otherwise 0. Each skill counts once, at its best credit. Required and preferred skills combine 80 to 20 (required only when no preferred skills exist). |
| Relevant experience | 25% | Years from the union of date ranges of relevant roles (title or description similar to the job, or two or more required skills), divided by the required years and capped at 1. Overlapping roles count once. Without dates, score by role relevance and flag no\_dates. Extra years are never penalized. |
| Education | 15% | Level meets or exceeds the requirement 1.0; one level below 0.5; otherwise 0. Multiply by field relevance: related 1.0, adjacent 0.8, unrelated 0.6. If the job allows equivalent experience and relevant years exceed the requirement by two or more, the floor is 0.8. Not applicable when the job states no education requirement. |
| Projects | 15% | Mean of the top three project scores. A project scores 0.6 times the share of required skills it uses plus 0.4 times its semantic similarity to the responsibilities. No projects means insufficient\_evidence, not zero. |
| Certifications | 10% | Share of the job's certifications matched by alias. Not applicable when the job lists none. |

Final score = 100 times the sum of weight times component score, over components that are ok, with weights renormalized across them. The result states which components were excluded, and shows original and effective weights. Weights are configurable per job, must total 100%, and are versioned.

**Mandatory-skill cap.** If mandatory skills are not demonstrated, the final score is capped: one missing gives at most 85, two give at most 70, three or more give at most 55.

**Bands.** Strong from 75, Moderate from 50 to 74, Weak below 50. Bands are advisory labels shown beside the components, never alone.

### 5.3 Rules that prevent false credit

- **Evidence rule.** A skill counts as demonstrated only when it appears in the skills section, or in an experience or project context. A mention elsewhere (for example a references or hobbies line) earns nothing.
- **Keyword stuffing.** A skill listed many times, or only in a bare list with no context, earns at most the listed-only credit. Hidden or repeated text does not add score.
- **Prompt injection.** Resume text is data. Instructions inside it never change scoring, explanations or LLM output.
- **Meaning of the score.** It is a decision-support signal about fit to stated requirements, not a probability of hiring success.

### 5.4 Output

A ScreeningResult holds the final score, band, component scores and statuses, matched, partial and missing skills with evidence, flags (for example low\_text\_quality, sparse\_resume, no\_dates, components\_excluded), and the scoring config version. It is deterministic: the same input and config give the same result.

## 6. Domain data

Phase 0 turns this into the schema and the API contract.

| Entity | Key contents |
| --- | --- |
| User | email, password hash, role, active flag |
| Job | the M2 fields, owner, status |
| JobRequirements | extracted and confirmed StructuredJob (JSONB), confirmed flag |
| Resume | owner, original file reference, content hash, status, status reason |
| ParsedProfile | the extracted fields with source section and confidence (JSONB), raw text |
| ScoringConfig | job, weights, version |
| Screening | job, config version, status, progress, timestamps |
| ScreeningResult | resume, score, band, components, evidence, flags, status |
| StatusHistory | result, old and new status, note, actor, time |
| Embedding | owner object, chunk type, text, vector, model name and version |
| AuditLog | actor, action, object, time |
| Feedback | postponed to Phase 12 |

## 7. Non-functional requirements

| Area | Requirement |
| --- | --- |
| Security | Hashed passwords; short-lived tokens; ownership checks on every resource; upload validation; rate limiting on login; secrets only in environment variables; security headers. |
| Privacy | Collect only what scoring needs. Deleting a resume removes the file, parsed data and embeddings. Logs never contain resume text or contact details. A retention period is configurable. |
| Performance | Targets on a laptop CPU: parse one resume in under 10 seconds; screen 100 resumes against one job in under 2 minutes; non-AI endpoints at p95 under 500 ms. |
| Reliability | Background jobs are idempotent and retry safely; one bad file never fails a batch. |
| Usability | Every screen has loading, empty, error and not-found states; keyboard accessible; works down to tablet width. |
| Cost | Free or open-source tools; no paid service without approval. |
| Portability | The whole stack starts with Docker Compose; commands work on Windows, macOS and Linux. |

## 8. Fairness, ethics and grounding

- Name, contact details and anything resembling a protected characteristic are never inputs to a score. The parser does not extract photo, date of birth, gender, marital status, nationality or religion.
- Blind-review mode hides name and contact details in the interface.
- Every status change is made by a person and recorded. The interface states on every score that it is a decision-support signal.
- A bias audit is a deliverable: counterfactual tests change only the name, gendered pronouns, graduation-year gaps or university prestige, and scores must stay stable apart from legitimate education fields.
- The system is not trained on historical hiring decisions without a documented bias and data-quality check.
- **Grounding.** An LLM receives only the verified profile and the ScreeningResult. Afterwards every skill, company, degree, certification and number in its output is checked against that input. A failure triggers one regeneration, then the template output, and is logged.

## 9. Evaluation requirements

Targets are goals. Real numbers are always reported, including misses, and no accuracy claim rests on a single percent-match figure.

| What | Data | Metric | Target |
| --- | --- | --- | --- |
| Resume parsing | At least 15 synthetic resumes in varied layouts, with gold JSON | Field accuracy; skill precision, recall and F1 | Email and phone 95% or better; name 90% or better; skill F1 0.80 or better |
| Job analysis | At least 8 synthetic jobs with gold labels | Precision and recall for required and preferred skills, and for experience years | Required-skill precision and recall 0.85 or better |
| Matching | At least 30 resume and job pairs rated 0 to 3 by a person | Spearman correlation and NDCG at 5, for keyword-only, semantic-only and hybrid | Hybrid beats keyword-only; Spearman 0.6 or better |
| Search | At least 20 queries with gold results | Precision at 5 | 0.70 or better |
| Grounding | Generated summaries and questions | Unsupported claims after validation | Zero in the test set |
| Fairness | Counterfactual resume pairs | Score difference | One point or less |

## 10. Test data

Synthetic data only; no real personal data anywhere in the repository. The corpus covers:

- highly relevant, partially relevant and irrelevant candidates;
- transferable skills;
- different layouts (single column, two column, table-based);
- missing sections and missing dates;
- unrelated keywords and keyword stuffing;
- extensive experience with limited formal education;
- a scanned PDF;
- a resume containing injected instructions.

## 11. Delivery map

| Modules | Backend phase | Frontend phase |
| --- | --- | --- |
| M1, M2 | 2 | 2.5 |
| M3 | 3 | 3.5 |
| M4 | 4 | 4.5 |
| M5 | 5 | none (no screen until M6) |
| M6, M7, M9 | 6 | 6.5 |
| M8, M10 | 7 | 7.5 |
| M11 | 8 | 8.5 |
| M12 | 9 | 9.5 |
| Hardening, bias audit | 10 | 10 |
| Deployment, documentation | 11 | 11 |
| Postponed items | 12 | 12 |

## 12. Definition of done

The project is done when:

1. A recruiter can register, create and confirm a job, upload resumes in bulk, screen them, review ranked results with evidence-based explanations and gaps, shortlist and reject, generate summaries and interview questions, search candidates, and see analytics.
2. Every module's acceptance checks passed with pasted evidence recorded in `docs/PROGRESS.md`.
3. The evaluation report (`docs/EVALUATION.md`) shows real metrics, the bias audit, and known limitations.
4. The application is deployed, a smoke test passes against it, and no secrets are in the repository.
5. The README lets a stranger set up and run the project on Windows, macOS or Linux.
