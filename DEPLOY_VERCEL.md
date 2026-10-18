# Deploy TraceMatrix with Vercel

Vercel hosts the static React/Vite frontend. The FastAPI backend, PostgreSQL
database, and (if using queued background jobs) Redis and a worker must run on
a separate hosting provider.

## 1. Deploy the backend and database

1. Provision PostgreSQL and Redis with a host that supports Docker containers.
2. Create a backend service from this repository with its Docker build context
   set to `backend` (the directory containing `Dockerfile`).
3. Set the backend environment variables in that host's service settings:

   - `APP_ENV=production`
   - `SECRET_KEY` to a new, long, randomly generated secret
   - `DATABASE_URL` to the provider's PostgreSQL URL, using the
     `postgresql+psycopg://` SQLAlchemy URL scheme
   - `REDIS_URL` to the provider's Redis URL if running queued background jobs
   - `FRONTEND_ORIGIN` to the exact Vercel production origin, for example
     `https://your-project.vercel.app` (no trailing slash)
   - `BOOTSTRAP_ADMIN_EMAIL` and `BOOTSTRAP_ADMIN_PASSWORD` for the first admin
     account

   Add any optional integration credentials (such as `ETH_RPC_URL` or
   `GITHUB_TOKEN`) through the provider's secret/environment settings, not by
   committing them to the repository.
4. Deploy the backend and confirm its `/health` endpoint responds successfully.
   The backend container runs `alembic upgrade head` before starting the API.
5. If `REDIS_URL` is configured, create a separate worker service from the same
   backend image with the command `python -m app.worker`. Give it the same
   database and Redis environment variables. Without Redis, jobs run in-process
   and no worker service is needed.
6. Create the initial administrator once the database is reachable. Run
   `python -m app.cli bootstrap` in the backend service/container, with
   `BOOTSTRAP_ADMIN_EMAIL` and `BOOTSTRAP_ADMIN_PASSWORD` set. Do not use the
   example or development credentials in production.

## 2. Deploy the frontend to Vercel

1. Push this project to a Git provider and import the repository into Vercel.
2. Set **Root Directory** to `tracematrix-fixed-v2/tracematrix/frontend`.
3. Use the Vite framework preset, build command `npm run build`, and output
   directory `dist`.
4. Add the Vercel project environment variable:

   - `VITE_API_BASE_URL` = the backend's public origin, for example
     `https://your-api.example.com` (no trailing slash)

   Set it for each Vercel environment you intend to use. Vite embeds this value
   at build time, so redeploy after changing it.
5. Deploy. The included `vercel.json` serves the SPA entry point for client-side
   routes.

The backend's `FRONTEND_ORIGIN` must match the deployed Vercel origin exactly
because the API restricts cross-origin requests. If you attach a custom domain,
update `FRONTEND_ORIGIN` to that domain and redeploy the backend. Preview
deployments use different origins; either set up a separate backend environment
for previews or use the production deployment for normal user access.

## Local development

Leave `VITE_API_BASE_URL` empty (or unset) to keep API requests on the frontend
origin. Vite's existing development proxy forwards `/api`, `/health`, and
`/ready` requests to `http://127.0.0.1:8000`.

Never commit `.env` files containing real credentials. Use the checked-in
`.env.example` files only as templates.
