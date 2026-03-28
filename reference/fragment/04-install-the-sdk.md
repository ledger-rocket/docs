# Install the SDK

## CLI Setup

The Fragment CLI generates GraphQL queries specific to your Schema. Install via NPM:

```bash
npm install -g @fragment-dev/cli
```

Or using Homebrew:

```bash
brew tap fragment-dev/tap && brew install fragment-dev/tap/fragment-cli
```

Authenticate by running:

```bash
fragment login
```

## TypeScript

Install the Node SDK:

```bash
npm install --save @fragment-dev/node-client
```

Initialize with credentials from your Fragment dashboard:

```typescript
import { createFragmentClient } from '@fragment-dev/node-client';

const fragment = createFragmentClient({
  params: {
    clientId: "<Client ID>",
    clientSecret: "<Client Secret>",
    apiUrl: "<API URL>",
    authUrl: "<OAuth URL>",
    scope: "<OAuth Scope>",
  },
});

const { workspace } = await fragment.getWorkspace();
console.log('Workspace Name:', workspace.name);
```

### Generate Queries

```bash
fragment get-schema --output=fragment/schema.jsonc
fragment gen-graphql --path=fragment/schema.jsonc --output=fragment/queries.graphql
```

### Generate Client

```bash
npx fragment-node-client-codegen --input=fragment/queries.graphql --outputFilename=fragment/fragment-client.ts
```

## Python

Install via pip:

```bash
pip install fragment-python
```

Initialize the client:

```python
from fragment.sdk.client import Client

client = Client(
  client_id="<Client ID>",
  client_secret="<Client Secret>",
  api_url="<API URL>",
  auth_url="<OAuth URL>",
  auth_scope="<OAuth Scope>",
)

async def print_workspace():
  response = await client.get_workspace()
  print(response.workspace.name)

import asyncio
asyncio.get_event_loop().run_until_complete(print_workspace())
```

### Generate Queries and Client

```bash
fragment get-schema --output=fragment/schema.jsonc
fragment gen-graphql --path=fragment_lib/schema.jsonc --output=fragment_lib/queries/queries.graphql
fragment-python-client-codegen --input-dir=fragment_lib/queries --target-package-name=sdk --output-dir=fragment_lib
```

## Go

Install the SDK:

```bash
go get 'github.com/fragment-dev/fragment-go/v3'
```

Initialize with credentials:

```go
import (
  "github.com/fragment-dev/fragment-go/v3/client"
  "github.com/fragment-dev/fragment-go/v3/queries"
)

func main() {
  graphqlClient, err := client.NewClient(
    &client.GetTokenParams{
      ClientID:     "<Client ID>",
      ClientSecret: "<Client Secret>",
      Scope:        "<OAuth Scope>",
      AuthURL:      "<OAuth URL>",
      ApiURL:       "<API URL>",
    },
  )
  response, _ := queries.GetWorkspace(context.Background(), graphqlClient)
}
```

### Generate Code

```bash
fragment get-schema --output=fragment/schema.jsonc
fragment gen-graphql --path=fragment/schema.jsonc --output=fragment/queries.graphql
go run github.com/fragment-dev/fragment-go/v3 --input=fragment/queries.graphql --output=fragment/generated.go --package=main
```

## Ruby

Install the gem:

```bash
gem install fragment-dev
```

Initialize:

```ruby
require 'fragment_client'

fragment = FragmentClient.new(
  "<Client ID>",
  "<Client Secret>",
  api_url: "<API URL>",
  oauth_url: "<OAuth URL>",
  oauth_scope: "<OAuth Scope>"
)

workspace = fragment.get_workspace()
```

### Use Custom Queries

```bash
fragment get-schema --output=fragment/schema.jsonc
fragment gen-graphql --path=fragment/schema.jsonc --output=fragment/queries.graphql
```

Then initialize with the generated file:

```ruby
fragment = FragmentClient.new(
  "<Client ID>",
  "<Client Secret>",
  api_url: "<API URL>",
  oauth_url: "<OAuth URL>",
  oauth_scope: "<OAuth Scope>",
  extra_queries_filenames: ["path/to/queries.graphql"]
)
```

## Other Languages

Generate GraphQL queries using the CLI:

```bash
fragment get-schema --output=fragment/schema.jsonc
fragment gen-graphql --path=fragment/schema.jsonc --output=fragment/queries.graphql
```

Use any GraphQL codegen tool for your language. The Fragment API schema is hosted at:

```text
https://api.fragment.dev/schema.graphql
```

Implement OAuth2 client credentials flow for authentication and add error handling and retry logic as needed.
