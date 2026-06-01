# Implementation Plan: Explainable & Fair AI-Powered Loan Decision Support Platform

## Overview

This implementation plan breaks down the AI Loan Decision Platform into phased, incremental tasks organized by domain (Database, Backend/ML, Frontend). The approach follows an MVP-first strategy: database schema → backend core → ML pipeline → frontend auth → frontend features → advanced features. Each task builds on previous steps, ensuring no orphaned code. Python (FastAPI) is used for backend/ML, TypeScript (Next.js) for frontend, and SQL for database migrations.

## Tasks

- [x] 1. Database Schema and Supabase Setup
  - [x] 1.1 Create Supabase project configuration and database migration for core tables
    - Create SQL migration file with `profiles`, `loan_applications`, `predictions`, `shap_values`, `counterfactuals`, `audit_logs`, and `fairness_metrics` tables
    - Include all CHECK constraints, foreign keys, and indexes as defined in the design
    - Include the `emi_not_exceeding_income` constraint on `loan_applications`
    - _Requirements: 1.4, 4.1, 12.2, 22.2_

  - [x] 1.2 Create Row Level Security (RLS) policies for all tables
    - Enable RLS on all tables
    - Create policies for `profiles`: users read own, admins read all
    - Create policies for `loan_applications`: applicants read/insert own, officers/admins read all
    - Create policies for `predictions`, `shap_values`, `counterfactuals`: users read own via joins, officers read all
    - Create policies for `audit_logs`: admins read all, officers read application-specific, system insert only (no update/delete)
    - Create policies for `fairness_metrics`: admins read, system insert
    - _Requirements: 3.6, 22.2, 12.2_

  - [x] 1.3 Create Supabase Auth configuration and trigger for profile creation
    - Configure Supabase Auth email/password provider
    - Create a database trigger that automatically creates a `profiles` row with default 'Applicant' role when a new user registers via Supabase Auth
    - _Requirements: 1.1, 1.4_

- [ ] 2. Backend Project Setup and Core Infrastructure
  - [x] 2.1 Initialize FastAPI project with configuration and dependencies
    - Create `backend/` directory structure as defined in design (routers, services, models, ml, middleware, utils)
    - Create `requirements.txt` with FastAPI, uvicorn, pydantic, supabase-py, xgboost, scikit-learn, shap, dice-ml, fairlearn, optuna, fpdf2, numpy, pandas, joblib, slowapi, python-jose, httpx
    - Create `app/main.py` with FastAPI app, CORS configuration (allow Vercel frontend domain), and lifespan handler for model loading
    - Create `app/config.py` with environment variable configuration (Supabase URL, keys, frontend URL)
    - _Requirements: 18.1, 21.1_

  - [x] 2.2 Implement JWT authentication middleware and dependency injection
    - Create `app/middleware/auth.py` with JWT verification using Supabase Auth public key
    - Create `app/dependencies.py` with `get_current_user` dependency that extracts user ID and role from JWT
    - Implement role-checking dependencies: `require_applicant`, `require_officer`, `require_admin`
    - Return 401 for missing/invalid/expired tokens, 403 for insufficient role
    - _Requirements: 18.1, 18.2, 18.3, 3.1, 3.2, 3.3, 3.4_

  - [ ] 2.3 Implement rate limiting on authentication endpoints
    - Create `app/middleware/rate_limiter.py` using slowapi
    - Configure rate limit: max 5 failed auth attempts per minute per IP
    - Return 429 Too Many Requests when limit exceeded
    - Reset counter after 1-minute cooldown with no failed attempts
    - _Requirements: 18.5, 18.6_

  - [x] 2.4 Create Pydantic request/response schemas
    - Create `app/models/schemas.py` with all Pydantic models: `LoanApplicationInput`, `PredictionResult`, `SimulationResult`, `ShapValue`, `Counterfactual`, `FairnessMetrics`, `AuditFilters`, `PaginatedResponse`, error response models
    - Include all validators (e.g., `emi_not_exceeding_income`) and field constraints matching Requirement 4
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9, 4.10, 4.11, 18.4, 19.1, 19.3_

  - [x] 2.5 Create database connection and repository layer
    - Create `app/models/database.py` with Supabase client initialization
    - Implement repository functions for CRUD operations on all tables
    - Include pagination helpers (20 items per page for applications/users, 50 for audit logs)
    - _Requirements: 14.2, 15.2, 16.2, 12.3_

  - [ ]* 2.6 Write property tests for input validation (Property 1)
    - **Property 1: Input Validation Accepts Only Valid Ranges**
    - Use Hypothesis to generate valid and invalid inputs for all 10 fields
    - Verify acceptance for in-range values and rejection for out-of-range values
    - **Validates: Requirements 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9, 4.10, 4.11, 19.1, 19.4, 19.6, 19.7**

  - [ ]* 2.7 Write property tests for rate limiting (Property 5)
    - **Property 5: Rate Limiting Enforcement**
    - Verify that 6th failed attempt within 1 minute returns 429
    - Verify attempts accepted after cooldown
    - **Validates: Requirements 18.5, 18.6**

- [ ] 3. Checkpoint - Backend infrastructure verified
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. ML Pipeline - Model Training and Prediction
  - [x] 4.1 Create model training script with XGBoost and Random Forest
    - Create `app/ml/train.py` that loads Give Me Some Credit dataset
    - Implement preprocessing: handle missing values, feature engineering
    - Train XGBoost with Optuna hyperparameter tuning (primary model)
    - Train Random Forest (secondary model) with same train/test split (80/20)
    - Serialize models with joblib to `app/ml/artifacts/`
    - Compute and store performance metrics (AUC_ROC, F1 Score, KS_Statistic) for both models
    - _Requirements: 5.4, 5.5, 6.1, 6.3_

  - [x] 4.2 Implement PredictionService with XGBoost and Random Forest inference
    - Create `app/services/prediction_service.py`
    - Load serialized models at startup
    - Implement `predict()`: run XGBoost inference, compute Approval_Probability = (1 - Default_Probability) * 100
    - Implement Risk_Score = Default_Probability * 100
    - Map Risk_Score to Risk_Level categories (Very Low Risk 0-20, Low Risk 21-40, Moderate Risk 41-60, High Risk 61-80, Very High Risk 81-100)
    - Decision: Approved if Approval_Probability >= 50%, else Rejected
    - Also run Random Forest for comparison probability
    - Implement `simulate()` for What-If (same inference, no SHAP/DiCE/audit)
    - Handle 5-second timeout with error response
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 10.2_

  - [ ]* 4.3 Write property tests for decision threshold (Property 2)
    - **Property 2: Decision Threshold Correctness**
    - Use Hypothesis to generate approval probabilities and verify decision is "Approved" iff probability >= 50%
    - **Validates: Requirements 5.1**

  - [x] 4.4 Implement SHAP Explainer Service
    - Create `app/services/shap_service.py`
    - Use `shap.TreeExplainer` with the trained XGBoost model
    - Implement `compute_local_shap()`: returns exactly 10 ShapValue objects (one per feature) with feature name, value, SHAP contribution, and direction
    - Implement `compute_global_importance()`: compute across all stored predictions (batch, on-demand)
    - Rank features by absolute SHAP value magnitude, identify top 3
    - Handle SHAP computation failure gracefully (return prediction without explanations)
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.6_

  - [ ]* 4.5 Write property tests for SHAP output (Properties 6, 7)
    - **Property 6: SHAP Output Completeness** - verify exactly 10 SHAP values returned per prediction
    - **Property 7: SHAP Ranking Correctness** - verify top 3 are highest absolute magnitude
    - **Validates: Requirements 7.1, 7.4**

  - [x] 4.6 Implement DiCE Counterfactual Service
    - Create `app/services/counterfactual_service.py`
    - Use `dice_ml` library with trained XGBoost model
    - Generate 1-3 counterfactuals for rejected applications only
    - Configure immutable features: Age only
    - Configure mutable features with valid ranges from Requirement 4
    - Prioritize minimal changes via DiCE proximity weight
    - Re-run each counterfactual through XGBoost for projected approval probability, risk score, risk level
    - Implement 10-second timeout with fallback to top-3 SHAP features
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8_

  - [ ]* 4.7 Write property tests for counterfactuals (Properties 8, 9, 10)
    - **Property 8: Counterfactual Immutability Constraint** - Age never modified
    - **Property 9: Counterfactual Range Constraint** - all values within valid ranges
    - **Property 10: Counterfactual Estimated Impact Accuracy** - projected probability matches re-prediction within ±1%
    - **Validates: Requirements 8.3, 8.5, 8.8**

  - [x] 4.8 Implement Loan Readiness Score Service
    - Create `app/services/readiness_service.py`
    - Implement weighted formula: Credit Score (30%), DTI Ratio (25%), Credit Utilization (20%), Employment Length (15%), Existing Loans (10%)
    - Implement normalization: credit_score_normalized = (credit_score - 300) / 550 * 100, dti_ratio_score = (1 - dti_ratio/100) * 100, credit_utilization_score = (1 - credit_utilization/100) * 100, employment_length_score = min(employment_length/10, 1) * 100, existing_loans_score = max(0, (1 - existing_loans/10)) * 100
    - Map to categories: Poor (0-25), Fair (26-50), Good (51-75), Excellent (76-100)
    - When score < 50, identify top 3 improvement areas by largest weighted factor deficit
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5_

  - [ ]* 4.9 Write property tests for Loan Readiness Score (Properties 11, 12, 13)
    - **Property 11: Loan Readiness Score Formula Correctness** - verify weighted sum matches expected output
    - **Property 12: Loan Readiness Score Category Mapping** - verify category boundaries
    - **Property 13: Loan Readiness Improvement Areas** - verify top 3 deficits when score < 50
    - **Validates: Requirements 9.1, 9.2, 9.3, 9.4**

- [ ] 5. Checkpoint - ML pipeline verified
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Backend API Endpoints
  - [ ] 6.1 Implement authentication router (register, login, logout)
    - Create `app/routers/auth.py`
    - POST `/api/v1/auth/register`: validate input, create user via Supabase Auth, profile trigger creates profile row
    - POST `/api/v1/auth/login`: authenticate via Supabase Auth, return JWT tokens
    - POST `/api/v1/auth/logout`: invalidate session (requires JWT)
    - Handle duplicate email (error message), invalid password length (8-128 chars), invalid email format, name validation (1-100 chars)
    - Log auth events to audit_logs
    - _Requirements: 1.1, 1.2, 1.3, 1.5, 1.6, 2.1, 2.2, 2.3, 2.4, 2.5, 12.5_

  - [x] 6.2 Implement loan applications router
    - Create `app/routers/applications.py`
    - POST `/api/v1/applications`: validate input (Pydantic), run prediction pipeline (XGBoost + RF + SHAP + DiCE + LRS), store application + prediction + SHAP values + counterfactuals + audit log, return PredictionResult
    - GET `/api/v1/applications`: list own applications (Applicant), paginated 20/page, sorted by date desc
    - GET `/api/v1/applications/{id}`: get full application detail with prediction, SHAP, counterfactuals
    - GET `/api/v1/applications/review`: list all applications for Officer review, paginated 20/page
    - Enforce role-based access via dependencies
    - _Requirements: 4.2, 4.12, 5.1, 14.2, 14.3, 14.4, 15.2, 15.3_

  - [ ] 6.3 Implement predictions router (What-If Simulator endpoint)
    - Create `app/routers/predictions.py`
    - POST `/api/v1/predictions/simulate`: validate input, run XGBoost inference + LRS only (no SHAP, no DiCE), return SimulationResult
    - Do NOT store results in database, do NOT trigger audit logging
    - Target response time < 2 seconds
    - Handle timeout: retry once, then return error
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.7_

  - [ ]* 6.4 Write property test for Simulator No-Persistence (Property 17)
    - **Property 17: Simulator No-Persistence Guarantee**
    - Verify that calling simulate endpoint creates no records in loan_applications, predictions, shap_values, counterfactuals, or audit_logs
    - **Validates: Requirements 10.5**

  - [x] 6.5 Implement audit service and logging
    - Create `app/services/audit_service.py`
    - Implement `log_prediction()`: record inputs, outputs, SHAP values, counterfactuals, timestamp, user ID
    - Implement `log_auth_event()`: record login, logout, failed attempts with timestamp and IP
    - Implement `get_logs()`: paginated (50/page), filterable by date range, user, decision outcome, sorted by timestamp desc
    - Implement retry logic: 3 retries with exponential backoff (1s, 2s, 4s), queue for deferred writing on failure
    - _Requirements: 12.1, 12.2, 12.3, 12.4, 12.5, 12.6_

  - [ ] 6.6 Implement admin router (fairness, users, audit, model stats)
    - Create `app/routers/admin.py`
    - GET `/api/v1/admin/fairness`: compute and return fairness metrics (require Admin role)
    - GET `/api/v1/admin/users`: paginated user list (20/page), sorted by registration date desc (require Admin role)
    - PUT `/api/v1/admin/users/{id}/role`: update user role with validation (require Admin, prevent removing last Admin)
    - GET `/api/v1/admin/audit-logs`: paginated, filterable audit logs (require Admin role)
    - GET `/api/v1/admin/model-stats`: return pre-computed model performance metrics (require Admin or Officer role)
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 11.1, 11.2, 12.3, 16.2, 16.3, 16.4, 16.5, 16.6_

  - [ ] 6.7 Implement Fairness Monitoring Service
    - Create `app/services/fairness_service.py`
    - Compute Demographic Parity difference and Equalized Odds difference using Fairlearn
    - Detect proxy bias: Pearson correlation between protected attributes and input features, flag |r| > 0.7
    - Return insufficient data notice if fewer than 30 predictions exist
    - Threshold: 0.1 for both metrics
    - Computation: on-demand when Admin requests
    - _Requirements: 11.1, 11.2, 11.3, 11.4, 11.5_

  - [ ]* 6.8 Write property tests for fairness (Properties 14, 15, 16)
    - **Property 14: Fairness Threshold Indicator** - green if ≤ 0.1, red if > 0.1
    - **Property 15: Proxy Bias Detection Threshold** - flag iff |r| > 0.7
    - **Property 16: Insufficient Data Notice** - display notice iff < 30 predictions
    - **Validates: Requirements 11.2, 11.3, 11.4, 11.5**

  - [ ] 6.9 Implement PDF Report Generation Service
    - Create `app/services/pdf_service.py` using fpdf2
    - Create `app/routers/reports.py` with GET `/api/v1/reports/{assessment_id}/pdf`
    - PDF content: platform logo, report title, sections (Applicant Information, Risk Assessment, Explanation with top 5 SHAP contributions, Recommendations)
    - If Rejected: include counterfactual improvement suggestions
    - If Approved: include top 3 positive SHAP factors as key strengths
    - Generate within 10 seconds, return as downloadable file
    - Handle generation errors with error message and logging
    - _Requirements: 13.1, 13.2, 13.3, 13.4, 13.5_

  - [ ]* 6.10 Write property test for PDF content completeness (Property 20)
    - **Property 20: PDF Report Content Completeness**
    - Verify PDF contains: applicant name, all input values, Risk Score, Approval Probability, Decision, top 5 SHAP contributions, and Recommendations section
    - **Validates: Requirements 13.1, 13.5**

  - [ ] 6.11 Implement input sanitization utilities
    - Create `app/utils/validators.py` with HTML entity encoding for text inputs (&, <, >, ", ')
    - Enforce max 500 characters on text fields
    - Reject whitespace-only strings in required fields
    - Apply sanitization in all routers before processing
    - _Requirements: 19.2, 19.6, 19.7_

  - [ ]* 6.12 Write property test for HTML sanitization (Property 18)
    - **Property 18: HTML Sanitization**
    - Use Hypothesis to generate strings with HTML special characters
    - Verify all special characters are encoded as HTML entities
    - **Validates: Requirements 19.2**

- [ ] 7. Checkpoint - Backend API complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 8. Frontend Project Setup and Authentication
  - [x] 8.1 Initialize Next.js project with TypeScript, Tailwind CSS, and shadcn/ui
    - Create `frontend/` directory with Next.js App Router structure as defined in design
    - Configure Tailwind CSS with mobile-first breakpoints (320px+, 768px+, 1024px+)
    - Install and configure shadcn/ui component library
    - Set up consistent typography scale, spacing system, and color palette
    - Create `src/types/index.ts` with all TypeScript interfaces matching backend schemas
    - _Requirements: 17.1, 17.4_

  - [x] 8.2 Implement Supabase Auth client and session management
    - Create `src/lib/supabase/client.ts` and `src/lib/supabase/server.ts`
    - Implement `AuthService` interface: register, login, logout, getSession, onAuthStateChange
    - Implement automatic token refresh at least 60 seconds before expiration
    - Handle token refresh failure: redirect to login with session ended message
    - Clear all client-side auth state on logout
    - _Requirements: 2.1, 2.3, 2.4, 2.5_

  - [x] 8.3 Implement registration and login pages
    - Create `src/app/(auth)/register/page.tsx`: form with name, email, password fields
    - Create `src/app/(auth)/login/page.tsx`: form with email, password fields
    - Implement client-side validation: email format, password length (8-128), name (1-100 chars)
    - Display inline validation errors within 200ms
    - Handle duplicate email error, generic auth failure message (don't reveal which credential is wrong)
    - Redirect to role-appropriate dashboard on success
    - _Requirements: 1.1, 1.2, 1.3, 1.5, 1.6, 2.1, 2.2, 19.5_

  - [x] 8.4 Implement RoleGuard component and protected route layout
    - Create `src/components/layout/RoleGuard.tsx`: check user role against allowed roles for current route
    - Create `src/app/(protected)/layout.tsx`: wrap all protected pages with auth check
    - Redirect unauthenticated users to login page
    - Redirect unauthorized users to their role-appropriate dashboard with 5-second access denied notification
    - _Requirements: 3.2, 3.3, 3.4, 3.5, 3.7_

  - [ ]* 8.5 Write property test for role-based access (Property 3)
    - **Property 3: Role-Based Access Enforcement**
    - Use fast-check to generate role/route combinations
    - Verify access granted iff route is in role's permitted set
    - **Validates: Requirements 3.2, 3.3, 3.4, 18.3**

  - [ ] 8.6 Implement responsive navigation (Navbar and MobileNav)
    - Create `src/components/layout/Navbar.tsx`: desktop navigation with role-appropriate links
    - Create `src/components/layout/MobileNav.tsx`: collapsible hamburger menu for viewports < 768px
    - Ensure 44x44px minimum tap targets for all interactive elements
    - No horizontal scrolling at any viewport width (320px+)
    - _Requirements: 17.2, 17.3, 17.5, 17.6_

  - [x] 8.7 Create API client service
    - Create `src/lib/api-client.ts` implementing the `ApiClient` interface
    - Include JWT token in Authorization header for all requests
    - Handle error responses (401 → redirect to login, 403 → access denied, 422 → field errors, 429 → rate limit message, 500 → generic error)
    - Implement retry logic for transient failures
    - _Requirements: 18.1, 18.2, 18.3_

- [ ] 9. Checkpoint - Frontend auth and navigation complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 10. Frontend - Applicant Features
  - [x] 10.1 Implement Loan Application Form
    - Create `src/components/forms/LoanApplicationForm.tsx`
    - Include all 10 input fields: Age, Monthly Income, Employment Status (dropdown), Employment Length, Credit Score, Existing Loans, Monthly EMI, DTI Ratio, Credit Utilization, Loan Amount Requested
    - Implement client-side validation matching all Requirement 4 rules
    - Display inline validation errors within 200ms adjacent to invalid fields
    - On valid submission, call API client and navigate to assessment result
    - Responsive layout: mobile (320px+), tablet (768px+), desktop (1024px+)
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9, 4.10, 4.11, 4.12, 17.1, 19.5_

  - [ ] 10.2 Implement Explanation Translator service
    - Create `src/lib/explanation-translator.ts` implementing `ExplanationTranslator` interface
    - Map all 10 technical feature names to plain-language equivalents (e.g., dti_ratio → "Portion of income going toward debt")
    - Translate SHAP values to plain-language statements ("This factor helped/hurt your application")
    - Translate counterfactuals to actionable recommendations with projected outcomes
    - Translate readiness score to interpretation text
    - Ensure no technical terms appear in applicant-facing output
    - _Requirements: 20.1, 20.2, 20.3, 20.4, 20.5, 20.6_

  - [ ]* 10.3 Write property test for feature name translation (Property 19)
    - **Property 19: Feature Name Translation Completeness**
    - Use fast-check to generate all technical feature names
    - Verify plain-language output contains no technical terms
    - **Validates: Requirements 20.1, 20.2**

  - [ ] 10.4 Implement SHAP visualization components with Plotly
    - Create `src/components/charts/ShapWaterfallChart.tsx`: waterfall chart showing SHAP value, direction, and feature name for each feature
    - Create `src/components/charts/ShapGlobalImportance.tsx`: bar chart for global feature importance (Officer view)
    - Use Plotly with interactive tooltips (feature name, value, SHAP contribution)
    - Highlight top 3 most influential factors
    - Responsive layout for mobile (320px+), tablet (768px+), desktop (1024px+)
    - Minimum 12px font for labels, 44x44px tap targets for interactive elements
    - _Requirements: 7.2, 7.3, 7.4, 7.5, 17.2, 17.3_

  - [ ] 10.5 Implement Readiness Gauge and Counterfactual components
    - Create `src/components/charts/ReadinessGauge.tsx`: color-coded gauge (red 0-25, orange 26-50, yellow 51-75, green 76-100) with textual interpretation
    - Create `src/components/explanations/CounterfactualCard.tsx`: display feature name, current value, recommended value, estimated impact in plain language
    - Create `src/components/explanations/ImprovementAreas.tsx`: top 3 improvement areas when readiness < 50
    - Create `src/components/explanations/PlainLanguageExplanation.tsx`: answers the 4 questions for rejected applications
    - _Requirements: 8.2, 8.5, 9.3, 9.4, 20.3, 20.5_

  - [ ] 10.6 Implement Assessment Detail page
    - Create `src/app/(protected)/applicant/assessment/[id]/page.tsx`
    - Display full assessment: input values, Risk Score with Risk Level label, Approval Probability, Decision, Loan Readiness Score with gauge
    - Display SHAP waterfall chart with plain-language explanations
    - If Rejected: display counterfactual recommendations with projected outcomes (Approval Probability, Risk Score, Risk Level, Loan Readiness Score)
    - If Approved: display positive factors
    - Include PDF download button
    - _Requirements: 5.3, 7.2, 8.2, 8.5, 9.3, 13.1, 14.3, 14.4, 20.4, 20.5_

  - [ ] 10.7 Implement Applicant Dashboard
    - Create `src/app/(protected)/applicant/dashboard/page.tsx`
    - Sections: New Assessment link, What-If Simulator link, Assessment History, Risk Reports, SHAP Explanation View, Recommendations
    - Mobile-first responsive layout
    - Handle loading states (skeleton loaders) and error states (retry option)
    - _Requirements: 14.1, 14.6, 14.7_

  - [ ] 10.8 Implement Assessment History page
    - Create `src/app/(protected)/applicant/history/page.tsx`
    - Paginated list (20 items/page) sorted by date descending
    - Show date, decision, Risk Score, Approval Probability per entry
    - Empty state with link to New Assessment when no history exists
    - Click to navigate to Assessment Detail page
    - _Requirements: 14.2, 14.3, 14.4, 14.5_

  - [ ] 10.9 Implement What-If Simulator page
    - Create `src/app/(protected)/applicant/simulator/page.tsx`
    - Create `src/components/forms/WhatIfForm.tsx` and `src/hooks/useSimulator.ts`
    - Pre-populate with most recent assessment inputs (or empty if none)
    - On any mutable field change, call simulate endpoint and display updated results within 2 seconds
    - Side-by-side comparison: original vs simulated values for Approval Probability, Risk Score, Risk Level, Loan Readiness Score
    - Visual indicators (arrows/color) showing improvement or decline
    - Validate inputs against Requirement 4 rules before simulation
    - Handle timeout: loading indicator → retry once → error message
    - Responsive layout for mobile, tablet, desktop
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7_

  - [ ]* 10.10 Write unit tests for frontend applicant components
    - Test LoanApplicationForm renders all fields with correct labels
    - Test validation error display for invalid inputs
    - Test ExplanationTranslator outputs plain language
    - Test ReadinessGauge color mapping
    - Test CounterfactualCard displays all required fields
    - _Requirements: 4.1, 9.3, 20.1_

- [ ] 11. Checkpoint - Applicant features complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 12. Frontend - Bank Officer Features
  - [ ] 12.1 Implement Bank Officer Dashboard
    - Create `src/app/(protected)/officer/dashboard/page.tsx`
    - Sections: Application Review, Risk Analytics, Explainability Dashboard, Audit View
    - Mobile-first responsive layout
    - Handle loading and error states
    - _Requirements: 15.1, 15.4, 15.6_

  - [ ] 12.2 Implement Application Review list page
    - Create `src/app/(protected)/officer/applications/page.tsx`
    - Paginated list (20/page) sorted by submission date descending
    - Show applicant name, submission date, Risk Score, decision status (Pending Review, Approved, Rejected)
    - Empty state when no applications exist
    - _Requirements: 15.2, 15.5_

  - [ ] 12.3 Implement Application Detail page (Officer view)
    - Create `src/app/(protected)/officer/applications/[id]/page.tsx`
    - Display complete application details (all form fields), prediction results (Approval Probability, Risk Score, Default Probability, Decision)
    - Display local SHAP waterfall chart AND global feature importance bar chart
    - Display counterfactual recommendations alongside SHAP explanations (for rejected)
    - Display audit trail for the specific application (prediction events, review actions)
    - _Requirements: 7.3, 8.6, 12.4, 15.3_

  - [ ] 12.4 Implement Risk Analytics and Model Statistics page
    - Create `src/app/(protected)/officer/analytics/page.tsx`
    - Side-by-side comparison table: XGBoost vs Random Forest
    - Display AUC_ROC, F1 Score, KS_Statistic for both models (4 decimal places)
    - Highlight which model has higher value for each metric
    - Handle metrics unavailable error state
    - Render within 3 seconds
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

- [ ] 13. Frontend - Admin Features
  - [ ] 13.1 Implement Admin Dashboard
    - Create `src/app/(protected)/admin/dashboard/page.tsx`
    - Sections: Fairness Monitoring, User Management, Audit Logs, Model Statistics
    - Display warning indicator if any fairness metric exceeds threshold
    - Display warning if audit logging failure detected
    - Mobile-first responsive layout
    - _Requirements: 16.1, 16.7, 11.4, 12.6_

  - [ ] 13.2 Implement Fairness Monitoring page
    - Create `src/app/(protected)/admin/fairness/page.tsx`
    - Create `src/components/charts/FairnessIndicator.tsx`
    - Display Demographic Parity difference and Equalized Odds difference
    - Green indicator if ≤ 0.1, red if > 0.1
    - Display proxy bias correlations with flag for |r| > 0.7
    - Show insufficient data notice if < 30 predictions
    - _Requirements: 11.1, 11.2, 11.3, 11.4, 11.5_

  - [ ] 13.3 Implement User Management page
    - Create `src/app/(protected)/admin/users/page.tsx`
    - Paginated list (20/page) sorted by registration date descending
    - Display name, email, role, registration date
    - Role change: confirmation prompt showing user name, current role, new role
    - Prevent removing last Admin (display error)
    - Handle role update errors gracefully
    - _Requirements: 16.2, 16.3, 16.4, 16.5, 16.6_

  - [ ] 13.4 Implement Audit Logs page
    - Create `src/app/(protected)/admin/audit/page.tsx`
    - Paginated list (50/page) sorted by timestamp descending
    - Filters: date range, user, decision outcome
    - Display event type, user, timestamp, event data summary
    - _Requirements: 12.3_

  - [ ] 13.5 Implement Model Statistics page (Admin view)
    - Create `src/app/(protected)/admin/models/page.tsx`
    - Reuse model comparison component from Officer analytics
    - Display AUC_ROC, F1 Score, KS_Statistic for both models
    - _Requirements: 6.1, 6.2_

- [ ] 14. Checkpoint - All dashboards complete
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 15. Integration, Error Handling, and Polish
  - [ ] 15.1 Implement comprehensive error handling across frontend
    - Toast notifications for transient API errors
    - Inline error messages for persistent errors
    - Full-page error state with retry button for network failures
    - Skeleton loaders for initial page loads, spinners for actions
    - Loading indicators for all async operations
    - Handle 401 (redirect to login), 403 (access denied), 422 (field errors), 429 (rate limit), 500 (generic)
    - _Requirements: 5.7, 7.6, 8.7, 9.5, 10.7, 13.4, 14.6, 15.6, 16.5, 21.4_

  - [ ] 15.2 Implement PDF download flow in frontend
    - Add download button on Assessment Detail page
    - Call API client to fetch PDF blob
    - Initiate browser download within 10 seconds
    - Handle generation errors with user-friendly message
    - _Requirements: 13.1, 13.3, 13.4_

  - [ ] 15.3 Wire all frontend components to backend API
    - Connect LoanApplicationForm submission to POST /api/v1/applications
    - Connect What-If Simulator to POST /api/v1/predictions/simulate
    - Connect Assessment History to GET /api/v1/applications
    - Connect Officer Application Review to GET /api/v1/applications/review
    - Connect Admin pages to respective admin endpoints
    - Connect PDF download to GET /api/v1/reports/{id}/pdf
    - Verify end-to-end data flow for all user journeys
    - _Requirements: 4.12, 10.2, 14.2, 15.2, 16.2, 12.3, 13.3_

  - [ ]* 15.4 Write integration tests for full prediction flow
    - Test: submit application → receive prediction with SHAP + counterfactuals + LRS
    - Test: auth flow (register → login → access protected route → logout)
    - Test: RLS enforcement (applicant cannot see other applicant's data)
    - Test: audit trail created on prediction
    - Test: PDF generation and download
    - _Requirements: 4.12, 5.1, 12.1, 13.3, 22.2_

  - [ ]* 15.5 Write frontend unit tests for dashboard components
    - Test Applicant Dashboard renders all sections
    - Test Officer Dashboard renders all sections
    - Test Admin Dashboard renders all sections with warning indicators
    - Test empty states display correctly
    - Test pagination controls work
    - _Requirements: 14.1, 15.1, 16.1_

- [ ] 16. Final Checkpoint - All features integrated and tested
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation at each phase boundary
- Property tests validate universal correctness properties from the design document
- Unit tests validate specific examples and edge cases
- The implementation order follows MVP-first: Database → Backend Infrastructure → ML Pipeline → Frontend Auth → Applicant Features → Officer Features → Admin Features → Integration

### Development Phases

| Phase | Tasks | Focus Area | Estimated Effort |
|-------|-------|-----------|-----------------|
| Phase 1: Foundation | 1.1-1.3 | Database schema, RLS, Auth triggers | 2-3 days |
| Phase 2: Backend Core | 2.1-2.7 | FastAPI setup, auth middleware, rate limiting, schemas | 3-4 days |
| Phase 3: ML Pipeline | 4.1-4.9 | Model training, prediction, SHAP, DiCE, LRS | 5-7 days |
| Phase 4: Backend API | 6.1-6.12 | All API endpoints, services, PDF generation | 5-6 days |
| Phase 5: Frontend Auth | 8.1-8.7 | Next.js setup, auth pages, navigation, API client | 3-4 days |
| Phase 6: Applicant UI | 10.1-10.10 | Application form, explanations, simulator, dashboard | 5-7 days |
| Phase 7: Officer UI | 12.1-12.4 | Officer dashboard, application review, analytics | 3-4 days |
| Phase 8: Admin UI | 13.1-13.5 | Admin dashboard, fairness, users, audit, models | 3-4 days |
| Phase 9: Integration | 15.1-15.5 | Error handling, wiring, integration tests | 3-4 days |

### Team Assignment Recommendations

| Domain | Recommended Skills | Tasks |
|--------|-------------------|-------|
| Database/DevOps | SQL, Supabase, RLS policies | 1.1-1.3 |
| Backend (Python) | FastAPI, Pydantic, JWT, API design | 2.1-2.7, 6.1-6.12 |
| ML Engineer | XGBoost, SHAP, DiCE, Fairlearn, Optuna | 4.1-4.9, 6.7 |
| Frontend (TypeScript) | Next.js, React, Tailwind, Plotly, shadcn/ui | 8.1-8.7, 10.1-10.10, 12.1-12.4, 13.1-13.5 |
| QA/Testing | Hypothesis, fast-check, integration testing | All `*` tasks, 15.4-15.5 |

### Milestones

1. **M1 - Backend MVP** (end of Phase 2): Auth + validation working, API skeleton deployed
2. **M2 - ML Pipeline Complete** (end of Phase 3): Models trained, predictions generating correctly
3. **M3 - Full Backend** (end of Phase 4): All endpoints functional, PDF generation working
4. **M4 - Frontend Auth** (end of Phase 5): Users can register, login, navigate role-based routes
5. **M5 - Applicant Journey** (end of Phase 6): Full applicant flow from application to explanation
6. **M6 - Officer & Admin** (end of Phase 8): All dashboards functional
7. **M7 - Production Ready** (end of Phase 9): Fully integrated, tested, error handling complete

### MVP-First Implementation Order

1. Database schema + RLS (foundation for everything)
2. Backend auth + validation (enables frontend development)
3. XGBoost prediction + SHAP (core value proposition)
4. Loan application submission + results display (minimum viable user flow)
5. What-If Simulator (key differentiator)
6. DiCE counterfactuals (actionable recommendations)
7. Officer review dashboard (business stakeholder value)
8. Admin fairness monitoring (compliance requirement)
9. PDF reports, audit logs, model comparison (supporting features)

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2", "1.3"] },
    { "id": 2, "tasks": ["2.1", "8.1"] },
    { "id": 3, "tasks": ["2.2", "2.3", "2.4", "2.5", "8.2"] },
    { "id": 4, "tasks": ["2.6", "2.7", "4.1", "8.3", "8.6"] },
    { "id": 5, "tasks": ["4.2", "4.4", "4.8", "8.4", "8.7"] },
    { "id": 6, "tasks": ["4.3", "4.5", "4.6", "4.9", "8.5"] },
    { "id": 7, "tasks": ["4.7", "6.1", "6.5", "6.11"] },
    { "id": 8, "tasks": ["6.2", "6.3", "6.6", "6.7", "6.9", "6.12"] },
    { "id": 9, "tasks": ["6.4", "6.8", "6.10", "10.1", "10.2"] },
    { "id": 10, "tasks": ["10.3", "10.4", "10.5", "10.6"] },
    { "id": 11, "tasks": ["10.7", "10.8", "10.9", "10.10"] },
    { "id": 12, "tasks": ["12.1", "12.2", "12.3", "12.4"] },
    { "id": 13, "tasks": ["13.1", "13.2", "13.3", "13.4", "13.5"] },
    { "id": 14, "tasks": ["15.1", "15.2", "15.3"] },
    { "id": 15, "tasks": ["15.4", "15.5"] }
  ]
}
```
