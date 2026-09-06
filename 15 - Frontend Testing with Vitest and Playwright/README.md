# PulseVote – Frontend Testing with Vitest and Playwright

In the previous activities, you already:

* Built and secured the PulseVote API
* Built the PulseVote frontend
* Added authentication and RBAC
* Added rate limiting
* Added backend linting and unit testing
* Dockerised the API and frontend
* Added GitHub Actions CI with Newman
* Added SonarQube Cloud to the API pipeline
* Deployed the API to Render

We have already added automated checks around the API. We will now do the same for the frontend.

For the frontend, we will use:

* **Vitest** with **React Testing Library** for unit/component testing
* **Playwright** for end-to-end browser testing

For now, everything will run locally. In the next activity, we will add these checks to the frontend GitHub Actions pipeline.

## Research

Before you code, spend 10–15 minutes researching frontend testing, Vitest, React Testing Library and Playwright.

Answer briefly:

* What is frontend unit testing?
* What is component testing?
* What is end-to-end testing?
* What is Vitest?
* What is React Testing Library?
* What is Playwright?
* What is mocking?
* Why would we mock an API while testing the frontend?
* What is a flaky test?
* Why should we test both successful and unsuccessful behaviour?
* Why is an end-to-end test different from a component test?

Write a 4–6 sentence summary in your own words and commit it to your repo.

## Requirements

After this activity, your PulseVote frontend will:

* Run unit/component tests using Vitest
* Use React Testing Library to interact with React components
* Test login and registration
* Test protected routes
* Test role-based dashboard behaviour
* Test voting and poll management
* Generate frontend test coverage
* Run end-to-end tests using Playwright
* Test the application in a real browser
* Produce a Playwright HTML report

We will keep the Vitest tests next to the React files that they test.

The Playwright tests will be placed separately under:

```text
tests/e2e/
```

## Code Changes

### 1. Install the testing dependencies

Open a terminal in:

```text
pulsevote-frontend
```

Install Vitest, React Testing Library, jsdom and the coverage provider:

```bash
npm i -D vitest jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event @vitest/coverage-v8
```

Install Playwright:

```bash
npm i -D @playwright/test
```

Install the Chromium browser that Playwright will use:

```bash
npx playwright install chromium
```

### 2. Add the frontend testing scripts

Open:

```text
pulsevote-frontend/package.json
```

Update the `scripts` section so that it includes:

```json
"scripts": {
  "dev": "vite",
  "build": "vite build",
  "lint": "eslint .",
  "preview": "vite preview",
  "test": "vitest run",
  "test:watch": "vitest",
  "test:coverage": "vitest run --coverage",
  "test:e2e": "playwright test",
  "test:e2e:report": "playwright show-report"
}
```

Do not remove your existing dependencies.

Your testing commands will now be:

```bash
npm run test
npm run test:watch
npm run test:coverage
npm run test:e2e
npm run test:e2e:report
```


### 3. Configure Vitest

Create:

```text
pulsevote-frontend/vitest.config.js
```

Add:

```js
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: "./src/test/setup.js",
    include: ["src/**/*.test.{js,jsx}"],
    coverage: {
      provider: "v8",
      reporter: ["text", "html", "lcov"],
      reportsDirectory: "coverage",
      include: ["src/**/*.{js,jsx}"],
      exclude: [
        "src/main.jsx",
        "src/test/**",
        "src/**/*.test.{js,jsx}"
      ],
    },
  },
});
```

The important settings are:

```js
environment: "jsdom"
```

This gives our tests a browser-like DOM.

```js
plugins: [react()]
```

This allows JSX to be transformed correctly during the tests.

```js
include: ["src/**/*.test.{js,jsx}"]
```

This tells Vitest to run the tests under `src`.

It also means Vitest will not try to run our Playwright tests under `tests/e2e`.

### 4. Add a shared test setup file

Create the folder:

```text
src/test/
```

Create:

```text
src/test/setup.js
```

Add:

```js
import "@testing-library/jest-dom/vitest";
import { cleanup } from "@testing-library/react";
import { afterEach, vi } from "vitest";

afterEach(() => {
  cleanup();
  localStorage.clear();
  vi.clearAllMocks();
});
```

This:

* clears the rendered React components after each test
* clears `localStorage`
* clears mock call history

This helps make sure that one test does not affect the next one.

## Testing Login

### 5. Create the Login test

Create:

```text
src/components/Login.test.jsx
```

Add:

```jsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { MemoryRouter } from "react-router-dom";
import { beforeEach, describe, expect, it, vi } from "vitest";
import Login from "./Login";
import api from "../api/api";

const navigate = vi.fn();

vi.mock("react-router-dom", async () => {
  const actual = await vi.importActual("react-router-dom");

  return {
    ...actual,
    useNavigate: () => navigate
  };
});

vi.mock("../api/api", () => ({
  default: {
    post: vi.fn()
  }
}));

describe("Login", () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("renders the login form and can show the password", async () => {
    const user = userEvent.setup();

    render(
      <MemoryRouter>
        <Login />
      </MemoryRouter>
    );

    const emailInput = screen.getByLabelText("Email");
    const passwordInput = screen.getByLabelText("Password");

    expect(emailInput).toBeInTheDocument();
    expect(passwordInput).toHaveAttribute("type", "password");

    await user.click(screen.getByLabelText("Show password"));

    expect(passwordInput).toHaveAttribute("type", "text");
  });

  it("shows the API error when login fails", async () => {
    const user = userEvent.setup();

    api.post.mockRejectedValueOnce({
      response: {
        data: {
          message: "Invalid credentials"
        }
      }
    });

    render(
      <MemoryRouter>
        <Login />
      </MemoryRouter>
    );

    await user.type(
      screen.getByLabelText("Email"),
      "student@test.com"
    );

    await user.type(
      screen.getByLabelText("Password"),
      "WrongPassword1"
    );

    await user.click(
      screen.getByRole("button", { name: "Login" })
    );

    expect(api.post).toHaveBeenCalledWith("/auth/login", {
      email: "student@test.com",
      password: "WrongPassword1"
    });

    expect(
      await screen.findByText("Invalid credentials")
    ).toBeInTheDocument();
  });

  it("stores the token and navigates to the dashboard after a successful login", async () => {
    const user = userEvent.setup();

    api.post.mockResolvedValueOnce({
      data: {
        token: "test.jwt.token"
      }
    });

    render(
      <MemoryRouter>
        <Login />
      </MemoryRouter>
    );

    await user.type(
      screen.getByLabelText("Email"),
      "student@test.com"
    );

    await user.type(
      screen.getByLabelText("Password"),
      "Password123!"
    );

    await user.click(
      screen.getByRole("button", { name: "Login" })
    );

    expect(
      localStorage.getItem("token")
    ).toBe("test.jwt.token");

    expect(
      navigate
    ).toHaveBeenCalledWith("/dashboard");
  });
});
```

Here we are not calling the real API.

Instead, we mock:

```js
api.post
```

For the failed login test, we force the API to reject the request:

```js
api.post.mockRejectedValueOnce(...)
```

For the successful login test, we return a test token:

```js
api.post.mockResolvedValueOnce(...)
```

This allows us to test how the frontend behaves without depending on the backend or database.


## Testing Registration

### 6. Create the Register test

Create:

```text
src/components/Register.test.jsx
```

Add:

```jsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { MemoryRouter } from "react-router-dom";
import { beforeEach, describe, expect, it, vi } from "vitest";
import Register from "./Register";
import api from "../api/api";

const navigate = vi.fn();

vi.mock("react-router-dom", async () => {
  const actual = await vi.importActual("react-router-dom");

  return {
    ...actual,
    useNavigate: () => navigate
  };
});

vi.mock("../api/api", () => ({
  default: {
    post: vi.fn()
  }
}));

describe("Register", () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("rejects mismatched passwords before calling the API", async () => {
    const user = userEvent.setup();

    render(
      <MemoryRouter>
        <Register />
      </MemoryRouter>
    );

    await user.type(
      screen.getByLabelText("Email"),
      "newuser@test.com"
    );

    await user.type(
      screen.getByLabelText("Password", { selector: "input" }),
      "Password123!"
    );

    await user.type(
      screen.getByLabelText("Confirm password"),
      "Different123!"
    );

    await user.click(
      screen.getByRole("button", { name: "Register" })
    );

    expect(
      screen.getByText("Passwords do not match.")
    ).toBeInTheDocument();

    expect(
      api.post
    ).not.toHaveBeenCalled();
  });

  it("registers the user, stores the token and navigates to the dashboard", async () => {
    const user = userEvent.setup();

    api.post.mockResolvedValueOnce({
      data: {
        token: "registered.jwt.token"
      }
    });

    render(
      <MemoryRouter>
        <Register />
      </MemoryRouter>
    );

    await user.type(
      screen.getByLabelText("Email"),
      "newuser@test.com"
    );

    await user.type(
      screen.getByLabelText("Password", { selector: "input" }),
      "Password123!"
    );

    await user.type(
      screen.getByLabelText("Confirm password"),
      "Password123!"
    );

    await user.click(
      screen.getByRole("button", { name: "Register" })
    );

    expect(api.post).toHaveBeenCalledWith(
      "/auth/register-user",
      {
        email: "newuser@test.com",
        password: "Password123!"
      }
    );

    expect(
      localStorage.getItem("token")
    ).toBe("registered.jwt.token");

    expect(
      navigate
    ).toHaveBeenCalledWith("/dashboard");
  });
});
```

The first test checks validation before an API request is sent.

The second test confirms that the frontend:

1. sends the expected registration request
2. saves the returned token
3. navigates to the dashboard


## Testing Protected Routes

### 7. Create the ProtectedRoute test

Create:

```text
src/components/ProtectedRoute.test.jsx
```

Add:

```jsx
import { render, screen } from "@testing-library/react";
import {
  MemoryRouter,
  Route,
  Routes
} from "react-router-dom";
import { describe, expect, it } from "vitest";
import ProtectedRoute from "./ProtectedRoute";

function makeToken(payload) {
  const encode = (value) =>
    btoa(JSON.stringify(value))
      .replace(/\+/g, "-")
      .replace(/\//g, "_");

  return `${encode({
    alg: "none",
    typ: "JWT"
  })}.${encode(payload)}.signature`;
}

describe("ProtectedRoute", () => {
  it("redirects an unauthenticated visitor to login", () => {
    render(
      <MemoryRouter initialEntries={["/dashboard"]}>
        <Routes>
          <Route
            path="/login"
            element={<p>Login page</p>}
          />

          <Route
            path="/dashboard"
            element={
              <ProtectedRoute>
                <p>Private dashboard</p>
              </ProtectedRoute>
            }
          />
        </Routes>
      </MemoryRouter>
    );

    expect(
      screen.getByText("Login page")
    ).toBeInTheDocument();

    expect(
      screen.queryByText("Private dashboard")
    ).not.toBeInTheDocument();
  });

  it("shows protected content when a valid token is present", () => {
    localStorage.setItem(
      "token",
      makeToken({
        email: "student@test.com",
        roles: [
          {
            role: "user"
          }
        ],
        exp: Math.floor(Date.now() / 1000) + 3600
      })
    );

    render(
      <MemoryRouter initialEntries={["/dashboard"]}>
        <Routes>
          <Route
            path="/login"
            element={<p>Login page</p>}
          />

          <Route
            path="/dashboard"
            element={
              <ProtectedRoute>
                <p>Private dashboard</p>
              </ProtectedRoute>
            }
          />
        </Routes>
      </MemoryRouter>
    );

    expect(
      screen.getByText("Private dashboard")
    ).toBeInTheDocument();
  });
});
```

The test token is only used by the frontend test.

It is not a properly signed JWT.

The backend must still verify the real JWT signature and enforce access control.


## Testing the Role-Based Dashboard

### 8. Create the DashboardPage test

Create:

```text
src/pages/DashboardPage.test.jsx
```

Add:

```jsx
import { render, screen } from "@testing-library/react";
import {
  beforeEach,
  describe,
  expect,
  it,
  vi
} from "vitest";

import DashboardPage from "./DashboardPage";
import {
  getCurrentUser,
  hasRole
} from "../utils/auth";

vi.mock("../components/AdminDashboard", () => ({
  default: () => <div>ADMIN AREA</div>
}));

vi.mock("../components/ManagerDashboard", () => ({
  default: () => <div>MANAGER AREA</div>
}));

vi.mock("../components/UserDashboard", () => ({
  default: () => <div>USER AREA</div>
}));

vi.mock("../utils/auth", () => ({
  getCurrentUser: vi.fn(),
  hasRole: vi.fn()
}));

describe("DashboardPage", () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("shows the dashboards for every role assigned to the account", () => {
    getCurrentUser.mockReturnValue({
      email: "multi@test.com"
    });

    hasRole.mockImplementation(
      (role) =>
        role === "manager" ||
        role === "user"
    );

    render(<DashboardPage />);

    expect(
      screen.getByText("multi@test.com")
    ).toBeInTheDocument();

    expect(
      screen.getByText("MANAGER AREA")
    ).toBeInTheDocument();

    expect(
      screen.getByText("USER AREA")
    ).toBeInTheDocument();

    expect(
      screen.queryByText("ADMIN AREA")
    ).not.toBeInTheDocument();
  });
});
```

PulseVote supports more than one role on an account.

This test therefore confirms that the frontend can display more than one role-specific dashboard at the same time.


## Testing PollCard

### 9. Create the PollCard test

Create:

```text
src/components/PollCard.test.jsx
```

Add:

```jsx
import {
  render,
  screen,
  waitFor
} from "@testing-library/react";

import userEvent from "@testing-library/user-event";

import {
  beforeEach,
  describe,
  expect,
  it,
  vi
} from "vitest";

import PollCard from "./PollCard";
import api from "../api/api";

vi.mock("../api/api", () => ({
  default: {
    get: vi.fn(),
    post: vi.fn()
  }
}));

const poll = {
  _id: "poll-1",
  question: "Which deployment platform?",
  options: [
    "Render",
    "Other"
  ],
  status: "open"
};

describe("PollCard", () => {
  beforeEach(() => {
    vi.resetAllMocks();

    api.get.mockResolvedValue({
      data: {
        results: {
          counts: [0, 0],
          percentages: [0, 0],
          totalVotes: 0,
          userVoteIndex: null
        }
      }
    });
  });

  it("requires an option before voting", async () => {
    const user = userEvent.setup();

    render(
      <PollCard
        poll={poll}
        canManage={false}
        canVote
        onPollChanged={vi.fn()}
      />
    );

    await screen.findByText(
      "Total votes: 0"
    );

    await user.click(
      screen.getByRole(
        "button",
        { name: "Vote" }
      )
    );

    expect(
      screen.getByText(
        "Select an option before voting."
      )
    ).toBeInTheDocument();

    expect(
      api.post
    ).not.toHaveBeenCalled();
  });

  it("submits the selected option and refreshes the results", async () => {
    const user = userEvent.setup();

    api.get
      .mockResolvedValueOnce({
        data: {
          results: {
            counts: [0, 0],
            percentages: [0, 0],
            totalVotes: 0,
            userVoteIndex: null
          }
        }
      })
      .mockResolvedValueOnce({
        data: {
          results: {
            counts: [1, 0],
            percentages: [100, 0],
            totalVotes: 1,
            userVoteIndex: 0
          }
        }
      });

    api.post.mockResolvedValueOnce({
      data: {
        message: "Vote recorded"
      }
    });

    render(
      <PollCard
        poll={poll}
        canManage={false}
        canVote
        onPollChanged={vi.fn()}
      />
    );

    await screen.findByText(
      "Total votes: 0"
    );

    await user.click(
      screen.getByLabelText("Render")
    );

    await user.click(
      screen.getByRole(
        "button",
        { name: "Vote" }
      )
    );

    expect(
      api.post
    ).toHaveBeenCalledWith(
      "/polls/vote/poll-1",
      {
        optionIndex: 0
      }
    );

    expect(
      await screen.findByText(
        "Vote recorded."
      )
    ).toBeInTheDocument();

    expect(
      await screen.findByText(
        "Total votes: 1"
      )
    ).toBeInTheDocument();

    expect(
      screen.getByText(
        "Render (your vote)"
      )
    ).toBeInTheDocument();
  });

  it("allows a manager to close an open poll", async () => {
    const user = userEvent.setup();

    const onPollChanged =
      vi.fn().mockResolvedValue(undefined);

    api.post.mockResolvedValueOnce({
      data: {
        message: "Poll closed"
      }
    });

    render(
      <PollCard
        poll={poll}
        canManage
        canVote={false}
        onPollChanged={onPollChanged}
      />
    );

    await screen.findByText(
      "Total votes: 0"
    );

    await user.click(
      screen.getByRole(
        "button",
        { name: "Close Poll" }
      )
    );

    expect(
      api.post
    ).toHaveBeenCalledWith(
      "/polls/close/poll-1"
    );

    await waitFor(() =>
      expect(
        onPollChanged
      ).toHaveBeenCalled()
    );
  });
});
```

Notice that this test uses:

```js
vi.resetAllMocks();
```

rather than:

```js
vi.clearAllMocks();
```

`clearAllMocks()` only clears information about calls that have already happened.

`resetAllMocks()` also clears any queued return values.

That matters here because different PollCard tests return different mocked API responses.


### 10. Run the Vitest tests

Run:

```bash
npm run test
```

You should see tests for:

```text
Login
Register
ProtectedRoute
DashboardPage
PollCard
```

At this point, do not assume that a failed test means that the test is wrong.

A test may also have found a problem in the application.


## Fixing a Bug Found by the Test

### 11. Fix the first-option voting bug

When the PollCard test selects the first option, it exposes a small problem in the original component.

The selected option starts as:

```js
const [selectedOptionIndex, setSelectedOptionIndex] =
  useState("");
```

The original radio button used:

```jsx
checked={
  Number(selectedOptionIndex) === index
}
```

But:

```js
Number("") === 0
```

This means the first radio button can appear selected even though the selected value in state is still an empty string.

The test has found a real bug.

Open:

```text
src/components/PollCard.jsx
```

Find the radio input.

Replace:

```jsx
checked={
  Number(selectedOptionIndex) === index
}
```

with:

```jsx
checked={
  selectedOptionIndex !== "" &&
  Number(selectedOptionIndex) === index
}
```

The full input should now be:

```jsx
<input
  type="radio"
  name={`poll-${poll._id}`}
  value={index}
  checked={
    selectedOptionIndex !== "" &&
    Number(selectedOptionIndex) === index
  }
  onChange={(e) =>
    setSelectedOptionIndex(e.target.value)
  }
/>
```

Run the tests again:

```bash
npm run test
```

All the Vitest tests should now pass.

## Test Coverage

### 12. Generate the coverage report

Run:

```bash
npm run test:coverage
```

Vitest will display a coverage summary in the terminal.

It will also create:

```text
coverage/
```

The LCOV file will be:

```text
coverage/lcov.info
```

We will use this again when we add SonarQube Cloud to the frontend pipeline.


# Playwright End-to-End Testing

Vitest has tested individual pieces of the frontend.

We will now test complete user flows in a real Chromium browser.


### 13. Configure Playwright

Create:

```text
pulsevote-frontend/playwright.config.js
```

Add:

```js
import { defineConfig } from "@playwright/test";

export default defineConfig({
  workers: 1,

  testDir: "./tests/e2e",

  fullyParallel: true,

  reporter: [
    [
      "html",
      {
        open: "never"
      }
    ]
  ],

  use: {
    baseURL: "https://localhost:5173",
    ignoreHTTPSErrors: true,
    trace: "on-first-retry"
  },

  webServer: {
    command:
      "npm run dev -- --host 0.0.0.0",

    url:
      "https://localhost:5173",

    reuseExistingServer: true,

    ignoreHTTPSErrors: true,

    timeout: 120000
  }
});
```

We are using:

```js
workers: 1
```

for this activity.

This keeps the test run simple and predictable while we are learning how the suite works.

Playwright will automatically start the Vite frontend before the tests run.

Because PulseVote uses the self-signed HTTPS certificate from the earlier SSL activity, Playwright also needs:

```js
ignoreHTTPSErrors: true
```


### 14. Create the Playwright test folder

Create:

```text
tests/
└── e2e/
```

Create:

```text
tests/e2e/pulsevote.spec.js
```

### 15. Add the complete Playwright test file

Add:

```js
import {
  expect,
  test
} from "@playwright/test";

function makeToken({
  email = "user@test.com",
  roles = [
    {
      role: "user",
      organisationId: "org-1"
    }
  ]
} = {}) {
  const encode = (value) =>
    Buffer
      .from(JSON.stringify(value))
      .toString("base64")
      .replace(/\+/g, "-")
      .replace(/\//g, "_");

  return `${encode({
    alg: "none",
    typ: "JWT"
  })}.${encode({
    email,
    roles,
    exp:
      Math.floor(Date.now() / 1000) +
      3600
  })}.signature`;
}

async function mockApi(
  page,
  role = "user"
) {
  const token = makeToken({
    email: `${role}@test.com`,
    roles: [
      {
        role,
        organisationId:
          role === "admin"
            ? null
            : "org-1"
      }
    ]
  });

  let results = {
    counts: [0, 0],
    percentages: [0, 0],
    totalVotes: 0,
    userVoteIndex: null
  };

  await page.route(
    "**/*",
    async (route) => {
      const request =
        route.request();

      const url =
        request.url();

      const pathname =
        new URL(url).pathname;

      const method =
        request.method();

      // Only mock real backend API calls.
      // Do not intercept frontend files such as:
      // /src/api/api.js
      if (
        !pathname.startsWith(
          "/api/"
        )
      ) {
        return route.continue();
      }

      if (
        url.endsWith(
          "/api/auth/login"
        ) &&
        method === "POST"
      ) {
        const body =
          request.postDataJSON();

        if (
          body.password ===
          "WrongPassword1"
        ) {
          return route.fulfill({
            status: 401,
            contentType:
              "application/json",
            body: JSON.stringify({
              message:
                "Invalid credentials"
            })
          });
        }

        return route.fulfill({
          status: 200,
          contentType:
            "application/json",
          body: JSON.stringify({
            token
          })
        });
      }

      if (
        url.endsWith(
          "/api/auth/register-user"
        ) &&
        method === "POST"
      ) {
        return route.fulfill({
          status: 201,
          contentType:
            "application/json",
          body: JSON.stringify({
            token
          })
        });
      }

      if (
        url.endsWith(
          "/api/organisations/my-organisations"
        ) &&
        method === "GET"
      ) {
        return route.fulfill({
          status: 200,
          contentType:
            "application/json",
          body: JSON.stringify([
            {
              _id: "org-1",
              name: "INSY Class",
              joinCode: "JOIN123"
            }
          ])
        });
      }

      if (
        url.includes(
          "/api/polls/get-polls/org-1"
        ) &&
        method === "GET"
      ) {
        return route.fulfill({
          status: 200,
          contentType:
            "application/json",
          body: JSON.stringify([
            {
              _id: "poll-1",
              organisationId:
                "org-1",
              question:
                "Best CI tool?",
              options: [
                "GitHub Actions",
                "Other"
              ],
              status: "open"
            }
          ])
        });
      }

      if (
        url.includes(
          "/api/polls/get-poll-results/poll-1"
        ) &&
        method === "GET"
      ) {
        return route.fulfill({
          status: 200,
          contentType:
            "application/json",
          body: JSON.stringify({
            results
          })
        });
      }

      if (
        url.includes(
          "/api/polls/vote/poll-1"
        ) &&
        method === "POST"
      ) {
        results = {
          counts: [1, 0],
          percentages: [100, 0],
          totalVotes: 1,
          userVoteIndex: 0
        };

        return route.fulfill({
          status: 200,
          contentType:
            "application/json",
          body: JSON.stringify({
            message:
              "Vote recorded"
          })
        });
      }

      if (
        url.includes(
          "/api/polls/close/poll-1"
        ) &&
        method === "POST"
      ) {
        return route.fulfill({
          status: 200,
          contentType:
            "application/json",
          body: JSON.stringify({
            message:
              "Poll closed"
          })
        });
      }

      return route.fulfill({
        status: 200,
        contentType:
          "application/json",
        body: JSON.stringify({})
      });
    }
  );
}

async function login(
  page,
  role = "user"
) {
  await mockApi(
    page,
    role
  );

  await page.goto(
    "/login"
  );

  await page
    .getByLabel("Email")
    .fill(
      `${role}@test.com`
    );

  await page
    .getByLabel(
      "Password",
      {
        exact: true
      }
    )
    .fill(
      "Password123!"
    );

  await page
    .getByRole(
      "button",
      {
        name: "Login"
      }
    )
    .click();

  await expect(
    page
  ).toHaveURL(
    /\/dashboard$/
  );
}

test(
  "home page loads and exposes the authentication navigation",
  async ({ page }) => {
    await page.goto("/");

    await expect(
      page.getByRole(
        "heading",
        {
          name: "PulseVote",
          exact: true
        }
      )
    ).toBeVisible();

    await expect(
      page.getByRole(
        "link",
        {
          name: "Login"
        }
      )
    ).toBeVisible();

    await expect(
      page.getByRole(
        "link",
        {
          name: "Register"
        }
      )
    ).toBeVisible();
  }
);

test(
  "protected dashboard redirects an unauthenticated visitor to login",
  async ({ page }) => {
    await page.goto(
      "/dashboard"
    );

    await expect(
      page
    ).toHaveURL(
      /\/login$/
    );
  }
);

test(
  "failed login shows the backend error without leaving the login page",
  async ({ page }) => {
    await mockApi(page);

    await page.goto(
      "/login"
    );

    await page
      .getByLabel("Email")
      .fill(
        "user@test.com"
      );

    await page
      .getByLabel(
        "Password",
        {
          exact: true
        }
      )
      .fill(
        "WrongPassword1"
      );

    await page
      .getByRole(
        "button",
        {
          name: "Login"
        }
      )
      .click();

    await expect(
      page.getByText(
        "Invalid credentials"
      )
    ).toBeVisible();

    await expect(
      page
    ).toHaveURL(
      /\/login$/
    );
  }
);

test(
  "successful login opens the role-specific dashboard",
  async ({ page }) => {
    await login(
      page,
      "user"
    );

    await expect(
      page.getByRole(
        "heading",
        {
          name: "Dashboard",
          exact: true
        }
      )
    ).toBeVisible();

    await expect(
      page.getByRole(
        "heading",
        {
          name:
            "User Dashboard"
        }
      )
    ).toBeVisible();

    await expect(
      page.getByText(
        "Signed in as"
      )
    ).toContainText(
      "user@test.com"
    );
  }
);

test(
  "a user can view a poll, vote and see updated results",
  async ({ page }) => {
    await login(
      page,
      "user"
    );

    await expect(
      page.getByRole(
        "heading",
        {
          name:
            "Best CI tool?"
        }
      )
    ).toBeVisible();

    await page
      .getByLabel(
        "GitHub Actions"
      )
      .check();

    await page
      .getByRole(
        "button",
        {
          name: "Vote"
        }
      )
      .click();

    await expect(
      page.getByText(
        "Vote recorded."
      )
    ).toBeVisible();

    await expect(
      page.getByText(
        "Total votes: 1"
      )
    ).toBeVisible();

    await expect(
      page.getByText(
        "GitHub Actions (your vote)"
      )
    ).toBeVisible();
  }
);

test(
  "logout clears the session and protects the dashboard again",
  async ({ page }) => {
    await login(
      page,
      "user"
    );

    await page
      .getByRole(
        "link",
        {
          name: "Logout"
        }
      )
      .click();

    await expect(
      page
    ).toHaveURL(
      /\/$/
    );

    await page.goto(
      "/dashboard"
    );

    await expect(
      page
    ).toHaveURL(
      /\/login$/
    );
  }
);
```

### 16. Why the API mock checks the pathname

Notice this section:

```js
const pathname =
  new URL(url).pathname;

if (
  !pathname.startsWith("/api/")
) {
  return route.continue();
}
```

Do not replace this with:

```js
page.route("**/api/**", ...)
```

Our React frontend contains:

```text
src/api/api.js
```

When Vite serves that file in development, the browser requests:

```text
/src/api/api.js
```

A broad `**/api/**` route can therefore intercept the frontend JavaScript module itself.

The React application then fails before the test even reaches the login form.

We only want to mock real backend URLs beginning with:

```text
/api/
```


### 17. Why some heading selectors use `exact: true`

The home page contains headings such as:

```text
PulseVote
Welcome to PulseVote
```

If we use:

```js
page.getByRole(
  "heading",
  {
    name: "PulseVote"
  }
)
```

Playwright may match both headings.

Instead, use:

```js
page.getByRole(
  "heading",
  {
    name: "PulseVote",
    exact: true
  }
)
```

The same applies to:

```text
Dashboard
User Dashboard
```

Use:

```js
page.getByRole(
  "heading",
  {
    name: "Dashboard",
    exact: true
  }
)
```

when you specifically want the main Dashboard heading.


### 18. Run the Playwright tests

Run:

```bash
npm run test:e2e
```

You should see:

```text
6 passed
```

The tests run using one worker.

To watch the browser while Playwright runs:

```bash
npx playwright test --headed
```

To open the Playwright HTML report:

```bash
npm run test:e2e:report
```


## Ignore Test Output

### 19. Update `.gitignore`

Open:

```text
pulsevote-frontend/.gitignore
```

Make sure it includes:

```text
node_modules
dist
coverage/
playwright-report/
test-results/
```

These are generated files and should not be committed.

### 20. Update ESLint Config

Update the eslint packages:
```
npm install --save-dev eslint@10.10.0 eslint-plugin-react-hooks@7.1.1 eslint-plugin-react-refresh@0.5.6
```

Open:
```
eslint.config.js
```

Update it to use this config which caters for additional directories and files that we do not need lint to consider:
```js
import js from '@eslint/js'
import globals from 'globals'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'
import { defineConfig, globalIgnores } from 'eslint/config'

export default defineConfig([
  globalIgnores(['dist/', 'coverage/']),

  {
    files: ['src/**/*.{js,jsx}'],

    extends: [
      js.configs.recommended,
      reactRefresh.configs.vite,
    ],

    plugins: {
      'react-hooks': reactHooks,
    },

    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,

      parserOptions: {
        ecmaVersion: 'latest',
        ecmaFeatures: {
          jsx: true,
        },
        sourceType: 'module',
      },
    },

    rules: {
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn',

      'no-unused-vars': [
        'error',
        {
          varsIgnorePattern: '^[A-Z_]',
        },
      ],
    },
  },

  {
    files: ['tests/**/*.js'],

    extends: [
      js.configs.recommended,
    ],

    languageOptions: {
      globals: globals.node,

      parserOptions: {
        ecmaVersion: 'latest',
        sourceType: 'module',
      },
    },
  },
])
```

Fix any errors and warnings if they come up.

## Run All Frontend Checks

### 21. Run everything locally

Run:

```bash
npm run lint
```

Then:

```bash
npm run test
```

Then:

```bash
npm run test:coverage
```

Then:

```bash
npm run test:e2e
```

Finally:

```bash
npm run build
```

Everything should pass before you push your changes.


## Final Project Structure

Your frontend should now look similar to:

```text
pulsevote-frontend/
├── src/
│   ├── api/
│   │   └── api.js
│   ├── components/
│   │   ├── AdminDashboard.jsx
│   │   ├── Layout.jsx
│   │   ├── Login.jsx
│   │   ├── Login.test.jsx
│   │   ├── ManagerDashboard.jsx
│   │   ├── OrganisationSelector.jsx
│   │   ├── PollCard.jsx
│   │   ├── PollCard.test.jsx
│   │   ├── ProtectedRoute.jsx
│   │   ├── ProtectedRoute.test.jsx
│   │   ├── Register.jsx
│   │   ├── Register.test.jsx
│   │   └── UserDashboard.jsx
│   ├── pages/
│   │   ├── DashboardPage.jsx
│   │   ├── DashboardPage.test.jsx
│   │   ├── HomePage.jsx
│   │   ├── LoginPage.jsx
│   │   ├── LogoutPage.jsx
│   │   └── RegisterPage.jsx
│   ├── test/
│   │   └── setup.js
│   ├── utils/
│   │   ├── auth.js
│   │   └── messages.js
│   ├── App.jsx
│   └── main.jsx
├── tests/
│   └── e2e/
│       └── pulsevote.spec.js
├── playwright.config.js
├── vitest.config.js
└── package.json
```


## Test Your Tests

### 22. Make a test fail deliberately

Do not just trust a green test result.

Temporarily change one assertion in:

```text
tests/e2e/pulsevote.spec.js
```

For example, change:

```js
await expect(
  page
).toHaveURL(
  /\/login$/
);
```

to:

```js
await expect(
  page
).toHaveURL(
  /\/wrong-page$/
);
```

Run:

```bash
npm run test:e2e
```

The test should fail.

Undo your temporary change.

Run:

```bash
npm run test:e2e
```

The test should pass again.

You have now proved that the test can detect incorrect behaviour.


## Git Check

Before committing:

```bash
git status
```

Make sure you are not committing:

* `.env`
* `node_modules`
* `coverage`
* `playwright-report`
* `test-results`
* passwords
* real JWTs
* private keys

Commit and push your changes.


## What We Have Done

You have now added:

* frontend component testing with Vitest
* React Testing Library
* mocked API calls
* authentication tests
* registration tests
* protected-route tests
* role-based dashboard tests
* polling and voting tests
* coverage reporting
* Playwright end-to-end tests
* mocked browser API requests
* a real-browser authentication and voting flow

We also used a test to find and fix a real bug in `PollCard`.

In the next activity, we will move the frontend checks into GitHub Actions so that linting, unit tests, vulnerability checks, SonarQube, Playwright and the production build run automatically.
