# Selecting a Numscript Interpreter

> Source: <https://docs.formance.com/modules/numscript/selecting-an-interpreter>

## Overview

The Ledger service offers two Numscript interpreter options as of version 2.2:

1. **Original interpreter** (default): The established built-in interpreter. Enabled by
   default but is no longer evolving.
2. **New interpreter**: A newer, portable version designed for independent or integrated
   operation. Will receive ongoing updates and must be manually enabled.

## Enabling the New Interpreter

To activate the newer interpreter version, configure two settings:

- `ledger.experimental-features`
- `ledger.experimental-numscript`

These settings are available in the operator configuration reference (settings documentation).

## Playground Difference

The online Numscript Playground at <https://playground.numscript.org/> uses the new
interpreter, so results there may differ from those in production if you are still using the
original interpreter. Always verify scripts work with your active interpreter version.

## Recommendation

The new interpreter is recommended for new projects as it will receive ongoing feature
updates and improvements. The original interpreter remains available for backward
compatibility.
