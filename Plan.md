# Fengshui-Checker — v1 Implementation Plan

## Context

This is a greenfield repo (currently just a README) for a new cross-platform mobile app. Users will either photograph their house's floor plan (stored as a reference image) or build a room manually in-app by placing walls, doors, and windows with size/position data. The app then runs a fengshui recommendation engine against the structured room data and shows the user recommendations. The user will supply the actual fengshui rules/algorithm later — so the system needs a clean, swappable interface for that analysis step now, backed by a placeholder implementation.

Confirmed decisions driving this plan: React Native + Expo + TypeScript frontend, NativeWind (Tailwind) for styling, Node/Express + MongoDB (Mongoose) backend, full email/password JWT auth, photo stored as reference only (no CV/OCR), fengshui analysis runs server-side as a pluggable module, and projects sync to the backend per-account (not local-only). Repo is an npm workspaces monorepo: `/app`, `/server`, `/shared`.

## Monorepo Layout

```
Fengshui-Checker/
├── package.json                # npm workspaces: shared, server, app
├── tsconfig.base.json
├── shared/                     # @fengshui/shared — types only, no build step (consumed as TS source)
│   └── src/{geometry,room,project,user,recommendation,analyzer}.ts, index.ts
├── server/                     # @fengshui/server — Express + TypeScript + Mongoose
│   └── src/{config,models,middleware,routes,controllers,services,fengshui,utils}/...
└── app/                        # @fengshui/app — Expo RN + TypeScript + NativeWind
    └── src/{navigation,screens,components,api,state,styles}/...
```

Root `package.json` uses `"workspaces": ["shared", "server", "app"]`. `server` and `app` both depend on `"@fengshui/shared": "*"` and import it as TS source directly (via `tsconfig` path mapping + Metro `watchFolders` for Expo's monorepo support) — no build/publish step needed for `shared` in v1.

## Shared Domain Model (`shared/src/*.ts`)

The core seam of the whole system is the analyzer contract in `shared/src/analyzer.ts`:

```typescript
export interface FengshuiAnalysisInput {
  project: Pick<Project, 'id' | 'name' | 'sourceType' | 'rooms'>;
}
export interface FengshuiAnalysisResult {
  projectId: string;
  generatedAt: string;
  recommendations: FengshuiRecommendation[];
}
export type FengshuiAnalyzer = (input: FengshuiAnalysisInput) => Promise<FengshuiAnalysisResult> | FengshuiAnalysisResult;
```

Supporting types: `Position`/`Size`/`CompassOrientation`/`WallSegment` (geometry.ts), `Wall`/`Door`/`Window`/`Room` (room.ts — doors/windows attach to a `wallId` + `positionOnWall` fraction), `Project`/`ReferencePhoto`/`ProjectSourceType` (project.ts), `PublicUser` (user.ts), `FengshuiRecommendation` with `category`/`severity` (recommendation.ts). These are the types both the RN app and the Node server import directly — no duplication.

## Backend (`server/`)

- **Framework:** Express (simplest fit for a small v1 API — auth + CRUD + one analysis route + file upload).
- **DB:** MongoDB via Mongoose. `models/User.ts` (email, passwordHash, timestamps). `models/Project.ts` embeds rooms/walls/doors/windows as subdocuments mirroring the shared types (they're always read/written together — no need for separate collections).
- **Auth:** `services/auth.service.ts` (bcrypt hash/verify, JWT sign/verify), `middleware/auth.ts` (`requireAuth` reads `Authorization: Bearer`), controllers for signup/login/me/logout (logout is a stateless client-side token discard).
- **Photo storage:** multer → local disk (`UPLOAD_DIR`), served via `express.static('/uploads')`, wrapped behind `services/storage.service.ts` (`saveFile(buffer, filename): Promise<string>`) so swapping to a real object store later is a one-file change — no GridFS, unnecessary complexity for v1.
- **Fengshui module:** `fengshui/stubAnalyzer.ts` implements `FengshuiAnalyzer`, returning a placeholder recommendation. `controllers/analysis.controller.ts` references it via a single import — this is the line the user swaps when the real algorithm is ready.

**REST API:**

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/auth/signup` | – | create user, return JWT |
| POST | `/api/auth/login` | – | verify credentials, return JWT |
| GET | `/api/auth/me` | ✓ | current `PublicUser` |
| GET | `/api/projects` | ✓ | list current user's projects |
| POST | `/api/projects` | ✓ | create project (`sourceType`: manual/photo) |
| GET/PUT/DELETE | `/api/projects/:id` | ✓ | fetch/update/delete (ownership-checked) |
| POST | `/api/projects/:id/photo` | ✓ | multipart upload, sets `referencePhoto` |
| PUT | `/api/projects/:id/rooms` | ✓ | save the room-builder's `Room[]` |
| POST | `/api/projects/:id/analyze` | ✓ | run analyzer, return `FengshuiAnalysisResult` |

## Mobile App (`app/`)

- **Navigation:** React Navigation — `AuthStack` (Login/Signup) vs `AppStack` (ProjectList → ProjectDetail → PhotoCapture/RoomBuilder), switched on auth token presence.
- **State:** React Query for server state (projects, auth calls — gives caching/refetch/mutation status for free since everything syncs to the backend); Zustand for the thin client-only slice (JWT token in `expo-secure-store`, in-progress room-builder draft).
- **Photo capture:** `expo-image-picker` (camera or library) — simpler than `expo-camera` since there's no live-preview requirement, just "attach one photo."
- **Manual room builder (`components/canvas/RoomCanvas.tsx`):** `react-native-svg` + `react-native-gesture-handler` on a snapped grid — tap-to-place walls, tap-a-wall-to-attach doors/windows. Chosen over Skia since v1 only needs lines/rects, not freehand drawing.
- **Styling:** NativeWind — `tailwind.config.js` scanning `src/**/*.{ts,tsx}`, `nativewind/babel` plugin, global CSS import in `App.tsx`.
- **API client (`api/client.ts`):** thin fetch/axios wrapper attaching the JWT and normalizing errors.

**Data flow:**
- *Manual:* create project (`sourceType: 'manual'`) → build room in `RoomCanvas` → `PUT /rooms` → `ProjectDetail` → "Run Analysis" → `POST /analyze` → render `RecommendationCard[]`.
- *Photo:* create project (`sourceType: 'photo'`) → pick/capture image → `POST /:id/photo` → `ProjectDetail` shows the photo; "Run Analysis" is gated on `rooms.length > 0` since there's no structured data yet (users can still add rooms manually to a photo project via the same `/rooms` endpoint).

## Build Order

1. **Monorepo + shared types** — scaffold workspaces, write all `shared/src` interfaces, `npm run typecheck -w shared` passes.
2. **Backend auth + MongoDB** — Express skeleton, `User` model, signup/login/me, JWT middleware, bcrypt. Verify via curl.
3. **Project/Room CRUD + photo upload** — `Project` model, CRUD routes, `PUT /rooms`, multer + `storage.service.ts`. Verify via curl/Postman.
4. **Fengshui stub + analysis endpoint** — `stubAnalyzer.ts`, `POST /:id/analyze`. Verify against a project with sample room JSON.
5. **Expo shell: auth + project list** — Expo init, NativeWind, React Navigation, Zustand + React Query, `api/client.ts`, Login/Signup/ProjectList wired to the live backend.
6. **Photo capture flow** — `PhotoCaptureScreen`, upload, `ProjectDetail` shows photo.
7. **Manual room builder** — `RoomCanvas` with wall/door/window placement, save/reload round-trip via `/rooms`.
8. **Recommendations UI + polish** — `RecommendationCard`, wire "Run Analysis", loading/empty/error states, styling pass.

Each phase should be runnable/demoable before starting the next.

## Verification

- `npm run typecheck -w shared|server|app` at each phase boundary.
- Backend: run against a local MongoDB (or Atlas free tier); curl smoke tests for signup → login → `/me`, project create → `PUT /rooms` → `GET` round-trip, photo upload → confirm file on disk and reachable at `/uploads/<file>`, and `/analyze` returning the stub's `FengshuiRecommendation[]` shape.
- Mobile: `npx expo start`, run in iOS Simulator and Android Emulator; manual smoke flow covering both project-creation paths (signup → manual project → place walls/door/window → save → view stub recommendations; and signup → photo project → attach image → view on detail screen).

## Critical Files

- `shared/src/analyzer.ts` — the `FengshuiAnalyzer` contract everything is built around
- `shared/src/room.ts`, `shared/src/project.ts` — core domain model shared by client and server
- `server/src/models/Project.ts` — Mongoose schema mirroring the shared types
- `server/src/fengshui/stubAnalyzer.ts` — the seam the user replaces with their real algorithm later
- `app/src/components/canvas/RoomCanvas.tsx` — the manual room builder, the most novel UI piece
