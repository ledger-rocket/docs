# Tutorial: Setting Up ACH Processing

> Source: https://www.twisp.com/docs/tutorials/ach/setting-up-ach

## Overview

This tutorial walks through setting up ACH processing on the Twisp platform, resulting in a fully configured ACH processor. Takes approximately 15 minutes.

## Prerequisites

**Required:**

- Twisp account with API access
- GraphQL API endpoint URL
- API authentication token

**Not Required:**

- ODFI relationship
- Double-entry accounting knowledge
- Prior ACH experience

## Step-by-Step Setup

### Step 1: Create the Journal

Create a journal to track all ACH-related accounting entries via GraphQL mutation with status ACTIVE.

### Step 2: Create Required Accounts

Four accounts must be established:

1. **Settlement Account** - Where funds transit during ACH processing
2. **Suspense Account** - Holds transactions when destination accounts cannot be located
3. **Exception Account** - Receives transactions that fail processing rules
4. **Fee Account** - Collects ACH processing fees

All accounts are configured with `enableConcurrentPosting: true` to support high transaction volumes.

### Step 3: Create the Webhook Endpoint

A webhook endpoint receives requests for transaction decisioning when ACH files are processed. For testing, use webhook.site to create a free test URL.

### Step 4: Create the ACH Configuration

This connects the journal, all four accounts, the webhook endpoint, and ODFI header information. Use placeholder values for testing, replace with real bank information before production.

### Step 5: Verify the Configuration

A verification query confirms all components were created correctly.

## Important IDs to Save

- Journal ID
- Settlement Account ID
- Suspense Account ID
- Exception Account ID
- Fee Account ID
- Webhook Endpoint ID
- ACH Configuration ID

## Production Requirements Checklist

Before deploying to production:

- Establish an ODFI relationship with a bank
- Obtain real ODFI header values from the bank
- Update configuration with production values
- Set up SFTP/FTPS file transmission
- Implement a production webhook server
- Test with the ODFI's validation process

## Troubleshooting

- **Account Already Exists**: Use existing accounts or select different UUIDs
- **Webhook Endpoint Failure**: Verify URL format starts with `https://` and check network connectivity
- **Configuration Creation Failure**: Confirm all referenced IDs exist and UUIDs are properly formatted
