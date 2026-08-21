# R5 - Database reachable · Tasks

First non-pure requirement. Proves the connection before any schema exists, so a connection failure and a schema failure are never diagnosed as one problem.

- [ ] 1. Create the Supabase project
  - Free tier. Note that projects pause after ~7 days idle; four fetches a day means this never fires
  - _Requirements: 5.4_

- [ ] 2. Take the **pooler** connection string
  - Transaction mode, port `6543`. Not direct `5432` - a per-invocation Lambda connection exhausts the limit
  - Keep it local for now; it moves to SSM in R9
  - _Requirements: 5.1, 5.4_

- [ ] 3. Write `requirements.txt`
  - Exactly `psycopg[binary]`, `httpx`, `tzdata`. Nothing else
  - Supabase client libraries are not a dependency; this is plain Postgres via `psycopg` v3
  - _Requirements: 5.2, 5.3_

- [ ] 4. Set up the local environment
  - Virtualenv, `pip install -r requirements.txt`
  - Also `pip install boto3` as a **dev-only** dependency - needed from R7 onward, and it must not enter `requirements.txt` or the layer
  - _Requirements: 5.3_

## Gate

A one-liner from the laptop against the pooler string returns `SELECT 1`, and the port in the string is `6543`.
