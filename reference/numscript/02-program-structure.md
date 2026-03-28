# Numscript Program Structure

> Source: <https://docs.formance.com/modules/numscript/program-structure>

## Overview

A Numscript program's primary purpose is to output financial transactions, based on the rules
and constraints defined in the script, along with an initial state of the system of accounts
provided as input.

The interpreter processes transactions, which are then applied to relevant accounts by the
system utilizing Numscript.

## Program Components

A Numscript program consists of the following building blocks:

### 1. Feature Declarations (Optional)

Feature flags can be declared at the top of a program to enable experimental features:

```numscript
#![feature("experimental-overdraft-function")]
```

### 2. Variables Declaration (Optional)

Enables definition of variables for use throughout the program, eliminating the need to
hardcode specific values:

```numscript
vars {
  monetary $price
  account  $seller
  portion  $commission
}
```

### 3. Statements

#### Send Statements

The core of a Numscript program. Define how value moves between different accounts:

```numscript
send [USD/2 100] (
  source = @world
  destination = @users:001
)
```

#### Save Statements

Define minimum balance protections:

```numscript
save [USD/2 100] from @merchants:1234
```

#### Metadata Statements (Optional)

Allow developers to assign metadata to either accounts or the transaction itself:

```numscript
set_tx_meta("reference", "order-12345")
set_account_meta(@users:001, "status", "active")
```

## Formal Grammar

The complete Numscript grammar specification is available as an ANTLR grammar file
(`Numscript.g4`) in the project repository at:
<https://github.com/formancehq/numscript/blob/main/Numscript.g4>
