# Adding Authentication to the Frontend

In the previous activity, you added registration and login functionality to the PulseVote API.

In this activity, you will connect the React frontend to those authentication endpoints.

By the end of the activity, a user should be able to:

1. Register with an email address and password
2. Log in
3. Store the returned JWT in `localStorage`
4. Access a protected dashboard
5. Log out
6. See different navigation links depending on whether they are logged in

## Research

Before writing the code, spend 10–15 minutes researching the following:

* What is Axios?
* What is `localStorage`?
* What is React Router?
* What do `useState` and `useEffect` do?
* What is a protected route?
* Why should a React frontend not connect directly to MongoDB?

Write a short summary of 4–6 sentences in your own words and add it to your repository.

Suggested links:

* https://axios-http.com/docs/intro
* https://reactrouter.com/
* https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage
* https://react.dev/reference/react/useState
* https://react.dev/reference/react/useEffect

## Code Integration

You will make the following frontend changes:

1. Install Axios and React Router
2. Configure React Router
3. Create the application pages
4. Create registration and login forms
5. Store the JWT in `localStorage`
6. Protect the dashboard route
7. Add logout functionality
8. Update the navigation menu

## 1. Install the dependencies

Open a terminal in the frontend project and run:

```bash
npm install axios react-router-dom
```

Research what each package provides before continuing.

* Axios will be used to send requests to the API.
* React Router will be used to move between pages.

## 2. Create the project structure

Create the following files:

```text
src/
├── components/
│   ├── Layout.jsx
│   ├── Login.jsx
│   ├── ProtectedRoute.jsx
│   └── Register.jsx
├── pages/
│   ├── DashboardPage.jsx
│   ├── HomePage.jsx
│   ├── LoginPage.jsx
│   ├── LogoutPage.jsx
│   └── RegisterPage.jsx
├── App.css
├── App.jsx
└── main.jsx
```

The components folder contains reusable parts of the application.

The pages folder contains the content displayed for each route.

## 3. Configure React Router

Update `src/main.jsx` so that the application is wrapped in `BrowserRouter`:

```jsx
import { StrictMode } from "react";
import { BrowserRouter } from "react-router-dom";
import { createRoot } from "react-dom/client";
import App from "./App.jsx";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>
);
```

`BrowserRouter` allows the application to display different pages based on the URL.

## 4. Configure the application routes

Replace the existing content in `src/App.jsx` with the application routes:

```jsx
import "./App.css";
import { Routes, Route } from "react-router-dom";
import Layout from "./components/Layout";
import ProtectedRoute from "./components/ProtectedRoute";
import HomePage from "./pages/HomePage";
import LoginPage from "./pages/LoginPage";
import RegisterPage from "./pages/RegisterPage";
import LogoutPage from "./pages/LogoutPage";
import DashboardPage from "./pages/DashboardPage";

function App() {
  return (
    <Routes>
      <Route path="/" element={<Layout />}>
        <Route index element={<HomePage />} />
        <Route path="login" element={<LoginPage />} />
        <Route path="register" element={<RegisterPage />} />
        <Route path="logout" element={<LogoutPage />} />

        <Route
          path="dashboard"
          element={
            <ProtectedRoute>
              <DashboardPage />
            </ProtectedRoute>
          }
        />
      </Route>
    </Routes>
  );
}

export default App;
```

The application now has the following routes:

| Route        | Page                |
| ------------ | ------------------- |
| `/`          | Home                |
| `/login`     | Login               |
| `/register`  | Register            |
| `/dashboard` | Protected dashboard |
| `/logout`    | Logout              |

Notice that `DashboardPage` is wrapped inside `ProtectedRoute`.

## 5. Create the main layout

Create `src/components/Layout.jsx`:

```jsx
import {
  Outlet,
  NavLink,
  useLocation
} from "react-router-dom";
import { useEffect, useState } from "react";

export default function Layout() {
  const [loggedIn, setLoggedIn] = useState(false);
  const location = useLocation();

  useEffect(() => {
    setLoggedIn(!!localStorage.getItem("token"));
  }, [location]);

  return (
    <div className="app-shell">
      <header>
        <h1>PulseVote</h1>

        <nav>
          <NavLink
            to="/"
            className={({ isActive }) =>
              isActive ? "active" : ""
            }
          >
            Home
          </NavLink>

          {!loggedIn && (
            <>
              <NavLink
                to="/login"
                className={({ isActive }) =>
                  isActive ? "active" : ""
                }
              >
                Login
              </NavLink>

              <NavLink
                to="/register"
                className={({ isActive }) =>
                  isActive ? "active" : ""
                }
              >
                Register
              </NavLink>
            </>
          )}

          {loggedIn && (
            <>
              <NavLink
                to="/dashboard"
                className={({ isActive }) =>
                  isActive ? "active" : ""
                }
              >
                Dashboard
              </NavLink>

              <NavLink
                to="/logout"
                className={({ isActive }) =>
                  isActive ? "active" : ""
                }
              >
                Logout
              </NavLink>
            </>
          )}
        </nav>
      </header>

      <main>
        <Outlet />
      </main>
    </div>
  );
}
```

The `Layout` component provides the heading and navigation menu for every page.

`Outlet` displays the current page inside the layout.

The `useEffect` checks whether a token exists whenever the route changes:

```js
setLoggedIn(!!localStorage.getItem("token"));
```

When the user is logged out, the menu displays:

* Home
* Login
* Register

When the user is logged in, the menu displays:

* Home
* Dashboard
* Logout

## 6. Create the home page

Create `src/pages/HomePage.jsx`:

```jsx
export default function HomePage() {
  return (
    <div className="card">
      <h2>Welcome to PulseVote</h2>

      <p>
        You can log in or register using the menu.
      </p>
    </div>
  );
}
```

## 7. Create the registration component

Create `src/components/Register.jsx`:

```jsx
import { useState } from "react";
import axios from "axios";

export default function Register() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");
  const [success, setSuccess] = useState("");

  const register = async () => {
    setError("");
    setSuccess("");

    try {
      const response = await axios.post(
        "https://localhost:5000/api/auth/register",
        {
          email,
          password
        }
      );

      localStorage.setItem("token", response.data.token);
      setSuccess("Registered and logged in!");
    } catch (err) {
      setError(
        err.response?.data?.message ||
        "Error registering."
      );
    }
  };

  return (
    <div>
      {error && (
        <div className="error-message">
          {error}
        </div>
      )}

      {success && (
        <div className="success-message">
          {success}
        </div>
      )}

      <input
        type="email"
        placeholder="Email"
        onChange={(event) => setEmail(event.target.value)}
      />

      <input
        type="password"
        placeholder="Password"
        onChange={(event) => setPassword(event.target.value)}
      />

      <button onClick={register}>
        Register
      </button>
    </div>
  );
}
```

The component uses `useState` to store:

* The email address
* The password
* An error message
* A success message

When the user selects Register, Axios sends a `POST` request to:

```text
https://localhost:5000/api/auth/register
```

The request body contains:

```json
{
  "email": "user@example.com",
  "password": "password"
}
```

After successful registration, the token returned by the API is stored using:

```js
localStorage.setItem("token", response.data.token);
```

The user is now treated as logged in.

## 8. Create the registration page

Create `src/pages/RegisterPage.jsx`:

```jsx
import Register from "../components/Register";

export default function RegisterPage() {
  return (
    <div className="card">
      <h2>Register</h2>
      <Register />
    </div>
  );
}
```

This page displays the reusable `Register` component.

## 9. Create the login component

Create `src/components/Login.jsx`:

```jsx
import { useState } from "react";
import axios from "axios";
import { useNavigate } from "react-router-dom";

export default function Login() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");
  const [success, setSuccess] = useState("");

  const navigate = useNavigate();

  const login = async () => {
    setError("");
    setSuccess("");

    try {
      const response = await axios.post(
        "https://localhost:5000/api/auth/login",
        {
          email,
          password
        }
      );

      localStorage.setItem("token", response.data.token);
      setSuccess("Successfully logged in.");

      setTimeout(() => {
        navigate("/dashboard");
      }, 1000);
    } catch (err) {
      setError(
        err.response?.data?.message ||
        "Login failed."
      );
    }
  };

  return (
    <div>
      {error && (
        <div className="error-message">
          {error}
        </div>
      )}

      {success && (
        <div className="success-message">
          {success}
        </div>
      )}

      <input
        type="email"
        placeholder="Email"
        onChange={(event) => setEmail(event.target.value)}
      />

      <input
        type="password"
        placeholder="Password"
        onChange={(event) => setPassword(event.target.value)}
      />

      <button onClick={login}>
        Login
      </button>
    </div>
  );
}
```

Axios sends the login details to:

```text
https://localhost:5000/api/auth/login
```

After a successful login:

1. The JWT is stored in `localStorage`.
2. A success message is displayed.
3. The user is redirected to the dashboard.

The redirect is performed using:

```js
navigate("/dashboard");
```

## 10. Create the login page

Create `src/pages/LoginPage.jsx`:

```jsx
import Login from "../components/Login";

export default function LoginPage() {
  return (
    <div className="card">
      <h2>Login</h2>
      <Login />
    </div>
  );
}
```

## 11. Create the protected route

Create `src/components/ProtectedRoute.jsx`:

```jsx
import { Navigate } from "react-router-dom";

export default function ProtectedRoute({ children }) {
  const token = localStorage.getItem("token");

  return token
    ? children
    : <Navigate to="/login" />;
}
```

The component checks whether a token exists in `localStorage`.

When a token exists, the protected page is displayed:

```jsx
return children;
```

When no token exists, the user is redirected to the login page:

```jsx
<Navigate to="/login" />
```

This prevents users who are not logged in from opening the dashboard through the frontend.

The API must still validate the JWT before returning any sensitive or protected information. Frontend route protection alone is not sufficient security.

## 12. Create the dashboard

Create `src/pages/DashboardPage.jsx`:

```jsx
import { useEffect, useState } from "react";

export default function DashboardPage() {
  const [authorized, setAuthorized] = useState(null);

  useEffect(() => {
    const token = localStorage.getItem("token");

    if (token) {
      setAuthorized(true);
    } else {
      setAuthorized(false);
    }
  }, []);

  return (
    <div className="card">
      <h2>Dashboard</h2>

      {authorized === false && (
        <div className="error-message">
          You are not authorized to access this content.
        </div>
      )}

      {authorized === true && (
        <div className="success-message">
          You are authorized and can see this protected content.
        </div>
      )}
    </div>
  );
}
```

The dashboard performs a simple check to determine whether a token exists.

At this stage, it displays a confirmation message rather than requesting protected information from the API.

## 13. Add logout functionality

Create `src/pages/LogoutPage.jsx`:

```jsx
import { useEffect } from "react";
import { useNavigate } from "react-router-dom";

export default function LogoutPage() {
  const navigate = useNavigate();

  useEffect(() => {
    localStorage.removeItem("token");

    const timer = setTimeout(() => {
      navigate("/");
    }, 3000);

    return () => clearTimeout(timer);
  }, [navigate]);

  return (
    <div className="card">
      <h2>Logged Out</h2>

      <p>
        You have been successfully logged out.
        Redirecting to the homepage...
      </p>
    </div>
  );
}
```

Logging out removes the token:

```js
localStorage.removeItem("token");
```

After three seconds, the user is redirected to the home page.

The cleanup function removes the timer if the component is closed before the timer finishes:

```js
return () => clearTimeout(timer);
```

## 14. Add consistent styling

Update `src/App.css` to include consistent styling for the application.

Your stylesheet should include styles for:

* The page background
* Text and headings
* Navigation links
* Active navigation links
* Input fields
* Buttons
* Cards
* Error messages
* Success messages

At a minimum, include clear styles for the two message types:

```css
.error-message {
  background-color: #2f1c1c;
  color: #ff6b6b;
  padding: 0.75rem;
  border-radius: 6px;
  margin-bottom: 1rem;
  border: 1px solid #ff6b6b;
}

.success-message {
  background-color: #1e2f1e;
  color: #4afc75;
  padding: 0.75rem 1rem;
  border-radius: 6px;
  margin-bottom: 1rem;
  border: 1px solid #4afc75;
}
```

Use consistent layout and styling across all pages.

## 15. Test the application

Complete the following tests.

### Registration

1. Open the Register page.
2. Enter an email address and password.
3. Select Register.
4. Confirm that a success message appears.
5. Confirm that the user was added to MongoDB through the API.
6. Confirm that a token appears in `localStorage`.

### Login

1. Log out or remove the token.
2. Open the Login page.
3. Enter incorrect login details.
4. Confirm that an error message appears.
5. Enter the correct login details.
6. Confirm that a success message appears.
7. Confirm that you are redirected to the dashboard.
8. Confirm that the token appears in `localStorage`.

You can inspect `localStorage` through:

```text
Developer Tools → Application → Local Storage
```

### Protected route

1. Confirm that you can open the dashboard while logged in.
2. Remove the token from `localStorage`.
3. Try to open `/dashboard`.
4. Confirm that you are redirected to `/login`.

### Navigation

When logged out, confirm that the menu displays:

* Home
* Login
* Register

When logged in, confirm that the menu displays:

* Home
* Dashboard
* Logout

### Logout

1. Select Logout.
2. Confirm that the token is removed.
3. Confirm that the logout message appears.
4. Confirm that you are redirected to the home page.

## Key Requirements

Your completed frontend must:

1. Include a registration page using an email address and password.
2. Include a login page using an email address and password.
3. Use Axios to communicate with the API.
4. Store the returned JWT in `localStorage`.
5. Include a protected dashboard route.
6. Redirect unauthenticated users to the login page.
7. Include logout functionality.
8. Remove the token when the user logs out.
9. Update the navigation menu according to the authentication state.
10. Display appropriate success and error messages.
11. Use React Router to manage the application pages.
12. Use consistent layout and styling.
13. Never connect directly to MongoDB from the frontend.

## Important Security Note

The frontend only checks whether a token exists.

This makes it useful for controlling what the user sees, but it does not provide complete security.

Any protected API endpoint must still verify that the JWT is valid before returning protected data.

The frontend must communicate with the API. It must never communicate directly with MongoDB.

## Submission

Commit and push the completed frontend to GitHub.

Your repository should include:

* React Router configuration
* The main application layout
* Registration functionality
* Login functionality
* JWT storage using `localStorage`
* A protected dashboard route
* Logout functionality
* Conditional navigation
* Success and error messages
* Consistent styling
* Your research summary
* Instructions for running the frontend
