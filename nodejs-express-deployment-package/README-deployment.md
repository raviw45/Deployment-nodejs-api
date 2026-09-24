# Deployment setup — dev / qa / prod on separate AWS accounts

## How the environment is selected

The branch name IS the environment name. `deploy.yml` sets:

```yaml
environment: ${{ github.ref_name }}
```

Pushing to `dev` runs the job under GitHub's `dev` **Environment**; pushing to
`qa` uses `qa`; pushing to `prod` uses `prod`. Each Environment (repo Settings
→ Environments) has its own secrets/variables, so the same workflow file
authenticates into a completely different AWS account depending on which
branch triggered it — no branch-name `if/else` logic needed in the workflow.

For prod specifically, add **required reviewers** on the `prod` Environment.
That gives you a manual approval gate before anything deploys to production,
without touching the workflow file at all.

## Per-environment secrets/variables to configure (Settings → Environments → dev/qa/prod)

**Secrets** (sensitive — never shown in logs):
- `AWS_ROLE_ARN` — the IAM role in *that* environment's AWS account, assumed via OIDC
- `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` — only if `PUSH_TO_DOCKERHUB` is enabled

**Variables** (non-sensitive config, visible in logs):
- `AWS_REGION`
- `ECR_REPOSITORY`
- `ECS_CLUSTER`
- `ECS_SERVICE`
- `CONTAINER_NAME`
- `PUSH_TO_DOCKERHUB` (`"true"`/`"false"`)
- `DOCKERHUB_NAMESPACE`

## Why secrets never touch the pipeline

The actual `.env`-style values (JWT secret, Mongo URI, etc.) are **not**
fetched by GitHub Actions and are **not** baked into the Docker image. They
live in each environment's AWS Secrets Manager, and the non-secret config
(third-party URLs) lives in that account's SSM Parameter Store.

`ecs/task-def-<env>.json` references them via `secrets[].valueFrom` with the
ARN. ECS itself resolves these at container startup and injects them as
environment variables inside the container — the CI pipeline only ever
handles an image URI, nothing sensitive.

This requires the task's `executionRoleArn` to have `secretsmanager:GetSecretValue`
and `ssm:GetParameters` permissions scoped to the `myapp/<env>/*` path in that
account.

## Setting up OIDC trust (once per AWS account)

In each of the dev/qa/prod AWS accounts, create an IAM role trusted by
GitHub's OIDC provider, scoped to this repo:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com" },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
        "StringLike": { "token.actions.githubusercontent.com:sub": "repo:<org>/<repo>:environment:dev" }
      }
    }
  ]
}
```

Repeat with `environment:qa` / `environment:prod` for the other two roles.
This means even if the `prod` role ARN leaked, it can only be assumed by a
workflow run explicitly tied to the `prod` GitHub Environment — not just any
branch or workflow in the repo.

## Replace before use

In each `ecs/task-def-<env>.json`:
- `<DEV_ACCOUNT_ID>` / `<QA_ACCOUNT_ID>` / `<PROD_ACCOUNT_ID>` — that account's AWS account ID
- Secret/parameter ARNs — match whatever you actually name your secrets in Secrets Manager and SSM
- `taskRoleArn` — the role your running container assumes for its own AWS API calls (e.g., S3, SES), separate from `executionRoleArn` which ECS uses just to pull secrets/images
