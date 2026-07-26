# 08 - Building the PulseVote RBAC Frontend

## Overview

In Activity 07, you completed the backend role-based access control workflow for PulseVote. The backend supports administrators, managers, normal users, organisations, join codes, polls, voting, results, and opening or closing polls.

In this activity, you will complete the React frontend and connect it to the Activity 07 backend.

By the end of the activity:
- An administrator can create manager accounts.
- A manager can create an organisation and receive a join code.
- A manager can create, open, and close polls.
- A user can join an organisation using a join code.
- A user can vote once in each open poll.
- Organisation members can view poll results.
- The frontend reads roles from the JWT and displays the relevant dashboard.
- Every protected Axios request automatically includes the JWT.
- Organisation access is still enforced by the backend.

> Conditional rendering is not security. React only adjusts the interface. Express must still reject requests that the user is not authorised to perform.

## Research

Before coding, research:
- Axios instances and interceptors
- Bearer authentication
- JWT payloads
- Conditional rendering in React
- Role-based access control
- The difference between decoding and verifying a JWT

## Before You Start
Confirm that:
- Activity 07 is working.
- MongoDB is running.
- The backend runs on `https://localhost:5000`.
- The frontend runs on `https://localhost:5173`.
- The browser trusts or has accepted both local development certificates.
- The Activity 07 Postman workflow works.
- You have seeded your database using the postman script provided in Activity 07.

> Do not create a public first-admin form in React.

# Part A - Complete the Backend Support

The Activity 07 API already contains most of the required behaviour. However, the frontend also needs to retrieve the organisations available to the logged-in account. Poll listing and poll status changes must also be checked against the correct organisation.

## 1. Add a JWT Helper

Create: `pulsevote-backend/utils/generateToken.js`

```js
const jwt = require("jsonwebtoken");

function generateToken(user) {
  return jwt.sign(
    { id: user._id, email: user.email, roles: user.roles },
    process.env.JWT_SECRET,
    { expiresIn: "1h" }
  );
}

module.exports = generateToken;
```
This helper keeps token creation consistent. The token contains the user ID, email address, and roles.

## 2. Update the Authentication Controller

Replace: `pulsevote-backend/controllers/authController.js`

```js
const User = require("../models/User");
const { validationResult } = require("express-validator");
const generateToken = require("../utils/generateToken");

exports.registerUser = async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ message: "Invalid input", errors: errors.array() });
  }

  const { email, password } = req.body;

  try {
    const existing = await User.findOne({ email });
    if (existing) return res.status(400).json({ message: "Email already exists" });

    const user = await User.create({
      email,
      password,
      roles: [{ organisationId: null, role: "user" }]
    });

    return res.status(201).json({
      message: "User registered",
      token: generateToken(user)
    });
  } catch (err) {
    return res.status(500).json({ error: "Server error" });
  }
};

exports.registerManager = async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ message: "Invalid input", errors: errors.array() });
  }

  try {
    const { email, password } = req.body;
    const existing = await User.findOne({ email });
    if (existing) return res.status(400).json({ message: "Email already exists" });

    const managerUser = await User.create({
      email,
      password,
      roles: [{ organisationId: null, role: "manager" }]
    });

    // The token belongs to the new manager. The frontend must not replace
    // the currently logged-in admin token with this token.
    return res.status(201).json({
      message: "Manager registered",
      token: generateToken(managerUser)
    });
  } catch (err) {
    return res.status(500).json({ error: "Server error" });
  }
};

exports.registerAdmin = async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ message: "Invalid input", errors: errors.array() });
  }

  try {
    const { email, password } = req.body;
    const adminExists = await User.exists({ "roles.role": "admin" });

    // Activity 08 keeps first-admin creation as a Postman-only bootstrap step.
    // Once an admin exists, this public bootstrap endpoint is disabled.
    if (adminExists) {
      return res.status(403).json({ message: "The first admin has already been created" });
    }

    const existing = await User.findOne({ email });
    if (existing) return res.status(400).json({ message: "Email already exists" });

    const adminUser = await User.create({
      email,
      password,
      roles: [{ organisationId: null, role: "admin" }]
    });

    return res.status(201).json({
      message: "Admin registered",
      token: generateToken(adminUser)
    });
  } catch (err) {
    return res.status(500).json({ error: "Server error" });
  }
};

exports.login = async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ message: "Invalid input", errors: errors.array() });
  }

  const { email, password } = req.body;

  try {
    const user = await User.findOne({ email });
    if (!user || !(await user.comparePassword(password))) {
      return res.status(400).json({ message: "Invalid credentials" });
    }

    return res.json({ token: generateToken(user) });
  } catch (err) {
    return res.status(500).json({ error: "Server error" });
  }
};
```
Important changes:

- Normal users still register through `/register-user`.
- Administrators create managers through `/register-manager`.
- The manager token returned by registration belongs to the new manager and must not replace the administrator token.
- The first-admin endpoint is disabled after the first administrator exists.

## 3. Update the Organisation Controller

Replace: `pulsevote-backend/controllers/organisationController.js`

```js
const Organisation = require("../models/Organisation");
const User = require("../models/User");
const generateToken = require("../utils/generateToken");

exports.createOrganisation = async (req, res) => {
  try {
    const name = req.body.name?.trim();
    if (!name) return res.status(400).json({ message: "Organisation name is required" });

    const existing = await Organisation.findOne({ name });
    if (existing) return res.status(400).json({ message: "Organisation name already exists" });

    const org = new Organisation({
      name,
      createdBy: req.user.id,
      members: [req.user.id]
    });
    org.generateJoinCode();
    await org.save();

    const user = await User.findById(req.user.id);
    const alreadyManager = user.roles.some(
      (role) => role.role === "manager" && role.organisationId?.toString() === org._id.toString()
    );

    if (!alreadyManager) {
      user.roles.push({ organisationId: org._id, role: "manager" });
      await user.save();
    }

    return res.status(201).json({
      message: "Organisation created",
      organisation: org,
      token: generateToken(user)
    });
  } catch (err) {
    return res.status(500).json({ error: "Server error" });
  }
};

exports.getMyOrganisations = async (req, res) => {
  try {
    const user = await User.findById(req.user.id).lean();
    if (!user) return res.status(401).json({ message: "User not found" });

    const isAdmin = user.roles.some((role) => role.role === "admin");
    const organisationIds = user.roles
      .filter((role) => role.organisationId)
      .map((role) => role.organisationId);

    const query = isAdmin ? {} : { _id: { $in: organisationIds } };
    const organisations = await Organisation.find(query).sort({ name: 1 }).lean();

    return res.json(organisations);
  } catch (err) {
    return res.status(500).json({ error: "Server error" });
  }
};

exports.generateJoinCode = async (req, res) => {
  try {
    const { organisationId } = req.params;
    const org = await Organisation.findById(organisationId);
    if (!org) return res.status(404).json({ message: "Organisation not found" });

    org.generateJoinCode();
    await org.save();

    return res.json({ message: "Join code regenerated", joinCode: org.joinCode });
  } catch (err) {
    return res.status(500).json({ error: "Server error" });
  }
};

exports.joinOrganisation = async (req, res) => {
  try {
    const joinCode = req.body.joinCode?.trim();
    if (!joinCode) return res.status(400).json({ message: "Join code is required" });

    const org = await Organisation.findOne({ joinCode });
    if (!org) return res.status(404).json({ message: "Invalid join code" });

    const user = await User.findById(req.user.id);
    const alreadyJoined = user.roles.some(
      (role) => role.role === "user" && role.organisationId?.toString() === org._id.toString()
    );

    if (alreadyJoined) {
      return res.status(409).json({ message: "You have already joined this organisation" });
    }

    if (!org.members.some((memberId) => memberId.toString() === req.user.id)) {
      org.members.push(req.user.id);
      await org.save();
    }

    user.roles.push({ organisationId: org._id, role: "user" });
    await user.save();

    return res.json({
      message: "Joined organisation",
      organisation: org,
      token: generateToken(user)
    });
  } catch (err) {
    return res.status(500).json({ error: "Server error" });
  }
};
```
The controller now:

- Returns a refreshed JWT after creating an organisation.
- Returns a refreshed JWT after joining an organisation.
- Provides `getMyOrganisations` so the frontend can retrieve organisation names, IDs, and join codes after later logins.
- Rejects duplicate organisation membership.

The refreshed JWT means the user does not need to log out and log in again after the role array changes.

## 4. Update the Organisation Routes

Replace: `pulsevote-backend/routes/organisationRoutes.js`

```js
const express = require("express");
const { protect } = require("../middleware/authMiddleware");
const { requireRole } = require("../middleware/roleMiddleware");
const {
  createOrganisation,
  getMyOrganisations,
  generateJoinCode,
  joinOrganisation
} = require("../controllers/organisationController");

const router = express.Router();

router.get("/my-organisations", protect, getMyOrganisations);
router.post("/create-organisation", protect, requireRole("manager"), createOrganisation);
router.post("/generate-join-code/:organisationId", protect, requireRole("manager"), generateJoinCode);
router.post("/join-organisation", protect, joinOrganisation);

module.exports = router;
```
The new endpoint is:

```text
GET /api/organisations/my-organisations
```

It returns organisations associated with the authenticated account.

## 5. Update the Poll Controller

Replace: `pulsevote-backend/controllers/pollController.js`

```js
const Poll = require("../models/Poll");
const Organisation = require("../models/Organisation");
const User = require("../models/User");

function hasOrganisationRole(user, organisationId, roleName) {
  return user.roles.some(
    (role) => role.role === roleName && role.organisationId?.toString() === organisationId.toString()
  );
}

exports.createPoll = async (req, res) => {
  try {
    const { organisationId } = req.body;
    const question = req.body.question?.trim();
    const options = Array.isArray(req.body.options)
      ? req.body.options.map((option) => String(option).trim()).filter(Boolean)
      : [];

    if (!question) return res.status(400).json({ message: "Poll question is required" });
    if (options.length < 2) {
      return res.status(400).json({ message: "A poll must have at least two options" });
    }
    if (new Set(options.map((option) => option.toLowerCase())).size !== options.length) {
      return res.status(400).json({ message: "Poll options must be unique" });
    }

    const org = await Organisation.findById(organisationId);
    if (!org) return res.status(404).json({ message: "Organisation not found" });

    const poll = await Poll.create({
      organisationId,
      question,
      options,
      createdBy: req.user.id
    });

    return res.status(201).json({ message: "Poll created", poll });
  } catch (err) {
    return res.status(500).json({ error: "Server error" });
  }
};

exports.votePoll = async (req, res) => {
  try {
    const { pollId } = req.params;
    const { optionIndex } = req.body;

    const poll = await Poll.findById(pollId);
    if (!poll) return res.status(404).json({ message: "Poll not found" });
    if (poll.status !== "open") return res.status(400).json({ message: "Poll is closed" });

    const user = await User.findById(req.user.id).lean();
    if (!hasOrganisationRole(user, poll.organisationId, "user")) {
      return res.status(403).json({ message: "Only organisation users may vote in this poll" });
    }

    if (!Number.isInteger(optionIndex) || optionIndex < 0 || optionIndex >= poll.options.length) {
      return res.status(400).json({ message: "Invalid poll option" });
    }

    const alreadyVoted = poll.votes.some((vote) => vote.userId.toString() === req.user.id);
    if (alreadyVoted) return res.status(409).json({ message: "You have already voted" });

    poll.votes.push({ userId: req.user.id, optionIndex });
    await poll.save();

    return res.json({ message: "Vote recorded", poll });
  } catch (err) {
    return res.status(500).json({ error: "Server error" });
  }
};

exports.getPollResults = async (req, res) => {
  try {
    const { pollId } = req.params;
    const poll = await Poll.findById(pollId).lean();
    if (!poll) return res.status(404).json({ message: "Poll not found" });

    const user = await User.findById(req.user.id).lean();
    const isAdmin = user.roles.some((role) => role.role === "admin");
    const isMember = user.roles.some(
      (role) => role.organisationId?.toString() === poll.organisationId.toString()
    );

    if (!isAdmin && !isMember) {
      return res.status(403).json({ message: "Not a member of this organisation" });
    }

    const counts = Array(poll.options.length).fill(0);
    for (const vote of poll.votes || []) {
      if (Number.isInteger(vote.optionIndex) && vote.optionIndex >= 0 && vote.optionIndex < counts.length) {
        counts[vote.optionIndex] += 1;
      }
    }

    const totalVotes = counts.reduce((sum, count) => sum + count, 0);
    const percentages = counts.map((count) =>
      totalVotes ? Number(((count / totalVotes) * 100).toFixed(2)) : 0
    );
    const userVote = poll.votes?.find((vote) => vote.userId?.toString() === req.user.id);

    return res.json({
      poll: {
        _id: poll._id,
        organisationId: poll.organisationId,
        question: poll.question,
        options: poll.options,
        status: poll.status
      },
      results: {
        counts,
        percentages,
        totalVotes,
        userVoteIndex: userVote ? userVote.optionIndex : null
      }
    });
  } catch (err) {
    return res.status(500).json({ error: "Server error" });
  }
};

exports.getOrgPolls = async (req, res) => {
  try {
    const { organisationId } = req.params;
    const user = await User.findById(req.user.id).lean();
    const isAdmin = user.roles.some((role) => role.role === "admin");
    const isMember = user.roles.some(
      (role) => role.organisationId?.toString() === organisationId
    );

    if (!isAdmin && !isMember) {
      return res.status(403).json({ message: "Not a member of this organisation" });
    }

    const polls = await Poll.find({ organisationId }).sort({ _id: -1 });
    return res.json(polls);
  } catch (err) {
    return res.status(500).json({ error: "Server error" });
  }
};

async function setPollStatus(req, res, status) {
  try {
    const poll = await Poll.findById(req.params.pollId);
    if (!poll) return res.status(404).json({ message: "Poll not found" });

    const user = await User.findById(req.user.id).lean();
    const isAdmin = user.roles.some((role) => role.role === "admin");
    const isManager = hasOrganisationRole(user, poll.organisationId, "manager");

    if (!isAdmin && !isManager) {
      return res.status(403).json({ message: "Not a manager of this organisation" });
    }

    poll.status = status;
    await poll.save();
    return res.json({ message: `Poll ${status}`, poll });
  } catch (err) {
    return res.status(500).json({ error: "Server error" });
  }
}

exports.closePoll = (req, res) => setPollStatus(req, res, "closed");
exports.openPoll = (req, res) => setPollStatus(req, res, "open");
```
The updated controller adds these checks:

- Poll options must contain at least two non-empty, unique choices.
- A user must belong to the poll's organisation before voting.
- The selected option index must be valid.
- A user may vote only once.
- A user must belong to an organisation before loading its polls.
- A manager may open or close only polls belonging to an organisation they manage.

## 6. Update the Poll Routes

Replace: `pulsevote-backend/routes/pollRoutes.js`

```js
const express = require("express");
const { protect } = require("../middleware/authMiddleware");
const { requireRole } = require("../middleware/roleMiddleware");
const {
  createPoll,
  votePoll,
  getPollResults,
  getOrgPolls,
  closePoll,
  openPoll
} = require("../controllers/pollController");

const router = express.Router();

router.post("/create-poll", protect, requireRole("manager"), createPoll);
router.post("/vote/:pollId", protect, requireRole("user"), votePoll);
router.get("/get-poll-results/:pollId", protect, getPollResults);
router.get("/get-polls/:organisationId", protect, getOrgPolls);
router.post("/close/:pollId", protect, closePoll);
router.post("/open/:pollId", protect, openPoll);

module.exports = router;
```
The controller performs the organisation-specific check for opening and closing polls because the organisation ID is obtained from the poll itself.

## 7. Correct the CSP Connection Source

Open: `pulsevote-backend/app.js`

Find:

```js
connectSrc: ["'self'", "http://localhost:5000"]
```

Replace it with:

```js
connectSrc: ["'self'", "https://localhost:5000"]
```

The backend is using HTTPS, so the CSP configuration should use the same protocol.

## 8. Test the Backend

Start the backend: `npm run dev`

Test the following before working on React:

1. Log in as administrator.
2. Create a manager.
3. Log in as manager.
4. Create an organisation.
5. Confirm that the response contains `organisation` and a refreshed `token`.
6. Call `GET /api/organisations/my-organisations` with the manager token.
7. Register and log in as a normal user.
8. Join the organisation.
9. Confirm that the join response contains a refreshed `token`.
10. Call `GET /api/organisations/my-organisations` with the refreshed user token.

Commit your changes if all works fine.

# Part B - Build the Frontend

## 9. Create the Frontend Structure

Inside `pulsevote-frontend/src`, create this structure:

```text
src/
├── api/
│   └── api.js
├── components/
│   ├── AdminDashboard.jsx
│   ├── Layout.jsx
│   ├── Login.jsx
│   ├── ManagerDashboard.jsx
│   ├── OrganisationSelector.jsx
│   ├── PollCard.jsx
│   ├── ProtectedRoute.jsx
│   ├── Register.jsx
│   └── UserDashboard.jsx
├── pages/
│   ├── DashboardPage.jsx
│   ├── HomePage.jsx
│   ├── LoginPage.jsx
│   ├── LogoutPage.jsx
│   └── RegisterPage.jsx
├── utils/
│   ├── auth.js
│   └── messages.js
├── App.css
├── App.jsx
└── main.jsx
```

No additional npm package is required. Axios and React Router are already installed from the previous activities.

## 10. Create the Axios API Client

Create: `pulsevote-frontend/src/api/api.js`

```js
import axios from "axios";

const api = axios.create({
  baseURL: "https://localhost:5000/api",
  headers: { "Content-Type": "application/json" }
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

export default api;
```
The interceptor runs before each request. When a token exists, it adds:

```text
Authorization: Bearer <token>
```

Do not repeat the backend URL or token logic in every component.

## 11. Add Authentication Helpers

Create: `pulsevote-frontend/src/utils/auth.js`

```js
export function getToken() {
  return localStorage.getItem("token");
}

export function saveToken(token) {
  localStorage.setItem("token", token);
}

function decodeToken(token) {
  const payload = token.split(".")[1];
  const base64 = payload.replace(/-/g, "+").replace(/_/g, "/");
  const json = decodeURIComponent(
    window.atob(base64)
      .split("")
      .map((character) => `%${(`00${character.charCodeAt(0).toString(16)}`).slice(-2)}`)
      .join("")
  );
  return JSON.parse(json);
}

export function getCurrentUser() {
  const token = getToken();
  if (!token) return null;

  try {
    const user = decodeToken(token);
    if (user.exp && user.exp * 1000 < Date.now()) {
      logout();
      return null;
    }
    return user;
  } catch {
    logout();
    return null;
  }
}

export function getRoles() {
  return getCurrentUser()?.roles ?? [];
}

export function hasRole(roleName) {
  return getRoles().some((role) => role.role === roleName);
}

export function logout() {
  localStorage.removeItem("token");
}
```
This code decodes the payload so React can display the appropriate interface. It does not verify the JWT signature. Signature verification remains a backend responsibility.

## 12. Add a Reusable Error Helper

Create: `pulsevote-frontend/src/utils/messages.js`

```js
export function getErrorMessage(error) {
  const validationMessage = error.response?.data?.errors?.[0]?.msg;
  return validationMessage ||
    error.response?.data?.message ||
    error.response?.data?.error ||
    "The request could not be completed.";
}
```
This helper supports both ordinary API errors and validation errors returned by `express-validator`.

## 13. Replace the Login Component

Replace: `pulsevote-frontend/src/components/Login.jsx`

```jsx
import { useState } from "react";
import { useNavigate } from "react-router-dom";
import api from "../api/api";
import { saveToken } from "../utils/auth";
import { getErrorMessage } from "../utils/messages";

export default function Login() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [showPassword, setShowPassword] = useState(false);
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);
  const navigate = useNavigate();

  async function handleSubmit(event) {
    event.preventDefault();
    setError("");
    setLoading(true);

    try {
      const response = await api.post("/auth/login", { email, password });
      saveToken(response.data.token);
      navigate("/dashboard");
    } catch (requestError) {
      setError(getErrorMessage(requestError));
    } finally {
      setLoading(false);
    }
  }

  return (
    <form onSubmit={handleSubmit} className="form-stack">
      {error && <div className="error-message">{error}</div>}
      <label>
        Email
        <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
      </label>
      <label>
        Password
        <input type={showPassword ? "text" : "password"} value={password} onChange={(e) => setPassword(e.target.value)} required />
      </label>
      <label className="inline-control">
        <input type="checkbox" checked={showPassword} onChange={(e) => setShowPassword(e.target.checked)} />
        Show password
      </label>
      <button disabled={loading}>{loading ? "Logging in..." : "Login"}</button>
    </form>
  );
}
```
The component now uses the shared Axios instance and sends the user directly to the dashboard after login.

## 14. Replace the Registration Component

Replace: `pulsevote-frontend/src/components/Register.jsx`

```jsx
import { useState } from "react";
import { useNavigate } from "react-router-dom";
import api from "../api/api";
import { saveToken } from "../utils/auth";
import { getErrorMessage } from "../utils/messages";

export default function Register() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [confirmPassword, setConfirmPassword] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);
  const navigate = useNavigate();

  async function handleSubmit(event) {
    event.preventDefault();
    setError("");

    if (password !== confirmPassword) {
      setError("Passwords do not match.");
      return;
    }

    setLoading(true);
    try {
      const response = await api.post("/auth/register-user", { email, password });
      saveToken(response.data.token);
      navigate("/dashboard");
    } catch (requestError) {
      setError(getErrorMessage(requestError));
    } finally {
      setLoading(false);
    }
  }

  return (
    <form onSubmit={handleSubmit} className="form-stack">
      {error && <div className="error-message">{error}</div>}
      <label>Email<input type="email" value={email} onChange={(e) => setEmail(e.target.value)} required /></label>
      <label>Password<input type="password" value={password} onChange={(e) => setPassword(e.target.value)} required /></label>
      <label>Confirm password<input type="password" value={confirmPassword} onChange={(e) => setConfirmPassword(e.target.value)} required /></label>
      <button disabled={loading}>{loading ? "Registering..." : "Register"}</button>
    </form>
  );
}
```
The important endpoint is:

```text
POST /api/auth/register-user
```

Do not use the old `/api/auth/register` endpoint.

## 15. Replace the Protected Route

Replace: `pulsevote-frontend/src/components/ProtectedRoute.jsx`

```jsx
import { Navigate } from "react-router-dom";
import { getCurrentUser } from "../utils/auth";

export default function ProtectedRoute({ children }) {
  return getCurrentUser() ? children : <Navigate to="/login" replace />;
}
```
An absent, invalid, or expired token sends the visitor to the login page.

## 16. Update the Main Layout

Replace: `pulsevote-frontend/src/components/Layout.jsx`

```jsx
import { Outlet, NavLink, useLocation } from "react-router-dom";
import { getCurrentUser } from "../utils/auth";

export default function Layout() {
  useLocation();
  const loggedIn = Boolean(getCurrentUser());

  return (
    <div className="app-shell">
      <header>
        <h1>PulseVote</h1>
        <nav>
          <NavLink to="/">Home</NavLink>
          {!loggedIn && <NavLink to="/login">Login</NavLink>}
          {!loggedIn && <NavLink to="/register">Register</NavLink>}
          {loggedIn && <NavLink to="/dashboard">Dashboard</NavLink>}
          {loggedIn && <NavLink to="/logout">Logout</NavLink>}
        </nav>
      </header>
      <main><Outlet /></main>
    </div>
  );
}
```
This removes the undefined `logout()` call from the old layout. Logout is handled by `LogoutPage.jsx`.

## 17. Update the Logout Page

Replace: `pulsevote-frontend/src/pages/LogoutPage.jsx`

```jsx
import { useEffect } from "react";
import { useNavigate } from "react-router-dom";
import { logout } from "../utils/auth";

export default function LogoutPage() {
  const navigate = useNavigate();

  useEffect(() => {
    logout();
    navigate("/", { replace: true });
  }, [navigate]);

  return <div className="card"><p>Logging out...</p></div>;
}
```
## 18. Create the Organisation Selector

Create: `pulsevote-frontend/src/components/OrganisationSelector.jsx`

```jsx
export default function OrganisationSelector({ organisations, selectedId, onChange }) {
  if (!organisations.length) return null;

  return (
    <label>
      Organisation
      <select value={selectedId} onChange={(event) => onChange(event.target.value)}>
        {organisations.map((organisation) => (
          <option key={organisation._id} value={organisation._id}>{organisation.name}</option>
        ))}
      </select>
    </label>
  );
}
```
An account may belong to more than one organisation. The selector allows the manager or user to choose which organisation is currently active.

## 19. Build the Administrator Dashboard

Create: `pulsevote-frontend/src/components/AdminDashboard.jsx`

```jsx
import { useState } from "react";
import api from "../api/api";
import { getErrorMessage } from "../utils/messages";

export default function AdminDashboard() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [confirmPassword, setConfirmPassword] = useState("");
  const [message, setMessage] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);

  async function handleSubmit(event) {
    event.preventDefault();
    setMessage("");
    setError("");

    if (password !== confirmPassword) {
      setError("Passwords do not match.");
      return;
    }

    setLoading(true);
    try {
      await api.post("/auth/register-manager", { email, password });
      setMessage("Manager account created. The admin remains logged in.");
      setEmail("");
      setPassword("");
      setConfirmPassword("");
    } catch (requestError) {
      setError(getErrorMessage(requestError));
    } finally {
      setLoading(false);
    }
  }

  return (
    <section className="panel">
      <h3>Admin: Create Manager</h3>
      {message && <div className="success-message">{message}</div>}
      {error && <div className="error-message">{error}</div>}
      <form onSubmit={handleSubmit} className="form-stack">
        <label>Manager email<input type="email" value={email} onChange={(e) => setEmail(e.target.value)} required /></label>
        <label>Password<input type="password" value={password} onChange={(e) => setPassword(e.target.value)} required /></label>
        <label>Confirm password<input type="password" value={confirmPassword} onChange={(e) => setConfirmPassword(e.target.value)} required /></label>
        <button disabled={loading}>{loading ? "Creating..." : "Create Manager"}</button>
      </form>
    </section>
  );
}
```
Notice that the response token is deliberately ignored. It belongs to the newly created manager. Saving it would log the administrator in as the manager.

## 20. Build the Reusable Poll Card

Create: `pulsevote-frontend/src/components/PollCard.jsx`

```jsx
import { useEffect, useState } from "react";
import api from "../api/api";
import { getErrorMessage } from "../utils/messages";

export default function PollCard({ poll, canManage, canVote, onPollChanged }) {
  const [selectedOptionIndex, setSelectedOptionIndex] = useState("");
  const [results, setResults] = useState(null);
  const [message, setMessage] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);

  async function loadResults() {
    try {
      const response = await api.get(`/polls/get-poll-results/${poll._id}`);
      setResults(response.data.results);
    } catch (requestError) {
      setError(getErrorMessage(requestError));
    }
  }

  useEffect(() => {
    loadResults();
  }, [poll._id]);

  async function vote(event) {
    event.preventDefault();
    if (selectedOptionIndex === "") {
      setError("Select an option before voting.");
      return;
    }

    setLoading(true);
    setError("");
    setMessage("");
    try {
      await api.post(`/polls/vote/${poll._id}`, { optionIndex: Number(selectedOptionIndex) });
      setMessage("Vote recorded.");
      await loadResults();
    } catch (requestError) {
      setError(getErrorMessage(requestError));
    } finally {
      setLoading(false);
    }
  }

  async function changeStatus() {
    const action = poll.status === "open" ? "close" : "open";
    setLoading(true);
    setError("");
    try {
      await api.post(`/polls/${action}/${poll._id}`);
      await onPollChanged();
    } catch (requestError) {
      setError(getErrorMessage(requestError));
    } finally {
      setLoading(false);
    }
  }

  const alreadyVoted = results?.userVoteIndex !== null && results?.userVoteIndex !== undefined;

  return (
    <article className="poll-card">
      <div className="poll-heading">
        <h4>{poll.question}</h4>
        <span className={`status ${poll.status}`}>{poll.status}</span>
      </div>
      {message && <div className="success-message">{message}</div>}
      {error && <div className="error-message">{error}</div>}

      {canVote && poll.status === "open" && !alreadyVoted && (
        <form onSubmit={vote} className="option-list">
          {poll.options.map((option, index) => (
            <label key={`${poll._id}-${index}`} className="option-row">
              <input type="radio" name={`poll-${poll._id}`} value={index} checked={Number(selectedOptionIndex) === index} onChange={(e) => setSelectedOptionIndex(e.target.value)} />
              {option}
            </label>
          ))}
          <button disabled={loading}>{loading ? "Submitting..." : "Vote"}</button>
        </form>
      )}

      {canVote && poll.status === "closed" && <p>This poll is closed.</p>}
      {alreadyVoted && <p>Your vote has already been recorded.</p>}

      {results && (
        <div className="results">
          <strong>Total votes: {results.totalVotes}</strong>
          {poll.options.map((option, index) => (
            <div className="result-row" key={`${poll._id}-result-${index}`}>
              <span>{option}{results.userVoteIndex === index ? " (your vote)" : ""}</span>
              <span>{results.counts[index]} votes — {results.percentages[index]}%</span>
            </div>
          ))}
        </div>
      )}

      {canManage && (
        <button className="secondary" onClick={changeStatus} disabled={loading}>
          {poll.status === "open" ? "Close Poll" : "Open Poll"}
        </button>
      )}
    </article>
  );
}
```
Each poll card owns its selected radio option. This prevents selections in one poll from changing another poll.

The component can operate in two modes:
- `canVote`: show voting controls to an organisation user.
- `canManage`: show opening and closing controls to an organisation manager.

Results are loaded for both managers and users.

## 21. Build the Manager Dashboard

Create: `pulsevote-frontend/src/components/ManagerDashboard.jsx`

```jsx
import { useEffect, useState } from "react";
import api from "../api/api";
import { saveToken } from "../utils/auth";
import { getErrorMessage } from "../utils/messages";
import OrganisationSelector from "./OrganisationSelector";
import PollCard from "./PollCard";

export default function ManagerDashboard() {
  const [organisations, setOrganisations] = useState([]);
  const [selectedId, setSelectedId] = useState("");
  const [polls, setPolls] = useState([]);
  const [organisationName, setOrganisationName] = useState("");
  const [question, setQuestion] = useState("");
  const [options, setOptions] = useState(["", ""]);
  const [message, setMessage] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);

  const selectedOrganisation = organisations.find((organisation) => organisation._id === selectedId);

  async function loadOrganisations(preferredId) {
    const response = await api.get("/organisations/my-organisations");
    setOrganisations(response.data);
    const nextId = preferredId || selectedId || response.data[0]?._id || "";
    setSelectedId(nextId);
  }

  async function loadPolls() {
    if (!selectedId) {
      setPolls([]);
      return;
    }
    const response = await api.get(`/polls/get-polls/${selectedId}`);
    setPolls(response.data);
  }

  useEffect(() => {
    loadOrganisations().catch((requestError) => setError(getErrorMessage(requestError)));
  }, []);

  useEffect(() => {
    loadPolls().catch((requestError) => setError(getErrorMessage(requestError)));
  }, [selectedId]);

  async function createOrganisation(event) {
    event.preventDefault();
    setLoading(true);
    setMessage("");
    setError("");
    try {
      const response = await api.post("/organisations/create-organisation", { name: organisationName });
      saveToken(response.data.token);
      setOrganisationName("");
      setMessage("Organisation created and access token refreshed.");
      await loadOrganisations(response.data.organisation._id);
    } catch (requestError) {
      setError(getErrorMessage(requestError));
    } finally {
      setLoading(false);
    }
  }

  async function regenerateJoinCode() {
    setLoading(true);
    setMessage("");
    setError("");
    try {
      await api.post(`/organisations/generate-join-code/${selectedId}`);
      setMessage("A new join code was generated. The old code no longer works.");
      await loadOrganisations(selectedId);
    } catch (requestError) {
      setError(getErrorMessage(requestError));
    } finally {
      setLoading(false);
    }
  }

  function updateOption(index, value) {
    setOptions((current) => current.map((option, optionIndex) => optionIndex === index ? value : option));
  }

  async function createPoll(event) {
    event.preventDefault();
    setLoading(true);
    setMessage("");
    setError("");
    try {
      const cleanedOptions = options.map((option) => option.trim()).filter(Boolean);
      await api.post("/polls/create-poll", { organisationId: selectedId, question, options: cleanedOptions });
      setQuestion("");
      setOptions(["", ""]);
      setMessage("Poll created.");
      await loadPolls();
    } catch (requestError) {
      setError(getErrorMessage(requestError));
    } finally {
      setLoading(false);
    }
  }

  return (
    <section className="panel">
      <h3>Manager Dashboard</h3>
      {message && <div className="success-message">{message}</div>}
      {error && <div className="error-message">{error}</div>}

      <form onSubmit={createOrganisation} className="form-stack compact-form">
        <h4>Create Organisation</h4>
        <label>Name<input value={organisationName} onChange={(e) => setOrganisationName(e.target.value)} required /></label>
        <button disabled={loading}>Create Organisation</button>
      </form>

      <OrganisationSelector organisations={organisations} selectedId={selectedId} onChange={setSelectedId} />

      {selectedOrganisation && (
        <div className="organisation-summary">
          <p><strong>ID:</strong> {selectedOrganisation._id}</p>
          <p><strong>Join code:</strong> <code>{selectedOrganisation.joinCode}</code></p>
          <button className="secondary" onClick={regenerateJoinCode} disabled={loading}>Generate New Join Code</button>
        </div>
      )}

      {selectedId && (
        <form onSubmit={createPoll} className="form-stack compact-form">
          <h4>Create Poll</h4>
          <label>Question<input value={question} onChange={(e) => setQuestion(e.target.value)} required /></label>
          {options.map((option, index) => (
            <div className="option-editor" key={index}>
              <input value={option} onChange={(e) => updateOption(index, e.target.value)} placeholder={`Option ${index + 1}`} required={index < 2} />
              {options.length > 2 && <button type="button" className="danger" onClick={() => setOptions(options.filter((_, optionIndex) => optionIndex !== index))}>Remove</button>}
            </div>
          ))}
          <button type="button" className="secondary" onClick={() => setOptions([...options, ""])}>Add Option</button>
          <button disabled={loading}>Create Poll</button>
        </form>
      )}

      <div className="poll-list">
        {polls.map((poll) => <PollCard key={poll._id} poll={poll} canManage canVote={false} onPollChanged={loadPolls} />)}
      </div>
    </section>
  );
}
```
The manager dashboard supports:

- Creating an organisation
- Saving the refreshed JWT
- Loading the manager's organisations
- Selecting an organisation
- Viewing and regenerating its join code
- Creating polls with two or more options
- Loading organisation polls
- Opening and closing polls

## 22. Build the User Dashboard

Create: `pulsevote-frontend/src/components/UserDashboard.jsx`

```jsx
import { useEffect, useState } from "react";
import api from "../api/api";
import { saveToken } from "../utils/auth";
import { getErrorMessage } from "../utils/messages";
import OrganisationSelector from "./OrganisationSelector";
import PollCard from "./PollCard";

export default function UserDashboard() {
  const [joinCode, setJoinCode] = useState("");
  const [organisations, setOrganisations] = useState([]);
  const [selectedId, setSelectedId] = useState("");
  const [polls, setPolls] = useState([]);
  const [message, setMessage] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);

  async function loadOrganisations(preferredId) {
    const response = await api.get("/organisations/my-organisations");
    setOrganisations(response.data);
    setSelectedId(preferredId || selectedId || response.data[0]?._id || "");
  }

  async function loadPolls() {
    if (!selectedId) {
      setPolls([]);
      return;
    }
    const response = await api.get(`/polls/get-polls/${selectedId}`);
    setPolls(response.data);
  }

  useEffect(() => {
    loadOrganisations().catch((requestError) => setError(getErrorMessage(requestError)));
  }, []);

  useEffect(() => {
    loadPolls().catch((requestError) => setError(getErrorMessage(requestError)));
  }, [selectedId]);

  async function joinOrganisation(event) {
    event.preventDefault();
    setLoading(true);
    setMessage("");
    setError("");
    try {
      const response = await api.post("/organisations/join-organisation", { joinCode });
      saveToken(response.data.token);
      setJoinCode("");
      setMessage(`Joined ${response.data.organisation.name}. Access token refreshed.`);
      await loadOrganisations(response.data.organisation._id);
    } catch (requestError) {
      setError(getErrorMessage(requestError));
    } finally {
      setLoading(false);
    }
  }

  return (
    <section className="panel">
      <h3>User Dashboard</h3>
      {message && <div className="success-message">{message}</div>}
      {error && <div className="error-message">{error}</div>}

      <form onSubmit={joinOrganisation} className="form-stack compact-form">
        <h4>Join Organisation</h4>
        <label>Join code<input value={joinCode} onChange={(e) => setJoinCode(e.target.value)} required /></label>
        <button disabled={loading}>{loading ? "Joining..." : "Join"}</button>
      </form>

      <OrganisationSelector organisations={organisations} selectedId={selectedId} onChange={setSelectedId} />

      {!organisations.length && <p>Join an organisation to view its polls.</p>}
      <div className="poll-list">
        {polls.map((poll) => <PollCard key={poll._id} poll={poll} canManage={false} canVote onPollChanged={loadPolls} />)}
      </div>
    </section>
  );
}
```
The user dashboard supports:

- Joining an organisation with a code
- Saving the refreshed JWT
- Loading joined organisations
- Selecting an organisation
- Viewing polls
- Voting once
- Viewing results

## 23. Replace the Dashboard Page

Replace: `pulsevote-frontend/src/pages/DashboardPage.jsx`

```jsx
import AdminDashboard from "../components/AdminDashboard";
import ManagerDashboard from "../components/ManagerDashboard";
import UserDashboard from "../components/UserDashboard";
import { getCurrentUser, hasRole } from "../utils/auth";

export default function DashboardPage() {
  const user = getCurrentUser();

  return (
    <div className="dashboard-page">
      <section className="card account-summary">
        <h2>Dashboard</h2>
        <p>Signed in as <strong>{user?.email}</strong></p>
      </section>
      {hasRole("admin") && <AdminDashboard />}
      {hasRole("manager") && <ManagerDashboard />}
      {hasRole("user") && <UserDashboard />}
    </div>
  );
}
```
Do not use a single `if/else` chain. Roles are stored in an array, so an account may match more than one dashboard component.

## 24. Replace the Styles

Replace: `pulsevote-frontend/src/App.css`

```css
:root {
  font-family: "Segoe UI", system-ui, sans-serif;
  color: #f5f7fb;
  background: #121821;
  color-scheme: dark;
}

* { box-sizing: border-box; }
body { margin: 0; min-width: 320px; min-height: 100vh; background: #121821; }
button, input, select { font: inherit; }
button { border: 0; border-radius: 8px; padding: .7rem 1rem; background: #4e8cff; color: white; cursor: pointer; }
button:disabled { opacity: .55; cursor: not-allowed; }
button.secondary { background: #34445d; }
button.danger { background: #8f3540; }
input, select { width: 100%; padding: .75rem; border: 1px solid #41506a; border-radius: 8px; background: #1c2533; color: white; }
label { display: grid; gap: .35rem; }
code { padding: .2rem .4rem; border-radius: 4px; background: #111722; }

.app-shell { width: min(1120px, 94%); margin: auto; padding: 1.5rem 0 3rem; }
header { display: flex; align-items: center; justify-content: space-between; gap: 1rem; flex-wrap: wrap; }
nav { display: flex; gap: .8rem; flex-wrap: wrap; }
nav a { color: #a9c9ff; text-decoration: none; padding: .4rem .6rem; border-radius: 6px; }
nav a.active { background: #263753; color: white; }
main { margin-top: 1.5rem; }

.card, .panel, .poll-card { background: #1a2330; border: 1px solid #2c3a4f; border-radius: 14px; padding: 1.25rem; }
.dashboard-page { display: grid; gap: 1rem; }
.panel { display: grid; gap: 1rem; }
.account-summary { margin-bottom: .25rem; }
.form-stack { display: grid; gap: .8rem; }
.compact-form { max-width: 620px; }
.inline-control, .option-row { display: flex; align-items: center; gap: .6rem; }
.inline-control input, .option-row input { width: auto; }
.option-editor { display: flex; gap: .6rem; align-items: center; }
.option-editor input { flex: 1; }
.organisation-summary { padding: 1rem; border: 1px dashed #475a77; border-radius: 10px; }
.poll-list { display: grid; gap: 1rem; }
.poll-heading { display: flex; justify-content: space-between; gap: 1rem; align-items: start; }
.status { text-transform: uppercase; font-size: .78rem; padding: .3rem .5rem; border-radius: 999px; }
.status.open { background: #244e39; }
.status.closed { background: #63353a; }
.option-list { display: grid; gap: .65rem; margin: 1rem 0; }
.results { display: grid; gap: .5rem; margin: 1rem 0; }
.result-row { display: flex; justify-content: space-between; gap: 1rem; padding: .55rem 0; border-bottom: 1px solid #2c3a4f; }
.error-message, .success-message { padding: .75rem; border-radius: 8px; }
.error-message { background: #512b31; color: #ffd7dc; }
.success-message { background: #234936; color: #d6ffe7; }

@media (max-width: 620px) {
  .option-editor, .result-row { align-items: stretch; flex-direction: column; }
}
```
The existing `App.jsx`, page wrappers, and `main.jsx` may remain unchanged.

## 25. Start the Frontend

From `pulsevote-frontend`:

```bash
npm install
npm run dev
```

Open: `https://localhost:5173`

Accept the local certificate warning if required.

# Part C - Test the Complete Activity

## 26. Test Administrator Features

1. Create the first administrator through Postman if it does not already exist.
2. Log in through React as the administrator.
3. Confirm that the administrator dashboard appears.
4. Create a manager using a strong password.
5. Confirm that the administrator remains logged in.
6. Try registering the same manager email again.
7. Confirm that the backend rejects the duplicate.

## 27. Test Manager Features

1. Log out.
2. Log in as the manager.
3. Create an organisation.
4. Confirm that its name, ID, and join code appear.
5. Create a poll with at least two options.
6. Add a third option.
7. Confirm that the poll appears immediately.
8. Close the poll.
9. Reopen the poll.
10. Generate a new join code.
11. Confirm that the old join code no longer works.

## 28. Test User Features

1. Register a new normal user.
2. Enter the organisation join code.
3. Confirm that the organisation appears without requiring another login.
4. Select the organisation.
5. Open one of its polls.
6. Choose an option and vote.
7. Confirm that the results update.
8. Confirm that the selected option is marked as the user's vote.
9. Try to vote again.
10. Confirm that duplicate voting is rejected.

## 29. Test Rejected Requests

Use the browser Network tab and Postman to test:
- A normal user tries to register a manager.
- A normal user tries to create an organisation.
- A normal user tries to create a poll.
- A user tries to load polls from an organisation they have not joined.
- A user tries to vote in a different organisation's poll.
- A manager tries to open or close a different organisation's poll.
- A request is sent without a token.
- A request is sent with an invalid token.
- A user tries to join the same organisation twice.
- A user submits an invalid option index.
- A poll is created with blank or duplicate options.

The interface may hide invalid actions, but the backend response is the actual security control.

## 30. Security Review

Answer in your repository:

1. Why is hiding a React button not sufficient RBAC?
2. Why does decoding a token not verify its authenticity?
3. Why must the backend check the organisation as well as the role name?
4. Why must the administrator ignore the token returned when creating a manager?
5. Why is the first-admin endpoint treated as a bootstrap operation?
6. What information can a user read from a JWT stored in `localStorage`?
7. How could an XSS vulnerability expose a token stored in `localStorage`?
8. How does the CSP from Activity 06 reduce this risk?
9. Why are refreshed tokens returned after organisation roles change?

## 31. Final Git Checks

Do not commit:

- `.env`
- `node_modules`
- JWT values
- passwords
- private keys
- local certificates unless your lecturer explicitly supplied them for the activity
- screenshots containing sensitive values
