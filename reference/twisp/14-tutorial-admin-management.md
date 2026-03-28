# Tutorial: Managing Tenants, Users, and Groups

> Source: https://www.twisp.com/docs/tutorials/admin-management

## Introduction

### Tenants

"A Tenant in Twisp represents an environment within an organization, typically associated with a specific application, service, or set of resources." Each tenant contains isolated ledgers deployed to specific regions. A unique `accountId` combined with an AWS region enables data isolation.

### Users

"Users are human members within an organization who interact with the Twisp accounting core." Each user has a unique email identifier and can belong to multiple groups. Their permissions derive from combined policies across all associated groups.

### Groups

"Groups are a logical grouping of users within an organization. They are used to manage access control and permissions for users." Each group associates with policies defining allowed member actions.

## Working with Users and Groups

### Managing Users

```graphql
mutation AdminCreateUser {
  admin {
    createUser(
      input: {
        id: "9cc8bd28-a36d-502e-89fd-7f1410c1b90a"
        groupIds: ["d57bd759-73d5-4452-a73e-12b590324e35"]
        email: "george@twisp.com"
      }
    ) {
      id
      email
      groupIds
    }
  }
}
```

New users automatically receive invitation emails with console access links.

### Managing Groups

```graphql
mutation AdminCreateGroup {
  admin {
    createGroup(
      input: {
        id: "917e123a-b89d-4ab5-b11c-cdf6aac80b63"
        name: "empty-policy-1"
        description: "A group with an empty policy"
        policy: "[]"
      }
    ) {
      name
      description
      policy
    }
  }
}
```

Update group policies:

```graphql
mutation AdminUpdateGroup {
  admin {
    updateGroup(
      input: {
        name: "empty-policy-1"
        description: "An empty policy layer will default to the base policy."
        policy: "[{\"actions\": [\"*\"],\"effect\": \"DENY\",\"resources\":[\"*\"], \"assertions\": {\"always false\": \"1 == 0\"}}]"
      }
    ) {
      name
      description
      policy
    }
  }
}
```

### Deleting Users and Groups

```graphql
mutation AdminDeleteUser {
  admin {
    deleteUser(email: "george@twisp.com") {
      email
    }
  }
}
```

## Creating a Tenant

```graphql
mutation AdminCreateTenant {
  admin {
    createTenant(
      input: {
        id: "72a0097f-239e-48ec-a417-49c318332ed6"
        accountId: "sandbox"
        name: "Sandbox"
        description: "Sandbox tenant for testing"
      }
    ) {
      accountId
      name
    }
  }
}
```

## Assigning Permissions

"Permissions are determined by the policies associated with each Group a User belongs to." Users in multiple groups receive cumulative permissions.

```graphql
mutation AdminUpdateUser {
  admin {
    updateUser(
      input: {
        email: "george@twisp.com"
        groupIds: ["152f0c89-6cba-53c9-955e-16d7cbc1f35e"]
      }
    ) {
      email
      groupIds
    }
  }
}
```

### Verifying Effective Permissions

"The user must have at least one ALLOW policy for each desired action and resource but will be blocked by any DENY policy on the same action and resource."

### Monitoring Access

```graphql
query GetAdminUsers {
  admin {
    users(first: 5) {
      nodes {
        email
        organizationId
        groupIds
      }
    }
  }
}
```
