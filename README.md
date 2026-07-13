# Admin Frontend Login Flow — Beginner's Guide

This guide explains how login works between the **Admin Frontend**, **Admin Backend**, Google, PostgreSQL, and Redis.

It is written for someone who is new to Astro, backend APIs, authentication, JWT, cookies, and OAuth.

## 1. The applications involved

When you use the Admin login page, several applications work together:

```text
Admin Frontend (Astro + React)
http://localhost:4322
        |
        | HTTP/API requests
        v
Admin Backend (Fastify + TypeScript)
http://localhost:4000/admin
        |
        +----> PostgreSQL: permanent user data
        |
        +----> Redis: temporary login session data
        |
        +----> Google: Google account verification
```

### Admin Frontend

The Admin Frontend displays the login form and dashboard. Astro handles pages and server-side page protection, while React handles the interactive login form.

Local URL:

```text
http://localhost:4322
```

Project directory:

```text
Admin-Frontend/
```

### Admin Backend

The Admin Backend receives login requests, checks users, creates login tokens, and sends responses to the frontend.

Local base URL:

```text
http://localhost:4000/admin
```

Project directory:

```text
Admin-Backend/
```

### PostgreSQL

PostgreSQL permanently stores admin users. Admin authentication searches the table named:

```text
adminUser
```

It does **not** search the normal `User` table for Admin login.

### Redis

Redis stores temporary session information. PostgreSQL remembers who the user is permanently; Redis helps the backend track the current login session quickly.

## 2. Important authentication terms

### API

An API allows the frontend and backend to communicate.

For example:

```text
Frontend: "Please log in this user."
Backend:  "The details are valid. Here is the logged-in user."
```

### Endpoint

An endpoint is one specific backend URL that performs one job.

Examples:

```text
POST /admin/auth/login            Email/password login
GET  /admin/auth/google           Start Google login
GET  /admin/auth/google/callback  Receive Google's response
GET  /admin/auth/verify           Verify a login token
```

### JWT

JWT means **JSON Web Token**. It is a signed value created by the backend after successful login.

The token contains information such as:

```text
user ID
email
name
expiration time
session identifier
```

The signature prevents a user from safely changing the token themselves.

### Cookie

A cookie is a small value saved by the browser. This application stores the JWT in a cookie named:

```text
token
```

The browser sends this cookie when it requests protected pages such as `/dashboard`.

### OAuth

OAuth lets Google confirm the user's identity without giving the user's Google password to FC Labs.

```text
FC Labs asks Google: "Who is this user?"
Google asks the user to sign in and approve.
Google securely returns the user's identity to FC Labs.
```

## 3. How frontend API URLs are created

The Admin Frontend `.env` contains:

```env
PUBLIC_BACKEND_ADMIN_SERVICE_URL=http://localhost:4000/admin
```

The Axios client is created in:

```text
Admin-Frontend/src/lib/api.ts
```

Its simplified configuration is:

```ts
const api = axios.create({
  baseURL: import.meta.env.PUBLIC_BACKEND_ADMIN_SERVICE_URL,
  withCredentials: true,
});
```

The base URL and endpoint path are joined together:

```text
Base URL:      http://localhost:4000/admin
Endpoint path: /auth/login

Final URL:     http://localhost:4000/admin/auth/login
```

Common URL mappings:

| Frontend call | Final backend URL | Purpose |
|---|---|---|
| `/auth/public-key` | `http://localhost:4000/admin/auth/public-key` | Get password encryption key |
| `/auth/login` | `http://localhost:4000/admin/auth/login` | Email/password login |
| `/auth/loginWithGoogle` | `http://localhost:4000/admin/auth/loginWithGoogle` | Google ID-token login |
| `/auth/google` | `http://localhost:4000/admin/auth/google` | Start redirect-based Google login |
| `/auth/google/callback` | `http://localhost:4000/admin/auth/google/callback` | Receive Google callback |
| `/auth/verify` | `http://localhost:4000/admin/auth/verify` | Verify JWT |
| `/auth/profile` | `http://localhost:4000/admin/auth/profile` | Fetch current user profile |

## 4. What happens when the login page opens

The user opens:

```text
http://localhost:4322/login
```

The flow is:

```text
Browser requests /login
        |
        v
Astro runs Admin-Frontend/src/pages/login.astro
        |
        v
Astro checks whether a token cookie already exists
        |
        +----> Token exists: redirect to /dashboard
        |
        +----> No token: continue loading login page
        |
        v
Frontend calls GET /admin/auth/public-key
        |
        v
Backend returns its RSA public key
        |
        v
React login form is displayed
```

Why is the public key requested? The frontend uses it to encrypt the password before sending the password across the network.

Important files:

```text
Admin-Frontend/src/pages/login.astro
Admin-Frontend/src/components/signin/Login.tsx
Admin-Frontend/src/service/admin.service.ts
Admin-Backend/src/api/auth/auth.controller.ts
```

## 5. Email and password login flow

### Complete pattern

```text
User enters email and password
        |
        v
Login.tsx validates that both fields are present
        |
        v
Browser encrypts the password with the RSA public key
        |
        v
Frontend calls POST /admin/auth/login
        |
        v
Backend decrypts the password with its RSA private key
        |
        v
Backend searches the "adminUser" table by email
        |
        v
Backend uses bcrypt to compare the password
        |
        +----> Invalid: return an error
        |
        +----> Valid: continue
        |
        v
Backend creates a JWT
        |
        v
Backend stores the session identifier in Redis
        |
        v
Backend sets the JWT in the "token" cookie
        |
        v
Backend returns user information
        |
        v
Frontend redirects to /dashboard
```

### Where the frontend API call occurs

The React component calls `loginWithEmailPassword()` in:

```text
Admin-Frontend/src/components/signin/Login.tsx
```

The actual Axios request is in:

```text
Admin-Frontend/src/service/admin.service.ts
```

Simplified request:

```ts
api.post('/auth/login', {
  email,
  encryptedPassword,
});
```

Final request:

```text
POST http://localhost:4000/admin/auth/login
```

### What the backend does

The backend handler is `loginWithEmailPasswordHandler()` in:

```text
Admin-Backend/src/api/auth/auth.controller.ts
```

It performs these jobs:

1. Checks that email and encrypted password were supplied.
2. Decrypts the password with the private key.
3. searches `adminUser` using the email.
4. Compares the entered password with the stored bcrypt password hash.
5. Creates a JWT if the password is valid.
6. Saves session information in Redis.
7. Sets the JWT cookie.
8. Returns the user information to the frontend.

## 6. The current Google button flow

The current Admin Frontend uses `@react-oauth/google`.

### Complete pattern

```text
User clicks the Google button
        |
        v
Google popup opens
        |
        v
User selects a Google account
        |
        v
Google gives the frontend an ID token
        |
        v
Frontend calls POST /admin/auth/loginWithGoogle
        |
        v
Backend verifies the ID token with Google
        |
        v
Backend reads the email from the verified token
        |
        v
Backend searches "adminUser" for that exact email
        |
        +----> User missing: return "User not found"
        |
        +----> User found: continue
        |
        v
Backend creates JWT and Redis session
        |
        v
Backend sends token cookie and user data
        |
        v
Frontend redirects to /dashboard
```

The Google account must satisfy both conditions:

1. It must be allowed under Google Cloud's OAuth test users while the app is in testing mode.
2. The exact same email must exist in PostgreSQL's `adminUser` table.

### Where the current Google API call occurs

The Google component is in:

```text
Admin-Frontend/src/components/signin/Login.tsx
```

After Google supplies a credential, it calls `loginWithGoogle()`.

The Axios request is in:

```text
Admin-Frontend/src/service/admin.service.ts
```

Simplified request:

```ts
api.post('/auth/loginWithGoogle', {
  token,
});
```

Final request:

```text
POST http://localhost:4000/admin/auth/loginWithGoogle
```

### Localhost cookie problem in this flow

The `loginWithGoogleHandler()` currently uses production cookie settings:

```ts
secure: true,
sameSite: 'none',
domain: 'quickrecruit.com',
```

Those settings are meant for an HTTPS production website on `quickrecruit.com`. A browser rejects that cookie when the application runs on `http://localhost`.

The result looks like this:

```text
Google login succeeds
        |
        v
Backend sends production-only cookie
        |
        v
Browser rejects cookie on localhost
        |
        v
Frontend opens /dashboard
        |
        v
Dashboard cannot find token cookie
        |
        v
User returns to /login
```

## 7. Recommended Google redirect flow

The backend already contains another Google OAuth flow that is more suitable for the current local setup.

It starts at:

```text
http://localhost:4000/admin/auth/google
```

### Complete pattern

```text
User clicks "Continue with Google"
        |
        v
Browser navigates to GET /admin/auth/google
        |
        v
Admin Backend creates a Google authorization URL
        |
        v
Backend redirects the browser to Google
        |
        v
User signs in and approves
        |
        v
Google redirects to /admin/auth/google/callback
        |
        v
Backend exchanges Google's code for an access token
        |
        v
Backend requests the user's profile from Google
        |
        v
Backend searches "adminUser" by email
        |
        v
Backend creates JWT and Redis session
        |
        v
Backend sets a localhost-compatible token cookie
        |
        v
Backend redirects to http://localhost:4322/dashboard
```

In this flow, the frontend does not need an Axios request to begin login. It sends the whole browser to the backend:

```ts
window.location.href =
  `${import.meta.env.PUBLIC_BACKEND_ADMIN_SERVICE_URL}/auth/google`;
```

With the current `.env`, the generated URL is:

```text
http://localhost:4000/admin/auth/google
```

Google must be configured with this callback:

```text
http://localhost:4000/admin/auth/google/callback
```

The Admin Backend `.env` should contain:

```env
GOOGLE_REDIRECT_URI=http://localhost:4000/admin/auth/google/callback
FRONTEND_URL=http://localhost:4322
BACKEND_URL=http://localhost:4000
```

Never commit the Google client secret or other `.env` secrets.

## 8. Dashboard protection flow

The dashboard is not displayed merely because React redirects to `/dashboard`. Astro checks the authentication cookie before rendering it.

### Complete pattern

```text
Browser requests /dashboard
        |
        v
Astro runs Admin-Frontend/src/pages/dashboard.astro
        |
        v
dashboard.astro calls protectPage()
        |
        v
protectPage() reads the "token" cookie
        |
        +----> Cookie missing: redirect to /login
        |
        +----> Cookie exists: continue
        |
        v
Frontend calls GET /admin/auth/verify
and sends the token in the "token" request header
        |
        v
Backend checks the JWT signature and expiration
        |
        +----> Invalid/expired: return 401
        |                       and redirect to /login
        |
        +----> Valid: return decoded user data
        |
        v
Astro renders the dashboard
```

Page protection is implemented in:

```text
Admin-Frontend/src/lib/ProtectedModule.ts
```

The important logic is:

```ts
const token = cookies.get('token')?.value;

if (!token) {
  return {
    redirect: {
      method: 'redirect',
      path: '/login',
    },
  };
}
```

The frontend verifies an existing token through:

```text
GET http://localhost:4000/admin/auth/verify
```

## 9. Why a successful login can return to `/login`

This loop usually means the login credentials were accepted, but the dashboard could not find or verify the token.

```text
Login succeeds
   |
   v
Redirect to /dashboard
   |
   v
Cookie missing or invalid
   |
   v
protectPage() redirects to /login
```

Common causes:

1. The browser rejected the cookie because it has production-only settings.
2. Admin Backend is not running on port `4000`.
3. Redis is not running.
4. `JWT_SECRET` differs between login and verification processes.
5. The JWT has expired.
6. Frontend and backend are using different hosts, such as `localhost` and `127.0.0.1`, causing cookie confusion.
7. Google email does not exist in `adminUser`.

For local development, consistently use:

```text
Admin Frontend: http://localhost:4322
Admin Backend:  http://localhost:4000
```

Avoid mixing `localhost` and `127.0.0.1` in browser URLs when testing cookies.

## 10. How to run and test the complete Admin application

Start PostgreSQL:

```bash
sudo systemctl start postgresql
```

Start Redis:

```bash
sudo systemctl start redis-server
redis-cli ping
```

Expected Redis response:

```text
PONG
```

Start Admin Backend in terminal 1:

```bash
cd ~/Documents/Q-Connect/Admin-Backend
npm run dev
```

Test backend health:

```text
http://localhost:4000/health
```

Start Admin Frontend in terminal 2:

```bash
cd ~/Documents/Q-Connect/Admin-Frontend
npm run dev -- --port 4322
```

Open the login page:

```text
http://localhost:4322/login
```

Test the backend-managed Google flow directly:

```text
http://localhost:4000/admin/auth/google
```

## 11. Quick file map

| File | Responsibility |
|---|---|
| `Admin-Frontend/.env` | Admin Backend base URL and public Google client ID |
| `Admin-Frontend/src/pages/login.astro` | Server-renders login page and loads public key |
| `Admin-Frontend/src/components/signin/Login.tsx` | Login form and Google button behavior |
| `Admin-Frontend/src/lib/api.ts` | Axios base URL and shared HTTP settings |
| `Admin-Frontend/src/service/admin.service.ts` | Functions that make authentication API calls |
| `Admin-Frontend/src/lib/ProtectedModule.ts` | Protects Astro pages using the token cookie |
| `Admin-Frontend/src/pages/dashboard.astro` | Protected dashboard page |
| `Admin-Backend/.env` | Database, Google, JWT, frontend, and backend configuration |
| `Admin-Backend/src/app.ts` | Creates Fastify and registers routes/CORS |
| `Admin-Backend/src/api/auth/auth.routes.ts` | Defines authentication endpoint paths |
| `Admin-Backend/src/api/auth/auth.controller.ts` | Performs login, callback, token, cookie, and verification work |
| `Admin-Backend/src/api/auth/auth.dao.ts` | Queries `adminUser`, checks passwords, and manages Redis sessions |
| `Admin-Backend/src/pre-handler/auth.prehandler.ts` | Reads JWT from cookie or request headers |

## 12. Final recommended Google login pattern

```text
Admin login page
        |
        v
Continue with Google button
        |
        v
GET http://localhost:4000/admin/auth/google
        |
        v
Google sign-in and consent
        |
        v
GET http://localhost:4000/admin/auth/google/callback
        |
        v
Find matching PostgreSQL adminUser
        |
        v
Create JWT + save Redis session
        |
        v
Set token cookie
        |
        v
Redirect to http://localhost:4322/dashboard
        |
        v
Astro reads token cookie
        |
        v
GET http://localhost:4000/admin/auth/verify
        |
        v
Render dashboard
```

The most important idea is:

> A redirect to `/dashboard` does not prove that login is complete. The dashboard is displayed only when the browser has a valid `token` cookie and the backend successfully verifies that JWT.
