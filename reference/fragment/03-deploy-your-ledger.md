# Deploy Your Ledger

Once you've designed your Ledger, you can deploy it using three primary methods: the dashboard, the API, or the CLI embedded in your CI/CD pipeline.

## Dashboard

The Fragment Dashboard provides an interface for editing and storing your Schema and creating Ledgers directly. This approach works well during development but isn't recommended for production workflows due to limited automation capabilities.

## API

The API enables automated Ledger deployment, making it suitable for managing multiple environments and keeping them synchronized. You can call two key operations:

**storeSchema** - Stores your Schema configuration. Depending on your use case, you may either share one Schema across all users or create a Schema per user.

**createLedger** - Creates a new Ledger instance using a stored Schema. When calling this mutation, reference the `key` from your `storeSchema` call. You can also set the `balanceUTCOffset` parameter to specify the timezone for balance calculations.

These API operations allow you to create many Schemas and Ledgers programmatically from within your product.

## CLI

The Fragment CLI can be integrated into your CI/CD pipeline for automated Schema storage. A typical workflow includes:

1. Installing Node.js 20.x or later
2. Installing the CLI via npm (`npm install -g @fragment-dev/cli`)
3. Authenticating with Fragment using your credentials
4. Running `fragment store-schema --path my-schema.jsonc` to deploy your Schema

This approach keeps your Schema definitions in version control and automates deployment across environments. For additional CLI commands and options, consult the CLI Command Reference documentation.
