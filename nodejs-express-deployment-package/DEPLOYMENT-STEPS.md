# Node.js + Express + MongoDB → AWS ECS deployment — steps followed

This file is a running log of the setup we walked through, so it can be
handed back to Claude in a future session as context.

---

## 1. Project scaffolding

```bash
npm init -y
npm i express cookie-parser
npm i -D nodemon
```

Add to `package.json`:
```json
{ "type": "module" }
```

Scripts:
```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  }
}
```

## 2. App structure

```
project/
├── app.js
├── server.js
├── models/
│   ├── user.model.js
│   └── enquiry.model.js
├── controllers/
│   ├── auth.controller.js
│   └── enquiry.controller.js
├── middleware/
│   ├── auth.middleware.js
│   └── role.middleware.js
├── routes/
│   ├── auth.routes.js
│   ├── user.routes.js
│   └── enquiry.routes.js
├── ecs/
│   ├── task-def-dev.json
│   ├── task-def-qa.json
│   └── task-def-prod.json
└── .github/workflows/deploy.yml
```

## 3. `app.js` — core middleware setup

```javascript
import express from "express";
import cookieParser from "cookie-parser";
import cors from "cors";

const app = express();

app.use(cors({
  origin: ["https://myapp.com", "http://localhost:5173"],
  credentials: true,
}));
app.use(cookieParser(process.env.COOKIE_SECRET));
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

app.use("/api/auth", authRouter);
app.use("/api/users", userRouter);
app.use("/api/enquiries", enquiryRouter);

export default app;
```

`server.js`:
```javascript
import app from "./app.js";
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

## 4. Authentication (register/login) — `controllers/auth.controller.js`

- Passwords hashed with `bcryptjs` before saving (never store plain text).
- Login compares hash with `bcrypt.compare`, returns the SAME generic
  "Invalid credentials" error whether the email doesn't exist or the
  password is wrong (avoids leaking which emails are registered).
- On success, issues a JWT with `jsonwebtoken`, signed with `process.env.JWT_SECRET`,
  containing `{ id, email, role }`.

## 5. Authorization — two separate middleware layers

`middleware/auth.middleware.js` — verifies the Bearer token, attaches `req.user`:
```javascript
export const authMiddleware = (req, res, next) => {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith("Bearer ")) return res.status(401).json({ message: "No token provided" });
  const token = authHeader.split(" ")[1];
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    return res.status(401).json({ message: "Invalid or expired token" });
  }
};
```

`middleware/role.middleware.js` — gates by role, kept separate on purpose
(single responsibility — auth changes independently of permission rules):
```javascript
export const authorizeRoles = (...allowedRoles) => (req, res, next) => {
  if (!allowedRoles.includes(req.user.role)) {
    return res.status(403).json({ message: "Access denied" });
  }
  next();
};
```

Applied in order on routes: `authMiddleware` always runs before `authorizeRoles`,
since the role check reads `req.user` set by the auth middleware.

## 6. Listing API — pagination, search, filters (MongoDB/Mongoose)

Query string convention: `?page=&limit=&status=&search=&sortBy=&order=`

Key implementation details:
- `page`/`limit` parsed with `parseInt`, `limit` capped at 100 to prevent abuse.
- `find()` and `countDocuments()` run in parallel via `Promise.all` (independent
  operations shouldn't be awaited sequentially).
- `.lean()` used on reads — skips Mongoose document overhead, faster for
  read-only JSON responses.
- Search uses a MongoDB **text index** (`$text: { $search }`) — not `$regex`,
  since regex on an unindexed field forces a full collection scan.
- `sortBy` is validated against a **whitelist** of allowed fields before being
  used in the sort object — never pass raw client input into a Mongo query key.
- For serious validation at scale: a Zod schema (`z.coerce.number()`, `z.enum()`)
  replacing manual `if` checks, giving typed values, defaults, and clean 400s.
- Indexes added on the schema for every field actually filtered/sorted/searched
  on (`status`, `createdAt`, text index on `studentName`/`email`) — without
  these, queries fall back to full collection scans as data grows.

## 7. CORS

```javascript
app.use(cors({
  origin: ["https://myapp.com"],   // never "*" once credentials/cookies are involved
  credentials: true,
}));
```

Client must also opt in: `fetch(url, { credentials: "include" })`. Both sides
must agree or cookies silently never arrive cross-origin.

## 8. CI/CD — GitHub Actions → ECR → ECS

**Environment selection mechanism:** the branch name IS the GitHub Environment
name (`environment: ${{ github.ref_name }}`). Each Environment (`dev`/`qa`/`prod`,
configured in repo Settings → Environments) holds its own:
- `AWS_ROLE_ARN` (secret) — a role in that environment's own separate AWS account
- `AWS_REGION`, `ECR_REPOSITORY`, `ECS_CLUSTER`, `ECS_SERVICE`, `CONTAINER_NAME` (variables)

Auth into AWS uses **OIDC** (`aws-actions/configure-aws-credentials`), not
long-lived access keys — the trust policy on each role is scoped to
`repo:<org>/<repo>:environment:<env>`, so a role can only be assumed by a
workflow run tied to that specific GitHub Environment.

**Build happens on a self-hosted runner** (the company/local build machine),
tagged and pushed to ECR, optionally mirrored to Docker Hub.

**Task definition selection:** `ecs/task-def-${{ github.ref_name }}.json` —
plain string interpolation of the branch name into a file path already
checked into the repo. This is a *different* mechanism from the Environment
secrets above — it's just picking which committed JSON file to read.

**Real `.env` secrets are never handled by the pipeline at all.** Each
`task-def-<env>.json` has a `secrets` array with `valueFrom` ARNs pointing at
that environment's AWS Secrets Manager (real secrets: JWT_SECRET, MONGO_URI)
and SSM Parameter Store (non-secret config: third-party service URLs). ECS
resolves these itself at container startup — nothing sensitive ever appears
in CI logs or gets baked into the image.

**Deploy flow, in order:**
1. Checkout code
2. Assume environment-specific AWS role via OIDC
3. Login to ECR
4. `docker build` on the self-hosted runner
5. Tag + push to ECR (+ optional Docker Hub push)
6. `amazon-ecs-render-task-definition` — injects the new image URI into
   `ecs/task-def-<env>.json`
7. `amazon-ecs-deploy-task-definition` — registers the new task def revision,
   updates the ECS service, waits for stability

See the accompanying files in this package:
- `.github/workflows/deploy.yml`
- `ecs/task-def-dev.json`, `ecs/task-def-qa.json`, `ecs/task-def-prod.json`
- `README-deployment.md` (OIDC trust policy, secrets/variables checklist)

## Open follow-ups not yet built (good next steps)

- Refresh token flow (short-lived access token + longer-lived refresh token)
- Cursor-based pagination for very large collections (`_id: { $gt: lastId }`
  instead of `skip()`, to avoid slow deep-page scans)
- Rate limiting on `/login` (`express-rate-limit`) against brute force
- Forgot-password flow (single-use expiring reset token)
