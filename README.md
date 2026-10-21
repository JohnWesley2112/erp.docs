# Systra School ERP — Technical Case Study

> **Implementation status:** This README describes functionality present in
> the checked-in source. Builds, migrations, tests, deployed endpoints, and
> end-to-end behavior have not been verified as part of this documentation
> update. Source presence must not be interpreted as evidence of production
> readiness.

## 1. Project Overview & Technology Stack

Systra is a school ERP under active development. The frontend is a React 19,
TypeScript, Vite single-page application using Material UI, Redux Toolkit,
Axios, and React Router. The backend is a TypeScript / Express 5 API using
Prisma 6 and PostgreSQL.

The source includes workflows for academic structure, students and enrollments,
teachers and assignments, attendance, user administration, and permissions.
The backend separates a central **admin database** from per-tenant databases.
Tenant-aware middleware validates a user's membership and resolves a tenant
database client before protected domain routes use it.

Package manifests declare React 19, TypeScript 5.9, Vite 8, MUI 9, Redux
Toolkit, React Router 7, and Axios for the web application. The API manifest
declares Express 5, Prisma 6, bcrypt, JWT, Swagger UI, Vitest, and Supertest.
These declarations do not establish that builds or tests pass.

## 2. System Architecture & Folder Pattern

The frontend organizes routed experiences under `src/pages`, shared UI under
`src/components`, API helpers under `src/api`, and state under `src/store`.
The API separates Express setup and shared middleware from feature modules
under `src/modules`. Modules commonly contain routes, controllers, and
services. Prisma calls provide the persistence boundary; validation and data
access responsibilities can live in services as well as controllers, so this
is not a strict universal layering rule.

```text
E:\erp
├── web.erp/                              # React/Vite client
│   ├── src/
│   │   ├── api/                           # Axios client and feature API helpers
│   │   ├── components/                    # Shared UI and route guards (gaurd)
│   │   ├── layout/                        # Authenticated layout and navigation
│   │   ├── pages/                         # Academic, student, teacher, attendance,
│   │   │                                  # user, permissions, and audit screens
│   │   ├── router/                        # React Router routes and guards
│   │   └── store/                         # Redux Toolkit state
├── api.erp/                              # Express/TypeScript API
│   ├── prisma/
│   │   ├── admin/                         # Admin schema, migrations, and seed
│   │   └── tenant/                        # Tenant schema, migrations, and seed
│   └── src/
│       ├── middlewares/                   # Authentication, tenant, permissions
│       ├── modules/                       # Auth, IAM, users, academics, students,
│       │                                  # teachers, attendance, audit, tenant
│       ├── infrastructure/                # Admin and tenant Prisma clients
│       ├── errors/                        # Application errors and error handler
│       └── app.ts / server.ts             # Middleware, routes, and startup
└── api.documentation/                    # API and static case-study documentation
```

```text
React page → feature API helper / Axios → Express route
         → authenticate → resolve tenant → require permission
         → controller / service → tenant Prisma client → tenant PostgreSQL DB
```

Most feature routers are mounted under both `/api` and `/api/v1`; auth and
tenant routes also have versioned aliases. `/health` and Swagger UI at
`/api-docs` are mounted separately. This documents route wiring in source, not
deployment availability.

## 3. Database Architecture & Relationships

The API configures two PostgreSQL Prisma clients:

- **Admin database:** generated from `prisma/admin/schema.prisma` and
  configured through `ADMIN_DATABASE_URL`.
- **Tenant database:** generated from `prisma/tenant/schema.prisma`; the API
  resolves a tenant's database configuration and creates/reuses a Prisma client
  for that tenant.

Migrations are configured separately under `prisma/admin/migrations` and
`prisma/tenant/migrations`. The separate files under `prisma/schemas` are not
the schema paths selected by the current Prisma configs.

| Database / domain | Models and relationship facts |
| --- | --- |
| **Admin database** | `User`, `Tenant`, `UserTenantMembership`, `TenantDatabaseConfig`, `RefreshSession`, and `RefreshSessionToken`. Memberships link users and tenants through Prisma foreign keys; database configuration and refresh sessions also have tenant/user relations. Refresh-session tokens reference sessions and include a self-reference for token rotation history. |
| **Tenant database: school setup** | `Institution`, `AcademicYear`, `Class`, `Section`, and `Subject`. `Section` has foreign-key relations to `Class` and `AcademicYear`. |
| **Tenant database: school operations** | `Teacher`, `TeacherAssignment`, `Student`, `Guardian`, `StudentGuardian`, `Enrollment`, `AdmissionSerial`, `AttendanceSession`, `AttendanceRecord`, and `AuditLog`. Teacher assignments, student/guardian joins, enrollments, admission serials, and attendance records have in-database relations as declared in the tenant Prisma schema. |
| **Tenant database: access control** | `UserProfile`, `Role`, `Permission`, `UserRole`, and `RolePermission`. User-role and role-permission join rows have tenant-database foreign keys; `RolePermission` has a composite primary key. |

Cross-database identifiers are **logical references, not foreign keys**.
Tenant `UserProfile.userId`, `Teacher.userId`,
`AttendanceSession.createdByUserId`, and `AuditLog.actorUserId` do not have
Prisma relations to admin-database records. A database foreign key cannot span
these separately connected databases, so identity and lifecycle consistency
must be enforced by application logic.

The active tenant schema also declares `User`, `Tenant`,
`UserTenantMembership`, and `TenantDatabaseConfig`. Its tenant ID is a UUID,
whereas the admin schema's `Tenant.id` is an integer. The current login and
tenant-directory flow uses the admin Prisma client. The purpose and lifecycle
of these parallel declarations should be clarified before relying on them or
adding data migrations.

No generated Mermaid ER diagrams were found in the case-study documentation
directory; this README summarizes the schema in a table rather than linking a
diagram that would not render natively on GitHub Pages.

## 4. Authentication, Tenant Resolution & RBAC

### Authentication and tenant resolution

The API source implements the following flow:

1. `POST /api/v1/auth/login` checks an active admin account using bcrypt and
   returns its active tenant memberships. A single active membership is
   selected automatically; users with multiple memberships can choose one with
   `POST /api/v1/auth/select-tenant`.
2. Access JWTs expire after **15 minutes** and include user ID, email, optional
   tenant ID, and session ID. The random opaque refresh token is stored as a
   hash in the admin database and sent in an `HttpOnly`, `Secure` cookie.
3. `POST /api/v1/auth/refresh` rotates refresh tokens; replay of a consumed
   token revokes its session. Refresh sessions last seven days. Logout revokes
   the associated session.
4. Protected feature routes authenticate the bearer JWT, validate active
   tenant membership and tenant status, resolve tenant database configuration,
   and attach the tenant Prisma client to the request.
5. Feature routes check a tenant-local permission code before invoking their
   controller. Refresh and logout additionally enforce the configured client
   origin.

These are source-level findings, not a security certification. Deployment
cookie behavior, secret management, CSRF posture, and end-to-end session flows
remain to be verified.

### Roles, permission codes, and assignments

The tenant seed defines three default roles:

| Role | Seeded permissions |
| --- | --- |
| `INSTITUTION_ADMIN` | All seeded institution, academic, student, teacher, user, role, attendance, and audit permissions. |
| `CAMPUS_ADMIN` | Student read/create/update; teacher read; academic read; user read; attendance read/mark/submit. |
| `TEACHER` | Student read; teacher read; academic read; attendance read/mark/submit. |

The permission catalog uses explicit codes:

```text
institution.read       institution.manage
academic.read          academic.manage
student.read           student.create          student.update
teacher.read           teacher.create          teacher.update
user.read              user.manage
role.read              role.manage
attendance.read        attendance.mark         attendance.submit
audit.read
```

Tenant-local `UserProfile`, `UserRole`, and `Role` records represent role
assignments. The IAM API includes role and permission reads and role-permission
grant/revoke operations. API middleware performs permission checks; frontend
guards provide additional navigation and presentation controls, not the
security boundary. Attendance services also check teacher assignments for
section access.

Current route gates include `student.read/create/update`,
`teacher.read/create/update`, `attendance.read`, `attendance.mark`, and
`attendance.submit`. Enrollment creation uses `student.create`; guardian
association uses `student.update`. These policy choices should be reviewed
against product requirements.

## 5. Implemented School Modules

The following areas have corresponding frontend pages/routes and backend API
modules in source. Their presence does not demonstrate that the app builds, the
database is migrated, or the user workflows succeed at runtime.

| Area | Source-visible functionality | Status |
| --- | --- | --- |
| **Academics** | Academic years, classes, sections, and subjects expose list/detail/create/update routes and pages, using `academic.read` and `academic.manage`. | Implemented in source; runtime unverified |
| **Students & enrollments** | Student list/detail/create/update, enrollment creation, guardian association, admission and enrollment UI workflows. | Implemented in source; runtime unverified |
| **Teachers** | Teacher list/detail/create/update and subject/section assignment routes and UI. | Implemented in source; runtime unverified |
| **Attendance** | Session list/detail/create, record entry, and submission. The service checks draft state, active enrollment, duplicate records, and teacher section assignments. | Implemented in source; runtime unverified |
| **User administration** | User list/detail/create/update, activation/deactivation, and role assignment/removal; admin accounts and memberships coordinate with tenant-local profiles and roles. | Implemented in source; runtime unverified |
| **Security administration** | Role/permission reads, role-permission grant/revoke routes, permission-matrix UI, audit-log read route, and audit model. | Implemented in source; runtime unverified |

## 6. Current Status, Limitations & Next Steps

### Current implementation status

The repository contains frontend pages and API routes/services for academic
setup, students and enrollments, teachers and assignments, attendance, user
administration, IAM, and audit-log reads. It also contains separate admin and
tenant Prisma schemas, migration directories, seeds, and API tests. This
documentation update did **not** run application builds, Prisma validation,
migrations, or test suites. No feature should be treated as tested,
production-ready, or deployed based solely on source presence.

### Limitations and open questions

- The tenant schema declares user, tenant, membership, and database-config
  models in addition to corresponding admin-database models. Confirm ownership,
  seed behavior, and migration intent.
- Several tenant-side identity IDs are logical references without
  cross-database foreign keys. Account, membership, profile, teacher, and audit
  lifecycle consistency requires application-level validation.
- Route declarations and the presence of tests do not establish coverage or
  passing behavior. Permission-policy consistency, tenant isolation, cookie
  configuration, and failure handling need verification using controlled test
  databases.
- No generated Mermaid ER files were found in the case-study documentation
  directory.

### Prioritized next steps

1. Run Prisma validation and client generation, API and frontend builds, lint,
   and targeted auth, tenant-resolution, user-administration, RBAC, and domain
   tests in a safe non-production environment; record actual results.
2. Resolve the duplicate admin/tenant identity model declarations and document
   authoritative user, membership, profile, and role lifecycles, including
   cross-database failure recovery.
3. Add authorization allow/deny and tenant-isolation tests for protected
   routes; reconcile the seeded role matrix with route permissions and
   attendance/teacher scope rules.
4. Review request validation, refresh-cookie and origin/CSRF configuration,
   login throttling, secret management, and audit coverage before production
   use.
5. Verify migrations, seed procedures, backup/restore, health checks, and API
   contracts against non-production infrastructure.
