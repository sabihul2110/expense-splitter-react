# PROJECT_CONTEXT.md
# SplitEase — College Expense Splitter
# Last updated: 2026-03-25
# Use this file to onboard a fresh Claude instance.

---

## 1. WHAT THIS PROJECT IS

A full-stack web app for splitting expenses among groups of people.
Think Splitwise, but built from scratch as a portfolio project.

**Live stack:**
- Frontend: React (Vite) at http://localhost:5173
- Backend:  FastAPI (Python) at http://localhost:8000
- Database: MySQL (local, schema: expense_splitter_react)
- Developer machine: MacBook Air M2

**Start commands:**
```bash
# Terminal 1 — Backend
cd backend && source venv/bin/activate
uvicorn main:app --reload --loop asyncio   # --loop asyncio required on Python 3.14

# Terminal 2 — Frontend
cd frontend && npm run dev
```

---

## 2. FOLDER STRUCTURE

```
Expense-Splitter-React/
├── backend/                        Python + FastAPI
│   ├── main.py                     App entry, registers all routers, CORS config
│   ├── config.py                   Reads .env → exports DB_CONFIG, JWT settings
│   ├── auth.py                     bcrypt hashing, JWT create/decode, FastAPI deps
│   ├── db.py                       ALL raw SQL functions (single source of truth)
│   ├── requirements.txt
│   ├── .env                        DB credentials + JWT secret (not in git)
│   └── routers/
│       ├── __init__.py
│       ├── auth_router.py          POST /auth/signup, POST /auth/login, GET /auth/me
│       ├── users.py                GET /users/, GET /users/all, PUT, DELETE
│       ├── groups.py               CRUD groups + members + categories/subcategories
│       ├── expenses.py             Add/list/delete expenses + splits
│       ├── payments.py             Record/list/delete settlement payments
│       ├── settlements.py          Calls MySQL stored procedure Calculate_Settlements
│       ├── invites.py              POST /groups/{id}/invite, GET+POST /invite/{token}
│       └── notifications.py        Bell notifications + POST /groups/{id}/remind
│
└── frontend/                       React + Vite + Tailwind CSS v4
    ├── vite.config.js              Includes @tailwindcss/vite plugin
    ├── package.json
    └── src/
        ├── main.jsx                Root: wraps app in BrowserRouter + AuthProvider
        ├── App.jsx                 ALL route definitions (flat, no nesting)
        ├── index.css               Global design system (CSS variables + utility classes)
        │                           NO Tailwind utility classes in JSX — all custom CSS
        ├── api/
        │   └── axios.js            Axios instance with JWT interceptor + 401 auto-logout
        ├── context/
        │   └── AuthContext.jsx     Global auth state; validates token via /auth/me on load
        ├── components/
        │   ├── AppShell.jsx        User app layout: sidebar + topbar (bell + avatar)
        │   ├── AdminLayout.jsx     Admin layout: purple sidebar + <Outlet /> for nested routes
        │   ├── NotificationBell.jsx Bell icon in topbar; polls unread count every 30s
        │   └── Sidebar.jsx         Standalone sidebar (currently imported by AppShell)
        └── pages/
            ├── Login.jsx           /login — supports ?next= param for post-login redirect
            ├── Signup.jsx          /signup — first user becomes admin automatically
            ├── Dashboard.jsx       /dashboard — balance hero + recent groups
            ├── Groups.jsx          /groups — grid of group cards + "New Group" modal
            ├── GroupDetail.jsx     /groups/:id — ledger table + settlements + invite modal
            ├── AddExpense.jsx      /groups/:id/add-expense — split-screen form + live preview
            ├── AddPayment.jsx      /groups/:id/add-payment — record a settlement payment
            ├── Settlements.jsx     /settlements — global view across all groups
            ├── Activity.jsx        /activity — timeline feed of all expenses/payments
            ├── JoinGroup.jsx       /join/:token — handles invite link join flow
            └── admin/
                ├── AdminOverview.jsx    /admin — stats overview
                ├── AdminUsers.jsx       /admin/users — view/delete all users
                ├── AdminGroups.jsx      /admin/groups — view/delete all groups
                └── AdminTransactions.jsx /admin/transactions — all expenses across groups
```

---

## 3. TECH STACK & LIBRARIES

### Backend
| Library | Version | Purpose |
|---------|---------|---------|
| FastAPI | 0.135.1 | Web framework + auto /docs |
| uvicorn | 0.42.0 | ASGI server (use --loop asyncio on Python 3.14) |
| mysql-connector-python | 9.6.0 | MySQL driver |
| python-jose[cryptography] | 3.5.0 | JWT signing/verification |
| passlib[bcrypt] | 1.7.4 | Password hashing (bcrypt MUST be pinned to 4.0.1) |
| bcrypt | 4.0.1 | ⚠️ MUST be this version — 5.x breaks passlib on Python 3.14 |
| pydantic[email] | 2.12.5 | Request validation + EmailStr |
| python-dotenv | 1.2.2 | Load .env file |

### Frontend
| Library | Version | Purpose |
|---------|---------|---------|
| React | 19.x | UI framework |
| Vite | 8.x | Build tool + dev server |
| react-router-dom | 7.x | Client-side routing |
| axios | 1.x | HTTP client |
| tailwindcss | v4 | CSS framework (used via @tailwindcss/vite plugin) |
| @tailwindcss/vite | v4 | Vite plugin for Tailwind v4 |

---

## 4. DATABASE SCHEMA

Database name: `expense_splitter_react`

### Tables
```
Users            user_id, name, email, password_hash, upi_id, role, created_at
Groups           group_id, group_name, created_by, created_at
Group_Members    group_id + user_id (PK), joined_at
Categories       category_id, category_name
Subcategories    subcategory_id, category_id, subcategory_name
Expenses         expense_id, group_id, payer_id, category_id, subcategory_id,
                 total_amount, description, split_type (equal|custom), expense_date
Expense_Splits   split_id, expense_id, user_id, amount_owed, share_pct
Payments         payment_id, group_id, payer_id, payee_id, amount, note, payment_date
Invites          invite_id, token (UNIQUE), group_id, created_by, expires_at, created_at
Notifications    notification_id, user_id, from_user_id, type, message, group_id,
                 is_read (default 0), created_at
```

### Stored Procedure
`Calculate_Settlements(group_id)` — returns per-member balances:
```
net_balance = total_paid - total_owed - payments_sent + payments_received
positive = owed to you | negative = you owe | zero = settled
```

### Seed data in DB
- 6 Categories: Travel, Accommodation, Food & Dining, Activities, Utilities, Groceries
- 17 Subcategories across those categories

---

## 5. AUTH SYSTEM

**Flow:**
1. User signs up → password hashed with bcrypt → stored in Users table
2. Login returns a JWT containing `{user_id, email, role}`
3. JWT stored in `localStorage` as `expense_user` JSON object
4. Every API request: axios interceptor reads token → adds `Authorization: Bearer <token>`
5. On app load: `AuthContext` calls `GET /auth/me` to validate token is still good
6. If token is invalid/expired → localStorage cleared → redirect to /login (no empty state)
7. Any 401 response from any API call → axios interceptor clears localStorage + redirects

**Role system:**
- `role: "user"` — normal user
- `role: "admin"` — first user to sign up gets this automatically
- Admin sees "Admin Panel" link in sidebar → goes to /admin (separate layout)
- Admin permissions: view all users/groups, delete users/groups
- Admin does NOT bypass financial logic (must be group member to add expenses)

**JWT payload:** `{user_id: int, email: str, role: str}` — name is NOT in JWT
- ⚠️ If you need the user's name in a backend route, fetch it from DB using user_id

---

## 6. API ENDPOINTS

### Auth
```
POST /auth/signup        body: {name, email, password, upi_id?}
POST /auth/login         body: {email, password}
GET  /auth/me            validates token, returns fresh user data
```

### Users
```
GET  /users/             all users (id, name, upi_id) — for dropdowns
GET  /users/all          full details — admin only
PUT  /users/{id}         update own profile (or any if admin)
DELETE /users/{id}       admin only
```

### Groups
```
GET  /groups/                      my groups
GET  /groups/all                   all groups (admin)
POST /groups/                      create group, creator auto-added as member
GET  /groups/{id}/members          member list
PUT  /groups/{id}                  rename (admin)
PUT  /groups/{id}/members          replace members (admin)
DELETE /groups/{id}                admin only
GET  /groups/categories            all categories
GET  /groups/subcategories/{cat}   subcategories for a category
```

### Expenses
```
GET  /expenses/{group_id}                list expenses (membership required)
GET  /expenses/{group_id}/{id}/splits    split breakdown for one expense
POST /expenses/{group_id}               add expense + splits
DELETE /expenses/{id}                   delete expense
```

### Payments
```
GET  /payments/{group_id}   list payments
POST /payments/{group_id}   record a payment
DELETE /payments/{id}       delete payment
```

### Settlements
```
GET /settlements/{group_id}             raw balances (calls stored procedure)
GET /settlements/{group_id}/simplified  minimal payment transactions
```

### Invites
```
POST /groups/{group_id}/invite   generate invite token (members only)
GET  /invite/{token}             get group name for token (public, no auth)
POST /invite/{token}/join        join group (auth required)
```

### Notifications
```
GET  /notifications/             my notifications (latest 50)
GET  /notifications/unread-count lightweight count for bell badge
POST /notifications/read/{id}    mark one as read
POST /notifications/read-all     mark all as read
POST /groups/{group_id}/remind   send in-app reminder to a debtor
```

---

## 7. STATE MANAGEMENT

No Redux or Zustand. Just React built-ins:

- **AuthContext** (`src/context/AuthContext.jsx`) — global user state
  - Provides: `{user, login, logout, authChecked}`
  - Persists to localStorage
  - Validates on mount via /auth/me
  - All components access via `useAuth()` hook

- **Local state** (`useState`) — all other state is component-local
  - Each page fetches its own data on mount with `useEffect`
  - No global data cache — data refetched on navigation

- **URL state** — group ID from `useParams()`, redirect target from `useSearchParams()`

---

## 8. ROUTING ARCHITECTURE

**Critical rule: flat routes, no nesting (except admin).**

```
/login                    → Login.jsx (no shell)
/signup                   → Signup.jsx (no shell)
/join/:token              → JoinGroup.jsx (no shell)
/dashboard                → Dashboard.jsx (in AppShell)
/groups                   → Groups.jsx (in AppShell)
/groups/:id               → GroupDetail.jsx (in AppShell)
/groups/:id/add-expense   → AddExpense.jsx (in AppShell)
/groups/:id/add-payment   → AddPayment.jsx (in AppShell)
/settlements              → Settlements.jsx (in AppShell)
/activity                 → Activity.jsx (in AppShell)
/admin                    → AdminLayout (with Outlet)
  /admin                  → AdminOverview.jsx
  /admin/users            → AdminUsers.jsx
  /admin/groups           → AdminGroups.jsx
  /admin/transactions     → AdminTransactions.jsx
```

Each page wraps itself in `<AppShell title="..." actions={...}>`.
Admin pages render inside `<AdminLayout>` via React Router `<Outlet />`.

**Route guards:**
- `UserRoute` — redirects to /login if no user
- `AdminRoute` — redirects to /login if no user, /dashboard if not admin

---

## 9. COMPONENT HIERARCHY

```
BrowserRouter
└── AuthProvider (context)
    └── App (routes)
        ├── Login / Signup / JoinGroup  (standalone, no shell)
        │
        ├── AppShell  (user app wrapper)
        │   ├── Sidebar (nav: Dashboard, Groups, Settlements, Activity)
        │   ├── Topbar (title | actions | NotificationBell | Avatar)
        │   └── <children> (the actual page content)
        │
        └── AdminLayout  (admin wrapper, uses React Router Outlet)
            ├── Admin Sidebar (nav: Overview, Users, Groups, Transactions)
            └── <Outlet /> (AdminOverview / AdminUsers / AdminGroups / AdminTransactions)
```

---

## 10. DESIGN SYSTEM (index.css)

**Important:** The project uses Tailwind v4 but almost exclusively via custom CSS classes in `index.css`. JSX files use class names like `card`, `btn btn-primary`, `stat-card`, etc. — not Tailwind utility classes directly.

**CSS variables:**
```css
--bg:        #0d0e14   /* page background */
--surface:   #13141c   /* cards, sidebar */
--surface2:  #1a1c26   /* inputs, hover states */
--surface3:  #21232f   /* deepest surface */
--border:    #252730
--border2:   #31333f
--primary:   #2563eb   /* blue — main CTA */
--primary-h: #3b82f6   /* hover */
--success:   #10b981   /* green */
--danger:    #ef4444   /* red */
--warning:   #f59e0b   /* amber */
--text:      #f0f1f5
--text2:     #9095a8   /* secondary text */
--text3:     #4e5260   /* muted / labels */
```

**Key class patterns:**
```
Layout:    .shell, .sidebar, .shell-main, .topbar, .page-area, .page-inner
Cards:     .card, .card-p, .card-hover, .stat-card, .stat-grid
Buttons:   .btn .btn-primary/success/danger/ghost/surface, .btn-sm/xs/lg/xl
Badges:    .badge .badge-primary/success/danger/neutral/amber
Forms:     .form-group, .form-label, .form-row (2-col)
Tables:    .table-wrap, th, td, .td-num (tabular numbers)
Tabs:      .tabs, .tab-btn, .tab-btn.active
Modals:    .modal-overlay, .modal-box, .modal-head, .modal-body
Sidebar:   .sb-logo, .sb-nav, .sb-item, .sb-item.active, .sb-footer
Type:      .t-money-lg/md/sm, .td-num, .c-success/danger/primary/muted
```

**Font:** Inter (Google Fonts, loaded in index.css)
**Font sizes:** titles 18px topbar, stat values 28px, amounts 22-36px, body 14-15px

---

## 11. FEATURE STATUS

### ✅ Fully Working
- User signup + login with JWT auth
- Auth persists across refresh (localStorage + /auth/me validation)
- Auto-logout on 401 (expired token)
- Create groups (creator auto-added as member)
- Invite system: generate link → /join/:token → join group
- Add expenses (equal split and custom split)
- Expense splits stored per-participant (payer ≠ participants is valid)
- Record settlement payments
- Settlement calculation via stored procedure
- Context-aware settlement UI:
  - Debtor sees "Pay via UPI" button
  - Creditor sees "🔔 Remind" button
  - Third party sees read-only amount
- In-app notifications: bell icon in topbar, unread count badge, 30s polling
- Send Reminder → creates notification for debtor
- Admin panel at /admin with separate layout
- Admin can view all users/groups/transactions, delete users/groups

### ⚠️ Known Issues / Fixed in Latest Files
- **Reminder bug (FIXED):** `notifications.py` was using `current_user['name']`
  but JWT only contains `{user_id, email, role}`. Fix: fetch name from DB via
  `get_user_name(user_id)`. The fixed file is `routers/notifications.py`.

### 🔲 Not Yet Built
- Email notifications (needs Resend/SendGrid)
- Push notifications
- User profile edit page
- Group name shown in topbar (currently shows "Group #1")
- Expense edit (only delete exists)
- Partial expense split for specific participants only (UI exists, backend works)
- Export to CSV
- Mobile-responsive layout

---

## 12. IMPORTANT PATTERNS & CONVENTIONS

### Backend patterns
```python
# All DB functions in db.py — routers never write SQL directly
# Every DB function opens/closes its own connection (no connection pooling)
# JWT payload = {user_id, email, role} — name is NOT stored in token
# To get name: call get_user_name(user_id) which queries the DB
# current_user = decoded JWT dict, not a full user object
# datetime from MySQL comes as naive datetime — always add timezone.utc before comparing

# Standard route pattern:
@router.get("/something")
def my_route(current_user: dict = Depends(get_current_user)):
    user_id = current_user["user_id"]  # always int
    # ...
```

### Frontend patterns
```jsx
// Every page wraps itself in AppShell:
export default function MyPage() {
  return (
    <AppShell title="Page Title" actions={<button>Action</button>}>
      {/* page content */}
    </AppShell>
  );
}

// Data fetching: useEffect on mount, local useState
const [data, setData] = useState([]);
useEffect(() => {
  api.get("/endpoint").then(r => setData(r.data));
}, []);

// Auth access:
const { user } = useAuth();
// user = { user_id, name, email, role, access_token }

// Settlements context-awareness pattern (GroupDetail.jsx):
// Compare user.name (string) against settlement row names
// This is intentional — backend returns names from DB, not IDs
```

### File change tracking
If the following files are modified, make sure the context stays updated:
- `backend/routers/notifications.py` — reminder bug fix (latest version fetches name from DB)
- `backend/routers/auth_router.py` — added GET /auth/me endpoint
- `frontend/src/context/AuthContext.jsx` — validates token on load, shows spinner
- `frontend/src/api/axios.js` — 401 interceptor for auto-logout
- `frontend/src/components/AppShell.jsx` — profile top-right, notification bell
- `frontend/src/components/NotificationBell.jsx` — new file, bell dropdown

---

## 13. .ENV FILE (backend/.env)

```
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=<your mysql password>
DB_NAME=expense_splitter_react

JWT_SECRET=<long random string>
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=60
```

This file is NOT in git. Frontend API base URL is hardcoded to `http://localhost:8000`
or read from `VITE_API_URL` env var (for deployment).

---

## 14. DEPLOYMENT NOTES (for future)

| Service | Use | Free tier notes |
|---------|-----|-----------------|
| Vercel | Frontend | ✅ Works great |
| Render | Backend (FastAPI) | ⚠️ Spins down after 15min — first request ~30s |
| Railway | MySQL DB | ⚠️ 1GB limit |

**Before deploying:**
1. Add `VITE_API_URL=https://your-backend.onrender.com` to Vercel env vars
2. Update backend `allow_origins` in `main.py` to include production frontend URL
3. Change `JWT_SECRET` to a real random secret
4. Run `pip freeze > requirements.txt` to capture all deps

**What works in production:**
- Invite links ✅ (just tokens in DB, no special infra)
- In-app notifications ✅ (polling, no websockets needed)
- Auth persistence ✅ (stateless JWT)

---

## 15. COMMON GOTCHAS

1. **Always start backend with `--loop asyncio`** on Python 3.14 (uvloop crashes)
   ```bash
   uvicorn main:app --reload --loop asyncio
   ```

2. **bcrypt must be version 4.0.1** — passlib is incompatible with bcrypt 5.x on Python 3.14
   ```bash
   pip install "bcrypt==4.0.1"
   ```

3. **JWT has no `name` field** — if you need the current user's name in a backend route,
   call `get_user_name(current_user["user_id"])` to fetch from DB.

4. **Groups query returns `created_at` as Python datetime** — convert to string before
   returning in JSON: `r["created_at"] = str(r["created_at"])`

5. **Vite proxy not configured** — frontend calls backend directly at localhost:8000.
   CORS is configured in FastAPI to allow localhost:5173.

6. **Tailwind v4 syntax** — uses `@import "tailwindcss"` not `@tailwind base/components/utilities`.
   Theme customization uses `@theme {}` block in CSS.
