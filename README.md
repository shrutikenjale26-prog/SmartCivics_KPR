# CivicConnect AI – Smart City Civic Issue Reporting Platform

CivicConnect AI is a small full-stack demo showing how citizens can report civic issues (potholes, water leakage, garbage, power outages), how departments process them, and how admins track city-wide performance.

- **Frontend**: HTML5, CSS3, minimal vanilla JavaScript
- **Backend**: Python FastAPI + MongoDB + JWT + bcrypt
- **Roles**: `citizen`, `department`, `admin`

---

## Project structure

```text
civicconnect-ai/
├── frontend/
│   ├── index.html        # Landing page
│   ├── login.html        # Shared login for all roles
│   ├── register.html     # Citizen registration
│   ├── dashboard.html    # Citizen dashboard
│   ├── department.html   # Department dashboard
│   ├── admin.html        # Admin dashboard + charts
│   ├── report.html       # Citizen complaint submission
│   ├── style.css         # Global blue/green theme
│   └── script.js         # API calls & auth handling
│
├── backend/
│   ├── main.py           # FastAPI app entrypoint
│   ├── database.py       # MongoDB connection (Motor)
│   ├── requirements.txt  # Backend dependencies
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py       # User schemas + seeding
│   │   └── complaint.py  # Complaint schemas + AI classification
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── auth.py       # Registration & login
│   │   ├── complaints.py # Citizen complaint endpoints
│   │   ├── department.py # Department-only endpoints
│   │   └── admin.py      # Admin-only endpoints & analytics
│   └── middleware/
│       ├── __init__.py
│       ├── auth.py       # JWT, bcrypt, role-based access helpers
│       └── file_size_limit.py # Upload size restriction middleware
│
└── README.md
```

---

## Backend setup

1. **Install dependencies**

   ```bash
   cd backend
   pip install -r requirements.txt
   ```

2. **Run MongoDB locally**

   Make sure a MongoDB instance is running on:

   ```text
   mongodb://localhost:27017
   ```

   The app uses database: `civicconnect_ai`.

3. **Run FastAPI with Uvicorn**

   From the `backend` directory:

   ```bash
   uvicorn main:app --reload
   ```

   The API will be available at `http://127.0.0.1:8000` and the interactive docs at `http://127.0.0.1:8000/docs`.

---

## Frontend usage

1. Open the `frontend` pages directly in your browser (e.g. via file explorer or a simple static server):

   - `frontend/index.html` – landing page
   - `frontend/login.html` – login portal
   - `frontend/register.html` – citizen registration

2. The frontend JavaScript (`script.js`) calls the FastAPI backend at:

   ```text
   http://127.0.0.1:8000
   ```

   Make sure the backend is running before using the UI.

---

## Seeded users & roles

On startup, the backend connects to MongoDB and seeds:

- **Admin user (role: admin)**
  - Email: `admin@civicconnect.com`
  - Password: `admin123`

- **Department users (role: department, password for all: `dept123`)**
  - Roads:       `roads@civicconnect.com`  (departmentName: `Roads Department`)
  - Water:       `water@civicconnect.com`  (departmentName: `Water Department`)
  - Sanitation:  `sanitation@civicconnect.com` (departmentName: `Sanitation Department`)
  - Electrical:  `electrical@civicconnect.com` (departmentName: `Electrical Department`)

Citizens register themselves via `register.html`. All newly registered users have role `citizen`.

---

## Core features

### 1. User Registration & Login

- **Registration** (`POST /auth/register`)
  - Creates a new `citizen` user with hashed password (bcrypt).
  - Fields: `name`, `email`, `password`.

- **Login** (`POST /auth/login`)
  - Returns a JWT access token with expiry and role information.
  - Response includes both the token and basic user profile, used by the frontend for redirects.

### 2. Citizen features

- **Submit complaint** (`POST /complaints/`)
  - Accepts `title`, `description`, optional `image` and `audio` uploads (multipart form-data).
  - Runs keyword-based AI classification (see below).
  - Stores `imagePath` and `audioPath` on disk under `uploads/images` and `uploads/audio`.

- **View own complaints** (`GET /complaints/my`)
  - Returns complaints where `userId` matches the logged-in user.

### 3. AI-based classification (no external APIs)

Inside `models/complaint.py` a simple classifier inspects complaint descriptions:

- **Category by keyword**
  - `pothole`, `road` → `Roads` (`Roads Department`)
  - `water`, `leakage` → `Water` (`Water Department`)
  - `garbage`, `trash` → `Sanitation` (`Sanitation Department`)
  - `electricity`, `power` → `Electrical` (`Electrical Department`)
  - Otherwise → `General` (`General Administration`)

- **Priority by keyword**
  - `urgent`, `dangerous` → `High`
  - `delay`, `issue` → `Medium`
  - Otherwise → `Low`

The backend sets `category`, `priority`, and `departmentAssigned` automatically when a complaint is created.

### 4. Department dashboard

- **List assigned complaints** (`GET /department/complaints`)
  - Only returns complaints where `departmentAssigned` equals the department user’s `departmentName`.

- **Update status and add response note** (`PATCH /department/complaints/{complaint_id}`)
  - Fields: `status` (`Pending`, `In Progress`, `Resolved`), optional `responseNote`.
  - The `Complaint` model stores `responseNote` for citizen visibility.

### 5. Admin dashboard

- **View all complaints** (`GET /admin/complaints`)
- **View all users** (`GET /admin/users`)
- **Create department users** (`POST /admin/departments`)
  - Admin can promote new department accounts by specifying name, email, password, and `departmentName`.
- **Delete complaints** (`DELETE /admin/complaints/{complaint_id}`)
- **Analytics** (`GET /admin/analytics`)
  - Returns:
    - `total` – total complaints
    - `pending` – pending complaints
    - `resolved` – resolved complaints
    - `highPriority` – high-priority complaints
  - Admin UI uses a small Chart.js doughnut chart to visualize this data.

---

## Security details

- **JWT authentication**
  - Implemented in `middleware/auth.py` using `python-jose`.
  - Tokens include `sub` (user id), `role`, optional `departmentName`, and an expiry (`ACCESS_TOKEN_EXPIRE_MINUTES`).

- **Password hashing (bcrypt)**
  - All passwords are hashed with bcrypt via `hash_password`/`verify_password` in `middleware/auth.py`.

- **Role-based access control (RBAC)**
  - Helper `role_required([...])` in `middleware/auth.py` is used as a FastAPI dependency.
  - Route-level enforcement:
    - Citizens only: `/complaints/*`
    - Departments only: `/department/*`
    - Admin only: `/admin/*`

- **Protected routes**
  - Frontend stores the JWT and user profile in `localStorage`.
  - `script.js` helper `requireRole([...])` checks role before rendering each dashboard page.
  - Requests to protected APIs include `Authorization: Bearer <token>` headers.

- **Input validation**
  - Pydantic models enforce basic constraints on strings, emails, and password lengths.

- **File upload size restriction**
  - `FileSizeLimitMiddleware` blocks requests with `Content-Length` above the configured limit (default 5 MB).

---

## Running the full stack locally

1. **Start MongoDB** (on `mongodb://localhost:27017`).
2. **Start the FastAPI backend** from the `backend` folder:

   ```bash
   uvicorn main:app --reload
   ```

3. **Open the frontend** by loading `frontend/index.html` in your browser.
   - For best results, use a simple static server so that the browser origin is `http://localhost` (but `file://` also works with the provided CORS configuration).

4. **Example flows**
   - Log in as **admin** → explore analytics, users, and complaints.
   - Log in as a **department** user → see only that department’s complaints and update statuses.
   - Register as a **citizen** → submit complaints with description, image, and audio; then watch them progress.

---

## Notes

- This project is intentionally kept simple and suitable for local demos, training, and academic projects.
- For production, you should:
  - Move `SECRET_KEY` into a secure environment variable.
  - Use robust logging, auditing, and HTTPS termination.
  - Replace the toy keyword-based classifier with a real ML model or rules engine if needed.

