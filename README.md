# AI Loan Decision Platform

Explainable & Fair AI-Powered Loan Decision Support Platform with transparent lending decisions.

## Architecture

- **Frontend**: Next.js 14 (TypeScript) + Tailwind CSS + shadcn/ui
- **Backend**: FastAPI (Python) + XGBoost + SHAP + DiCE + Fairlearn
- **Database**: Supabase PostgreSQL with Row Level Security
- **Auth**: Supabase Auth (JWT-based)

## Quick Start

### Prerequisites

- Python 3.12+
- Node.js 18+
- Supabase project (with migrations applied)

### Backend Setup

```bash
cd backend
python -m venv venv
.\venv\Scripts\activate  # Windows
# source venv/bin/activate  # Linux/Mac

pip install -r requirements.txt
cp .env.example .env  # Fill in Supabase credentials

# Train ML models (first time only)
python -m app.ml.train

# Start server
python -m uvicorn app.main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
cp .env.local.example .env.local  # Fill in Supabase credentials

npm run dev
```

### Access

- Frontend: http://localhost:3000
- Backend API: http://localhost:8000
- Swagger Docs: http://localhost:8000/docs

## Features

### Applicant
- Submit loan applications with instant credit risk assessment
- View SHAP-based explanations of decisions
- Receive counterfactual recommendations for rejected applications
- What-If Simulator for exploring approval scenarios
- Assessment history with full detail views
- PDF report download

### Bank Officer
- Review all submitted applications
- View detailed predictions with SHAP explanations
- Model performance analytics (XGBoost vs Random Forest)

### Admin
- Fairness monitoring (Demographic Parity, Equalized Odds)
- User management with role assignment
- Audit logs for compliance
- Model statistics

## API Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | /api/v1/applications | Applicant | Submit loan application |
| GET | /api/v1/applications | Applicant | List own applications |
| GET | /api/v1/applications/{id} | Any | Get application detail |
| GET | /api/v1/applications/review | Officer | List all applications |
| POST | /api/v1/predictions/simulate | Applicant | What-If simulation |
| GET | /api/v1/reports/{id}/pdf | Applicant | Download PDF report |
| GET | /api/v1/admin/audit-logs | Admin | View audit logs |
| GET | /api/v1/admin/model-stats | Officer/Admin | Model metrics |
| GET | /api/v1/health | Public | Health check |


```

## License

MIT
