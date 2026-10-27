# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

Record your plan here before editing code.

- Presentation tier decision: Keep route-specific rendering in React components and all HTTP handling in the shared API client. Each view exposes loading, success, empty, and accessible error states and consumes the existing `/api` proxy paths.
- Logic tier decision: Keep input validation and business rules in the Express service layer, with routes translating service results and deliberate errors into HTTP responses. Preserve the existing API contracts and return 4xx responses for invalid or unknown references.
- Data tier decision: Express relationships and required fields with Mongoose schemas. Seed representative users, teams, activities, and workouts with stable keys and idempotent upserts that preserve unrelated records.
- Activity tracking approach: Validate activity type, positive minutes, and well-formed IDs that resolve to an existing user and team before writing. Store the submitted references and activity data, calculate points from minutes, and set `occurredAt` when creating the record.
- Team management approach: Require a non-empty name, preserve the submitted name and mascot, and store member references as user IDs. Listing and creation responses return the team fields expected by the API contract.
- Leaderboard approach: Aggregate activity points by team, sort totals descending, and return `teamName`, `totalPoints`, and sequential 1-based ranks.
- Workout suggestions approach: Find the user by email, consider their fitness goal and recent activity when choosing matching workouts, and include a concise reason for the leading recommendation. Return a deliberate 4xx response when the user does not exist.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`: checks required model fields, relationships, and repeatable seed behavior using an in-memory MongoDB. A failure points to schema validation or seed correctness. After seeding local MongoDB, run `npm run seed --prefix octofit-tracker/backend` twice; the second run must not duplicate fixtures or remove unrelated records.
- API command: `npm run test:api --prefix octofit-tracker/backend`: checks request validation, persisted activity/team responses, leaderboard aggregation/ranking, and user-specific workout suggestions. A failure points to route/service behavior, status codes, or data mapping.
- Frontend command: `npm test --prefix octofit-tracker/frontend`: checks API-backed rendering and loading, empty, and accessible error states for each route. A failure points to view behavior or shared API-client integration.
- Backend build command: `npm run build --prefix octofit-tracker/backend`: type-checks the backend; a failure indicates a TypeScript or backend interface mismatch.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`: type-checks and bundles the app; a failure indicates a TypeScript, module, or Vite build issue.
- Baseline before implementation: backend tests reported 8 failures (4 model validation and 4 API contract tests); frontend tests reported 15 failures. These are the starter implementation gaps to resolve in data, logic, then presentation order.
