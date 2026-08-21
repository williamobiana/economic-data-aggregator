# R7 - Object store · Tasks

First AWS requirement, and it comes **before** the handler - R8's gate asserts against objects under `raw/`.

- [ ] 1. Create the S3 bucket
  - One bucket, prefixes `raw/<week_key>/<fetched_at>.json` and `exports/<date>/`
  - _Requirements: 7.1_

- [ ] 2. Configure it
  - Block all public access
  - **Versioning off** - `<fetched_at>` and `<date>` are then the only overwrite protection, so never simplify either key
  - No lifecycle expiry on `raw/`; lifecycle on `exports/` is permitted
  - _Requirements: 7.1, 7.2, 7.3_

- [ ] 3. Confirm local write access
  - Local runs use the developer's own admin credentials from `~/.aws/credentials`; Lambda uses the execution role from R9
  - `boto3` resolves both through its default chain, so no profile handling and no code branch
  - _Requirements: 7.4_

## Gate

- `aws s3api get-bucket-versioning` returns empty or `Suspended`
- `get-public-access-block` all true
- `get-bucket-lifecycle-configuration` contains no rule matching `raw/`
- A put and get round-trip from the laptop under `raw/` succeeds and the bytes match - this proves the local credentials, not just the bucket
