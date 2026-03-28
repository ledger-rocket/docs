# Numscript CLI

> Source: <https://docs.formance.com/modules/numscript/cli>

## Overview

The Numscript CLI provides tools for installing, checking, running, and testing Numscript
programs locally.

## Installation

### Via curl (Mac/Unix)

```bash
curl -sSf https://raw.githubusercontent.com/formancehq/numscript/main/install.sh | bash
```

### Via Go toolchain

```bash
go install github.com/formancehq/numscript/cmd/numscript@latest
```

## Commands

### `numscript check`

Run static analysis on a Numscript program to identify parsing errors, variable and type
misuse, and validate Numscript constructs:

```bash
numscript check my-file.num
```

Returns an error status if warnings or errors are detected.

### `numscript run`

Available from version 0.0.19+. Executes local scripts for prototyping.

#### Script File (`my-script.num`)

```numscript
vars {
  monetary $amt
}

send $amt (
  source = @alice
  destination = @world
)
```

#### Inputs File (`my-script.num.inputs.json`)

```json
{
  "variables": {
    "amt": "USD/2 100"
  },
  "balances": {
    "alice": { "USD/2": 9999 }
  }
}
```

#### Usage

```bash
numscript run my-script.num
```

Output displays a postings table with source, destination, asset, and amount columns.

### `numscript test`

Available from version 0.0.19+. Validates specifications against Numscript files using the
specs format (see [14-specs-format.md](14-specs-format.md)).

```bash
numscript test src/domain/numscript
```

This scans folders for `.num` files with corresponding `.num.specs.json` specification files.

## Language Server

The CLI includes a language server providing:

- **Diagnostics**: Error detection and reporting
- **Hover information**: Details about values on hover
- **Document symbols**: Outline of program structure
- **Go-to-definition**: Navigate to variable/account definitions

## VS Code Extension

A VS Code extension is available on the Marketplace for syntax highlighting and language
server integration.

## Repository

GitHub: <https://github.com/formancehq/numscript>

Latest release: v0.0.24 (as of March 2026)
