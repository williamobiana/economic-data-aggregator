## 1. Pick a database that avoids the VPC
Two good options. Aurora Serverless v2 with the Data API turned on lets Lambda talk to it over plain HTTPS, so no VPC and no NAT gateway; set minimum capacity to 0 ACU so it sleeps between fetches. Or keep Neon/Supabase for the database and use AWS only for compute — cheaper and simpler, and nothing else in this plan changes. Store the credentials in Secrets Manager either way.

## 2. Create an S3 bucket for raw payloads
One bucket, two prefixes: raw/ for the verbatim payloads and exports/ for the CSV and JSON dumps. On AWS it's cleaner to put the actual JSON in S3 and have the raw_snapshots row hold the object key alongside fetched_at, week_key and week_final. Turn on versioning and a lifecycle rule if you want, but never delete raw payloads.

## 3. Run the migration
Run the migration once from your laptop against the new database: raw_snapshots, events, fetch_log, including the four empty actual_* columns. Same schema as before apart from raw_snapshots pointing at S3 instead of holding jsonb.

## 4. Write the parser and the Lambda handler
The parser and its tests are ordinary Node code and carry over unchanged — write and green them locally before touching AWS. Then wrap the pipeline in a handler: fetch, reject HTML, filter to the 8 currencies, put the raw payload in S3, parse, upsert, write fetch_log. Read week_offset and week_final from the event payload so one function serves all three schedules.

## 5. Deploy with SAM or CDK
Use AWS SAM or CDK so the function, its IAM role, the schedules and the alarm all live in one template you can redeploy. Give the role only what it needs: read the secret, write to the bucket. Set timeout around 30 seconds, reserved concurrency to 1 so two runs can never overlap, and Lambda's own retry attempts to zero — you want the 10-minute floor, not an instant retry that gets you rate-limited.

## 6. Set up EventBridge schedules
Four rules for the daily harvest, one for the Saturday 23:00 sweep passing week_final true, and optionally the next-week probe. Add a matching retry rule 10 minutes after each main slot; the handler checks fetch_log first and exits immediately if the previous attempt already succeeded. EventBridge Scheduler takes cron in UTC directly.

## 7. Wire up failure emails
Have the handler emit a CloudWatch metric or log line on failure, then alarm on it and send to an SNS topic subscribed to your email. Add a second alarm for the Saturday sweep specifically — that's the one you rerun by hand the same evening if it fails.

## 8. Add export, then check after two weeks
A second small Lambda dumps the events table to CSV and JSON in the exports/ prefix on a Sunday schedule. Then after two weeks running, query fetch_log for gaps, confirm both week_final snapshots exist in S3, and confirm the week rollover added rows rather than overwriting the old ones.
