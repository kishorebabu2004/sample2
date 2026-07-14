
# Admin Application Features — Beginner-Friendly Analysis

This document explains what the Q Connect Admin application currently does by analyzing both:

```text
Admin-Frontend/
Admin-Backend/
```

The Admin application is mainly for a **product owner or Super Admin** who manages the institutions and colleges that subscribe to the product.

## Visual functional guide

The following diagrams explain how the application behaves. They are intentionally written as HTML/CSS so that a Markdown viewer that supports inline HTML can render them as a visual guide. The tables below remain the detailed implementation reference.

> **GitHub Preview Note**
>
> GitHub strips `<style>` tags and most custom CSS from Markdown for security reasons.
> The original visual HTML/CSS section has been removed because it cannot render correctly in GitHub's preview.
> If you want visual diagrams on GitHub, use Mermaid diagrams, Markdown tables, or images instead.


## 1. Simple overview

Think of the Admin application as the FC Labs **head office**:

```text
FC Labs Admin
   |
   +----> Manages institutions
   |
   +----> Manages colleges under institutions
   |
   +----> Manages users and roles inside colleges
   |
   +----> Creates subscription plans
   |
   +----> Assigns plans to colleges
   |
   +----> Manages the shared question bank
   |
   +----> Manages its own login and profile
```

The business hierarchy used by the code is:

```text
Institution (backend name: Enterprise)
   |
   +----> College (backend name: Workspace)
              |
              +----> Users
              +----> Roles and permissions
              +----> Assigned subscription plans
              +----> College-specific questions
```

## 2. Status legend

The tables below use these labels:

| Status | Meaning |
|---|---|
| **Implemented** | Frontend page, frontend API function, and backend route are present |
| **Partial** | Some code exists, but an important part is missing, disabled, or inconsistent |
| **Placeholder** | A visible page or menu exists, but it does not yet provide the intended functionality |
| **Not implemented** | The menu or frontend expects the feature, but the required page/API is absent |
| **Internal utility** | Developer/support functionality rather than a normal product feature |

## 3. Main Admin features

| No. | Feature area | What the Admin can do | Frontend page | Backend module | Status |
|---:|---|---|---|---|---|
| 1 | Authentication | Sign in with email/password or Google, sign out, verify session | `/login` | `auth` | **Implemented**, with a localhost Google-cookie issue in one flow |
| 2 | Password recovery | Request reset email, validate reset token, set a new password | `/forgot-password`, `/set-password` | `auth` | **Implemented** |
| 3 | Institution management | List, search, filter, add, view, edit, and change institution status | `/manageInstitution` | `enterprise` | **Implemented** |
| 4 | College management | List, search, filter, add, view, edit, and activate/deactivate colleges | `/manageColleges` | `workspace` | **Implemented** |
| 5 | Institution-college hierarchy | View colleges belonging to a selected institution and add colleges under it | `/manageInstitution/[institutionId]` | `enterprise`, `workspace` | **Implemented** |
| 6 | College users | List, add, edit, mark staff, assign role, and change user status | `/manageColleges/[collegeId]` | `user` | **Implemented** |
| 7 | College roles | Create roles, view roles, delete roles, and configure module permissions | `/manageColleges/[collegeId]` | `role` | **Implemented**, but application-wide permission enforcement is **partial** |
| 8 | Subscription plans | List, search, create, view, and edit product plans | `/managePlans` | `plans` | **Implemented** |
| 9 | Plan features | Attach credit/access identifiers and values to a plan | `/managePlans/addPlan`, `/managePlans/[id]/edit` | `plans` | **Implemented** |
| 10 | Assign plans to colleges | Assign a plan to a college with start/end dates and change subscription status | `/managePlans/[id]` | `plans` | **Implemented** |
| 11 | College plan view | View plans currently assigned to a selected college | `/manageColleges/[collegeId]` | `plans` | **Implemented** |
| 12 | Question bank | List, search, filter, paginate, view, create, edit, and delete questions | `/manageQuestions` | `question` | **Implemented** |
| 13 | MCQ authoring | Create single-answer or multiple-answer MCQs with options | `/manageQuestions/addQuestion` | `question` | **Implemented** |
| 14 | Coding-question authoring | Configure allowed languages, input/output structures, and visible/hidden test cases | `/manageQuestions/addQuestion` | `question` | **Implemented** |
| 15 | General questions | Create non-MCQ, non-coding question content | `/manageQuestions/addQuestion` | `question` | **Implemented** |
| 16 | Profile settings | View profile and update phone number | `/settings?tab=profile` | `auth` | **Implemented** |
| 17 | Password settings | Send a password reset link to the logged-in Admin | `/settings?tab=password` | `auth` | **Implemented** |
| 18 | Dashboard | Display product-wide statistics and performance | `/dashboard` | None | **Placeholder** — currently only displays a welcome message |
| 19 | Reports | Monthly and annual reports | `/reports/monthly`, `/reports/annual` | None | **Not implemented** — links exist, but pages/APIs do not |
| 20 | Contact form | Submit a contact request | `/contactUs` | Expected `contact` module | **Partial/broken** — frontend call exists, Admin Backend route is absent |
| 21 | Session import/export | Export or import session/cookie information | `/export-cookie`, `/import-cookie` | `session` | **Internal utility** |
| 22 | Error handling | Display 403, 404, and 500 pages | `/403`, `/404`, `/500` | Not required | **Implemented** |

## 4. Institution management

In the frontend, an institution represents a top-level organization. In the backend and database, it is called an `Enterprise`.

### What the Admin can do

| Action | Description | API pattern |
|---|---|---|
| List institutions | View institutions using pagination, searching, and filters | `GET /admin/enterprise` |
| Add institution | Create a new institution and its details/addresses | `POST /admin/enterprise` |
| View institution | Open the details of one institution | `GET /admin/enterprise/:enterpriseId` |
| Edit institution | Update institution information | `PUT /admin/enterprise/:enterpriseId` |
| Change status | Activate, suspend, or deactivate an institution | `PUT /admin/enterprise/status` |
| Simple list | Load institution IDs/names for dropdowns | `GET /admin/enterprise/simple` |
| View its colleges | List colleges belonging to the institution | `GET /admin/enterprise/:enterpriseId/workspaces` |

### Institution information handled by the UI

| Group | Example fields |
|---|---|
| Identity | Institution name, institution code, institution type |
| Contact | Primary contact name, email, phone number, country code |
| Public information | Website, logo, established year, description |
| Address | Address lines, city, state, country, pincode, primary-address flag |
| Workflow | Status and onboarding status |

### Main code locations

```text
Admin-Frontend/src/pages/manageInstitution/
Admin-Frontend/src/components/manageInstitutions/
Admin-Backend/src/api/enterprise/
```

## 5. College management

In the frontend, the entity is called a college. In the backend and database, it is called a `Workspace`.

### What the Admin can do

| Action | Description | API pattern |
|---|---|---|
| List colleges | View/search/filter all colleges | `GET /admin/workspace` |
| Add college | Create a college under an institution | `POST /admin/workspace/:enterpriseId` |
| View college | Load one college and its detailed information | `GET /admin/workspace/:workspaceId` |
| Edit college | Update college and accreditation details | `PUT /admin/workspace/:workspaceId` |
| Change status | Activate or deactivate a college | `PUT /admin/workspace/status` |
| Simple list | Load college IDs/names for plan assignment dropdowns | `GET /admin/workspace/simple` |

### College information handled by the UI

| Group | Example fields |
|---|---|
| Basic details | College name, code, description, type |
| Parent | Institution/enterprise ID |
| Contact | Email, phone number, website |
| Branding | College logo |
| Approval | UGC approved, AICTE approved |
| Accreditation | NAAC grade, NAAC score, NBA accredited, valid-until date |
| Ranking/status | NIRF ranking, autonomous status |
| Address | Address lines, city, state, country, pincode |

### College details page pattern

```text
College details page
   |
   +----> College information
   +----> Users
   +----> Roles and permissions
   +----> Assigned plans
```

### Main code locations

```text
Admin-Frontend/src/pages/manageColleges/
Admin-Frontend/src/components/manageColleges/
Admin-Backend/src/api/workspace/
```

## 6. College user management

These are users who belong to a college. They are stored in the normal `User` table, not the Super Admin `adminUser` table.

| Action | Description | API pattern |
|---|---|---|
| List users | Get users assigned to a college | `GET /admin/user/:workspaceId` |
| Add user | Create user and connect them to college and role | `POST /admin/user/:workspaceId` |
| Edit user | Change user details, role, or staff flag | `PUT /admin/user/:workspaceId` |
| Change status | Set active, inactive, suspended, or deleted | `PUT /admin/user/status` |

When a college user is created, the backend transaction can create:

```text
User
  |
  +----> UserWorkspace (connects user to college)
  |
  +----> UserRole (connects user to role and college)
  |
  +----> Staff (optional, when "is staff" is selected)
```

Important distinction:

| Table | Purpose |
|---|---|
| `adminUser` | Product owner/Super Admin login for the Admin application |
| `User` | Student/staff/college-level user records |

## 7. Roles and permissions

The Admin can define college-specific roles and choose allowed actions for each module.

| Action | Description | API pattern |
|---|---|---|
| Load available modules/actions | Get the permission catalogue | `GET /admin/role/modules-with-actions` |
| List college roles | View roles for one college | `GET /admin/role/:workspaceId` |
| Create role | Create a role with module permissions | `POST /admin/role/:workspaceId` |
| View permissions | Load one role's permissions | `GET /admin/role/:roleId/modules-with-actions` |
| Update permissions | Change allowed module actions | `POST /admin/role/:roleId/update-permissions` |
| Delete role | Soft-delete or remove a role | `DELETE /admin/role/:roleId` |
| User-role dropdown | Load roles that can be assigned to users | `GET /admin/role/:workspaceId/forUser` |

### Permission database pattern

```text
Role
  |
  +----> RoleModule
             |
             +----> Module
             |
             +----> RoleModulePermission
                          |
                          +----> ModuleAction
```

Example:

```text
Role: College Manager
Module: Students
Actions: View = allowed, Create = allowed, Delete = not allowed
```

### Current limitation

The role-management screens and data model exist. However, frontend module/permission stores contain commented code, and the JWT role/workspace data is also commented in several authentication paths. This means fine-grained permission enforcement appears incomplete even though role configuration is available.

## 8. Subscription plan management

Plans describe what a subscribed college can access or how many credits it receives.

### What the Admin can do

| Action | Description | API pattern |
|---|---|---|
| List plans | Search and paginate plans | `GET /admin/plans` |
| Add plan | Create a plan with feature values | `POST /admin/plans` |
| View plan | Display plan details and assigned colleges | `GET /admin/plans/:id` |
| Edit plan | Change name, type, price, description, and features | `PUT /admin/plans/:id` |
| Load identifiers | Load available feature/credit definitions | `GET /admin/plans/identifiers` |
| View assigned colleges | List colleges subscribed to the plan | `GET /admin/plans/:id/workspaces` |
| Assign college | Subscribe a college with start/end dates | `POST /admin/plans/:id/workspaces` |
| Change subscription status | Activate/deactivate a college's subscription | `PUT /admin/plans/workspaces/status` |
| View college plans | List subscriptions belonging to a college | `GET /admin/plans/workspace/:workspaceId` |
| Module access | Calculate module access from a college plan | `GET /admin/plans/workspace/:workspaceId/module-access` |

### Plan structure

```text
Plan
  |
  +----> PlanFeature
  |          |
  |          +----> PlanIdentifier
  |
  +----> WorkspacePlanSubscription
             |
             +----> College/Workspace
```

### Plan fields

| Area | Fields |
|---|---|
| Basic details | Name, description |
| Type | Main plan or add-on plan |
| Pricing | Monthly price, yearly price |
| Features | Access flags and credit/quantity values |
| Assignment | College, start date, end date, active status |

### Known mismatch

The frontend contains an `updatePlanStatus()` request to `PUT /admin/plans/status`, but the Admin Backend does not define that route. College-plan subscription status is supported, but direct plan status changing appears incomplete.

## 9. Question bank management

The Admin application includes a shared question-authoring system.

### General actions

| Action | Description | API pattern |
|---|---|---|
| List questions | Paginated list | `GET /admin/question` |
| Search/filter | Filter by text, difficulty, category, tags, and companies | `GET /admin/question?...` |
| View question | Open complete question details | `GET /admin/question/:questionId` |
| Add question | Create a general, MCQ, or coding question | `POST /admin/question` |
| Edit question | Update question and related settings | `PUT /admin/question/:questionId` |
| Delete question | Soft-delete/remove question | `DELETE /admin/question/:questionId` |
| Load filters | Get categories, tags, and companies | `GET /admin/question/filters` |

### Supported question types

| Type | Supported information |
|---|---|
| General | Title, description, category, difficulty, visibility, images, tags, companies |
| MCQ | General fields plus options and single/multiple correct answers |
| Coding | General fields plus languages, input/output definitions, and test cases |

### Question properties

| Property | Examples |
|---|---|
| Difficulty | Easy, Medium, Hard |
| Visibility | Public or college-only |
| Category | DSA, Aptitude, Logical, etc. |
| Tags | Arrays, Sorting, Dynamic Programming, etc. |
| Companies | Amazon, Google, Infosys, etc. |
| Images | Up to two images in the current UI |

### Coding question capabilities

The coding-question editor supports:

- Selecting allowed programming languages
- Allowing all languages
- Single values such as string, number, float, boolean, and character
- One-dimensional arrays
- Two-dimensional arrays
- Input labels and data types
- Expected output configuration
- Multiple test cases
- Visible sample cases and hidden evaluation cases

### Question database pattern

```text
Question
  |
  +----> Category
  +----> QuestionTag ----> Tag
  +----> QuestionCompany ----> Company
  +----> QuestionImage
  +----> QuestionMcqSettings ----> QuestionOptions
  +----> QuestionCodingSettings ----> QuestionCodingTestCases
```

## 10. Authentication and account features

| Feature | API pattern | Status/notes |
|---|---|---|
| Email/password login | `POST /admin/auth/login` | RSA-encrypted password, bcrypt comparison |
| Start Google redirect login | `GET /admin/auth/google` | Recommended localhost flow |
| Google callback | `GET /admin/auth/google/callback` | Exchanges code, loads Google profile |
| Google ID-token login | `POST /admin/auth/loginWithGoogle` | Implemented, but production cookie settings cause a localhost loop |
| Verify JWT | `GET /admin/auth/verify` | Used by protected Astro pages |
| Get current profile | `GET /admin/auth/profile` | Reads logged-in Admin identity |
| Update profile | `PUT /admin/auth/profile` | Currently updates phone number |
| Logout | `GET /admin/auth/logout` | Deletes Redis session and clears cookie |
| Get password public key | `GET /admin/auth/public-key` | Used for browser-side RSA password encryption |
| Forgot password | `POST /admin/auth/forgot-password` | Sends reset email |
| Validate reset token | `GET /admin/auth/validate-reset-token` | Checks link before showing reset form |
| Set password | `POST /admin/auth/set-password` | Saves bcrypt password hash |

### Login data pattern

```text
adminUser in PostgreSQL
        |
        v
Successful authentication
        |
        +----> JWT stored in browser token cookie
        |
        +----> Session identifier stored in Redis
```

## 11. Dashboard and reporting status

### Dashboard

The `/dashboard` page is protected by login, but its current content is only:

```text
Welcome to QConnect
```

There are no Admin dashboard statistics, charts, college counts, subscription summaries, student-performance totals, or reporting APIs in the Admin application yet.

Status: **Placeholder**.

### Reports

The navigation menu contains:

```text
/reports/monthly
/reports/annual
```

But there are no matching Astro pages or Admin Backend report routes.

Status: **Not implemented**.

## 12. Contact and session utilities

### Contact form

The Admin Frontend calls:

```text
POST /admin/contact/
```

The Admin Backend does not contain a `contact` module or matching route.

Status: **Partial/broken**.

### Session import/export

The backend provides:

```text
GET  /admin/session/export-all-cookies
POST /admin/session/import-cookies
```

The frontend includes:

```text
/export-cookie
/import-cookie
```

This appears to be an internal development/support feature used to move or restore session information, rather than a normal Super Admin business feature.

## 13. Frontend page map

| Page | Purpose | Protection |
|---|---|---|
| `/login` | Admin login | Public |
| `/forgot-password` | Request password reset | Public |
| `/set-password` | Set/reset password using token | Public with reset token |
| `/contactUs` | Contact form | Public |
| `/dashboard` | Admin home | Protected |
| `/manageInstitution` | Institution list and management | Protected |
| `/manageInstitution/[institutionId]` | Institution details and its colleges | Protected |
| `/manageColleges` | College list and management | Protected |
| `/manageColleges/[collegeId]` | College details, users, roles, and plans | Protected |
| `/managePlans` | Plan list | Protected |
| `/managePlans/addPlan` | Create plan | Protected |
| `/managePlans/[id]` | Plan details and college assignments | Protected |
| `/managePlans/[id]/edit` | Edit plan | Protected |
| `/manageQuestions` | Question list | Protected |
| `/manageQuestions/addQuestion` | Create question | Protected |
| `/manageQuestions/[id]` | Question details | Protected |
| `/manageQuestions/[id]/edit` | Edit question | Protected |
| `/settings` | Profile and password settings | Protected |
| `/export-cookie` | Export session utility | Unclear/internal |
| `/import-cookie` | Import session utility | Unclear/internal |
| `/403` | Forbidden page | Error page |
| `/404` | Not found page | Error page |
| `/500` | Server error page | Error page |

## 14. Backend module map

Every backend module follows roughly this pattern:

```text
routes.ts
   |
   v
controller.ts
   |
   v
dao.ts
   |
   v
PostgreSQL or Redis
```

| Module | Main responsibility |
|---|---|
| `auth` | Admin login, Google OAuth, JWT, profile, password reset |
| `enterprise` | Institution management |
| `workspace` | College management |
| `user` | College user management |
| `role` | Roles, modules, actions, and permissions |
| `plans` | Product plans, plan features, and college subscriptions |
| `question` | General, MCQ, and coding question bank |
| `session` | Internal session import/export |

## 15. Important database table map

| Business concept | Prisma/database model | Main purpose |
|---|---|---|
| Super Admin | `adminUser` | Login to Admin application |
| Institution | `Enterprise` | Top-level subscribed organization |
| Institution details | `EnterpriseDetails` | Contact, logo, website, description |
| College | `Workspace` | College belonging to institution |
| College details | `WorkSpaceDetails` | Accreditation and ranking information |
| Address | `Address` | Institution/college addresses |
| College user | `User` | Student/staff/college users |
| User-college link | `UserWorkspace` | Connects a user to a college |
| Role | `Role` | Named college role |
| User role | `UserRole` | Connects user, role, and college |
| Module | `Module` | Product capability used in permissions |
| Module action | `ModuleAction` | View/create/edit/delete-like action |
| Role module | `RoleModule` | Connects role and module |
| Permission | `RoleModulePermission` | Stores whether an action is allowed |
| Plan | `Plan` | Subscription plan definition |
| Plan identifier | `PlanIdentifier` | Access or credit feature definition |
| Plan feature | `PlanFeature` | Feature value assigned to plan |
| College subscription | `WorkspacePlanSubscription` | Assigns plan to college with dates/status |
| Question | `Question` | Common question information |
| MCQ settings | `QuestionMcqSettings` | Single/multiple-answer configuration |
| MCQ option | `QuestionOptions` | Answer choices |
| Coding settings | `QuestionCodingSettings` | Allowed languages |
| Coding test case | `QuestionCodingTestCases` | Inputs, outputs, visibility |
| Category | `Category` | Question category |
| Tag | `Tag` | Question topic tag |
| Company | `Company` | Company association for question |

## 16. Typical feature request pattern

For example, when an Admin creates a college:

```text
Admin clicks "Add College"
        |
        v
React opens CollegeModal
        |
        v
Admin fills and submits form
        |
        v
admin.service.ts calls POST /admin/workspace/:enterpriseId
        |
        v
workspace.routes.ts selects createWorkspace()
        |
        v
workspace.controller.ts validates/coordinates request
        |
        v
workspace.dao.ts writes database transaction
        |
        v
Workspace + WorkSpaceDetails + Address are saved
        |
        v
Backend sends success response
        |
        v
Frontend shows toast and refreshes college list
```

This same general pattern is used throughout the Admin application:

```text
Astro page
  -> React component
  -> admin.service.ts
  -> Axios API request
  -> Backend route
  -> Controller
  -> DAO
  -> PostgreSQL/Redis
  -> API response
  -> React updates UI
```

## 17. What remains to be completed for FC Labs

Based on the current Admin codebase, likely remaining work includes:

| Priority area | Current gap |
|---|---|
| Dashboard | Replace welcome placeholder with real multi-college metrics and charts |
| Reports | Build monthly/annual report pages and backend report APIs |
| Analytics | Add cross-college student, assessment, and coding-performance analytics |
| Contact form | Add missing Admin Backend contact endpoint or point frontend to correct service |
| Plan status | Add/fix missing direct plan-status backend endpoint or remove unused call |
| Authorization | Complete role/module permission enforcement in backend and frontend |
| Google login | Choose one OAuth flow and make cookie behavior environment-aware |
| Security | Remove full JWT logging and avoid production domains in local cookies |
| Admin management | Add a proper UI/API for creating and managing `adminUser` accounts if required |
| Product naming | Replace Q Connect branding, URLs, and wording with FC Labs |

## 18. One-sentence summary

> The Admin application currently manages institutions, colleges, college users, roles, subscriptions, and the question bank, while product-wide dashboards, reports, analytics, some authorization enforcement, and a few integrations still need to be completed for FC Labs.
