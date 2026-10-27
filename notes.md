# Festival Portal — Start-to-Finish Coding Guide

**Purpose:** A fast, practical reference for building the Festival Portal with React + Vite, Axios, and a Node/Express API. Use this as a coding map during your assessment. Learn the small patterns, then type them yourself.

> **Important:** This guide is a template, not a claim that every route already exists in your backend. Confirm your actual API routes and response shapes in your API Explorer/Postman before connecting login, joining festivals, or managing events. Replace example endpoints where needed.

---

## 0. Target features

### Public visitor
- Browse festivals.
- Search and sort festivals.
- View festival details.
- See a Login button.

### Participant
- Log in.
- See **My Festivals**.
- Join a festival.
- Cannot join the same festival twice.

### Event Manager
- Log in.
- See **Manage Festivals**.
- Create, edit, and delete festivals if permitted.
- View participants if the backend supports it.

### Security rule
The backend—not the React interface—must authenticate users, assign/verify roles, and authorize protected actions. Hiding a button is not security.

---

## 1. Project structure

A simple frontend structure:

```text
festival-portal/
├── src/
│   ├── main.jsx
│   ├── App.jsx
│   ├── Login.jsx
│   ├── api.js
│   └── App.css
├── package.json
└── index.html
```

Keep the existing backend in its own folder if it is a separate project.

---

## 2. Start the project

From the frontend folder:

```bash
npm install
npm run dev
```

If creating a new React + Vite project:

```bash
npm create vite@latest festival-portal -- --template react
cd festival-portal
npm install
npm install axios
npm run dev
```

Install Git tracking if this project is not already a Git repository:

```bash
git init
git add .
git commit -m "Initial Festival Portal setup"
```

After each working feature, save a checkpoint:

```bash
git add .
git commit -m "Describe the working feature"
```

Useful Git commands:

```bash
git status
git add .
git commit -m "Add festival search"
git log --oneline
```

---

## 3. Understand the React entry point — `src/main.jsx`

Vite loads `main.jsx`, which renders `App.jsx`.

```jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App.jsx";
import "./App.css";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

**Remember:** `main.jsx` starts the React app; `App.jsx` contains the main screen.

---

## 4. Create the API helper — `src/api.js`

Centralize Axios configuration so you do not repeat the base URL everywhere.

```js
import axios from "axios";

const api = axios.create({
  // Change this if your backend runs at another address or port.
  baseURL: "http://localhost:5000/api",
  headers: {
    "Content-Type": "application/json",
  },
});

export default api;
```

If your API uses a different port or prefix, update `baseURL`.

---

## 5. Fetch and display festivals

The example assumes the endpoint is `GET /festivals` and returns either an array or a response shaped like:

```json
{
  "data": [
    { "id": 100, "name": "Sample Festival", "location": "Philippines" }
  ],
  "pagination": { "page": 1, "limit": 15, "total": 100, "totalPages": 7 },
  "success": true
}
```

### Core pattern

```jsx
import { useEffect, useState } from "react";
import api from "./api";

function App() {
  const [festivals, setFestivals] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState("");

  useEffect(() => {
    async function loadFestivals() {
      try {
        const response = await api.get("/festivals");

        // This API wraps the array in response.data.data.
        // If your API returns an array directly, use response.data instead.
        setFestivals(response.data.data);
      } catch (err) {
        setError("Could not load festivals.");
        console.error(err);
      } finally {
        setLoading(false);
      }
    }

    loadFestivals();
  }, []);

  if (loading) return <p>Loading festivals...</p>;
  if (error) return <p>{error}</p>;

  return (
    <main>
      <h1>Festival Portal</h1>

      {festivals.map((festival) => (
        <article key={festival.id}>
          <h2>{festival.name}</h2>
          <p>{festival.location}</p>
          <p>{festival.description}</p>
        </article>
      ))}
    </main>
  );
}

export default App;
```

### Why these parts matter
- `useState([])` starts with an empty list.
- `useEffect(..., [])` loads the list once after the component mounts.
- `response.data.data` accesses the array in the example API response.
- `key={festival.id}` gives each rendered item a stable React key.
- `try/catch/finally` handles errors and loading state.

**Avoid:** `festivals.map is not a function`. Check the actual response and store the array, not the whole response object.

---

## 6. Add search

Add this state inside `App()`:

```jsx
const [search, setSearch] = useState("");
```

Before the `return`, create a filtered list:

```jsx
const filteredFestivals = festivals.filter((festival) => {
  const query = search.toLowerCase();

  return (
    festival.name?.toLowerCase().includes(query) ||
    festival.location?.toLowerCase().includes(query)
  );
});
```

Add the input above the list:

```jsx
<input
  type="text"
  placeholder="Search by festival or location"
  value={search}
  onChange={(event) => setSearch(event.target.value)}
/>
```

Then map `filteredFestivals` instead of `festivals`:

```jsx
{filteredFestivals.map((festival) => (
  <article key={festival.id}>
    <h2>{festival.name}</h2>
    <p>{festival.location}</p>
  </article>
))}
```

**Remember:** Controlled input = `value` + `onChange`.

---

## 7. Add sorting

Add state:

```jsx
const [sortOrder, setSortOrder] = useState("asc");
```

Create a sorted copy (do not sort the original state array directly):

```jsx
const sortedFestivals = [...filteredFestivals].sort((a, b) => {
  const dateA = a.start_date
    ? new Date(a.start_date).getTime()
    : Number.MAX_SAFE_INTEGER;

  const dateB = b.start_date
    ? new Date(b.start_date).getTime()
    : Number.MAX_SAFE_INTEGER;

  return sortOrder === "asc" ? dateA - dateB : dateB - dateA;
});
```

Add a dropdown:

```jsx
<select
  value={sortOrder}
  onChange={(event) => setSortOrder(event.target.value)}
>
  <option value="asc">Earliest Date First</option>
  <option value="desc">Latest Date First</option>
</select>
```

Map `sortedFestivals` in the display.

---

## 8. Simple page layout without Bootstrap

You can keep the layout plain while learning:

```jsx
return (
  <main>
    <header className="app-header">
      <h1>Festival Portal</h1>
      {/* Place the login/role-dependent buttons here. */}
    </header>

    <input
      type="text"
      placeholder="Search festivals"
      value={search}
      onChange={(event) => setSearch(event.target.value)}
    />

    {/* Place sorting controls and festival cards here. */}
  </main>
);
```

Optional basic CSS in `App.css`:

```css
body {
  font-family: Arial, sans-serif;
  margin: 0;
  padding: 24px;
}

.app-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
}

input,
select,
button {
  padding: 8px;
  margin: 4px;
}

.festival-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 16px;
}

.festival-card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 16px;
}
```

---

## 9. Create a simple login screen — `src/Login.jsx`

This is only the form UI. It does **not** authenticate a user until you connect it to a real backend endpoint.

```jsx
import { useState } from "react";

function Login({ onLogin, onBack }) {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");

  async function handleSubmit(event) {
    event.preventDefault();
    setError("");

    // onLogin should call the actual backend login endpoint.
    try {
      await onLogin({ email, password });
    } catch (err) {
      setError("Login failed. Check your credentials.");
    }
  }

  return (
    <main>
      <h2>Login</h2>

      <form onSubmit={handleSubmit}>
        <input
          type="email"
          placeholder="Email"
          value={email}
          onChange={(event) => setEmail(event.target.value)}
          required
        />

        <input
          type="password"
          placeholder="Password"
          value={password}
          onChange={(event) => setPassword(event.target.value)}
          required
        />

        <button type="submit">Sign In</button>
        <button type="button" onClick={onBack}>Back</button>
      </form>

      {error && <p role="alert">{error}</p>}
    </main>
  );
}

export default Login;
```

### Controlled form pattern
- `value={email}` shows the current state.
- `onChange` updates the state.
- `event.preventDefault()` stops the browser from reloading the page.
- `type="button"` prevents the Back button from submitting the form.

---

## 10. Login and role-based navigation

The backend should return the authenticated user and role. For example only:

```json
{
  "success": true,
  "user": {
    "id": 1,
    "name": "Sample User",
    "email": "sample@example.com",
    "role": "participant"
  },
  "token": "example-token"
}
```

An event manager might have `"role": "event_manager"`. Use the actual field names and token/session mechanism returned by your backend.

### State in `App.jsx`

```jsx
const [user, setUser] = useState(null);
const [showLogin, setShowLogin] = useState(false);
const [showMyFestivals, setShowMyFestivals] = useState(false);
const [showManageFestivals, setShowManageFestivals] = useState(false);
```

### Login function — only after confirming the real endpoint

The following is a **pattern only**. Replace `/login`, response fields, and token handling with the backend's actual contract.

```jsx
async function handleLogin(credentials) {
  const response = await api.post("/login", credentials);

  // Example only: adjust to the actual response structure.
  const loggedInUser = response.data.user;

  setUser(loggedInUser);
  setShowLogin(false);
}
```

Do not use this function until you confirm that `POST /api/login` exists and know the request and response shapes.

### Header buttons

```jsx
<header className="app-header">
  <h1>Festival Portal</h1>

  {!user ? (
    <button onClick={() => setShowLogin(true)}>Login</button>
  ) : (
    <>
      <span>Hello, {user.name}</span>

      {user.role === "participant" && (
        <button onClick={() => setShowMyFestivals(true)}>
          My Festivals
        </button>
      )}

      {user.role === "event_manager" && (
        <button onClick={() => setShowManageFestivals(true)}>
          Manage Festivals
        </button>
      )}

      <button onClick={handleLogout}>Logout</button>
    </>
  )}
</header>
```

### Logout pattern

```jsx
function handleLogout() {
  setUser(null);
  setShowMyFestivals(false);
  setShowManageFestivals(false);
}
```

If authentication uses a token or server session, logout must also clear/revoke it as appropriate. Prefer secure `HttpOnly` cookies for session tokens when supported by your backend. Avoid storing sensitive long-lived tokens in browser local storage unless the application's security design explicitly accepts that risk.

### Show login screen

Before the main `return`:

```jsx
if (showLogin) {
  return (
    <Login
      onLogin={handleLogin}
      onBack={() => setShowLogin(false)}
    />
  );
}
```

To support role-based pages cleanly, a later improvement is to use a router, but conditional rendering is sufficient for a small assessment app.

---

## 11. Join a festival as a participant

First confirm the real backend endpoint, such as whether it uses `POST /festivals/:id/join`, `POST /registrations`, or another route. Do not assume the route.

### Local UI prototype only

This sample stores joined festivals in React state; it will reset on refresh and is not persistent server-side membership.

```jsx
const [joinedFestivals, setJoinedFestivals] = useState([]);

function joinFestival(festival) {
  setJoinedFestivals((current) => {
    // Do not add a festival that is already in the list.
    if (current.some((item) => item.id === festival.id)) {
      return current;
    }

    return [...current, festival];
  });
}
```

Inside each festival card:

```jsx
{user?.role === "participant" && (
  <button
    onClick={() => joinFestival(festival)}
    disabled={joinedFestivals.some((item) => item.id === festival.id)}
  >
    {joinedFestivals.some((item) => item.id === festival.id)
      ? "Joined"
      : "Join Festival"}
  </button>
)}
```

The server must enforce duplicate-registration rules and associate the registration with the authenticated user.

### Real API pattern after confirming the endpoint

```jsx
async function joinFestival(festival) {
  // Example only. Replace the route with the documented API endpoint.
  await api.post(`/festivals/${festival.id}/join`);

  // Refresh the participant's joined-festival list from the server.
}
```

### My Festivals page

The real implementation should fetch the current user's joined festivals from the backend, rather than relying only on local state.

```jsx
{joinedFestivals.length === 0 ? (
  <p>You have not joined any festivals yet.</p>
) : (
  joinedFestivals.map((festival) => (
    <article key={festival.id}>
      <h2>{festival.name}</h2>
      <p>{festival.location}</p>
    </article>
  ))
)}
```

---

## 12. Event Manager features

Show manager-only controls when the server-confirmed user role is `event_manager`:

```jsx
{user?.role === "event_manager" && (
  <button onClick={() => setShowManageFestivals(true)}>
    Manage Festivals
  </button>
)}
```

Typical manager features:
- List festivals they are authorized to manage.
- Create a festival.
- Edit a festival.
- Delete a festival after confirmation.
- View registered participants.

Use the actual API routes and permission rules. The backend must verify the user's role and ownership/permissions for every protected operation.

A generic create request looks like this, but the endpoint and fields must match your API:

```jsx
await api.post("/festivals", {
  name,
  location,
  description,
  start_date,
});
```

---

## 13. API Explorer / Postman testing checklist

Test endpoints independently before wiring them into React.

For each endpoint, record:
- HTTP method (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`)
- Full URL
- Required headers
- Request body
- Expected status code
- Example response
- Whether authentication is required
- Which roles may call it

Recommended tests:

| Feature | Test |
|---|---|
| List festivals | `GET` festivals returns a list and pagination |
| Search | Search filters by name/location |
| Login | Valid credentials succeed; invalid credentials fail |
| Role | Participant and event manager receive correct roles |
| Join | Participant can join an eligible festival |
| Duplicate join | Joining twice is rejected or handled idempotently |
| My Festivals | Returns only the current participant's registrations |
| Create/update/delete | Only authorized managers can perform these actions |
| Logout | Session/token is invalidated or removed correctly |

---

## 14. Common React errors and quick fixes

### `festivals.map is not a function`
You probably stored the entire response object instead of the array.

```jsx
// If response is { data: [...], pagination: {...} }
setFestivals(response.data.data);
```

Inspect the actual response before choosing the correct property.

### `Cannot read properties of undefined`
A field or response property does not exist. Inspect:

```jsx
console.log(response.data);
```

Do not log passwords, tokens, or other secrets.

### `Functions are not valid as a React child`
You may have placed a function itself inside JSX. Use an explicit element:

```jsx
<option value="asc">Earliest Date First</option>
```

### Blank page or JSX error
Check the browser Console and terminal. Review the line number and ensure:
- JSX tags are closed.
- Each `.map()` element has a stable `key`.
- State hooks are inside the component function.
- Hooks are not called conditionally.

### Login button does nothing
If it is meant to show a separate screen, check that:
- `showLogin` state exists.
- The button calls `setShowLogin(true)`.
- The conditional login screen is before the main return.
- `Login.jsx` is imported correctly.

### Form reloads the page
Use:

```jsx
function handleSubmit(event) {
  event.preventDefault();
}
```

### Back button submits the form
Use:

```jsx
<button type="button" onClick={onBack}>Back</button>
```

---

## 15. Minimum working order for a time-limited assessment

Do these in order; test each one before adding the next:

1. **Run the project** — `npm run dev`.
2. **Fetch festivals** — confirm the list appears.
3. **Search** — filter by name/location.
4. **Sort** — use a dropdown with explicit options.
5. **Login UI** — show and hide a simple separate login page.
6. **Backend login** — use the documented endpoint.
7. **Role-based navigation** — participant versus event manager.
8. **Join Festival** — connect to the documented registration endpoint.
9. **My Festivals** — fetch the current user's registrations.
10. **Manager features** — only if time and API support allow.
11. **Test API calls** — verify method, URL, headers, body, and response.
12. **Git checkpoint** — commit the working version.

If time is short, prioritize a working festival list, one search control, login API integration, and role-specific navigation before optional styling.

---

## 16. Fast memory sheet

| Pattern | Meaning |
|---|---|
| `useState(value)` | Store data that can change |
| `useEffect(fn, [])` | Run an effect on mount |
| `onChange={(e) => setX(e.target.value)}` | Read input |
| `onSubmit={handleSubmit}` | Handle form submission |
| `event.preventDefault()` | Stop browser form reload |
| `api.get("/festivals")` | Send GET request |
| `api.post("/login", data)` | Send POST request |
| `response.data` | Read Axios response body |
| `array.map(...)` | Render a list |
| `array.filter(...)` | Filter a list |
| `[...array].sort(...)` | Sort a copy |
| `item.id` as `key` | Stable React list key |
| `user.role === "participant"` | Conditional participant UI |
| `git add . && git commit ...` | Save a Git checkpoint |

**Golden rule:** Make one small change, save, run, test, and commit. Do not replace the entire `App.jsx` each time. Confirm the backend API contract before connecting authentication or registration.
