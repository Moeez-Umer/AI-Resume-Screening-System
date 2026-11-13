# Backend–Frontend API Contract (`docs/api-contract.md`)

This document is the **single source of truth** between the FastAPI backend (`backend/app/api/v1/`) and the React + TypeScript frontend (`frontend/src/api/`).
Per `.agent/rules/project-rules.md`:
- Every route is mounted under `/api/v1`.
- `frontend/src/api/types.ts` is copied verbatim from **Section 4** of this document.
- If any mismatch is discovered between implementation and this contract, work must stop and be reported immediately.

---

## 1. Global API Conventions

### 1.1 Base URL and Transport
- **Backend Local Origin:** `http://localhost:8000`
- **API Base Path:** `/api/v1`
- **Frontend Dev Proxy:** Vite (`http://localhost:5173`) proxies `/api` $\rightarrow$ `http://localhost:8000`
- **Request / Response Content-Type:** `application/json; charset=utf-8` (except file upload endpoints which accept `multipart/form-data`)
- **Error Content-Type:** `application/problem+json` (RFC 7807)
- **Timestamps:** ISO-8601 UTC strings (`YYYY-MM-DDTHH:mm:ss.sssZ`)
- **Identifiers:** UUID v4 strings (`"550e8400-e29b-41d4-a716-446655440000"`)

### 1.2 Authentication & Authorization Header
All protected endpoints require a valid JWT access token in the `Authorization` header:
```http
Authorization: Bearer <access_token>
```
- **Missing, malformed, expired, or deactivated-user token:** returns `401 Unauthorized` (`ProblemDetails`).
- **Authenticated user lacking the required role (e.g., `recruiter` calling an `admin` endpoint):** returns `403 Forbidden` (`ProblemDetails`).
- **Cross-tenant resource access (a `recruiter` requesting or mutating another recruiter's job, resume, candidate, screening, or result):** returns **`404 Not Found`** (`ProblemDetails`), never `403`, to prevent resource enumeration (`SPEC.md §2`).

### 1.3 RFC 7807 `ProblemDetails` Error Shape
Every non-2xx response returns `Content-Type: application/problem+json` with the following shape:
```json
{
  "type": "https://api.ai-resume-screening.local/problems/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "One or more request fields failed validation.",
  "instance": "/api/v1/jobs",
  "errors": [
    {
      "field": "min_years_experience",
      "message": "Input should be greater than or equal to 0",
      "code": "greater_than_equal"
    }
  ]
}
```
Standard Problem `type` URIs:
- `.../problems/bad-request` (`400`)
- `.../problems/unauthorized` (`401`)
- `.../problems/forbidden` (`403`)
- `.../problems/not-found` (`404`)
- `.../problems/conflict` (`409`)
- `.../problems/payload-too-large` (`413`)
- `.../problems/unsupported-media-type` (`415`)
- `.../problems/validation-error` (`422`)
- `.../problems/rate-limit-exceeded` (`429`, includes `Retry-After` HTTP header)
- `.../problems/internal-error` (`500`)

### 1.4 Standard Pagination Shape (`PaginatedResponse<T>`)
All list endpoints accept `page` (1-indexed, default `1`, min `1`) and `page_size` (default `20`, min `1`, max `100`) and return:
```json
{
  "items": [],
  "total": 0,
  "page": 1,
  "page_size": 20,
  "total_pages": 0
}
```

### 1.5 Wording & Fairness Rules in API Payloads
- **Missing skills:** API labels and explanations always state `"Not demonstrated in the resume"` (`status: "missing"`), never `"Candidate does not have skill"`.
- **Advisory nature:** Every `ScreeningResult` response includes `decision_support_disclaimer: "Score is an advisory decision-support signal based on stated requirements, not a hiring prediction."`
- **Blind-review mode:** Passing `?blind_review=true` on candidate, result, explanation, summary, or search endpoints replaces `display_name` with `blind_code` (e.g., `"CAND-8F2A"`), sets `email`, `phone`, and `location` to `null`, and scrubs matching PII tokens from evidence snippets (`[REDACTED]`).

---

## 2. Shared Enums

| Enum Name | Allowed Values | Used In |
| --- | --- | --- |
| `UserRole` | `"recruiter"`, `"admin"` | `users`, Auth, RBAC |
| `JobStatus` | `"draft"`, `"open"`, `"closed"`, `"archived"` | `jobs` |
| `EmploymentType` | `"full_time"`, `"part_time"`, `"contract"`, `"internship"` | `jobs` |
| `EducationLevel` | `"none"`, `"high_school"`, `"associate"`, `"bachelor"`, `"master"`, `"phd"` | `jobs`, `job_requirements`, `parsed_profiles` |
| `FieldRelevance` | `"related"`, `"adjacent"`, `"unrelated"` | `screening_results` (Education component) |
| `RequirementImportance` | `"mandatory"`, `"important"`, `"nice_to_have"` | `job_requirements` |
| `ResumeStatus` | `"uploaded"`, `"parsing"`, `"parsed"`, `"needs_review"`, `"failed"` | `resumes` |
| `ResumeSectionName` | `"contact"`, `"summary"`, `"skills"`, `"experience"`, `"education"`, `"certifications"`, `"projects"`, `"achievements"`, `"other"` | `parsed_profiles`, `screening_results` |
| `SkillContextType` | `"experience_or_project"`, `"skills_section_only"`, `"semantic_high"`, `"semantic_low"`, `"unverified_section"` | `parsed_profiles`, `screening_results` |
| `ScreeningStatus` | `"pending"`, `"running"`, `"completed"`, `"failed"` | `screenings` |
| `ComponentStatus` | `"ok"`, `"insufficient_evidence"`, `"not_applicable"` | `screening_results` |
| `ScoreBand` | `"strong"`, `"moderate"`, `"weak"` | `screening_results`, `feedback` (`planned, not created` — Phase 12, `D-013`) |
| `ReviewStatus` | `"new"`, `"shortlisted"`, `"rejected"`, `"on_hold"` | `screening_results`, `status_history` |
| `SkillMatchClass` | `"strong_match"`, `"partial_match"`, `"missing"`, `"additional"` | `screening_results` (M9) |
| `AdvisoryRecommendation` | `"Review first"`, `"Review"`, `"Weak match on stated requirements"` | `screening_results` (M8) |
| `QuestionCategory` | `"technical"`, `"behavioral"`, `"experience_based"`, `"project_specific"` | `screening_results` (M10) |
| `QuestionDifficulty` | `"beginner"`, `"intermediate"`, `"advanced"` | `screening_results` (M10) |

---

## 3. Master Endpoint Index

All 53 endpoints across Infrastructure, M1–M12, and Phase 12 are indexed below:

| # | SPEC Module | Method | Path (`/api/v1...`) | Role | Backend Phase | Frontend Phase | ERD Tables Touched |
| --- | --- | --- | --- | --- | --- | --- | --- |
| E-01 | Infra / Core | `GET` | `/health` | `public` | 1 | 2.5 | `users` (DB ping) |
| E-02 | M1 Auth & Users | `POST` | `/auth/register` | `public` | 2 | 2.5 | `users`, `audit_log` |
| E-03 | M1 Auth & Users | `POST` | `/auth/login` | `public` | 2 | 2.5 | `users`, `audit_log` |
| E-04 | M1 Auth & Users | `GET` | `/auth/me` | `recruiter`, `admin` | 2 | 2.5 | `users` |
| E-05 | M1 Auth & Users | `PATCH` | `/auth/me` | `recruiter`, `admin` | 2 | 2.5 | `users`, `audit_log` |
| E-06 | M1 Auth & Users | `POST` | `/auth/change-password` | `recruiter`, `admin` | 2 | 2.5 | `users`, `audit_log` |
| E-07 | M1 Auth & Admin | `GET` | `/admin/users` | `admin` | 2 | 2.5 | `users` |
| E-08 | M1 Auth & Admin | `PATCH` | `/admin/users/{user_id}` | `admin` | 2 | 2.5 | `users`, `audit_log` |
| E-09 | M1 Auth & Admin | `GET` | `/admin/audit-logs` | `admin` | 2 | 2.5 | `audit_log`, `users` |
| E-10 | M1 Auth & Admin | `GET` | `/admin/config` | `admin` | 2 | 2.5 | `audit_log` |
| E-11 | M1 Auth & Admin | `PUT` | `/admin/config` | `admin` | 2 | 2.5 | `audit_log` |
| E-12 | M2 Job Mgmt | `POST` | `/jobs` | `recruiter` | 2 | 2.5 | `jobs`, `scoring_configs`, `audit_log` |
| E-13 | M2 Job Mgmt | `GET` | `/jobs` | `recruiter` | 2 | 2.5 | `jobs`, `job_requirements`, `screenings` |
| E-14 | M2 Job Mgmt | `GET` | `/jobs/{job_id}` | `recruiter` | 2 | 2.5 | `jobs`, `job_requirements`, `scoring_configs` |
| E-15 | M2 Job Mgmt | `PATCH` | `/jobs/{job_id}` | `recruiter` | 2 | 2.5 | `jobs`, `audit_log` |
| E-16 | M2 Job Mgmt | `DELETE` | `/jobs/{job_id}` | `recruiter` | 2 | 2.5 | `jobs`, `job_requirements`, `scoring_configs`, `screenings`, `screening_results`, `embeddings`, `audit_log` |
| E-17 | M2 Job Mgmt | `POST` | `/jobs/{job_id}/extract-file` | `recruiter` | 2 | 2.5 | `jobs`, `audit_log` |
| E-18 | M3 Resumes | `POST` | `/resumes/upload` | `recruiter` | 3 | 3.5 | `resumes`, `candidates`, `parsed_profiles`, `embeddings`, `audit_log` |
| E-19 | M3 Resumes | `GET` | `/resumes` | `recruiter` | 3 | 3.5 | `resumes`, `candidates` |
| E-20 | M3 Resumes | `GET` | `/resumes/{resume_id}` | `recruiter` | 3 | 3.5 | `resumes`, `candidates`, `parsed_profiles` |
| E-21 | M3 Resumes | `GET` | `/resumes/{resume_id}/profile` | `recruiter` | 3 | 3.5 | `resumes`, `parsed_profiles`, `candidates` |
| E-22 | M3 Resumes | `POST` | `/resumes/{resume_id}/reparse` | `recruiter` | 3 | 3.5 | `resumes`, `parsed_profiles`, `candidates`, `embeddings`, `audit_log` |
| E-23 | M3 Resumes | `DELETE` | `/resumes/{resume_id}` | `recruiter` | 3 | 3.5 | `resumes`, `parsed_profiles`, `candidates`, `embeddings`, `screening_results`, `status_history`, `feedback` (`planned, not created` until Phase 12, `D-013`), `audit_log` |
| E-24 | M3 Resumes | `GET` | `/candidates` | `recruiter` | 3 | 3.5 | `candidates`, `resumes`, `parsed_profiles` |
| E-25 | M3 Resumes | `GET` | `/candidates/{candidate_id}` | `recruiter` | 3 | 3.5 | `candidates`, `resumes`, `parsed_profiles`, `screening_results` |
| E-26 | M4 JD Intel | `POST` | `/jobs/{job_id}/analyze` | `recruiter` | 4 | 4.5 | `jobs`, `job_requirements`, `embeddings`, `audit_log` |
| E-27 | M4 JD Intel | `GET` | `/jobs/{job_id}/requirements` | `recruiter` | 4 | 4.5 | `jobs`, `job_requirements` |
| E-28 | M4 JD Intel | `PUT` | `/jobs/{job_id}/requirements/confirm` | `recruiter` | 4 | 4.5 | `jobs`, `job_requirements`, `embeddings`, `audit_log` |
| E-29 | M5 Matching | `POST` | `/jobs/{job_id}/score-preview` | `recruiter` | 6 | 6.5 | `jobs`, `job_requirements`, `scoring_configs`, `resumes`, `parsed_profiles`, `embeddings` |
| E-30 | M6 Screening | `GET` | `/jobs/{job_id}/scoring-config` | `recruiter` | 6 | 6.5 | `jobs`, `scoring_configs` |
| E-31 | M6 Screening | `PUT` | `/jobs/{job_id}/scoring-config` | `recruiter` | 6 | 6.5 | `jobs`, `scoring_configs`, `audit_log` |
| E-32 | M6 Screening | `POST` | `/jobs/{job_id}/screenings` | `recruiter` | 6 | 6.5 | `jobs`, `job_requirements`, `scoring_configs`, `resumes`, `parsed_profiles`, `candidates`, `embeddings`, `screenings`, `screening_results`, `audit_log` |
| E-33 | M6 Screening | `GET` | `/jobs/{job_id}/screenings` | `recruiter` | 6 | 6.5 | `jobs`, `screenings` |
| E-34 | M6 Screening | `GET` | `/screenings/{screening_id}` | `recruiter` | 6 | 6.5 | `screenings`, `jobs` |
| E-35 | M6 Screening | `GET` | `/jobs/{job_id}/results` | `recruiter` | 6 | 6.5 | `screening_results`, `screenings`, `candidates`, `resumes`, `parsed_profiles` |
| E-36 | M6/M7/M9 | `GET` | `/screening-results/{result_id}` | `recruiter` | 6 | 6.5 | `screening_results`, `candidates`, `resumes`, `parsed_profiles`, `status_history` |
| E-37 | M6 Screening | `PATCH` | `/screening-results/{result_id}/status` | `recruiter` | 6 | 6.5 | `screening_results`, `status_history`, `audit_log` |
| E-38 | M6 Screening | `GET` | `/screening-results/{result_id}/status-history` | `recruiter` | 6 | 6.5 | `status_history`, `screening_results`, `users` |
| E-39 | M6 Screening | `POST` | `/jobs/{job_id}/results/compare` | `recruiter` | 6 | 6.5 | `screening_results`, `candidates`, `parsed_profiles`, `job_requirements` |
| E-40 | M7 Explain | `GET` | `/screening-results/{result_id}/explanation` | `recruiter` | 6 | 6.5 | `screening_results`, `parsed_profiles`, `job_requirements` |
| E-41 | M7 Explain | `POST` | `/screening-results/{result_id}/explanation/rephrase` | `recruiter` | 7 | 7.5 | `screening_results`, `parsed_profiles`, `audit_log` |
| E-42 | M9 Skill Gaps | `GET` | `/screening-results/{result_id}/skill-gaps` | `recruiter` | 6 | 6.5 | `screening_results`, `parsed_profiles`, `job_requirements` |
| E-43 | M8 Summaries | `POST` | `/screening-results/{result_id}/summary` | `recruiter` | 7 | 7.5 | `screening_results`, `parsed_profiles`, `candidates`, `audit_log` |
| E-44 | M8 Summaries | `GET` | `/screening-results/{result_id}/summary` | `recruiter` | 7 | 7.5 | `screening_results`, `candidates` |
| E-45 | M10 Questions | `POST` | `/screening-results/{result_id}/interview-questions` | `recruiter` | 7 | 7.5 | `screening_results`, `parsed_profiles`, `job_requirements`, `candidates`, `audit_log` |
| E-46 | M10 Questions | `GET` | `/screening-results/{result_id}/interview-questions` | `recruiter` | 7 | 7.5 | `screening_results`, `candidates` |
| E-47 | M11 NL Search | `POST` | `/search/candidates` | `recruiter` | 8 | 8.5 | `candidates`, `resumes`, `parsed_profiles`, `embeddings`, `audit_log` |
| E-48 | M11 NL Search | `GET` | `/candidates/{candidate_id}/similar` | `recruiter` | 8 | 8.5 | `candidates`, `resumes`, `parsed_profiles`, `embeddings` |
| E-49 | M12 Analytics | `GET` | `/analytics/overview` | `recruiter` | 9 | 9.5 | `jobs`, `resumes`, `candidates`, `screenings`, `screening_results`, `parsed_profiles` |
| E-50 | M12 Analytics | `GET` | `/analytics/jobs/{job_id}` | `recruiter` | 9 | 9.5 | `jobs`, `screenings`, `screening_results`, `parsed_profiles` |
| E-51 | M12 Analytics | `GET` | `/admin/analytics/overview` | `admin` | 9 | 9.5 | `users`, `jobs`, `resumes`, `candidates`, `screenings`, `screening_results`, `audit_log` |
| E-52 | Phase 12 Feedback | `POST` | `/screening-results/{result_id}/feedback` | `recruiter` | 12 | 12 | `feedback` (`planned, not created` until Phase 12, `D-013`), `screening_results`, `audit_log` |
| E-53 | Phase 12 Feedback | `GET` | `/jobs/{job_id}/feedback` | `recruiter` | 12 | 12 | `feedback` (`planned, not created` until Phase 12, `D-013`), `screening_results`, `jobs` |

---

## 4. Detailed Endpoint Specifications

### Core Infrastructure (Phase 1)

#### `E-01` `GET /api/v1/health`
- **Role:** `public`
- **Phase:** Backend `1` | Frontend `2.5`
- **Tables:** `users` (`SELECT 1` DB connectivity check)
- **Request:** None
- **Response `200 OK`:** `HealthResponse` (`{ status: "ok", database: "connected", version: "1.0.0", timestamp: string }`)
- **Errors:** `500` (`database: "disconnected"`)

---

### Module M1: Authentication, Users & Administration (Backend Phase 2 / Frontend Phase 2.5)

#### `E-02` `POST /api/v1/auth/register`
- **Role:** `public` (Always creates a `recruiter`; `admin` accounts are created only via `scripts/seed_admin.py`)
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `users`, `audit_log`
- **Request Body:** `RegisterRequest` (`{ email: string, password: string (min 8 chars), full_name: string }`)
- **Response `201 Created`:** `AuthTokenResponse` (`{ access_token: string, token_type: "bearer", expires_in: number, user: UserProfile }`)
- **Errors:** `409 Conflict` (email already registered), `422 Validation Error`, `429 Rate Limit Exceeded`

#### `E-03` `POST /api/v1/auth/login`
- **Role:** `public`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `users`, `audit_log`
- **Request Body:** `LoginRequest` (`{ email: string, password: string }`)
- **Response `200 OK`:** `AuthTokenResponse`
- **Errors:** `401 Unauthorized` (invalid email/password or account deactivated), `422 Validation Error`, `429 Rate Limit Exceeded`

#### `E-04` `GET /api/v1/auth/me`
- **Role:** `recruiter`, `admin`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `users`
- **Request:** Bearer token
- **Response `200 OK`:** `UserProfile`
- **Errors:** `401 Unauthorized`

#### `E-05` `PATCH /api/v1/auth/me`
- **Role:** `recruiter`, `admin`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `users`, `audit_log`
- **Request Body:** `UpdateProfileRequest` (`{ full_name?: string, email?: string }`)
- **Response `200 OK`:** `UserProfile`
- **Errors:** `401 Unauthorized`, `409 Conflict` (email taken), `422 Validation Error`

#### `E-06` `POST /api/v1/auth/change-password`
- **Role:** `recruiter`, `admin`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `users`, `audit_log`
- **Request Body:** `ChangePasswordRequest` (`{ current_password: string, new_password: string }`)
- **Response `204 No Content`**
- **Errors:** `400 Bad Request` (incorrect `current_password`), `401 Unauthorized`, `422 Validation Error`

#### `E-07` `GET /api/v1/admin/users`
- **Role:** `admin`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `users`
- **Query Params:** `page?: number`, `page_size?: number`, `role?: UserRole`, `is_active?: boolean`, `search?: string`
- **Response `200 OK`:** `PaginatedResponse<UserProfile>`
- **Errors:** `401 Unauthorized`, `403 Forbidden` (non-admin)

#### `E-08` `PATCH /api/v1/admin/users/{user_id}`
- **Role:** `admin`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `users`, `audit_log`
- **Request Body:** `AdminUpdateUserRequest` (`{ is_active?: boolean, full_name?: string }`)
- **Response `200 OK`:** `UserProfile`
- **Errors:** `400 Bad Request` (admin attempting to deactivate own account), `401 Unauthorized`, `403 Forbidden`, `404 Not Found`

#### `E-09` `GET /api/v1/admin/audit-logs`
- **Role:** `admin`
- **Phase:** Backend `2` | Frontend `2.5` (and Phase `9.5` activity feed)
- **Tables:** `audit_log`, `users`
- **Query Params:** `page?: number`, `page_size?: number`, `actor_id?: string`, `action?: string`, `target_type?: string`, `date_from?: string`, `date_to?: string`
- **Response `200 OK`:** `PaginatedResponse<AuditLogItem>`
- **Errors:** `401 Unauthorized`, `403 Forbidden`

#### `E-10` `GET /api/v1/admin/config`
- **Role:** `admin`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `audit_log`
- **Response `200 OK`:** `SystemConfigResponse` (`{ max_upload_file_size_mb: number, max_upload_batch_size: number, data_retention_days: number, default_scoring_weights: ScoringWeights, llm_provider_configured: boolean, embedding_model: string }`)
- **Errors:** `401 Unauthorized`, `403 Forbidden`

#### `E-11` `PUT /api/v1/admin/config`
- **Role:** `admin`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `audit_log`
- **Request Body:** `UpdateSystemConfigRequest` (`{ max_upload_file_size_mb?: number, max_upload_batch_size?: number, data_retention_days?: number }`)
- **Response `200 OK`:** `SystemConfigResponse`
- **Errors:** `401 Unauthorized`, `403 Forbidden`, `422 Validation Error`

---

### Module M2: Job Management (Backend Phase 2 / Frontend Phase 2.5)

#### `E-12` `POST /api/v1/jobs`
- **Role:** `recruiter`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `jobs`, `scoring_configs`, `audit_log`
- **Request Body:** `CreateJobRequest`
- **Response `201 Created`:** `JobDetail` (automatically initializes `scoring_configs` version `1`)
- **Errors:** `401 Unauthorized`, `403 Forbidden`, `422 Validation Error`

#### `E-13` `GET /api/v1/jobs`
- **Role:** `recruiter`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `jobs`, `job_requirements`, `screenings`
- **Query Params:** `page?: number`, `page_size?: number`, `status?: JobStatus`, `search?: string`
- **Response `200 OK`:** `PaginatedResponse<JobSummary>` (scoped strictly to `owner_id == current_user.id`)
- **Errors:** `401 Unauthorized`, `403 Forbidden`

#### `E-14` `GET /api/v1/jobs/{job_id}`
- **Role:** `recruiter`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `jobs`, `job_requirements`, `scoring_configs`
- **Response `200 OK`:** `JobDetail`
- **Errors:** `401 Unauthorized`, `404 Not Found` (if not found or owned by another recruiter)

#### `E-15` `PATCH /api/v1/jobs/{job_id}`
- **Role:** `recruiter`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `jobs`, `audit_log`
- **Request Body:** `UpdateJobRequest` (partial M2 fields)
- **Response `200 OK`:** `JobDetail`
- **Errors:** `401 Unauthorized`, `404 Not Found`, `422 Validation Error`

#### `E-16` `DELETE /api/v1/jobs/{job_id}`
- **Role:** `recruiter`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `jobs`, `job_requirements`, `scoring_configs`, `screenings`, `screening_results`, `embeddings`, `audit_log`
- **Response `204 No Content`**
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-17` `POST /api/v1/jobs/{job_id}/extract-file`
- **Role:** `recruiter`
- **Phase:** Backend `2` | Frontend `2.5`
- **Tables:** `jobs`, `audit_log`
- **Request:** `multipart/form-data` with `file` (PDF, DOCX, or TXT $\le 5\text{ MB}$) and `apply_to_description?: boolean` (default `false` so existing fields are never overwritten without confirmation per `SPEC.md §4 M2`)
- **Response `200 OK`:** `JobFileExtractionResponse` (`{ job_id: string, filename: string, extracted_text: string, character_count: number, applied_to_job: boolean, existing_description_present: boolean }`)
- **Errors:** `400 Bad Request` (corrupt/empty file), `401 Unauthorized`, `404 Not Found`, `413 Payload Too Large`, `415 Unsupported Media Type`

---

### Module M3: Resume Upload, Parsing & Candidates (Backend Phase 3 / Frontend Phase 3.5)

#### `E-18` `POST /api/v1/resumes/upload`
- **Role:** `recruiter`
- **Phase:** Backend `3` | Frontend `3.5`
- **Tables:** `resumes`, `candidates`, `parsed_profiles`, `embeddings`, `audit_log`
- **Request:** `multipart/form-data` with `files: File[]` (up to 20 PDF/DOCX files, $\le 5\text{ MB}$ each by default)
- **Response `202 Accepted`:** `ResumeUploadBatchResponse` (per-file acceptance, SHA-256 duplicate detection, or validation rejection; accepted files begin background parsing immediately)
- **Errors:** `400 Bad Request` (0 files or $> 20$ files in batch), `401 Unauthorized`, `413 Payload Too Large`, `429 Rate Limit Exceeded`

#### `E-19` `GET /api/v1/resumes`
- **Role:** `recruiter`
- **Phase:** Backend `3` | Frontend `3.5`
- **Tables:** `resumes`, `candidates`
- **Query Params:** `page?: number`, `page_size?: number`, `status?: ResumeStatus`, `search?: string`, `blind_review?: boolean`
- **Response `200 OK`:** `PaginatedResponse<ResumeSummary>`
- **Errors:** `401 Unauthorized`

#### `E-20` `GET /api/v1/resumes/{resume_id}`
- **Role:** `recruiter`
- **Phase:** Backend `3` | Frontend `3.5`
- **Tables:** `resumes`, `candidates`, `parsed_profiles`
- **Query Params:** `blind_review?: boolean`
- **Response `200 OK`:** `ResumeDetail`
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-21` `GET /api/v1/resumes/{resume_id}/profile`
- **Role:** `recruiter`
- **Phase:** Backend `3` | Frontend `3.5`
- **Tables:** `resumes`, `parsed_profiles`, `candidates`
- **Query Params:** `blind_review?: boolean`
- **Response `200 OK`:** `ParsedProfileResponse` (structured fields with `source_section`, `snippet`, `confidence`, and `null_reason`)
- **Errors:** `401 Unauthorized`, `404 Not Found` (if resume does not exist, belongs to another recruiter, or has not yet produced a parsed profile)

#### `E-22` `POST /api/v1/resumes/{resume_id}/reparse`
- **Role:** `recruiter`
- **Phase:** Backend `3` | Frontend `3.5`
- **Tables:** `resumes`, `parsed_profiles`, `candidates`, `embeddings`, `audit_log`
- **Response `202 Accepted`:** `ResumeSummary` (`status: "parsing"`)
- **Errors:** `401 Unauthorized`, `404 Not Found`, `409 Conflict` (already `parsing`)

#### `E-23` `DELETE /api/v1/resumes/{resume_id}`
- **Role:** `recruiter`
- **Phase:** Backend `3` | Frontend `3.5`
- **Tables:** `resumes`, `parsed_profiles`, `candidates`, `embeddings`, `screening_results`, `status_history`, `feedback` (`planned, not created` until Phase 12 per `D-013`), `audit_log`
- **Behavior:** Removes the original file from `backend/storage/`, cascades deletion of `parsed_profiles`, `embeddings`, `screening_results`, `status_history` (and `feedback` once created in Phase 12 per `D-013`), and orphaned `candidates` row, and logs `resume.deleted` in `audit_log`.
- **Response `204 No Content`**
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-24` `GET /api/v1/candidates`
- **Role:** `recruiter`
- **Phase:** Backend `3` | Frontend `3.5`
- **Tables:** `candidates`, `resumes`, `parsed_profiles`
- **Query Params:** `page?: number`, `page_size?: number`, `search?: string`, `skill?: string`, `blind_review?: boolean`
- **Response `200 OK`:** `PaginatedResponse<CandidateSummary>`
- **Errors:** `401 Unauthorized`

#### `E-25` `GET /api/v1/candidates/{candidate_id}`
- **Role:** `recruiter`
- **Phase:** Backend `3` | Frontend `3.5`
- **Tables:** `candidates`, `resumes`, `parsed_profiles`, `screening_results`
- **Query Params:** `blind_review?: boolean`
- **Response `200 OK`:** `CandidateDetail` (includes linked resumes, latest `ParsedProfileResponse`, and screening history across jobs)
- **Errors:** `401 Unauthorized`, `404 Not Found`

---

### Module M4: Job Description Intelligence (Backend Phase 4 / Frontend Phase 4.5)

#### `E-26` `POST /api/v1/jobs/{job_id}/analyze`
- **Role:** `recruiter`
- **Phase:** Backend `4` | Frontend `4.5`
- **Tables:** `jobs`, `job_requirements`, `embeddings`, `audit_log`
- **Behavior:** Runs the M4 JD analyzer on `jobs` fields + `description`, populating `job_requirements.extracted_data` (`StructuredJob`) without clobbering any previously recruiter-confirmed requirements unless `reset_confirmed: true` is passed.
- **Request Body:** `AnalyzeJobRequest` (`{ reset_confirmed?: boolean }`)
- **Response `200 OK`:** `JobRequirementsResponse` (`{ job_id, extracted_data, confirmed_data, active_data, is_confirmed, analyzed_at, confirmed_at }`)
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-27` `GET /api/v1/jobs/{job_id}/requirements`
- **Role:** `recruiter`
- **Phase:** Backend `4` | Frontend `4.5`
- **Tables:** `jobs`, `job_requirements`
- **Response `200 OK`:** `JobRequirementsResponse` (`active_data` returns `confirmed_data` when `is_confirmed == true`, else `extracted_data`)
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-28` `PUT /api/v1/jobs/{job_id}/requirements/confirm`
- **Role:** `recruiter`
- **Phase:** Backend `4` | Frontend `4.5`
- **Tables:** `jobs`, `job_requirements`, `embeddings`, `audit_log`
- **Behavior:** Saves recruiter-edited requirements (`add`, `remove`, `rename`, move between `required_skills` and `preferred_skills`, update `importance`, `min_years_experience`, `allow_equivalent_experience`, `education`, `certifications`, `responsibilities`), sets `is_confirmed = true`, syncs top-level `jobs` fields, and refreshes job `embeddings`.
- **Request Body:** `ConfirmJobRequirementsRequest` (`StructuredJob`)
- **Response `200 OK`:** `JobRequirementsResponse`
- **Errors:** `401 Unauthorized`, `404 Not Found`, `422 Validation Error`

---

### Module M5: Matching Engine Preview (Engine in Phase 5 via Service/CLI; HTTP Endpoint in Backend Phase 6 / Frontend Phase 6.5)

> **Note:** Phase 5 builds the M5 matching engine (`backend/app/ai/matching/`) as a pure Python service plus CLI script (`scripts/eval_matching.py`) and exposes **no HTTP endpoints** in Phase 5. Until Phase 6, the matching engine is reached exclusively through the CLI script (`scripts/eval_matching.py`). The HTTP preview endpoint `E-29` is mounted in **Backend Phase 6** and consumed in **Frontend Phase 6.5**.

#### `E-29` `POST /api/v1/jobs/{job_id}/score-preview`
- **Role:** `recruiter`
- **Phase:** Backend `6` | Frontend `6.5` *(until Phase 6, the engine is reached through the CLI script `scripts/eval_matching.py`)*
- **Tables:** `jobs`, `job_requirements`, `scoring_configs`, `resumes`, `parsed_profiles`, `embeddings`
- **Behavior:** Synchronously invokes the pure M5 hybrid matcher for a single parsed resume against the job (with optional custom draft weights) without persisting a screening run.
- **Request Body:** `ScorePreviewRequest` (`{ resume_id: string, weights_override?: ScoringWeights }`)
- **Response `200 OK`:** `ScorePreviewResponse` (full 5-component score, effective weights, mandatory-skill cap status, matched/partial/missing skills, evidence citations, flags)
- **Errors:** `400 Bad Request` (resume not in `parsed` status), `401 Unauthorized`, `404 Not Found`, `422 Validation Error` (weights do not sum to 100%)

---

### Module M6, M7, M9: Screening, Ranking, Status, Explainability & Skill Gaps (Backend Phase 6 / Frontend Phase 6.5)

#### `E-30` `GET /api/v1/jobs/{job_id}/scoring-config`
- **Role:** `recruiter`
- **Phase:** Backend `6` | Frontend `6.5`
- **Tables:** `jobs`, `scoring_configs`
- **Response `200 OK`:** `ScoringConfigHistoryResponse` (`{ active_config: ScoringConfigResponse, history: ScoringConfigResponse[] }`)
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-31` `PUT /api/v1/jobs/{job_id}/scoring-config`
- **Role:** `recruiter`
- **Phase:** Backend `6` | Frontend `6.5`
- **Tables:** `jobs`, `scoring_configs`, `audit_log`
- **Behavior:** Validates that the 5 component weights sum to `1.0` (or `100%` within `1e-4`), marks prior config `is_active = false`, and inserts a new `scoring_configs` row with `version = prev + 1`.
- **Request Body:** `UpdateScoringConfigRequest` (`{ weights: ScoringWeights }`)
- **Response `200 OK`:** `ScoringConfigResponse`
- **Errors:** `401 Unauthorized`, `404 Not Found`, `422 Validation Error` (weights do not sum to 1.0)

#### `E-32` `POST /api/v1/jobs/{job_id}/screenings`
- **Role:** `recruiter`
- **Phase:** Backend `6` | Frontend `6.5`
- **Tables:** `jobs`, `job_requirements`, `scoring_configs`, `resumes`, `parsed_profiles`, `candidates`, `embeddings`, `screenings`, `screening_results`, `audit_log`
- **Request Body:** `TriggerScreeningRequest` (`{ resume_ids?: string[] | null }` — `null` or omitted screens all `parsed` resumes owned by the recruiter)
- **Response `202 Accepted`:** `ScreeningRunResponse`
- **Errors:** `400 Bad Request` (0 parsed resumes available), `401 Unauthorized`, `404 Not Found`, `409 Conflict` (screening run already in progress for this job), `429 Rate Limit Exceeded`

#### `E-33` `GET /api/v1/jobs/{job_id}/screenings`
- **Role:** `recruiter`
- **Phase:** Backend `6` | Frontend `6.5`
- **Tables:** `jobs`, `screenings`
- **Query Params:** `page?: number`, `page_size?: number`
- **Response `200 OK`:** `PaginatedResponse<ScreeningRunResponse>`
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-34` `GET /api/v1/screenings/{screening_id}`
- **Role:** `recruiter`
- **Phase:** Backend `6` | Frontend `6.5`
- **Tables:** `screenings`, `jobs`
- **Response `200 OK`:** `ScreeningRunResponse`
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-35` `GET /api/v1/jobs/{job_id}/results`
- **Role:** `recruiter`
- **Phase:** Backend `6` | Frontend `6.5`
- **Tables:** `screening_results`, `screenings`, `candidates`, `resumes`, `parsed_profiles`
- **Query Params:**
  - `screening_id?: string` (defaults to latest completed screening for `job_id`)
  - `page?: number`, `page_size?: number`
  - `sort_by?: "final_score" | "technical_skills" | "relevant_experience" | "education" | "projects" | "certifications" | "rank_position"` (default `"final_score"`)
  - `sort_order?: "asc" | "desc"` (default `"desc"`)
  - `min_score?: number` (`0`–`100`)
  - `band?: ScoreBand`
  - `review_status?: ReviewStatus`
  - `skills?: string` (comma-separated canonical skill names; candidate must have `strong_match` or `partial_match`)
  - `min_experience_years?: number`
  - `education_level?: EducationLevel`
  - `blind_review?: boolean`
- **Response `200 OK`:** `PaginatedResponse<ScreeningResultListItem>`
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-36` `GET /api/v1/screening-results/{result_id}`
- **Role:** `recruiter`
- **Phase:** Backend `6` | Frontend `6.5`
- **Tables:** `screening_results`, `candidates`, `resumes`, `parsed_profiles`, `status_history`
- **Query Params:** `blind_review?: boolean`
- **Response `200 OK`:** `ScreeningResultDetail` (includes full component breakdown, M7 explanation with snippets, M9 4-class skill gap analysis, flags, config version, and status history)
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-37` `PATCH /api/v1/screening-results/{result_id}/status`
- **Role:** `recruiter`
- **Phase:** Backend `6` | Frontend `6.5`
- **Tables:** `screening_results`, `status_history`, `audit_log`
- **Request Body:** `UpdateResultStatusRequest` (`{ status: ReviewStatus, note?: string | null }`)
- **Response `200 OK`:** `ScreeningResultDetail`
- **Errors:** `401 Unauthorized`, `404 Not Found`, `422 Validation Error`

#### `E-38` `GET /api/v1/screening-results/{result_id}/status-history`
- **Role:** `recruiter`
- **Phase:** Backend `6` | Frontend `6.5`
- **Tables:** `status_history`, `screening_results`, `users`
- **Response `200 OK`:** `StatusHistoryListResponse` (`{ result_id: string, items: StatusHistoryEntry[] }`)
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-39` `POST /api/v1/jobs/{job_id}/results/compare`
- **Role:** `recruiter`
- **Phase:** Backend `6` | Frontend `6.5`
- **Tables:** `screening_results`, `candidates`, `parsed_profiles`, `job_requirements`
- **Request Body:** `CompareCandidatesRequest` (`{ result_ids: string[], blind_review?: boolean }` — must contain `2` to `4` `result_ids` belonging to `job_id`)
- **Response `200 OK`:** `CompareCandidatesResponse` (`{ job_id: string, config_version_mismatch: boolean, results: ScreeningResultDetail[], shared_strong_skills: string[], skill_matrix: SkillComparisonRow[] }`)
- **Errors:** `400 Bad Request` (fewer than 2 or more than 4 IDs, or results from different jobs), `401 Unauthorized`, `404 Not Found`

#### `E-40` `GET /api/v1/screening-results/{result_id}/explanation`
- **Role:** `recruiter`
- **Phase:** Backend `6` | Frontend `6.5`
- **Tables:** `screening_results`, `parsed_profiles`, `job_requirements`
- **Query Params:** `blind_review?: boolean`
- **Response `200 OK`:** `ExplanationResponse` (M7 template-based strengths, gaps, per-component breakdown, and verbatim section/snippet evidence)
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-41` `POST /api/v1/screening-results/{result_id}/explanation/rephrase`
- **Role:** `recruiter`
- **Phase:** Backend `7` | Frontend `7.5` *(requires `LLMService` + `GroundingValidator`, which arrive in Phase 7; `E-40` template explanation stays in Phase 6 / 6.5)*
- **Tables:** `screening_results`, `parsed_profiles`, `audit_log`
- **Behavior:** Uses `LLMService` + `GroundingValidator` to polish explanation prose without adding any unverified facts; falls back to template if no LLM key or if grounding validation fails.
- **Query Params:** `blind_review?: boolean`
- **Response `200 OK`:** `ExplanationResponse`
- **Errors:** `401 Unauthorized`, `404 Not Found`, `429 Rate Limit Exceeded`

#### `E-42` `GET /api/v1/screening-results/{result_id}/skill-gaps`
- **Role:** `recruiter`
- **Phase:** Backend `6` | Frontend `6.5`
- **Tables:** `screening_results`, `parsed_profiles`, `job_requirements`
- **Query Params:** `blind_review?: boolean`
- **Response `200 OK`:** `SkillGapAnalysisResponse` (M9 4-class breakdown: `strong_match`, `partial_match`, `missing` labeled `"Not demonstrated in the resume"`, and `additional`)
- **Errors:** `401 Unauthorized`, `404 Not Found`

---

### Module M7 (LLM Rephrase), M8 & M10: Candidate Summaries & Interview Questions (Backend Phase 7 / Frontend Phase 7.5)

#### `E-43` `POST /api/v1/screening-results/{result_id}/summary`
- **Role:** `recruiter`
- **Phase:** Backend `7` | Frontend `7.5`
- **Tables:** `screening_results`, `parsed_profiles`, `candidates`, `audit_log`
- **Request Body:** `GenerateSummaryRequest` (`{ force_regenerate?: boolean, blind_review?: boolean }`)
- **Response `200 OK`:** `CandidateSummaryArtifactResponse` (M8 structured summary, advisory recommendation: `"Review first" | "Review" | "Weak match on stated requirements"`, `is_template_fallback`, grounding check status)
- **Errors:** `401 Unauthorized`, `404 Not Found`, `429 Rate Limit Exceeded`

#### `E-44` `GET /api/v1/screening-results/{result_id}/summary`
- **Role:** `recruiter`
- **Phase:** Backend `7` | Frontend `7.5`
- **Tables:** `screening_results`, `candidates`
- **Query Params:** `blind_review?: boolean`
- **Response `200 OK`:** `CandidateSummaryArtifactResponse` (generates on-the-fly with template fallback if not yet cached)
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-45` `POST /api/v1/screening-results/{result_id}/interview-questions`
- **Role:** `recruiter`
- **Phase:** Backend `7` | Frontend `7.5`
- **Tables:** `screening_results`, `parsed_profiles`, `job_requirements`, `candidates`, `audit_log`
- **Request Body:** `GenerateInterviewQuestionsRequest` (`{ categories?: QuestionCategory[], difficulty?: QuestionDifficulty, count?: number (default 8, max 15), force_regenerate?: boolean, blind_review?: boolean }`)
- **Response `200 OK`:** `InterviewQuestionsResponse` (each question names its `category`, `difficulty`, `target_skill_or_requirement`, `evidence_section`, `evidence_snippet`, `rationale`, and `is_template_fallback`)
- **Errors:** `401 Unauthorized`, `404 Not Found`, `422 Validation Error`, `429 Rate Limit Exceeded`

#### `E-46` `GET /api/v1/screening-results/{result_id}/interview-questions`
- **Role:** `recruiter`
- **Phase:** Backend `7` | Frontend `7.5`
- **Tables:** `screening_results`, `candidates`
- **Query Params:** `blind_review?: boolean`
- **Response `200 OK`:** `InterviewQuestionsResponse`
- **Errors:** `401 Unauthorized`, `404 Not Found`

---

### Module M11: Natural-Language Search & Similar Candidates (Backend Phase 8 / Frontend Phase 8.5)

#### `E-47` `POST /api/v1/search/candidates`
- **Role:** `recruiter`
- **Phase:** Backend `8` | Frontend `8.5`
- **Tables:** `candidates`, `resumes`, `parsed_profiles`, `embeddings`, `audit_log`
- **Request Body:** `NaturalLanguageSearchRequest` (`{ query: string, filters_override?: SearchStructuredFilters, page?: number, page_size?: number, blind_review?: boolean }`)
- **Response `200 OK`:** `NaturalLanguageSearchResponse` (returns `interpreted_query` showing extracted skills, `min_years_experience`, `location`, `education_level`, and `semantic_query_text`, plus paginated `SearchCandidateHit` items with `match_score`, `matched_skills`, and verbatim `evidence_snippets`)
- **Errors:** `401 Unauthorized`, `422 Validation Error`, `429 Rate Limit Exceeded`

#### `E-48` `GET /api/v1/candidates/{candidate_id}/similar`
- **Role:** `recruiter`
- **Phase:** Backend `8` | Frontend `8.5`
- **Tables:** `candidates`, `resumes`, `parsed_profiles`, `embeddings`
- **Query Params:** `limit?: number` (default `5`, max `20`), `blind_review?: boolean`
- **Response `200 OK`:** `SimilarCandidatesResponse` (`{ source_candidate_id: string, items: SearchCandidateHit[] }`)
- **Errors:** `401 Unauthorized`, `404 Not Found`

---

### Module M12: Analytics (Backend Phase 9 / Frontend Phase 9.5)

#### `E-49` `GET /api/v1/analytics/overview`
- **Role:** `recruiter`
- **Phase:** Backend `9` | Frontend `9.5`
- **Tables:** `jobs`, `resumes`, `candidates`, `screenings`, `screening_results`, `parsed_profiles`
- **Query Params:** `job_id?: string`, `date_from?: string`, `date_to?: string`
- **Response `200 OK`:** `AnalyticsOverviewResponse` (totals, average score, candidates per job, score histogram, top skill distribution, experience distribution, status breakdown; returns all `0`s and empty arrays when DB is empty)
- **Errors:** `401 Unauthorized`

#### `E-50` `GET /api/v1/analytics/jobs/{job_id}`
- **Role:** `recruiter`
- **Phase:** Backend `9` | Frontend `9.5`
- **Tables:** `jobs`, `screenings`, `screening_results`, `parsed_profiles`
- **Response `200 OK`:** `JobAnalyticsResponse`
- **Errors:** `401 Unauthorized`, `404 Not Found`

#### `E-51` `GET /api/v1/admin/analytics/overview`
- **Role:** `admin`
- **Phase:** Backend `9` | Frontend `9.5`
- **Tables:** `users`, `jobs`, `resumes`, `candidates`, `screenings`, `screening_results`, `audit_log`
- **Query Params:** `date_from?: string`, `date_to?: string`
- **Response `200 OK`:** `AdminAnalyticsOverviewResponse` (platform-wide user counts, active recruiters, total jobs/resumes/screenings, score/skill distributions, recent audit activity summary — never exposes candidate PII)
- **Errors:** `401 Unauthorized`, `403 Forbidden`

---

### Postponed Module: Recruiter Feedback Loop (Phase 12 — POSTPONED, `D-013`)

> **Note:** Per `D-013`, the `feedback` table is marked `planned, not created` in the ERD; its SQLAlchemy model, Alembic migration, and endpoints `E-52`–`E-53` are created only in **Phase 12**.

#### `E-52` `POST /api/v1/screening-results/{result_id}/feedback`
- **Role:** `recruiter`
- **Phase:** Backend `12` | Frontend `12` (`POSTPONED`)
- **Tables:** `feedback` (`planned, not created` until Phase 12, `D-013`), `screening_results`, `audit_log`
- **Request Body:** `CreateFeedbackRequest` (`{ rating: number (1-5), agrees_with_band: boolean, suggested_band?: ScoreBand | null, comment?: string | null, details?: Record<string, string> }`)
- **Response `201 Created`:** `FeedbackResponse`
- **Errors:** `401 Unauthorized`, `404 Not Found`, `422 Validation Error`

#### `E-53` `GET /api/v1/jobs/{job_id}/feedback`
- **Role:** `recruiter`
- **Phase:** Backend `12` | Frontend `12` (`POSTPONED`)
- **Tables:** `feedback` (`planned, not created` until Phase 12, `D-013`), `screening_results`, `jobs`
- **Query Params:** `page?: number`, `page_size?: number`
- **Response `200 OK`:** `PaginatedResponse<FeedbackResponse>`
- **Errors:** `401 Unauthorized`, `404 Not Found`

---

## 5. Exact TypeScript Types (`frontend/src/api/types.ts`)

Copy the block below verbatim into `frontend/src/api/types.ts`:

```typescript
// ============================================================================
// AUTO-SYNCED FROM docs/api-contract.md — DO NOT EDIT WITHOUT UPDATING CONTRACT
// ============================================================================

export type UserRole = "recruiter" | "admin";

export type JobStatus = "draft" | "open" | "closed" | "archived";

export type EmploymentType = "full_time" | "part_time" | "contract" | "internship";

export type EducationLevel =
  | "none"
  | "high_school"
  | "associate"
  | "bachelor"
  | "master"
  | "phd";

export type FieldRelevance = "related" | "adjacent" | "unrelated";

export type RequirementImportance = "mandatory" | "important" | "nice_to_have";

export type ResumeStatus =
  | "uploaded"
  | "parsing"
  | "parsed"
  | "needs_review"
  | "failed";

export type ResumeSectionName =
  | "contact"
  | "summary"
  | "skills"
  | "experience"
  | "education"
  | "certifications"
  | "projects"
  | "achievements"
  | "other";

export type SkillContextType =
  | "experience_or_project"
  | "skills_section_only"
  | "semantic_high"
  | "semantic_low"
  | "unverified_section";

export type ScreeningStatus = "pending" | "running" | "completed" | "failed";

export type ComponentStatus = "ok" | "insufficient_evidence" | "not_applicable";

export type ScoreBand = "strong" | "moderate" | "weak";

export type ReviewStatus = "new" | "shortlisted" | "rejected" | "on_hold";

export type SkillMatchClass =
  | "strong_match"
  | "partial_match"
  | "missing"
  | "additional";

export type AdvisoryRecommendation =
  | "Review first"
  | "Review"
  | "Weak match on stated requirements";

export type QuestionCategory =
  | "technical"
  | "behavioral"
  | "experience_based"
  | "project_specific";

export type QuestionDifficulty = "beginner" | "intermediate" | "advanced";

// ----------------------------------------------------------------------------
// RFC 7807 ProblemDetails & Common Wrappers
// ----------------------------------------------------------------------------

export interface ValidationFieldError {
  field: string;
  message: string;
  code: string;
}

export interface ProblemDetails {
  type: string;
  title: string;
  status: number;
  detail: string;
  instance: string;
  errors?: ValidationFieldError[];
}

export interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  page_size: number;
  total_pages: number;
}

export interface HealthResponse {
  status: "ok" | "degraded";
  database: "connected" | "disconnected";
  version: string;
  timestamp: string;
}

// ----------------------------------------------------------------------------
// M1: Authentication, Users & Admin
// ----------------------------------------------------------------------------

export interface UserProfile {
  id: string;
  email: string;
  full_name: string;
  role: UserRole;
  is_active: boolean;
  last_login_at: string | null;
  created_at: string;
  updated_at: string;
}

export interface RegisterRequest {
  email: string;
  password: string;
  full_name: string;
}

export interface LoginRequest {
  email: string;
  password: string;
}

export interface AuthTokenResponse {
  access_token: string;
  token_type: "bearer";
  expires_in: number;
  user: UserProfile;
}

export interface UpdateProfileRequest {
  full_name?: string;
  email?: string;
}

export interface ChangePasswordRequest {
  current_password: string;
  new_password: string;
}

export interface AdminUpdateUserRequest {
  is_active?: boolean;
  full_name?: string;
}

export interface AuditLogItem {
  id: string;
  actor_id: string | null;
  actor_email: string | null;
  actor_role: UserRole | "system";
  action: string;
  target_type: string;
  target_id: string | null;
  details: Record<string, string | number | boolean | null>;
  ip_address: string | null;
  created_at: string;
}

export interface ScoringWeights {
  technical_skills: number;
  relevant_experience: number;
  education: number;
  projects: number;
  certifications: number;
}

export interface SystemConfigResponse {
  max_upload_file_size_mb: number;
  max_upload_batch_size: number;
  data_retention_days: number;
  default_scoring_weights: ScoringWeights;
  llm_provider_configured: boolean;
  embedding_model: string;
}

export interface UpdateSystemConfigRequest {
  max_upload_file_size_mb?: number;
  max_upload_batch_size?: number;
  data_retention_days?: number;
}

// ----------------------------------------------------------------------------
// M2 & M4: Jobs & Structured Job Description Intelligence
// ----------------------------------------------------------------------------

export interface CreateJobRequest {
  title: string;
  department?: string | null;
  description: string;
  required_skills?: string[];
  preferred_skills?: string[];
  min_years_experience?: number;
  required_education_level?: EducationLevel;
  required_education_field?: string | null;
  allow_equivalent_experience?: boolean;
  certifications?: string[];
  responsibilities?: string[];
  employment_type?: EmploymentType;
  status?: JobStatus;
}

export interface UpdateJobRequest {
  title?: string;
  department?: string | null;
  description?: string;
  required_skills?: string[];
  preferred_skills?: string[];
  min_years_experience?: number;
  required_education_level?: EducationLevel;
  required_education_field?: string | null;
  allow_equivalent_experience?: boolean;
  certifications?: string[];
  responsibilities?: string[];
  employment_type?: EmploymentType;
  status?: JobStatus;
}

export interface JobSummary {
  id: string;
  owner_id: string;
  title: string;
  department: string | null;
  employment_type: EmploymentType;
  status: JobStatus;
  min_years_experience: number;
  required_skills_count: number;
  requirements_confirmed: boolean;
  latest_screening_id: string | null;
  candidates_screened_count: number;
  created_at: string;
  updated_at: string;
}

export interface JobSkillRequirement {
  skill_name: string;
  canonical_name: string;
  category: string;
  is_required: boolean;
  importance: RequirementImportance;
  cue_snippet: string | null;
}

export interface StructuredJob {
  required_skills: JobSkillRequirement[];
  preferred_skills: JobSkillRequirement[];
  min_years_experience: number;
  allow_equivalent_experience: boolean;
  education_level: EducationLevel;
  education_field: string | null;
  education_importance: RequirementImportance;
  certifications: string[];
  responsibilities: string[];
}

export interface JobRequirementsResponse {
  job_id: string;
  extracted_data: StructuredJob;
  confirmed_data: StructuredJob | null;
  active_data: StructuredJob;
  is_confirmed: boolean;
  analyzed_at: string;
  confirmed_at: string | null;
}

export interface AnalyzeJobRequest {
  reset_confirmed?: boolean;
}

export type ConfirmJobRequirementsRequest = StructuredJob;

export interface ScoringConfigResponse {
  id: string;
  job_id: string;
  version: number;
  weights: ScoringWeights;
  mandatory_cap_rules: {
    one_missing: number;
    two_missing: number;
    three_plus_missing: number;
  };
  is_active: boolean;
  created_by: string | null;
  created_at: string;
}

export interface ScoringConfigHistoryResponse {
  active_config: ScoringConfigResponse;
  history: ScoringConfigResponse[];
}

export interface UpdateScoringConfigRequest {
  weights: ScoringWeights;
}

export interface JobDetail {
  id: string;
  owner_id: string;
  title: string;
  department: string | null;
  description: string;
  required_skills: string[];
  preferred_skills: string[];
  min_years_experience: number;
  required_education_level: EducationLevel;
  required_education_field: string | null;
  allow_equivalent_experience: boolean;
  certifications: string[];
  responsibilities: string[];
  employment_type: EmploymentType;
  status: JobStatus;
  source_filename: string | null;
  requirements: JobRequirementsResponse | null;
  active_scoring_config: ScoringConfigResponse;
  created_at: string;
  updated_at: string;
}

export interface JobFileExtractionResponse {
  job_id: string;
  filename: string;
  extracted_text: string;
  character_count: number;
  applied_to_job: boolean;
  existing_description_present: boolean;
}

// ----------------------------------------------------------------------------
// M3: Resume Upload, ParsedProfile & Candidates
// ----------------------------------------------------------------------------

export interface ExtractedField<T> {
  value: T | null;
  source_section: ResumeSectionName | null;
  snippet: string | null;
  confidence: number;
  null_reason: string | null;
}

export interface ExtractedSkillItem {
  raw_name: string;
  canonical_name: string;
  category: string;
  is_soft_skill: boolean;
  context_type: SkillContextType;
  source_section: ResumeSectionName;
  snippet: string;
  confidence: number;
}

export interface ExtractedExperienceItem {
  company: string | null;
  title: string | null;
  start_date: string | null;
  end_date: string | null;
  is_current: boolean;
  duration_months: number | null;
  description: string;
  demonstrated_skills: string[];
  source_section: ResumeSectionName;
  snippet: string;
  confidence: number;
}

export interface ExtractedEducationItem {
  degree_raw: string | null;
  degree_level: EducationLevel;
  field_of_study: string | null;
  institution: string | null;
  graduation_year: number | null;
  source_section: ResumeSectionName;
  snippet: string;
  confidence: number;
}

export interface ExtractedCertificationItem {
  name: string;
  canonical_name: string;
  issuer: string | null;
  year: number | null;
  source_section: ResumeSectionName;
  snippet: string;
  confidence: number;
}

export interface ExtractedProjectItem {
  title: string;
  description: string;
  demonstrated_skills: string[];
  url: string | null;
  source_section: ResumeSectionName;
  snippet: string;
  confidence: number;
}

export interface ExtractedAchievementItem {
  text: string;
  source_section: ResumeSectionName;
  snippet: string;
  confidence: number;
}

export interface ParsedProfileResponse {
  id: string;
  resume_id: string;
  candidate_id: string;
  contact_info: {
    name: ExtractedField<string>;
    email: ExtractedField<string>;
    phone: ExtractedField<string>;
    location: ExtractedField<string>;
  };
  summary: ExtractedField<string>;
  skills: ExtractedSkillItem[];
  experience: ExtractedExperienceItem[];
  education: ExtractedEducationItem[];
  certifications: ExtractedCertificationItem[];
  projects: ExtractedProjectItem[];
  achievements: ExtractedAchievementItem[];
  total_experience_years: number | null;
  section_map: Record<string, string>;
  extraction_flags: string[];
  parser_version: string;
  created_at: string;
  updated_at: string;
}

export interface ResumeSummary {
  id: string;
  owner_id: string;
  candidate_id: string | null;
  candidate_display_name: string | null;
  candidate_blind_code: string | null;
  original_filename: string;
  mime_type: string;
  file_size_bytes: number;
  content_hash: string;
  status: ResumeStatus;
  status_reason: string | null;
  uploaded_at: string;
  parsed_at: string | null;
}

export interface ResumeDetail extends ResumeSummary {
  parsed_profile: ParsedProfileResponse | null;
}

export interface ResumeUploadBatchItem {
  original_filename: string;
  status: "accepted" | "duplicate_skipped" | "rejected";
  resume_id: string | null;
  duplicate_of_resume_id: string | null;
  reason: string | null;
}

export interface ResumeUploadBatchResponse {
  total_submitted: number;
  accepted_count: number;
  duplicate_count: number;
  rejected_count: number;
  items: ResumeUploadBatchItem[];
}

export interface CandidateSummary {
  id: string;
  owner_id: string;
  blind_code: string;
  display_name: string | null;
  primary_email: string | null;
  primary_phone: string | null;
  location: string | null;
  headline: string | null;
  latest_resume_id: string | null;
  latest_resume_status: ResumeStatus | null;
  top_skills: string[];
  total_experience_years: number | null;
  resumes_count: number;
  created_at: string;
  updated_at: string;
}

export interface CandidateJobHistoryItem {
  screening_result_id: string;
  job_id: string;
  job_title: string;
  final_score: number;
  band: ScoreBand;
  review_status: ReviewStatus;
  screened_at: string;
}

export interface CandidateDetail extends CandidateSummary {
  resumes: ResumeSummary[];
  latest_profile: ParsedProfileResponse | null;
  job_history: CandidateJobHistoryItem[];
}

// ----------------------------------------------------------------------------
// M5, M6, M7, M9: Matching, Screening, Ranking, Explainability & Skill Gaps
// ----------------------------------------------------------------------------

export interface EvidenceCitation {
  claim: string;
  source_section: ResumeSectionName;
  snippet: string;
  score_contribution: number | null;
}

export interface ComponentScoreDetail {
  component:
    | "technical_skills"
    | "relevant_experience"
    | "education"
    | "projects"
    | "certifications";
  score: number;
  status: ComponentStatus;
  original_weight: number;
  effective_weight: number;
  summary_reason: string;
  evidence: EvidenceCitation[];
}

export interface SkillGapItem {
  skill_name: string;
  canonical_name: string;
  category: string;
  classification: SkillMatchClass;
  is_required: boolean;
  importance: RequirementImportance | null;
  credit_awarded: number;
  status_label: string;
  matched_via:
    | "experience_context"
    | "project_context"
    | "skills_section_only"
    | "alias"
    | "semantic"
    | "not_demonstrated"
    | "additional_candidate_skill";
  similarity_score: number | null;
  source_section: ResumeSectionName | null;
  snippet: string | null;
}

export interface SkillGapAnalysisResponse {
  result_id: string;
  job_id: string;
  candidate_id: string;
  wording_notice: "Skills not found in the resume are classified as 'Not demonstrated in the resume', never confirmed absent.";
  strong_match: SkillGapItem[];
  partial_match: SkillGapItem[];
  missing: SkillGapItem[];
  additional: SkillGapItem[];
}

export interface ExplanationResponse {
  result_id: string;
  strengths: EvidenceCitation[];
  gaps: EvidenceCitation[];
  component_narrative: Record<string, string>;
  overall_narrative: string;
  is_llm_rephrased: boolean;
  is_template_fallback: boolean;
}

export interface ScorePreviewRequest {
  resume_id: string;
  weights_override?: ScoringWeights;
}

export interface ScorePreviewResponse {
  job_id: string;
  resume_id: string;
  final_score: number;
  uncapped_score: number;
  band: ScoreBand;
  mandatory_cap_applied: boolean;
  missing_mandatory_skills: string[];
  components: ComponentScoreDetail[];
  skill_analysis: Omit<SkillGapAnalysisResponse, "result_id">;
  flags: string[];
  decision_support_disclaimer: string;
}

export interface TriggerScreeningRequest {
  resume_ids?: string[] | null;
}

export interface ScreeningRunResponse {
  id: string;
  job_id: string;
  scoring_config_id: string;
  config_version: number;
  triggered_by: string | null;
  status: ScreeningStatus;
  total_resumes: number;
  processed_resumes: number;
  failed_resumes: number;
  summary_stats: {
    avg_score: number;
    strong_count: number;
    moderate_count: number;
    weak_count: number;
  } | null;
  error_message: string | null;
  started_at: string;
  completed_at: string | null;
}

export interface ScreeningResultListItem {
  id: string;
  screening_id: string;
  job_id: string;
  resume_id: string;
  candidate_id: string;
  candidate_display_name: string | null;
  candidate_blind_code: string;
  candidate_email: string | null;
  candidate_location: string | null;
  config_version: number;
  final_score: number;
  band: ScoreBand;
  rank_position: number;
  review_status: ReviewStatus;
  component_scores_map: {
    technical_skills: number | null;
    relevant_experience: number | null;
    education: number | null;
    projects: number | null;
    certifications: number | null;
  };
  strong_skills_count: number;
  partial_skills_count: number;
  missing_skills_count: number;
  total_experience_years: number | null;
  education_level: EducationLevel;
  flags: string[];
  decision_support_disclaimer: string;
  created_at: string;
  updated_at: string;
}

export interface StatusHistoryEntry {
  id: string;
  screening_result_id: string;
  old_status: ReviewStatus;
  new_status: ReviewStatus;
  note: string | null;
  changed_by_id: string | null;
  changed_by_name: string | null;
  changed_at: string;
}

export interface StatusHistoryListResponse {
  result_id: string;
  items: StatusHistoryEntry[];
}

export interface UpdateResultStatusRequest {
  status: ReviewStatus;
  note?: string | null;
}

export interface ScreeningResultDetail extends ScreeningResultListItem {
  scoring_config_id: string;
  uncapped_score: number;
  components: ComponentScoreDetail[];
  skill_analysis: SkillGapAnalysisResponse;
  explanation: ExplanationResponse;
  evidence: EvidenceCitation[];
  summary_data: CandidateSummaryArtifactResponse | null;
  interview_questions: InterviewQuestionsResponse | null;
  status_history: StatusHistoryEntry[];
}

export interface CompareCandidatesRequest {
  result_ids: string[];
  blind_review?: boolean;
}

export interface SkillComparisonRow {
  skill_name: string;
  is_required: boolean;
  importance: RequirementImportance;
  per_result_class: Record<string, SkillMatchClass>;
}

export interface CompareCandidatesResponse {
  job_id: string;
  config_version_mismatch: boolean;
  results: ScreeningResultDetail[];
  shared_strong_skills: string[];
  skill_matrix: SkillComparisonRow[];
}

// ----------------------------------------------------------------------------
// M8 & M10: Candidate Summaries & Interview Questions
// ----------------------------------------------------------------------------

export interface GenerateSummaryRequest {
  force_regenerate?: boolean;
  blind_review?: boolean;
}

export interface CandidateSummaryArtifactResponse {
  result_id: string;
  candidate_id: string;
  job_id: string;
  background_overview: string;
  main_skills_highlighted: string[];
  experience_recap: string;
  relevant_projects_recap: string;
  education_recap: string;
  key_strengths: EvidenceCitation[];
  potential_gaps: EvidenceCitation[];
  advisory_recommendation: AdvisoryRecommendation;
  decision_support_disclaimer: string;
  is_template_fallback: boolean;
  grounding_verified: boolean;
  generated_at: string;
}

export interface GenerateInterviewQuestionsRequest {
  categories?: QuestionCategory[];
  difficulty?: QuestionDifficulty;
  count?: number;
  force_regenerate?: boolean;
  blind_review?: boolean;
}

export interface InterviewQuestionItem {
  id: string;
  category: QuestionCategory;
  difficulty: QuestionDifficulty;
  question_text: string;
  target_skill_or_requirement: string;
  target_status: "partial_match" | "missing" | "experience_claim" | "project_claim";
  evidence_section: ResumeSectionName | null;
  evidence_snippet: string | null;
  evaluation_focus: string;
}

export interface InterviewQuestionsResponse {
  result_id: string;
  candidate_id: string;
  job_id: string;
  questions: InterviewQuestionItem[];
  is_template_fallback: boolean;
  grounding_verified: boolean;
  generated_at: string;
}

// ----------------------------------------------------------------------------
// M11: Natural-Language Candidate Search
// ----------------------------------------------------------------------------

export interface SearchStructuredFilters {
  skills?: string[];
  min_years_experience?: number | null;
  location?: string | null;
  education_level?: EducationLevel | null;
}

export interface NaturalLanguageSearchRequest {
  query: string;
  filters_override?: SearchStructuredFilters;
  page?: number;
  page_size?: number;
  blind_review?: boolean;
}

export interface InterpretedSearchQuery {
  raw_query: string;
  extracted_skills: string[];
  min_years_experience: number | null;
  location: string | null;
  education_level: EducationLevel | null;
  semantic_query_text: string;
}

export interface SearchCandidateHit {
  candidate_id: string;
  resume_id: string;
  display_name: string | null;
  blind_code: string;
  primary_email: string | null;
  location: string | null;
  headline: string | null;
  total_experience_years: number | null;
  education_level: EducationLevel;
  matched_skills: string[];
  relevance_score: number;
  evidence_snippets: EvidenceCitation[];
}

export interface NaturalLanguageSearchResponse {
  interpreted_query: InterpretedSearchQuery;
  results: PaginatedResponse<SearchCandidateHit>;
}

export interface SimilarCandidatesResponse {
  source_candidate_id: string;
  items: SearchCandidateHit[];
}

// ----------------------------------------------------------------------------
// M12: Analytics
// ----------------------------------------------------------------------------

export interface DistributionBucket {
  label: string;
  count: number;
  percentage: number;
}

export interface JobCandidateCountItem {
  job_id: string;
  job_title: string;
  status: JobStatus;
  candidates_screened: number;
  shortlisted_count: number;
  rejected_count: number;
  average_score: number;
}

export interface AnalyticsOverviewResponse {
  totals: {
    total_jobs: number;
    open_jobs: number;
    resumes_uploaded: number;
    resumes_parsed: number;
    candidates_screened: number;
    shortlisted: number;
    rejected: number;
    on_hold: number;
    average_score: number;
  };
  candidates_per_job: JobCandidateCountItem[];
  score_distribution: DistributionBucket[];
  skill_distribution: DistributionBucket[];
  experience_distribution: DistributionBucket[];
  status_distribution: DistributionBucket[];
}

export interface JobAnalyticsResponse {
  job_id: string;
  job_title: string;
  total_screenings: number;
  candidates_screened: number;
  average_score: number;
  median_score: number;
  score_distribution: DistributionBucket[];
  component_averages: ScoringWeights;
  most_common_missing_skills: DistributionBucket[];
  status_distribution: DistributionBucket[];
}

export interface AdminAnalyticsOverviewResponse {
  platform_totals: {
    total_users: number;
    active_recruiters: number;
    deactivated_users: number;
    total_jobs: number;
    total_resumes: number;
    total_candidates: number;
    total_screenings: number;
    total_screening_results: number;
    average_score: number;
  };
  score_distribution: DistributionBucket[];
  skill_distribution: DistributionBucket[];
  recent_activity_counts: DistributionBucket[];
}

// ----------------------------------------------------------------------------
// Phase 12: Feedback (POSTPONED)
// ----------------------------------------------------------------------------

export interface CreateFeedbackRequest {
  rating: number;
  agrees_with_band: boolean;
  suggested_band?: ScoreBand | null;
  comment?: string | null;
  details?: Record<string, string>;
}

export interface FeedbackResponse {
  id: string;
  screening_result_id: string;
  recruiter_id: string;
  rating: number;
  agrees_with_band: boolean;
  suggested_band: ScoreBand | null;
  comment: string | null;
  details: Record<string, string>;
  created_at: string;
}
```

