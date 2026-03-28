# Security and Authentication

> Source: https://www.twisp.com/docs/infrastructure/security-and-auth

## Overview

Security policies control access to encrypted resources in Twisp's infrastructure. The system provides "out of the box" encryption for data at rest and in transit, with all access authenticated via JWT tokens issued through the OpenID Connect 1.0 protocol.

Users can configure additional security settings by establishing tenants and clients through GraphQL mutations to define precise permissions on specific operations.

## The Authentication and Authorization Flow

Twisp applies policies to an authenticated principal before granting access to resources.

The authentication process consists of four key stages:

1. A principal submits an HTTPS request with their OIDC JWT and Twisp account IDs in the headers
2. The principal name is extracted from the JWT
3. Twisp locates corresponding policies using the principal name
4. The resulting policies are evaluated for authorization

### Making an Authenticated Request

All HTTPS requests to the GraphQL API must include these headers:

```text
Authorization: Bearer <JWT>
x-twisp-account-id: <Twisp account id>
```

The JWT can originate from either OpenID Connect or AWS IAM.

### Authorizing a Principal

Within a client configuration, the security principal represents the authenticated identity accessing the system. The authorization system retrieves policies based on the principal name -- typically the issuer from the `iss` claim for OIDC, or the AWS identity from the `sub` claim for IAM tokens.

## Provisioning Tenants

Each cloud environment requires its own tenant. Use the admin GraphQL namespace to create tenants:

```graphql
mutation CreateTenants {
  admin {
    staging:createTenant(
      input: {
        id: "d9d8f1c0-0299-4d5b-b2b6-85beafdda28b"
        accountId: "TwispStaging"
        name: "staging"
        description: "staging environment for Twisp"
      }
    ) {
      accountId
      name
    }
    production:createTenant(
      input: {
        id: "848df974-4133-4ee4-ab45-86e5e29b6822"
        accountId: "TwispProd"
        name: "production"
        description: "production environment for Twisp"
      }
    ) {
      accountId
      name
    }
  }
}
```

## Creating Clients and Policies

Authentication requires any OIDC-compliant token supplied through the `Authorization: Bearer <token>` header. A corresponding client must exist in Twisp for the token's principal so the system knows which policies to apply.

### Third-Party OIDC Client Example

For tokens from external identity providers, set the client principal to the OIDC issuer URL:

```graphql
mutation CreateGoogleCloudClient {
  auth {
    createClient(
      input: {
        name: "michael gcloud readonly"
        principal: "https://accounts.google.com"
        policies: [
          {
            actions: [SELECT]
            resources: ["*"]
            effect: ALLOW
            assertions: {
              isMike: "context.auth.claims.email == 'michael@twisp.com'"
            }
          }
          {
            actions: [SELECT, INSERT, UPDATE, DELETE]
            resources: ["*"]
            effect: ALLOW
            assertions: {
              isJarred: "context.auth.claims.email == 'jarred@twisp.com'"
            }
          }
        ]
      }
    ) {
      principal
    }
  }
}
```

### AWS IAM Principal Example

Twisp exchanges a presigned AWS STS `GetCallerIdentity` request for an OIDC token:

```graphql
mutation CreateIAMAuthClient {
  auth {
    createClient(
      input: {
        principal: "arn:aws:iam::012345678901:role/example-role"
        name: "example role policy"
        policies: [
          {
            effect: ALLOW
            actions: [SELECT, INSERT, UPDATE, DELETE]
            resources: ["*"]
          }
        ]
      }
    ) {
      principal
    }
  }
}
```

### Understanding Policies

Each policy defines:

- **Effect**: ALLOW or DENY
- **Actions**: Permitted or denied operations
- **Resources**: In-scope resources
- **Assertions**: Optional CEL expressions that must evaluate to true

**Logical Combination:** Policies evaluate as chained AND statements scoped to the requested resource and action. A principal requires at least one ALLOW policy for a resource/action pair. A single DENY policy on that pair blocks the operation.

**Supported wildcards:** `*` and `?` values work in resources and actions.

**Available actions:**

- `db:Select` (SELECT in GraphQL): read a document
- `db:Insert` (INSERT): create a document
- `db:Update` (UPDATE): change document fields
- `db:Delete` (DELETE): remove a document

**Resource format:** `namespace.ledger.<document|indexes|joins|references>.propertyName`. The current namespace is `financial`.

**Assertions:** Use Common Expression Language (CEL) to access `context.auth.claims` (token claims) and `context.document` (the document being acted on). If any assertion evaluates to false, the policy is skipped.

## Using OpenID Connect Tokens

Any JWT from an OpenID Connect 1.0-capable issuer is supported by Twisp. When an API endpoint receives the JWT, it validates the token signature against the issuer and invokes the endpoint with the security principal set to the issuer's `iss` claim. All claims are available in `context.auth.claims` for policy evaluation.

## Issuing Tokens with AWS IAM

Twisp vends an OpenID Connect token in exchange for an authenticated AWS IAM role or user, allowing services in AWS to retrieve Twisp-issued tokens for access. The issuer is `https://auth.${AWS::Region}.prod.twisp.com/token/iam`, and the principal name equals the original IAM identity stored in the token's `sub` field.

## Cloud Endpoints

- **AWS Twisp Token:** `https://auth.us-east-1.cloud.twisp.com/token/iam`
- **Financial GraphQL:** `https://api.us-east-1.cloud.twisp.com/financial/v1/graphql`
- **gRPC:** `https://api.us-east-1.cloud.twisp.com:50051`

## Encryption

### Encryption in Transit

Data is encrypted via HTTPS connections for all external and internal API operations.

### Encryption at Rest

All stored data in Twisp is encrypted at rest. Encryption keys reside in AWS Key Management Service.
