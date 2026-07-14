# FC Labs Admin — User Flows and Screen Guide

This document explains what a Super Admin does in the FC Labs Admin application. It also records the QConnect-to-FC-Labs migration so the removed Institution layer and the new learning-operations modules are easy to compare.

> GitHub renders the Mermaid diagrams below automatically. The HTML wireframes are intentionally simple so they remain readable in GitHub and other Markdown viewers.

## 1. Application map

```mermaid
flowchart TD
    A[Super Admin] --> B[Sign in]
    B --> C[Dashboard]
    C --> D[Colleges]
    D --> E[Users and Staff]
    D --> F[Batches]
    D --> G[Roles and Permissions]
    C --> H[Subscription Plans]
    H --> D
    C --> I[Question Bank]
    C --> J[Roadmaps]
    J --> K[Tasks and Assessments]
    C --> L[Trainers]
    C --> M[Reports and Monitoring]
    C --> N[Settings]
    N --> O[Password Recovery]
```

### Business hierarchy

```mermaid
flowchart TB
    A[FC Labs Admin]
    C[College / Workspace]
    U[Users and Staff]
    R[Roles and Permissions]
    P[Plan Subscription]
    B[Batches]
    M[Roadmaps]
    T[Trainers]
    Q[Question Bank]
    A --> C
    C --> U
    C --> R
    C --> P
    C --> B
    C --> M
    C --> T
    C --> Q
```

<table>
<tr><th>Layer</th><th>Purpose</th><th>Owner of the action</th></tr>
<tr><td>Authentication</td><td>Proves who the Admin is and creates a session.</td><td>Admin + Google/email</td></tr>
<tr><td>College</td><td>Top-level customer organization; backend name: Workspace.</td><td>Super Admin</td></tr>
<tr><td>Users/Roles</td><td>People and permissions inside a college.</td><td>Super Admin</td></tr>
<tr><td>Batches</td><td>Groups of students used for roadmap delivery.</td><td>Super Admin</td></tr>
<tr><td>Plans</td><td>Features, credits, and subscriptions assigned to colleges.</td><td>Super Admin</td></tr>
<tr><td>Roadmaps/Trainers</td><td>Learning journeys, delivery ownership, tasks, and assessments.</td><td>Admin + Trainer</td></tr>
<tr><td>Questions</td><td>Shared general, MCQ, and coding content used by assessments.</td><td>Admin + Trainer</td></tr>
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
│ FC Labs Admin              │
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

## 3. Legacy QConnect institution flow (removed in FC Labs)

### User goal

In QConnect, the Admin created an Institution/Enterprise first and then added colleges below it. FC Labs removes this step: the Admin creates a College/Workspace directly. The flow remains here only to make the migration explicit.

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

## 4. College management (FC Labs primary organization)

### User goal

Add and maintain a college directly. There is no Institution/Enterprise parent step in FC Labs.

```mermaid
flowchart LR
    A[Open college list] --> B[Add college]
    B --> C[Enter college and accreditation details]
    C --> D[POST /workspace]
    D --> E[College workspace created]
    E --> F[Open college details]
    F --> G[Manage users, batches, roles, plans, roadmaps, and trainers]
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
    A->>A: Sign in to FC Labs Admin
    A->>A: Open Colleges
    A->>A: Create College / Workspace
    A->>A: Open college details
    A->>A: Create batches
    A->>A: Add college users
    A->>A: Create roles and assign permissions
    A->>A: Create subscription plan
    A->>A: Assign plan to college
    A->>A: Create roadmap and content
    A->>A: Assign trainer to batch and roadmap
    A->>A: Monitor tasks, assessments, and progress
    A->>A: Review college status
```

### Completion checklist

- [ ] Admin account exists in `adminUser`.
- [ ] College is created and active.
- [ ] Initial users are added to the college.
- [ ] Batches are created and students are assigned.
- [ ] Roles and permissions are configured.
- [ ] A plan is assigned with valid start/end dates.
- [ ] Roadmaps, trainers, tasks, and assessments are configured.
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

---

# FC Labs evolution from QConnect

The original QConnect Admin application manages institutions, colleges, users, roles, plans, questions, and Admin settings. The FC Labs version changes the organization model: the Institution/Enterprise level is removed and College/Workspace becomes the top-level organization.

## QConnect versus FC Labs

| Capability | Existing QConnect Admin | New FC Labs Admin |
|---|---|---|
| Top-level organization | Institution (`Enterprise`) | College (`Workspace`) |
| College relationship | A college belongs to an institution | A college is managed directly by FC Labs |
| Users and roles | Belong to a college/workspace | Still belong to a college/workspace |
| Batches | Not a primary Admin workflow | Core entity for grouping students |
| Subscription plans | Create plans and assign them to colleges | Same capability, now college-first |
| Question bank | Shared general, MCQ, and coding questions | Same capability, with roadmap-linked tasks and assessments |
| Roadmaps | Not available | Create, assign, monitor, and report roadmap progress |
| Trainers | Not available | Create, assign, track visits, activity, and performance |
| Assessments and tasks | Question-authoring capability exists | Trainers create and monitor them from roadmap/batch context |
| Student progress | Limited in the Admin scope | Chapter, topic, task, assessment, batch, and roadmap progress |
| Dashboard | Welcome placeholder | Multi-college, trainer, student, assessment, task, and roadmap metrics |
| Reports | Monthly/annual routes are not implemented | Required for college, trainer, roadmap, and student monitoring |
| Branding | QConnect labels and URLs | FC Labs labels, URLs, and product language |
| Authorization | Fine-grained enforcement is incomplete | Add Admin, Trainer, and other restricted roles |

## Organization model comparison

```mermaid
flowchart LR
    subgraph Q[Existing QConnect]
        QA[FC Labs Admin] --> QE[Institution / Enterprise]
        QE --> QW[College / Workspace]
        QW --> QU[Users]
        QW --> QR[Roles]
        QW --> QP[Plans]
        QW --> QQ[Questions]
    end
    subgraph F[New FC Labs]
        FA[FC Labs Admin] --> FW[College / Workspace]
        FW --> FU[Users]
        FW --> FR[Roles]
        FW --> FP[Plans]
        FW --> FB[Batches]
        FW --> FM[Roadmaps]
        FW --> FT[Trainers]
        FM --> FQ[Tasks and assessments]
    end
```

## FC Labs target hierarchy

```mermaid
flowchart TD
    A[FC Labs Admin]
    A --> C[Colleges / Workspaces]
    A --> SQ[Shared question bank]
    A --> SP[Subscription management]
    A --> RM[Roadmap management]
    A --> TM[Trainer management]
    A --> RA[Reports and analytics]
    C --> U[Users and staff]
    C --> R[Roles and permissions]
    C --> B[Batches]
    C --> P[Assigned plans]
    C --> Q[College question access]
    C --> M[Roadmaps]
    C --> T[Trainers]
    M --> CH[Modules and chapters]
    M --> TO[Topics and materials]
    M --> TA[Tasks and assessments]
```

## 13. FC Labs college-first onboarding flow

### User goal

Onboard a college directly, create its student groups, and prepare learning delivery without an Institution/Enterprise step.

```mermaid
sequenceDiagram
    actor A as FC Labs Admin
    participant F as Admin App
    participant B as Backend
    participant DB as Database

    A->>F: Open Colleges
    A->>F: Add college
    F->>B: POST /workspace
    B->>DB: Create College / Workspace
    DB-->>B: College created
    B-->>F: College details
    A->>F: Add students and staff
    F->>B: Create users and college links
    A->>F: Create batches
    F->>B: Save batch and student assignments
    B-->>F: College onboarding ready
```

```mermaid
flowchart LR
    A[College list] --> B[Add college]
    B --> C[Save college details]
    C --> D[Create student users]
    D --> E[Create batches]
    E --> F[Assign students to batches]
    F --> G[Create roadmap assignments]
    G --> H[Assign trainers]
    H --> I[College ready for delivery]
```

<pre>
┌──────────────────────────────────────────────────────────────┐
│ FC Labs · Colleges                         [ + Add college ]   │
├───────────────┬──────────────┬───────────┬────────────────────┤
│ College       │ Batches      │ Students  │ Actions            │
│ ABC College   │ 4            │ 240       │ View · Edit        │
├───────────────┴──────────────┴───────────┴────────────────────┤
│ College details: Users | Batches | Roadmaps | Trainers | Plans │
└──────────────────────────────────────────────────────────────┘
</pre>

## 14. Batch management flow

### User goal

Divide a large college cohort into manageable groups before assigning roadmaps and trainers.

```mermaid
flowchart TD
    C[College] --> B1[Batch 1]
    C --> B2[Batch 2]
    C --> B3[Batch 3]
    C --> B4[Batch 4]
    B1 --> S1[Assigned students]
    B2 --> S2[Assigned students]
    B3 --> S3[Assigned students]
    B4 --> S4[Assigned students]
    B1 --> R1[Roadmap 1]
    B2 --> R2[Roadmap 2]
    B3 --> R3[Roadmap 3]
    B4 --> R4[Roadmap 4]
```

```mermaid
flowchart LR
    A[Open college] --> B[Batches tab]
    B --> C[Create batch]
    C --> D[Name batch and set schedule]
    D --> E[Select or import students]
    E --> F[Save batch]
    F --> G[Assign roadmap and trainer]
```

## 15. Roadmap management flow

### User goal

Build a learning journey for a batch, including content, deadlines, tasks, assessments, and completion tracking.

```mermaid
flowchart LR
    A[Roadmaps] --> B[Create roadmap]
    B --> C[Add modules]
    C --> D[Add chapters and topics]
    D --> E[Attach materials]
    E --> F[Add tasks and assessments]
    F --> G[Set deadlines]
    G --> H[Assign roadmap to batch]
    H --> I[Assign trainer]
    I --> J[Publish roadmap]
    J --> K[Monitor progress]
```

```mermaid
flowchart TD
    R[Roadmap]
    R --> M1[Module 1]
    R --> M2[Module 2]
    M1 --> C1[Chapter 1]
    M1 --> C2[Chapter 2]
    C1 --> T1[Topic + learning material]
    C2 --> T2[Topic + learning material]
    T1 --> A1[Task]
    T2 --> A2[Assessment]
    A1 --> P1[Submission and review]
    A2 --> P2[Attempt and result]
```

<pre>
┌──────────────────────────────────────────────────────────────┐
│ Roadmap: Full Stack Foundations              [Publish]        │
├──────────────────────────────────────────────────────────────┤
│ Module 1 · Web basics                         80% complete     │
│   ✓ Chapter 1 · HTML                           100%            │
│   ✓ Chapter 2 · CSS                            100%            │
│   ○ Chapter 3 · JavaScript                      40%            │
│ Module 2 · Backend                              0%             │
│                                                              │
│ Assigned: Batch 1 · Trainer: Trainer A · Deadline: 30 Jun    │
└──────────────────────────────────────────────────────────────┘
</pre>

## 16. Trainer management and tracking flow

### User goal

Allocate trainers to colleges, batches, and roadmaps, then monitor their delivery activity.

```mermaid
flowchart LR
    A[Trainers] --> B[Add trainer]
    B --> C[Enter profile and skills]
    C --> D[Activate trainer]
    D --> E[Assign college]
    E --> F[Assign batch]
    F --> G[Assign roadmap]
    G --> H[Trainer logs in with Trainer role]
    H --> I[Updates topics and chapters]
    I --> J[Creates tasks and assessments]
    J --> K[Reviews submissions]
    K --> L[Admin monitors activity and performance]
```

```mermaid
flowchart TD
    T[Trainer profile]
    T --> A[Current college]
    T --> V[Visited colleges]
    T --> B[Assigned batches]
    T --> R[Active roadmaps]
    T --> C[Completed roadmaps]
    T --> AS[Assessments created]
    T --> TA[Tasks created]
    T --> ST[Students trained]
    T --> PR[Progress and performance]
```

<pre>
┌──────────────────────────────────────────────────────────────┐
│ Trainer: Trainer A                              Status ACTIVE  │
├─────────────────────┬────────────────────────────────────────┤
│ Current college     │ ABC College                            │
│ Assigned batches    │ Batch 1, Batch 3                       │
│ Active roadmaps     │ Full Stack Foundations                  │
│ Students trained    │ 120                                     │
│ Assessments/tasks   │ 8 assessments · 14 tasks                │
├─────────────────────┴────────────────────────────────────────┤
│ Activity: topics taught · chapters completed · college visits │
└──────────────────────────────────────────────────────────────┘
</pre>

## 17. Role-based application access

```mermaid
flowchart TD
    L[Login] --> D{Role}
    D -->|FC Labs Admin| A[All colleges, plans, roadmaps, trainers, reports]
    D -->|Trainer| T[Assigned colleges, batches, roadmaps, tasks, assessments, progress]
    D -->|Student| S[User application: assigned roadmap, tasks, assessments, own progress]
```

| Role | Application | Main responsibilities |
|---|---|---|
| FC Labs Admin | Admin application | Create colleges, batches, roadmaps, plans, trainers; monitor delivery and outcomes. |
| Trainer | Admin application | Work only with assigned colleges/batches/roadmaps; teach topics; create assessments/tasks; review submissions. |
| Student | User application | View assigned roadmap; complete topics; attend assessments; submit tasks; track personal progress. |

## 18. Student delivery and monitoring flow

```mermaid
sequenceDiagram
    actor T as Trainer
    actor S as Student
    participant U as User App
    participant B as Backend
    participant A as Admin App

    T->>B: Mark topic/chapter complete
    B-->>U: Update assigned roadmap progress
    S->>U: Open roadmap
    U-->>S: Show completed and pending content
    T->>B: Create task or assessment
    B-->>U: Publish task/assessment to batch
    S->>U: Submit task / attend assessment
    U->>B: Save submission/result
    B-->>A: Update batch, student, and roadmap metrics
    A-->>A: Admin reviews trainer and completion dashboards
```

## 19. FC Labs implementation priorities

| Priority | Workstream | Required outcome |
|---:|---|---|
| 1 | Organization migration | Remove Institution/Enterprise dependencies and make Workspace/College primary. |
| 2 | Batch management | Create batches and assign students to batches. |
| 3 | Roadmaps | Build roadmap content, assignment, publishing, deadlines, and progress tracking. |
| 4 | Trainers | Add Trainer role, profiles, allocations, visits, activity, and performance. |
| 5 | Delivery objects | Link tasks and assessments to roadmap, chapter, topic, batch, and trainer. |
| 6 | Monitoring | Add student, batch, trainer, college, and roadmap dashboards. |
| 7 | Reports | Add monthly/annual and operational reports. |
| 8 | Security | Complete role/module authorization and environment-aware OAuth cookies. |
| 9 | Product migration | Replace QConnect branding, URLs, labels, and content with FC Labs. |

## 20. Final FC Labs operating model

```mermaid
flowchart TB
    A[FC Labs Admin] --> C[College]
    C --> B[Batches]
    B --> S[Students]
    C --> R[Roadmap]
    R --> M[Modules, chapters, topics]
    M --> TA[Tasks and assessments]
    C --> T[Trainer]
    T --> R
    S --> U[User application]
    U --> P[Submissions and progress]
    P --> X[Admin monitoring and reports]
    T --> X
```

FC Labs therefore moves from an institution-first administration model to a college-first learning-operations model. The new value is the complete delivery loop: college onboarding → batch creation → roadmap assignment → trainer delivery → student activity → task/assessment results → progress and performance reporting.
