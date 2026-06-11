# Deploying to Vercel with Supabase

This guide describes how to run this Rallly fork on Vercel using Supabase for the
Postgres database and for file storage. It is intended for a small self-hosted
instance without separate Redis or object-storage infrastructure.

The application code is already compatible with this setup. The only deployment
specific code change is an optional connection-pool size limit, controlled by
`DATABASE_POOL_MAX`.

## Architecture

| Concern        | Provider           | Notes                                                        |
| -------------- | ------------------ | ----------------------------------------------------------- |
| Hosting        | Vercel             | Next.js app in `apps/web`, serverless functions             |
| Database       | Supabase Postgres  | Transaction pooler at runtime, session pooler for migrations |
| File storage   | Supabase Storage   | S3-compatible bucket for avatars and space images           |
| Rate limiting  | In-memory          | No Redis; limits apply per function instance                 |
| Sessions       | Database           | Stored in Postgres when Redis is not configured             |
| Email          | SMTP or AWS SES    | Configure a real SMTP provider; Mailpit is local only       |

## 1. Create the Supabase project

1. Create a new project at https://supabase.com and choose a region.
2. Set a database password and save it.
3. From **Project Settings > Database > Connection string**, collect two URLs:
   - **Transaction pooler** (port `6543`) for `DATABASE_URL`.
   - **Session pooler** (port `5432`) for `DIRECT_DATABASE_URL`.

   Both use the `...pooler.supabase.com` host and provide IPv4, which Vercel
   requires. The session pooler is used by Prisma migrations; the transaction
   pooler is used by the running app.

4. Append `?sslmode=require` to each URL.

Example shape:

```
DATABASE_URL=postgresql://postgres.<ref>:<password>@aws-0-<region>.pooler.supabase.com:6543/postgres?sslmode=require
DIRECT_DATABASE_URL=postgresql://postgres.<ref>:<password>@aws-0-<region>.pooler.supabase.com:5432/postgres?sslmode=require
```

## 2. Create the storage bucket

1. In Supabase, go to **Storage** and create a bucket (for example `rallly`).
2. Go to **Storage > S3 access keys** and create an access key pair.
3. Note the S3 endpoint: `https://<ref>.supabase.co/storage/v1/s3`.

These map to the `S3_*` environment variables. The existing S3 client already
uses path-style addressing, which Supabase Storage requires.

## 3. Configure the Vercel project

1. Import the repository into Vercel.
2. Set **Root Directory** to `apps/web`. The build command in
   `apps/web/vercel.json` runs the workspace build and then
   `prisma migrate deploy`.
3. Set the **Framework Preset** to Next.js (auto-detected).

## 4. Set Vercel environment variables

Add the following to the Vercel project (Production, and Preview if used):

```
# Database
DATABASE_URL=<transaction pooler URL, port 6543>
DIRECT_DATABASE_URL=<session pooler URL, port 5432>
DATABASE_POOL_MAX=1

# Core
SECRET_PASSWORD=<at least 32 characters>
NEXT_PUBLIC_BASE_URL=https://<your-vercel-domain>
NEXT_PUBLIC_SELF_HOSTED=true
SUPPORT_EMAIL=<your support email>

# Email (example: SMTP)
EMAIL_PROVIDER=smtp
SMTP_HOST=<smtp host>
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=<smtp user>
SMTP_PWD=<smtp password>

# Storage (Supabase Storage)
S3_BUCKET_NAME=rallly
S3_ENDPOINT=https://<ref>.supabase.co/storage/v1/s3
S3_REGION=<project region, e.g. eu-west-2>
S3_ACCESS_KEY_ID=<from Supabase S3 access keys>
S3_SECRET_ACCESS_KEY=<from Supabase S3 access keys>
```

Generate `SECRET_PASSWORD` with one of:

```
openssl rand -hex 32
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Leave `KV_REST_API_URL` and `KV_REST_API_TOKEN` unset to use in-memory rate
limiting and database-backed sessions.

## 5. Deploy

Trigger a deploy. On build, Vercel runs `prisma migrate deploy` against
`DIRECT_DATABASE_URL`, creating the schema in Supabase. The running app connects
through `DATABASE_URL`.

The two cron jobs in `apps/web/vercel.json` (daily housekeeping) are registered
automatically by Vercel.

## 6. Verify

- Open the deployed URL and create an account.
- Create a poll and confirm it persists (check the `polls` table in Supabase).
- Upload an avatar or space image and confirm the object appears in the bucket.

## Notes

- **Migrations run during build.** A failed migration fails the deploy. To run
  migrations manually instead, remove `&& pnpm db:deploy` from
  `apps/web/vercel.json` and run `pnpm db:deploy` locally against
  `DIRECT_DATABASE_URL`.
- **In-memory rate limiting** is per function instance and resets on cold start.
  This is acceptable for a small instance. Add Upstash Redis later by setting
  `KV_REST_API_URL` and `KV_REST_API_TOKEN`.
- **Connection limits.** `DATABASE_POOL_MAX=1` keeps each serverless instance to
  a single pooled connection. Raise it only if you observe pool exhaustion under
  load.
