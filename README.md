# Frontend Base API

Spring Boot REST API được thiết kế theo contract của frontend Vue hiện có. API dùng cùng field name, HTTP method, route và kiểu response mà frontend đang gọi; backend đặt context path `/api`, khớp `VITE_GLOB_API_URL`.

## Frontend contract đã đối chiếu

- Vue 3 + TypeScript strict; Ant Design Vue; Pinia; Vue Router; Axios; Vue I18n; Dayjs.
- Auth Pinia lưu access/refresh token trong `localStorage`. Login và refresh dùng Axios trực tiếp, nhận token response không bọc envelope; `/auth/me` dùng Axios instance có `Authorization: Bearer`.
- Route permissions: `DASHBOARD_VIEW`, `USER_VIEW`, `USER_CREATE`, `USER_UPDATE`, `USER_DELETE`, `ROLE_VIEW`.
- Module hiện có: login, dashboard, user CRUD, role read-only, 403/404. Không có API logout, department, upload/download hay API load menu.
- Base URL development: `http://localhost:8080/api`; frontend Vite mặc định `http://localhost:5173`.

## API matrix

| Frontend API | Method + URL (bao gồm context `/api`) | Request | Response | Permission |
|---|---|---|---|---|
| `loginApi` | `POST /api/auth/login` | `{ username, password }` | `{ accessToken, refreshToken }` | Public |
| `refreshTokenApi` | `POST /api/auth/refresh` | `{ refreshToken }` | `{ accessToken, refreshToken }` (rotated) | Public; refresh token required |
| `getUserInfoApi` | `GET /api/auth/me` | `Authorization: Bearer <accessToken>` | `{ id, username, fullName, email, permissions: string[] }` | Authenticated |
| `getUserList` | `GET /api/users?page=&pageSize=&keyword=&status=` | Query: page (1-based), pageSize (1-100), optional keyword/status (0/1) | `{ items: User[], total, page, pageSize }` | `USER_VIEW` |
| `getUser` | `GET /api/users/{id}` | UUID string path ID | `User` | `USER_VIEW` |
| `createUser` | `POST /api/users` | `{ username, fullName, email, phone?, status }` | `User` (201) | `USER_CREATE` |
| `updateUser` | `PUT /api/users/{id}` | Same user payload | `User` | `USER_UPDATE` |
| `deleteUser` | `DELETE /api/users/{id}` | UUID string path ID | Empty body (204) | `USER_DELETE` |
| `getRoleList` | `GET /api/roles` | None | `Role[]` (direct array, not page/envelope) | `ROLE_VIEW` |

`User` is `{ id: string, username, fullName, email, phone?, status: number, createdAt?: ISO-8601 string }`. `Role` is `{ id: string, name, code, description? }`. Login, refresh, me, list, detail, and role responses are direct JSON values; they are not wrapped in `{ data: ... }`. Error bodies are `{ timestamp, status, code, message, details }`, so the current Axios error normalizer can consume `message`, `code`, and `details`.

Error status compatibility: 400 validation/request, 401 missing/invalid/expired session, 403 permission denied, 404 missing user, 500 unexpected server error. Duplicate account fields use 409 Conflict.

## Architecture and persistence

Packages are organized by domain (`auth`, `user`, `role`) with shared `security`, `config`, and `common` packages. Each business domain follows Controller → Service → Repository → JPA entity/DTO/mapper. Controllers contain routing and validation only. Production schema is exclusively Flyway-managed; Hibernate is configured with `ddl-auto: validate`.

| Frontend feature/API | Entity/table | Notes |
|---|---|---|
| Login, `/auth/me`, user CRUD | `UserAccount` / `users` | UUID ID, case-preserving username, `full_name`, normalized unique username/email keys, numeric status, ISO instant createdAt; nullable password hash because create-user form has no password field |
| Permission checks and `/roles` | `Role` / `roles` | Role response contains exactly the fields the UI consumes |
| `AuthUser.permissions` | `Permission` / `permissions` | Only the six permission codes currently referenced in frontend are seeded |
| User-to-role authorization | `user_roles` | Many-to-many relationship |
| Role permission authorization | `role_permissions` | Many-to-many relationship |
| `/auth/refresh` | `RefreshToken` / `refresh_tokens` | Stores SHA-256 hashes, expiration and revocation time; raw tokens are returned only once |

`V1__create_identity_and_rbac_schema.sql` creates the schema and seeds only current FE permission codes plus `ADMIN` and `USER_MANAGER` role definitions. A local-profile runner creates the configurable admin and attaches `ADMIN`; production profile never seeds accounts.

## Authentication and permission flow

1. `POST /auth/login` verifies BCrypt credentials and enabled account status.
2. Backend returns the exact FE token shape. Access JWT is signed using the configured HMAC secret and carries current FE permission codes; refresh token is cryptographically random and only its hash is persisted.
3. Axios sends the bearer token to protected endpoints and `/auth/me`.
4. `POST /auth/refresh` validates the stored refresh-token hash, locks/rotates the old token and returns a replacement pair. Invalid/replayed refresh tokens return 401; Axios clears the local session and redirects to `/login`.
5. `@PreAuthorize` enforces each permission server-side; UI route/menu/button checks are not trusted as authorization.

The FE has no logout request, so the backend does not invent one. Its current logout only deletes browser storage; already-issued refresh tokens remain usable until rotation/revocation or expiration. Change this only together with an explicit FE logout API call. Likewise, the user form does not submit an initial password or role assignment: new users are created without login credentials and without roles, so they cannot authenticate until an explicit account provisioning/password/role feature is designed.

## Run locally

Requirements: Java 21+, Docker with Compose, Maven 3.9+ (or use the Maven Wrapper if generated in this workspace).

From `backend/`:

```sh
docker compose up -d postgres
./mvnw spring-boot:run
```

On Windows PowerShell use `.\mvnw.cmd spring-boot:run`. The default `local` Spring profile uses:

- PostgreSQL: `localhost:5432`, database/user `frontend_base`, development password `frontend_base_dev`.
- Seeded login: `admin` / `ChangeMe123!`.
- CORS: `http://localhost:5173` và `http://127.0.0.1:5173` (có thể thay bằng `APP_CORS_ALLOWED_ORIGINS`).
- API: `http://localhost:8080/api`.
- Swagger UI: `http://localhost:8080/api/swagger-ui.html`.

Override the local seed credentials before sharing a running instance. Set `APP_SEED_ENABLED=false` to turn off local account seeding. Database migration runs on backend startup through Flyway; do not use `ddl-auto=create` in production.

### Production configuration

Activate `SPRING_PROFILES_ACTIVE=prod` and provide `DATABASE_URL`, `DATABASE_USERNAME`, `DATABASE_PASSWORD`, `APP_JWT_SECRET` (at least 32 bytes, or `base64:<key>`), and `APP_CORS_ALLOWED_ORIGINS`. Use a unique secret and strong database/admin credentials; do not use the local defaults. Restrict CORS to the deployed FE origin and terminate TLS at the deployment ingress.

### Build and tests

```sh
./mvnw clean install
```

Windows:

```powershell
.\mvnw.cmd clean install
```

The automated suite uses H2 in PostgreSQL compatibility mode and runs the Flyway migration for the auth/permission/API integration path, plus a JWT signature unit test. Because H2 is not PostgreSQL, run migration verification against PostgreSQL in a deployment/test environment as well.

## Frontend end-to-end

1. Start PostgreSQL and the backend as above; wait for the Flyway migration and application-ready log.
2. Start the existing frontend from the project root using `pnpm dev`.
3. Open `http://localhost:5173/login`; use the local seed admin credentials.
4. Confirm `/auth/me` populates the store and `/dashboard` loads.
5. Open `/system/user` to exercise server-side search/filter/pagination and CRUD; open `/system/role` to load the role array.
6. To exercise refresh, use a short `APP_ACCESS_TOKEN_MINUTES` value and make a protected API call after expiry; Axios refreshes and retries it.
7. Sign out in the header (frontend-only local token removal); the next protected route returns to `/login`.

The browser E2E pass exposed an existing FE form issue: the user modal's save handler read the outer v-model object, which remained empty while the BasicForm displayed entered values. `src/components/BasicForm/BasicForm.vue` now exposes its validated internal form values, and the user page submits those values. No API field, route, method, response or auth contract changed. End-to-end browser testing against real PostgreSQL still requires Docker/PostgreSQL; the FE/browser smoke flow was exercised against the same Flyway migration using in-memory H2.
