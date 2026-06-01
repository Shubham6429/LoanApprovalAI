# Requirements Document

## Introduction

This document defines the requirements for an Explainable & Fair AI-Powered Loan Decision Support Platform. The platform is a production-style, mobile-first responsive web application that helps loan applicants understand their approval chances and helps financial institutions make transparent, explainable, fair, and auditable lending decisions. The system uses machine learning models (XGBoost, Random Forest) with SHAP-based explainability, DiCE counterfactual explanations, and Fairlearn-based fairness monitoring. The platform supports three user roles: Loan Applicant, Bank Officer, and Admin.

## Glossary

- **Platform**: The Explainable & Fair AI-Powered Loan Decision Support Platform web application
- **Applicant**: A user with the Loan Applicant role who submits loan applications and views assessments
- **Bank_Officer**: A user with the Bank Officer role who reviews applications and makes lending decisions
- **Admin**: A user with the Admin role who monitors fairness, manages users, and views audit logs
- **Prediction_Engine**: The backend ML service that generates approval probability, risk score, and decision
- **SHAP_Explainer**: The module that generates global and local SHAP explanations for model predictions
- **DiCE_Engine**: The module that generates counterfactual explanations showing minimum changes to flip a decision
- **Fairness_Monitor**: The module that computes and tracks fairness metrics using Fairlearn
- **Audit_Logger**: The service that records all prediction inputs, outputs, explanations, and user actions
- **PDF_Generator**: The service that produces downloadable PDF assessment reports
- **Risk_Score**: A numeric score from 0 to 100 representing the credit risk of an applicant (0 = Very Low Risk, 100 = Very High Risk)
- **Risk_Level**: A categorical label derived from Risk_Score: Very Low Risk (0-20), Low Risk (21-40), Moderate Risk (41-60), High Risk (61-80), Very High Risk (81-100)
- **Approval_Probability**: A percentage (0-100) representing the likelihood of loan approval
- **Loan_Readiness_Score**: A user-friendly score (0-100) indicating how ready an applicant is for loan approval
- **Default_Probability**: A percentage representing the likelihood that an applicant will default on a loan
- **RLS**: Row Level Security, a Supabase PostgreSQL feature that restricts data access at the row level
- **RBAC**: Role-Based Access Control, a security model that restricts system access based on user roles
- **EMI**: Equated Monthly Installment, the fixed monthly payment amount for a loan
- **DTI_Ratio**: Debt-to-Income Ratio, the percentage of monthly income used for debt payments
- **AUC_ROC**: Area Under the Receiver Operating Characteristic Curve, a model performance metric
- **KS_Statistic**: Kolmogorov-Smirnov Statistic, a metric measuring model discrimination ability
- **What_If_Simulator**: An interactive tool that allows applicants to modify financial inputs and instantly view updated predictions without submitting a formal application

## Requirements

### Requirement 1: User Registration

**User Story:** As a new user, I want to create an account with my credentials, so that I can access the platform based on my assigned role.

#### Acceptance Criteria

1. WHEN a user submits a registration form with name, email, and password, THE Platform SHALL create a new user account via Supabase Authentication and assign the default Applicant role.
2. WHEN a registration request contains an email already associated with an existing account, THE Platform SHALL display an error message indicating the email is already registered.
3. WHEN a registration request contains a password shorter than 8 characters or longer than 128 characters, THE Platform SHALL reject the request and display a validation error specifying the allowed password length range.
4. THE Platform SHALL store user profile information (name, email, role) in the Supabase PostgreSQL database with RLS policies applied.
5. WHEN a registration request contains an email that does not conform to a valid email format, THE Platform SHALL reject the request and display a validation error indicating the email format is invalid.
6. WHEN a registration request contains a name field that is empty or exceeds 100 characters, THE Platform SHALL reject the request and display a validation error specifying the name length requirements.

### Requirement 2: User Authentication

**User Story:** As a registered user, I want to log in and log out securely, so that I can access my role-specific features.

#### Acceptance Criteria

1. WHEN a user submits valid email and password credentials, THE Platform SHALL authenticate the user via Supabase Authentication and redirect to the role-appropriate dashboard.
2. WHEN a user submits invalid credentials, THE Platform SHALL display an error message indicating authentication failure without revealing which credential is incorrect.
3. WHEN an authenticated user clicks the logout button, THE Platform SHALL invalidate the Supabase Auth session token, clear all client-side stored authentication state, and redirect to the login page.
4. THE Platform SHALL maintain session state using Supabase Auth tokens and SHALL initiate an automatic token refresh at least 60 seconds before token expiration.
5. IF the Supabase Auth token refresh fails or the session token has expired, THEN THE Platform SHALL redirect the user to the login page and display a message indicating the session has ended.

### Requirement 3: Role-Based Access Control

**User Story:** As a platform administrator, I want to enforce role-based access, so that users can only access features appropriate to their role.

#### Acceptance Criteria

1. THE Platform SHALL enforce three distinct roles: Applicant, Bank_Officer, and Admin.
2. WHILE a user is authenticated with the Applicant role, THE Platform SHALL grant access to only applicant-specific pages (dashboard, application form, assessment history, reports) and deny access to all other pages.
3. WHILE a user is authenticated with the Bank_Officer role, THE Platform SHALL grant access to application review, risk analytics, explainability dashboard, and audit view pages.
4. WHILE a user is authenticated with the Admin role, THE Platform SHALL grant access to fairness monitoring, user management, audit logs, and model statistics pages.
5. WHEN an authenticated user attempts to access a page outside their role permissions via UI navigation or direct URL access, THE Platform SHALL redirect the user to their role-appropriate dashboard and display an access denied notification for 5 seconds.
6. THE Platform SHALL enforce RLS policies on all database tables to prevent unauthorized data access at the row level.
7. WHEN an unauthenticated user attempts to access any protected page, THE Platform SHALL redirect the user to the login page.

### Requirement 4: Loan Application Submission

**User Story:** As a loan applicant, I want to submit my financial information, so that I can receive a credit risk assessment.

#### Acceptance Criteria

1. THE Platform SHALL present a loan application form collecting: Age, Monthly Income, Employment Status (one of: Employed, Self-Employed, Unemployed, Retired), Employment Length, Credit Score, Existing Loans, Monthly EMI, DTI_Ratio, Credit Utilization, and Loan Amount Requested.
2. WHEN an Applicant submits a loan application form, THE Platform SHALL validate all input fields on both the frontend and backend before sending data to the Prediction_Engine.
3. WHEN the Age field contains a value less than 18 or greater than 100, THE Platform SHALL reject the submission and display a validation error for the Age field.
4. WHEN the Monthly Income field contains a value less than or equal to zero or greater than 10,000,000, THE Platform SHALL reject the submission and display a validation error for the Monthly Income field.
5. WHEN the Credit Score field contains a value outside the range 300 to 850, THE Platform SHALL reject the submission and display a validation error for the Credit Score field.
6. WHEN the DTI_Ratio field contains a value less than 0 or greater than 100, THE Platform SHALL reject the submission and display a validation error for the DTI_Ratio field.
7. WHEN the Credit Utilization field contains a value less than 0 or greater than 100, THE Platform SHALL reject the submission and display a validation error for the Credit Utilization field.
8. WHEN the Loan Amount Requested field contains a value less than or equal to zero or greater than 10,000,000, THE Platform SHALL reject the submission and display a validation error for the Loan Amount field.
9. WHEN the Employment Length field contains a value less than 0 or greater than 50 years, THE Platform SHALL reject the submission and display a validation error for the Employment Length field.
10. WHEN the Existing Loans field contains a non-integer value or a value less than 0 or greater than 50, THE Platform SHALL reject the submission and display a validation error for the Existing Loans field.
11. WHEN the Monthly EMI field contains a value less than 0 or greater than the submitted Monthly Income value, THE Platform SHALL reject the submission and display a validation error for the Monthly EMI field.
12. WHEN all input validations pass, THE Platform SHALL send the application data to the Prediction_Engine and store the application record in the database.

### Requirement 5: Credit Risk Prediction

**User Story:** As a loan applicant, I want to receive a credit risk prediction, so that I can understand my approval chances.

#### Acceptance Criteria

1. WHEN the Prediction_Engine receives a validated loan application, THE Prediction_Engine SHALL generate an Approval_Probability (0-100%), a Risk_Score (0-100), a Default_Probability (0-100%), and a decision of Approved (if Approval_Probability is 50% or above) or Rejected (if Approval_Probability is below 50%).
2. THE Prediction_Engine SHALL classify the Risk_Score into a Risk_Level category: Very Low Risk (0-20), Low Risk (21-40), Moderate Risk (41-60), High Risk (61-80), or Very High Risk (81-100).
3. WHEN displaying prediction results to any user, THE Platform SHALL show both the numeric Risk_Score and the corresponding Risk_Level category label in plain language without technical jargon.
4. THE Prediction_Engine SHALL use XGBoost as the primary prediction model trained on the Give Me Some Credit dataset, and the XGBoost model output SHALL determine the final Approval_Probability, Risk_Score, Default_Probability, and decision.
5. THE Prediction_Engine SHALL use Random Forest as a secondary model and SHALL generate a separate Approval_Probability from Random Forest alongside the primary XGBoost prediction for comparison purposes.
6. WHEN a prediction is generated, THE Prediction_Engine SHALL return the result to the Platform within 5 seconds.
7. IF the Prediction_Engine encounters an error during prediction, THEN THE Platform SHALL display an error message indicating that the prediction could not be completed, retain the submitted application data, and log the error details for debugging.

### Requirement 6: Model Performance Comparison

**User Story:** As a bank officer, I want to compare model performance metrics, so that I can trust the prediction quality.

#### Acceptance Criteria

1. THE Platform SHALL display model performance metrics including AUC_ROC, F1 Score, and KS_Statistic for both XGBoost and Random Forest models, with each metric value displayed to 4 decimal places.
2. WHEN a Bank_Officer navigates to the model statistics page, THE Platform SHALL display a side-by-side comparison table of XGBoost and Random Forest performance metrics, showing each metric name, the value for each model, and which model has the higher value for each metric.
3. THE Platform SHALL compute model performance metrics based on the Give Me Some Credit test dataset using a fixed 80/20 train-test split, and display the metrics as pre-computed values that are refreshed each time the models are retrained.
4. IF the Platform is unable to retrieve or compute model performance metrics, THEN THE Platform SHALL display an error message indicating that metrics are temporarily unavailable and log the failure details.
5. WHEN a Bank_Officer navigates to the model statistics page, THE Platform SHALL render the complete metrics comparison within 3 seconds.

### Requirement 7: SHAP Explanations

**User Story:** As a loan applicant, I want to understand which factors influenced my credit decision, so that I can take informed actions to improve my profile.

#### Acceptance Criteria

1. WHEN a prediction is generated for a loan application, THE SHAP_Explainer SHALL compute local SHAP values for all 10 input features of that application and return the values as part of the prediction response.
2. WHEN an Applicant views their assessment result, THE Platform SHALL display a SHAP waterfall chart showing the SHAP value, direction (positive or negative contribution toward approval), and feature name for each of the 10 input features.
3. WHEN a Bank_Officer views an application, THE Platform SHALL display both the local SHAP waterfall chart for the specific application and a global feature importance bar chart computed across all stored historical predictions.
4. THE SHAP_Explainer SHALL rank features by absolute SHAP value magnitude and THE Platform SHALL highlight the top 3 most influential factors for each prediction.
5. THE Platform SHALL render SHAP visualizations using Plotly with interactive tooltips displaying the feature name, feature value, and SHAP contribution value, and with responsive layout for mobile (320px+), tablet (768px+), and desktop (1024px+) viewports.
6. IF the SHAP_Explainer fails to compute SHAP values for a prediction, THEN THE Platform SHALL display the prediction result without SHAP explanations and show a message indicating that the detailed explanation is temporarily unavailable.

### Requirement 8: Counterfactual Explanations

**User Story:** As a loan applicant, I want to know what minimum changes would flip my decision, so that I can take actionable steps toward approval.

#### Acceptance Criteria

1. WHEN a loan application receives a Rejected decision, THE DiCE_Engine SHALL generate between 1 and 3 counterfactual explanations showing the minimum feature changes required to achieve an Approved decision.
2. WHEN counterfactual explanations are generated, THE Platform SHALL display each recommendation showing: the feature name in plain language, the Applicant's current value, the recommended target value, and the estimated impact on approval probability.
3. THE DiCE_Engine SHALL generate counterfactual explanations that modify only mutable features (Monthly Income, Employment Status, Employment Length, Credit Score, Existing Loans, Monthly EMI, DTI_Ratio, Credit Utilization, and Loan Amount Requested), excluding only Age as an immutable feature.
4. THE DiCE_Engine SHALL prioritize recommendations that are realistic and actionable, favoring smaller incremental changes over large unrealistic jumps in feature values.
5. WHEN counterfactual explanations are displayed, THE Platform SHALL show a projected outcome summary after applying all recommended changes, including the estimated Approval_Probability, Risk_Score, Risk_Level, and Loan_Readiness_Score, presented in a user-friendly format without technical jargon.
6. WHEN a Bank_Officer views a rejected application, THE Platform SHALL display the counterfactual explanations alongside the SHAP explanations.
7. IF the DiCE_Engine cannot generate a valid counterfactual within 10 seconds, THEN THE Platform SHALL display a message indicating that specific recommendations are unavailable and display the top 3 features by SHAP value magnitude as general improvement areas.
8. THE DiCE_Engine SHALL constrain all counterfactual feature values to the valid input ranges defined in Requirement 4.

### Requirement 9: Loan Readiness Score

**User Story:** As a loan applicant, I want a simple readiness score, so that I can quickly understand my overall loan readiness without interpreting complex metrics.

#### Acceptance Criteria

1. WHEN a prediction is generated, THE Platform SHALL compute a Loan_Readiness_Score (0-100) that is distinct from the Approval_Probability.
2. THE Platform SHALL calculate the Loan_Readiness_Score based on weighted factors: Credit Score (30%), DTI_Ratio (25%), Credit Utilization (20%), Employment Length (15%), and Existing Loans (10%).
3. WHEN an Applicant views their assessment, THE Platform SHALL display the Loan_Readiness_Score with a color-coded gauge (red for 0-25 Poor, orange for 26-50 Fair, yellow for 51-75 Good, green for 76-100 Excellent) and the corresponding textual interpretation.
4. WHEN the Loan_Readiness_Score is below 50, THE Platform SHALL highlight the top 3 improvement areas ranked by their weighted factor contribution deficit.
5. IF the Platform fails to compute the Loan_Readiness_Score due to missing or invalid input data, THEN THE Platform SHALL display the prediction results without the readiness score and show a message indicating the readiness score is unavailable.

### Requirement 10: Interactive What-If Simulator

**User Story:** As a loan applicant, I want to modify my financial inputs and instantly see how changes affect my approval chances, so that I can understand what actions would improve my loan eligibility.

#### Acceptance Criteria

1. WHEN an Applicant navigates to the What-If Simulator from their dashboard or from an existing assessment result, THE Platform SHALL display an interactive form pre-populated with the Applicant's most recent assessment inputs (or empty if no prior assessment exists).
2. WHEN an Applicant modifies any mutable input field (Monthly Income, Employment Status, Employment Length, Existing Loans, Monthly EMI, DTI_Ratio, Credit Utilization, or Loan Amount Requested), THE Platform SHALL recalculate and display the updated Approval_Probability, Risk_Score, and Loan_Readiness_Score within 2 seconds without requiring a full form submission.
3. THE Platform SHALL display a side-by-side comparison showing the original values and simulated values for Approval_Probability, Risk_Score, Risk_Level, and Loan_Readiness_Score, with visual indicators (arrows or color changes) showing improvement or decline in a user-friendly format without technical jargon.
4. THE Platform SHALL validate all simulated input values against the same validation rules defined in Requirement 4 before computing updated predictions.
5. THE What-If Simulator SHALL NOT store simulation results as formal loan applications or trigger audit logging for simulated predictions.
6. THE Platform SHALL render the What-If Simulator with a responsive layout that adapts to mobile (320px+), tablet (768px+), and desktop (1024px+) viewports.
7. IF the Prediction_Engine fails to return a simulated result within 2 seconds, THEN THE Platform SHALL display a loading indicator and retry once, showing an error message if the retry also fails.

### Requirement 11: Fairness Monitoring

**User Story:** As an admin, I want to monitor fairness metrics, so that I can detect and address potential bias in the lending model.

#### Acceptance Criteria

1. THE Fairness_Monitor SHALL compute Demographic Parity difference and Equalized Odds difference metrics using Fairlearn on model predictions, with a default acceptable threshold of 0.1 for both metrics.
2. WHEN an Admin navigates to the fairness dashboard, THE Platform SHALL display current fairness metrics with visual indicators (green for within threshold, red for exceeding threshold) showing whether metrics are within acceptable thresholds.
3. THE Fairness_Monitor SHALL detect potential proxy bias by computing Pearson correlation coefficients between protected attributes (Age, Gender if available) and model input features, flagging any correlation exceeding 0.7 as a potential proxy.
4. WHEN a fairness metric exceeds the configured threshold of 0.1, THE Platform SHALL flag the metric with a warning indicator on the Admin dashboard.
5. IF fewer than 30 predictions exist in the system, THEN THE Platform SHALL display a notice indicating insufficient data for reliable fairness metrics.

### Requirement 12: Audit Logging

**User Story:** As an admin, I want comprehensive audit logs, so that the platform maintains regulatory compliance and decision traceability.

#### Acceptance Criteria

1. WHEN a prediction is generated, THE Audit_Logger SHALL record: Applicant inputs, prediction outputs (Approval_Probability, Risk_Score, Default_Probability, decision), SHAP explanation values, counterfactual recommendations, timestamp, and User ID.
2. THE Audit_Logger SHALL store all audit records in the Supabase PostgreSQL database with immutable write-only access (no update or delete operations permitted on audit records).
3. WHEN an Admin views the audit log page, THE Platform SHALL display audit records sorted by timestamp in descending order (most recent first) with pagination of 50 records per page and filtering by date range, user, and decision outcome.
4. WHEN a Bank_Officer views an application, THE Platform SHALL display the audit trail for that specific application showing all prediction events and Bank_Officer review actions in chronological order.
5. THE Audit_Logger SHALL record all user authentication events (login, logout, failed attempts) with timestamp and IP address.
6. IF the Audit_Logger fails to persist an audit record, THEN THE Platform SHALL retry the write operation up to 3 times, and if all retries fail, THE Platform SHALL queue the record for deferred writing and display a warning notification to the Admin dashboard indicating an audit logging failure.

### Requirement 13: PDF Report Generation

**User Story:** As a loan applicant, I want to download a PDF report of my assessment, so that I can keep a record and share it with financial advisors.

#### Acceptance Criteria

1. WHEN an Applicant clicks the download report button on an assessment result, THE PDF_Generator SHALL produce a PDF document containing: Applicant name and submitted application inputs (as defined in Requirement 4), Risk_Score, Approval_Probability, Decision, the top 5 SHAP feature contributions ranked by absolute SHAP value magnitude, Recommendations (counterfactual improvement suggestions if the decision is Rejected, or general positive factors if Approved), and the prediction Timestamp.
2. THE PDF_Generator SHALL format the report with the platform logo, a report title, and labeled section headings separating each content area (Applicant Information, Risk Assessment, Explanation, Recommendations).
3. WHEN the PDF is generated, THE Platform SHALL initiate a browser download of the PDF file within 10 seconds of the Applicant clicking the download button.
4. IF the PDF_Generator encounters an error during generation, THEN THE Platform SHALL display an error message indicating the report could not be generated and log the failure details including the assessment ID and error type.
5. IF the assessment decision is Approved and no counterfactual recommendations exist, THEN THE PDF_Generator SHALL include a Recommendations section stating the key positive factors that contributed to approval based on the top 3 positive SHAP values.

### Requirement 14: Applicant Dashboard

**User Story:** As a loan applicant, I want a centralized dashboard, so that I can access all my assessment features and history in one place.

#### Acceptance Criteria

1. WHEN an Applicant logs in, THE Platform SHALL display the Applicant Dashboard with sections for: New Assessment, What-If Simulator, Assessment History, Risk Reports, SHAP Explanation View, and Recommendations.
2. WHEN an Applicant navigates to Assessment History, THE Platform SHALL display a paginated list (maximum 20 items per page) of all previous applications sorted by date in descending order, showing date, decision, Risk_Score, and Approval_Probability for each entry.
3. WHEN an Applicant selects a previous assessment with a Rejected decision, THE Platform SHALL display the full assessment details including input values, Risk_Score, Approval_Probability, decision, SHAP explanations, and counterfactual recommendations.
4. WHEN an Applicant selects a previous assessment with an Approved decision, THE Platform SHALL display the full assessment details including input values, Risk_Score, Approval_Probability, decision, and SHAP explanations without counterfactual recommendations.
5. IF an Applicant has no previous assessments, THEN THE Platform SHALL display an empty state message in the Assessment History section indicating no assessments have been submitted and providing a link to start a New Assessment.
6. IF the Platform fails to load dashboard data due to a network or server error, THEN THE Platform SHALL display an error message indicating the data could not be retrieved and provide a retry option.
7. THE Platform SHALL render the Applicant Dashboard with a mobile-first responsive layout that adapts to mobile (320px+), tablet (768px+), and desktop (1024px+) viewports.

### Requirement 15: Bank Officer Dashboard

**User Story:** As a bank officer, I want a comprehensive review dashboard, so that I can efficiently review applications and make informed lending decisions.

#### Acceptance Criteria

1. WHEN a Bank_Officer logs in, THE Platform SHALL display the Bank Officer Dashboard with sections for: Application Review, Risk Analytics, Explainability Dashboard, and Audit View.
2. WHEN a Bank_Officer navigates to Application Review, THE Platform SHALL display a paginated list (20 items per page) of submitted applications sorted by submission date descending, showing applicant name, submission date, Risk_Score, and decision status (Pending Review, Approved, or Rejected) for each entry.
3. WHEN a Bank_Officer selects an application for review, THE Platform SHALL display the complete application details (all fields from the loan application form), prediction results (Approval_Probability, Risk_Score, Default_Probability, decision), SHAP explanations, and counterfactual recommendations on a single page.
4. THE Platform SHALL render the Bank Officer Dashboard with a mobile-first responsive layout that adapts to mobile (320px+), tablet (768px+), and desktop (1024px+) viewports.
5. IF no submitted applications exist when a Bank_Officer navigates to Application Review, THEN THE Platform SHALL display an empty state message indicating no applications are available for review.
6. IF the Platform fails to load application data for the Bank Officer Dashboard, THEN THE Platform SHALL display an error message indicating the data could not be retrieved and provide a retry option.

### Requirement 16: Admin Dashboard

**User Story:** As an admin, I want a monitoring dashboard, so that I can oversee system health, fairness, and user management.

#### Acceptance Criteria

1. WHEN an Admin logs in, THE Platform SHALL display the Admin Dashboard with sections for: Fairness Monitoring, User Management, Audit Logs, and Model Statistics.
2. WHEN an Admin navigates to User Management, THE Platform SHALL display a paginated list (maximum 20 users per page) of all registered users with name, email, role, and registration date, sorted by registration date descending.
3. WHEN an Admin confirms a user role change, THE Platform SHALL present a confirmation prompt displaying the target user's name, current role, and new role, and SHALL only proceed with the update after the Admin confirms the action.
4. WHEN an Admin confirms a user role change, THE Platform SHALL update the user role to one of the valid roles (Applicant, Bank_Officer, or Admin) in the database and the change SHALL take effect on the user's next authentication.
5. IF the Platform is unable to update a user role due to a database or network error, THEN THE Platform SHALL display an error message indicating the role change failed and SHALL retain the user's previous role unchanged.
6. IF an Admin attempts to remove the Admin role from the only remaining Admin account, THEN THE Platform SHALL reject the change and display an error message indicating that at least one Admin must exist in the system.
7. THE Platform SHALL render the Admin Dashboard with a mobile-first responsive layout that adapts to mobile (320px+), tablet (768px+), and desktop (1024px+) viewports.

### Requirement 17: Responsive Mobile-First Design

**User Story:** As a user on any device, I want the platform to work seamlessly on my screen size, so that I can access all features regardless of device.

#### Acceptance Criteria

1. THE Platform SHALL implement a mobile-first responsive design using Tailwind CSS breakpoints: mobile (320px+), tablet (768px+), and desktop (1024px+).
2. THE Platform SHALL render all interactive charts and visualizations (SHAP, fairness, analytics) with a minimum font size of 12px for labels, minimum tap target of 44x44 pixels for interactive chart elements, and no horizontal scrolling required to view chart content on mobile viewports (320px-767px).
3. WHILE the viewport width is below 768px, THE Platform SHALL provide touch-friendly interaction targets with a minimum tap area of 44x44 pixels for all interactive elements including buttons, links, form inputs, and chart data points.
4. THE Platform SHALL use shadcn/ui components styled with Tailwind CSS with a consistent typography scale, spacing system, and color palette applied uniformly across all pages.
5. WHILE the viewport width is below 768px, THE Platform SHALL display a collapsible navigation menu that provides access to all role-appropriate pages without horizontal scrolling.
6. THE Platform SHALL render all page content without horizontal overflow at any supported viewport width (320px and above), requiring only vertical scrolling to access content.

### Requirement 18: API Security and Protection

**User Story:** As a platform operator, I want the API to be secure, so that unauthorized access and abuse are prevented.

#### Acceptance Criteria

1. THE Platform SHALL require a valid Supabase Auth JWT token for all API endpoints except registration and login.
2. WHEN an API request is received without a valid token (missing, expired, or malformed), THE Platform SHALL return a 401 Unauthorized response with an error message indicating the authentication failure reason.
3. WHEN an API request is received with a valid token but insufficient role permissions, THE Platform SHALL return a 403 Forbidden response with an error message indicating that the user lacks permission for the requested resource.
4. THE Platform SHALL validate all API request bodies using Pydantic schemas on the FastAPI backend before processing.
5. THE Platform SHALL implement rate limiting on authentication endpoints to prevent brute-force attacks (maximum 5 failed attempts per minute per IP address), with the attempt counter resetting after 1 minute of no failed attempts from that IP address.
6. WHEN an IP address exceeds 5 failed authentication attempts within 1 minute, THE Platform SHALL return a 429 Too Many Requests response and reject further authentication attempts from that IP address until the 1-minute cooldown period has elapsed.

### Requirement 19: Input Sanitization and Validation

**User Story:** As a platform operator, I want all user inputs validated and sanitized, so that the system is protected from malicious input.

#### Acceptance Criteria

1. THE Platform SHALL validate all numeric input fields on both the frontend (TypeScript) and backend (Pydantic) before processing, rejecting any value that is not a valid number, contains non-numeric characters (excluding a single decimal point and leading negative sign where applicable), or exceeds the field's defined range.
2. THE Platform SHALL sanitize all text input fields by encoding HTML special characters (&, <, >, ", ') as their HTML entity equivalents before rendering, preventing cross-site scripting (XSS) attacks.
3. WHEN the backend receives input that fails Pydantic validation, THE Platform SHALL return a 422 Unprocessable Entity response containing an array of error objects, each identifying the field name and a human-readable description of the validation failure.
4. THE Platform SHALL reject any loan application input containing values outside the defined valid ranges specified in Requirement 4.
5. WHEN a frontend input field fails validation, THE Platform SHALL display an inline error message adjacent to the invalid field within 200 milliseconds, identifying the specific validation rule that failed.
6. THE Platform SHALL enforce a maximum length of 500 characters on all text input fields and reject any input exceeding this limit on both the frontend and backend.
7. IF a text input field contains only whitespace or becomes empty after sanitization, THEN THE Platform SHALL treat the field as blank and apply the same validation rules as an empty required field.

### Requirement 20: Human-Readable Explanations

**User Story:** As a loan applicant, I want all explanations and recommendations presented in simple, non-technical language, so that I can understand my assessment results without financial expertise.

#### Acceptance Criteria

1. THE Platform SHALL present all applicant-facing explanations, recommendations, SHAP summaries, and counterfactual suggestions in plain language understandable by the average user without financial or technical expertise.
2. THE Platform SHALL NOT display technical terms (such as Debt-to-Income Ratio, Credit Utilization Ratio, SHAP Value, KS Statistic, or Demographic Parity) in applicant-facing interfaces, and SHALL instead use plain-English equivalents (such as "portion of income going toward debt", "how much of your credit limit you are using", or "factors affecting your decision").
3. WHEN counterfactual recommendations are displayed to an Applicant, THE Platform SHALL present each recommendation with: a plain-language description of what to change, the current value, the recommended target value, and a brief explanation of why this change helps.
4. WHEN SHAP explanations are displayed to an Applicant, THE Platform SHALL present feature contributions as plain-language statements explaining how each factor helped or hurt their approval chances.
5. THE Platform SHALL present the applicant experience as a financial guidance assistant, clearly answering four questions for every rejected application: why was my application rejected, what should I improve, how much should I improve, and what will happen if I improve it.
6. WHEN counterfactual recommendations are displayed, THE Platform SHALL show the estimated Approval_Probability after applying all suggested changes, enabling the Applicant to understand the projected outcome.

## Non-Functional Requirements

### Requirement 21: Performance

**User Story:** As a user, I want the platform to respond quickly, so that I can complete my tasks without frustrating delays.

#### Acceptance Criteria

1. WHEN a user submits a loan application, THE Platform SHALL return the complete prediction result (including SHAP values) within 5 seconds.
2. THE Platform SHALL render any dashboard page to a time-to-interactive state within 3 seconds on a connection with 10 Mbps bandwidth and 50 ms latency.
3. WHILE 50 concurrent users are performing a mix of loan submissions and dashboard navigations, THE Platform SHALL maintain response times within 120% of the single-user response times specified in criteria 1 and 2.
4. IF a dashboard page fails to reach a time-to-interactive state within 3 seconds, THEN THE Platform SHALL display a loading indicator until the page becomes interactive.

### Requirement 22: Data Privacy and Security

**User Story:** As a loan applicant, I want my financial data protected, so that my sensitive information remains confidential.

#### Acceptance Criteria

1. THE Platform SHALL encrypt all data in transit using HTTPS/TLS.
2. THE Platform SHALL store all sensitive applicant data in Supabase PostgreSQL with RLS policies ensuring applicants can only access their own data.
3. THE Platform SHALL not expose raw model weights, training data, or internal model parameters through any API endpoint.
4. WHEN a user account is deleted, THE Platform SHALL retain anonymized audit records while removing all personally identifiable information.
