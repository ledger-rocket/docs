# Export Data

## S3 Export

Fragment enables exporting Ledger data to AWS S3, making data accessible for analytics, regulatory compliance, and integration with third-party systems.

### Export Format

Data exports are delivered approximately every 5 minutes, containing Ledger data created or updated since the previous export cycle.

Exports are organized by data type into three files with newline-separated JSON format, where each line represents a single instance:

- LedgerEntry
- LedgerLine
- LedgerAccount

Files follow the naming scheme: `{File Prefix}/{type}/day={day}/hour={hour}/{partition}.part`

### File Size Limits

Each file accommodates a maximum of 5,000 records. When batches exceed this threshold, records are split across multiple files:

- `LedgerLine/day=2024-01-15/hour=14/1705326000000-abc123.part` (records 1-5000)
- `LedgerLine/day=2024-01-15/hour=14/1705326001000-def456.part` (records 5001-10000)

### Onboarding

To enable data export:

1. Create an S3 bucket in AWS (no special configuration required)
2. Navigate to Settings -> S3 Exports in the Fragment dashboard
3. Provide the following information:
   - S3 Bucket Name
   - S3 Bucket Region (e.g., `us-east-1`)
   - Export Name (display purposes)
   - File Prefix (optional path for storing exports)

Apply the S3 bucket policy as instructed in the dashboard. The onboarding flow can be restarted at any time if configuration takes time to propagate.

### Testing

The dashboard allows testing the connection by writing a sample file to `test/testLedgerEntry.part`.

Possible test results:

- **Policy Not Applied**: Authorization error; policy not yet applied or bucket details are incorrect
- **Invalid Bucket Name**: Provided bucket name doesn't exist in AWS
- **Incorrect AWS Region**: Bucket exists in a different region than specified
- **Verified Permissions**: File successfully written

Note: Test files are not automatically removed after verification.

## Retool Integration

Fragment can be added as a GraphQL resource in Retool:

1. Create an API client in the Fragment dashboard
2. Add a new GraphQL Resource in Retool
3. Set Base URL to your API URL
4. Add `Authorization` Header: `Bearer OAUTH2_TOKEN` (Retool replaces token at runtime)
5. Set Authentication to `OAuth 2.0`
6. Enable `Use Client Credentials Flow`
7. Configure OAuth settings:
   - Access Token URL: OAuth URL
   - Client ID and Secret
   - Scopes: OAuth Scope
   - Prompt: `consent`
8. Save the resource

Note: Connection testing in Retool may fail even when configured correctly. Verify functionality by running queries within an app.
