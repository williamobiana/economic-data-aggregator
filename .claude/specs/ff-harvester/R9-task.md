# R9 - Remaining AWS prerequisites · Tasks

Three resources, least privilege, no monthly cost. The bucket is already up from R7.

- [ ] 1. Store the connection string in SSM
  - `SecureString`, the pooler string from R5
  - Not Secrets Manager - $0.40/secret/month breaks the $0 target
  - _Requirements: 9.1_

- [ ] 2. Create the Lambda execution role
  - Trusts `lambda.amazonaws.com`; shared by both functions
  - `AWSLambdaBasicExecutionRole`, `ssm:GetParameter` on that one parameter, `kms:Decrypt`, `s3:PutObject` on `<bucket>/raw/*` and `<bucket>/exports/*`
  - Nothing else, and no wildcards beyond those two prefixes
  - _Requirements: 9.2_

- [ ] 3. Build the dependency layer
  - `psycopg[binary]`, `httpx`, `tzdata`
  - Built for **manylinux** in Docker, CloudShell or EC2 - never from a laptop's `site-packages`
  - `tzdata` is mandatory; the Lambda image has no system tz database
  - _Requirements: 9.3_

## Gate

- `aws ssm get-parameter --with-decryption` returns the pooler string with `Type: SecureString`
- The role's inline policy JSON matches task 2 exactly - **presence is not the test, absence of anything else is**
- Unzipping the layer shows `psycopg_binary` `*.so` files with `manylinux` wheel provenance, and `zoneinfo` data present
