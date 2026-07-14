# QConnect Admin — User Flows and Screen Guide

This document explains what a Super Admin does in the QConnect Admin application. It is written as a product-flow guide: each module describes the user goal, the screens involved, the API/data path, and the expected result.

> GitHub renders the Mermaid diagrams below automatically. The HTML wireframes are intentionally simple so they remain readable in GitHub and other Markdown viewers.

## 1. Application map

```mermaid
flowchart TD
    A[Super Admin] --> B[Sign in]
    B --> C[Dashboard]
    C --> D[Institutions]
    D --> E[Colleges]
    E --> F[Users and Staff]
    E --> G[Roles and Permissions]
    C --> H[Subscription Plans]
    H --> E
    C --> I[Question Bank]
    C --> J[Settings]
    J --> K[Password Recovery]
```

### Business hierarchy

```mermaid
flowchart TB
    I[Institution / Enterprise]
    C[College / Workspace]
    U[Users and Staff]
    R[Roles and Permissions]
    P[Plan Subscription]
    Q[Question Bank]
    I --> C
    C --> U
    C --> R
    C --> P
    C --> Q
```

<table>
<tr><th>Layer</th><th>Purpose</th><th>Owner of the action</th></tr>
<tr><td>Authentication</td><td>Proves who the Admin is and creates a session.</td><td>Admin + Google/email</td></tr>
<tr><td>Institution</td><td>Top-level customer organization.</td><td>Super Admin</td></tr>
<tr><td>College</td><td>Workspace belonging to an institution.</td><td>Super Admin</td></tr>
<tr><td>Users/Roles</td><td>People and permissions inside a college.</td><td>Super Admin</td></tr>
<tr><td>Plans</td><td>Features, credits, and subscriptions assigned to colleges.</td><td>Super Admin</td></tr>
<tr><td>Questions</td><td>Shared general, MCQ, and coding content.</td><td>Super Admin</td></tr>
</table>

## 2. Sign in and session flow

### User goal

Sign in with Google or email/password, then open protected Admin pages.

```mermaid
sequenceDiagram
    actor A as Admin
    participant F as Admin Frontend
    participant G as Google
    participant B as Admin Backend
    participant DB as PostgreSQL
    participant R as Redis

    A->>F: Open /login
    F->>F: Astro checks token cookie
    A->>F: Click Google login
    F->>G: Open account picker
    G-->>F: Return Google ID token
    F->>B: POST /auth/loginWithGoogle { token }
    B->>G: Verify ID token and audience
    G-->>B: Verified email claim
    B->>DB: Find email in adminUser
    alt Email does not exist
        DB-->>B: No record
        B-->>F: 404 User not found
    else Email exists
        DB-->>B: Admin record
        B->>R: Save login session
        B-->>F: Set HttpOnly JWT cookie + user data
        F->>F: Redirect to /dashboard
    end
```

<table>
<tr><th>Screen wireframe</th><th>Functional behavior</th></tr>
<tr><td>
<pre>
┌────────────────────────────┐
│ QConnect Admin             │
│ Sign in to your account    │
│ [ Email                  ]  │
│ [ Password               ]  │
│ [ Sign in               ]   │
│ [ Continue with Google ]   │
│ Forgot password?           │
└────────────────────────────┘
</pre>
</td><td>Google returns an ID token to the browser. The browser sends only the token to the backend. The backend extracts the email, checks <code>adminUser</code>, creates the application JWT, and sets the cookie.</td></tr>
</table>

## 3. Institution management

### User goal

Create and maintain the top-level customer organization before adding colleges.

```mermaid
flowchart LR
    A[Institution list] --> B[Add institution]
    B --> C[Enter identity, contact, address]
    C --> D{Validation succeeds?}
    D -- No --> C
    D -- Yes --> E[POST /enterprise]
    E --> F[Institution created]
    F --> G[Open institution details]
    G --> H[View its colleges]
    A --> I[Search/filter/paginate]
    G --> J[Edit or change status]
```

```mermaid
flowchart TD
    Institution[Institution details]
    Institution --> Summary[Summary]
    Institution --> Contact[Contact and address]
    Institution --> Colleges[Child colleges]
    Institution --> Status[Active / inactive / suspended]
```

<pre>
┌─────────────────────────────────────────────────────────┐
│ Institutions                         [+ Add institution] │
│ [Search] [Status filter]                                  │
├──────────────┬─────────────┬──────────────┬──────────────┤
│ Name         │ Type        │ Status       │ Actions      │
│ ABC Group    │ University  │ ACTIVE       │ View Edit    │
└──────────────┴─────────────┴──────────────┴──────────────┘
</pre>

## 4. College management

### User goal

Add a college under an institution and maintain its accreditation, contact, and status information.

```mermaid
flowchart LR
    A[Select institution] --> B[Open college list]
    B --> C[Add college]
    C --> D[Choose parent institution]
    D --> E[Enter college and accreditation details]
    E --> F[POST /workspace/:enterpriseId]
    F --> G[College workspace created]
    G --> H[Open college details]
    H --> I[Manage users, roles, and plans]
```

```mermaid
flowchart TD
    College[College / Workspace]
    College --> Overview[Overview and contact]
    College --> Accreditation[UGC, AICTE, NAAC, NBA, NIRF]
    College --> Users[Users and staff]
    College --> Roles[Roles and permissions]
    College --> Plans[Assigned subscriptions]
```

## 5. User and staff management

### User goal

Create people inside a college, connect them to a role, and control their status.

```mermaid
flowchart LR
    A[College details] --> B[Users tab]
    B --> C[Add user]
    C --> D[Enter name, email, staff flag]
    D --> E[Choose role]
    E --> F{Valid data?}
    F -- No --> D
    F -- Yes --> G[POST /user/:workspaceId]
    G --> H[Create User]
    H --> I[Create UserWorkspace link]
    I --> J[Create UserRole link]
    J --> K[User appears in college list]
    B --> L[Edit user / change status]
```

<pre>
┌─────────────────────────────────────────────────────────┐
│ College: ABC Engineering                 [ + Add user ]  │
│ [Search users] [Role filter] [Status filter]             │
├────────────┬──────────────────┬───────────┬─────────────┤
│ Name       │ Email            │ Role      │ Status      │
│ Priya Shah │ priya@abc.edu    │ Manager   │ ACTIVE      │
└────────────┴──────────────────┴───────────┴─────────────┘
</pre>

## 6. Roles and permissions

### User goal

Define what a college role can view, create, edit, or delete.

```mermaid
flowchart LR
    A[College details] --> B[Roles tab]
    B --> C[Create role]
    C --> D[Name role]
    D --> E[Load modules and actions]
    E --> F[Select permissions]
    F --> G[POST /role/:workspaceId]
    G --> H[Role saved]
    H --> I[Assign role to a user]
    B --> J[Edit or delete role]
```

```mermaid
flowchart TD
    Role[College Manager]
    Role --> Students[Students]
    Role --> Assessments[Assessments]
    Role --> Reports[Reports]
    Students --> SV[View ✓]
    Students --> SC[Create ✓]
    Students --> SD[Delete ✗]
    Assessments --> AV[View ✓]
    Reports --> RV[View ✓]
```

## 7. Subscription plans

### User goal

Create a product plan, configure its features, and assign it to a college for a date range.

```mermaid
flowchart LR
    A[Plans list] --> B[Add or edit plan]
    B --> C[Enter name, type, price]
    C --> D[Add feature identifiers and values]
    D --> E[POST or PUT /plans]
    E --> F[Plan saved]
    F --> G[Open plan details]
    G --> H[Assign college]
    H --> I[Select college and dates]
    I --> J[POST /plans/:id/workspaces]
    J --> K[Active subscription]
```

```mermaid
flowchart TD
    Plan[Plan]
    Plan --> Features[Plan features]
    Features --> Credits[Credits / quantities]
    Features --> Access[Module access flags]
    Plan --> Subscription[Workspace subscription]
    Subscription --> College[College]
    Subscription --> Dates[Start and end dates]
```

<pre>
┌─────────────────────────────────────────────────────────┐
│ Professional Plan                         [Assign college]│
│ Monthly: ₹___   Yearly: ₹___   Status: ACTIVE           │
├───────────────────────────┬─────────────────────────────┤
│ Feature                   │ Value                       │
│ Assessment attempts       │ 100                         │
│ Coding module             │ Enabled                     │
├───────────────────────────┴─────────────────────────────┤
│ Assigned colleges: ABC Engineering · 01 Jan–31 Dec       │
└─────────────────────────────────────────────────────────┘
</pre>

## 8. Question bank and authoring

### User goal

Create reusable general, MCQ, and coding questions with tags, companies, visibility, and test cases.

```mermaid
flowchart LR
    A[Question bank] --> B[Search/filter existing questions]
    A --> C[Add question]
    C --> D{Choose type}
    D --> E[General]
    D --> F[MCQ]
    D --> G[Coding]
    E --> H[Enter content, difficulty, tags]
    F --> I[Add options and correct answers]
    G --> J[Add languages, input/output, test cases]
    H --> K[POST /question]
    I --> K
    J --> K
    K --> L[Question saved]
    L --> M[View, edit, or delete]
```

```mermaid
flowchart TD
    Q[Question editor]
    Q --> Common[Title, description, category, difficulty]
    Q --> Tags[Tags, companies, images, visibility]
    Q --> Type{Question type}
    Type --> MCQ[Options + one/multiple correct answers]
    Type --> Coding[Languages + input/output + visible/hidden tests]
    Type --> General[General response content]
```

<pre>
┌─────────────────────────────────────────────────────────┐
│ Add question                                             │
│ Type: ( General ▼ )   Difficulty: ( Medium ▼ )           │
│ Title: [                                              ]  │
│ Description: [                                        ]   │
│ Tags: [arrays] [sorting]       Visibility: [Public ▼]   │
│                                                         │
│ Type-specific editor                                     │
│   MCQ:     [Option A] [✓ correct]                       │
│   Coding:  [Language] [Input] [Output] [+ Test case]    │
│                                                         │
│                         [Cancel] [Save question]        │
└─────────────────────────────────────────────────────────┘
</pre>

## 9. Settings and password recovery

```mermaid
flowchart LR
    A[Settings] --> B[Profile tab]
    B --> C[View profile]
    C --> D[Update phone]
    D --> E[PUT /auth/profile]
    A --> F[Password tab]
    F --> G[Request reset email]
    G --> H[/forgot-password]
    H --> I[Validate reset token]
    I --> J[/set-password]
    J --> K[Save new bcrypt password]
    K --> L[Return to /login]
```

## 10. Dashboard, reports, contact, and utilities

```mermaid
flowchart TD
    A[Dashboard] --> B[Welcome placeholder]
    C[Reports menu] --> D[Monthly route not implemented]
    C --> E[Annual route not implemented]
    F[Contact form] --> G[Frontend submit exists]
    G --> H[Backend route currently absent]
    I[Session utility] --> J[Export cookies]
    I --> K[Import cookies]
```

| Module | Current user experience | Current status |
|---|---|---|
| Dashboard | Protected landing page with welcome content. | Placeholder |
| Reports | Navigation links exist, but report pages/APIs are absent. | Not implemented |
| Contact | Form exists, but the Admin Backend route is absent. | Partial/broken |
| Session import/export | Support utility for moving cookie/session information. | Internal utility |

## 11. End-to-end example: onboard a new college

```mermaid
sequenceDiagram
    actor A as Super Admin
    A->>A: Sign in
    A->>A: Open Institutions
    A->>A: Create Institution
    A->>A: Open institution details
    A->>A: Add College / Workspace
    A->>A: Add college users
    A->>A: Create Manager role
    A->>A: Assign Manager role to user
    A->>A: Create subscription plan
    A->>A: Assign plan to college
    A->>A: Add shared questions
    A->>A: Review college status
```

### Completion checklist

- [ ] Admin account exists in `adminUser`.
- [ ] Institution is created and active.
- [ ] College is created under the institution.
- [ ] Initial users are added to the college.
- [ ] Roles and permissions are configured.
- [ ] A plan is assigned with valid start/end dates.
- [ ] Required questions are created or reviewed.
- [ ] The college can proceed with its configured access.

## 12. Source map

| Area | Frontend | Backend |
|---|---|---|
| Authentication | `Admin-Frontend/src/pages/login.astro`, `src/components/signin/Login.tsx` | `Admin-Backend/src/api/auth/` |
| Institutions | `Admin-Frontend/src/pages/manageInstitution/` | `Admin-Backend/src/api/enterprise/` |
| Colleges | `Admin-Frontend/src/pages/manageColleges/` | `Admin-Backend/src/api/workspace/` |
| Users | `Admin-Frontend/src/components/manageColleges/` | `Admin-Backend/src/api/user/` |
| Roles | College detail role components | `Admin-Backend/src/api/role/` |
| Plans | `Admin-Frontend/src/pages/managePlans/` | `Admin-Backend/src/api/plans/` |
| Questions | `Admin-Frontend/src/pages/manageQuestions/` | `Admin-Backend/src/api/question/` |

