# Athenaeum — Library Management System

A full-stack library management system built to demonstrate a complete CRUD application with authentication, a relational database, a REST API, and a hand-built frontend — no frontend framework, no UI library, just HTML/CSS/JS talking to an ASP.NET Core API.

**Stack:** ASP.NET Core 8 Web API · Entity Framework Core · SQL Server · JWT Auth · vanilla HTML/CSS/JavaScript

---

## What it does

Librarians sign in and can:

- **Manage the catalog** — add, edit, delete books; search by title/author/ISBN; filter by category
- **Manage members** — register members, suspend/reactivate accounts, search the roster
- **Process loans** — check a book out to a member (auto-decrements available copies), mark it returned (auto-calculates a late fine if it's overdue)
- **See it all at a glance** — a dashboard with live counts (books, members, active loans, overdue loans, fines collected) and a recent-activity feed

Business rules enforced server-side: can't borrow a book with zero copies left, can't borrow as a suspended member, can't delete a book/member that has active loans, can't drop total copies below the number currently on loan, and overdue loans are flagged automatically and fined at $0.50/day on return.

---

## Architecture

```
┌─────────────────────┐        JSON over HTTP        ┌──────────────────────────┐        EF Core         ┌───────────────────┐
│   frontend/          │  ───────────────────────▶   │  backend/                │  ────────────────────▶ │   SQL Server       │
│   HTML + CSS + JS     │   Bearer <JWT> in header     │  ASP.NET Core Web API    │     LINQ queries        │   LibraryManagement │
│   (static, no build)  │  ◀───────────────────────   │  Controllers → DbContext │  ◀──────────────────── │   DB                │
└─────────────────────┘                              └──────────────────────────┘                         └───────────────────┘
```

- The **frontend** is plain static files — open them directly or serve with any static file server. Each page calls the API with `fetch()` and stores the JWT in `localStorage`.
- The **backend** is a REST API (no server-rendered views). Controllers are thin; they validate, talk to EF Core, and return DTOs (never raw entities, to avoid leaking internal shape or causing serialization cycles).
- The **database** schema is defined in plain SQL (`database/schema.sql`) so it's portable and reviewable independent of the ORM. EF Core maps onto that exact schema via Fluent API in `LibraryDbContext`.

---

## Project structure

```
library-management-system/
├── database/
│   └── schema.sql                 # Tables, constraints, indexes, seed data
├── backend/
│   └── LibraryManagementSystem.Api/
│       ├── Controllers/           # AuthController, BooksController, MembersController, LoansController, CategoriesController, DashboardController
│       ├── Models/                # Book, Member, Loan, User, Category (EF entities)
│       │   └── Dtos/              # Request/response shapes used by the API
│       ├── Data/                  # LibraryDbContext, DbSeeder (creates the default admin)
│       ├── Services/              # TokenService (JWT issuing)
│       └── Program.cs             # DI, JWT auth, CORS, Swagger
└── frontend/
    ├── index.html                 # Login
    ├── dashboard.html
    ├── books.html
    ├── members.html
    ├── loans.html
    ├── css/styles.css
    └── js/                        # api.js (fetch wrapper), auth.js, dashboard.js, books.js, members.js, loans.js
```

---

## Getting started

### Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download)
- SQL Server (LocalDB, which ships with Visual Studio, works fine — or full SQL Server / Express)
- Any way to serve static files for the frontend (a code editor's "Live Server" extension, or just opening the HTML files directly)

### 1. Create the database

Open `database/schema.sql` in SQL Server Management Studio or Azure Data Studio (connected to your LocalDB/SQL Server instance) and run the whole script. It creates the `LibraryManagementDB` database, all five tables with their constraints/indexes, and seeds a handful of sample books and members.

### 2. Run the API

```bash
cd backend/LibraryManagementSystem.Api
dotnet restore
dotnet run
```

By default it listens on `http://localhost:5236` (set in `Properties/launchSettings.json`) and opens Swagger UI at `http://localhost:5236/swagger` so you can explore/test every endpoint.

If your SQL Server isn't LocalDB, update the connection string in `appsettings.json` first:

```json
"DefaultConnection": "Server=YOUR_SERVER;Database=LibraryManagementDB;Trusted_Connection=True;TrustServerCertificate=True;"
```

On first run, the API automatically creates a default login:

| Username | Password   |
|----------|------------|
| `admin`  | `Admin@123` |

**Change this password (or the seeding logic in `Data/DbSeeder.cs`) before deploying anywhere public.**

### 3. Run the frontend

The frontend is just static files. Easiest options:

- **VS Code:** install the "Live Server" extension, right-click `frontend/index.html` → "Open with Live Server"
- **Python:** `cd frontend && python -m http.server 5500`, then visit `http://localhost:5500`
- Or just double-click `index.html` to open it in your browser — it works directly against the API too

If the API is running on a different port than `5236`, update the `API_BASE` constant at the top of `frontend/js/api.js`.

Log in with the admin credentials above and you're in.

---

## API reference

All routes except `/api/auth/login` require `Authorization: Bearer <token>`.

| Method | Endpoint                 | Description                          |
|--------|---------------------------|---------------------------------------|
| POST   | `/api/auth/login`         | Authenticate, returns a JWT           |
| GET    | `/api/books`              | List books (`?search=`, `?categoryId=`) |
| GET    | `/api/books/{id}`         | Get one book                          |
| POST   | `/api/books`               | Create a book                         |
| PUT    | `/api/books/{id}`          | Update a book                         |
| DELETE | `/api/books/{id}`          | Delete a book (blocked if on loan)    |
| GET    | `/api/members`             | List members (`?search=`)             |
| POST   | `/api/members`             | Register a member                     |
| PUT    | `/api/members/{id}`        | Update a member                       |
| DELETE | `/api/members/{id}`        | Delete a member (blocked if on loan)  |
| GET    | `/api/loans`               | List loans (`?status=`, `?memberId=`) |
| POST   | `/api/loans/borrow`        | Check out a book                      |
| PUT    | `/api/loans/{id}/return`   | Return a book, fine auto-calculated   |
| GET    | `/api/categories`          | List categories                       |
| GET    | `/api/dashboard/stats`     | Summary counts + recent activity      |

---

## Notes for showcasing this on a portfolio

- Take a few screenshots of the dashboard, books table, and the borrow modal in action — those make the strongest portfolio thumbnails.
- The default admin password is intentionally simple for local demos. If you deploy this anywhere public-facing, change it.
- `.gitignore` already excludes `bin/`, `obj/`, and any `appsettings.Production.json`/`secrets.json` so you won't accidentally commit build output or secrets.
- Feel free to swap the database name/branding ("Athenaeum") for your own — it's all in `appsettings.json` and the `<title>`/sidebar markup in the HTML files.

## License

MIT — see [LICENSE](LICENSE).
