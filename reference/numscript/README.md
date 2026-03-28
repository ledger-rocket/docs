# Numscript Documentation

Numscript is a Domain-Specific Language (DSL) by [Formance](https://www.formance.com) designed
to model complex financial transactions, replacing complex and error-prone custom code with
easy-to-read, declarative scripts.

## Documentation Index

| File | Description |
|------|-------------|
| [01-introduction.md](01-introduction.md) | Overview, design principles, and core concepts |
| [02-program-structure.md](02-program-structure.md) | Program components and structure |
| [03-send.md](03-send.md) | The `send` statement reference |
| [04-sources.md](04-sources.md) | Source specifications (single, ordered, portioned, nested) |
| [05-destinations.md](05-destinations.md) | Destination specifications (single, allocation, kept, ordered, nested) |
| [06-variables.md](06-variables.md) | Variable declarations, types, and injection |
| [07-metadata.md](07-metadata.md) | Reading and writing metadata on accounts and transactions |
| [08-overdraft.md](08-overdraft.md) | Unbounded and bounded overdraft controls |
| [09-save.md](09-save.md) | Minimum balance protection with `save` |
| [10-rounding.md](10-rounding.md) | Deterministic rounding and remainder distribution |
| [11-monetary-notation.md](11-monetary-notation.md) | Unambiguous Monetary Notation (UMN) specification |
| [12-accounting-model.md](12-accounting-model.md) | Source/destination accounting model, transactions, accounts, constraints |
| [13-cli.md](13-cli.md) | CLI installation, commands (`check`, `run`, `test`) |
| [14-specs-format.md](14-specs-format.md) | JSON specs format for testing Numscript programs |
| [15-interpreter.md](15-interpreter.md) | Selecting and configuring the Numscript interpreter |
| [16-grammar.md](16-grammar.md) | ANTLR grammar (Lexer.g4 and Numscript.g4) |
| [17-examples.md](17-examples.md) | Practical examples: ride-sharing, marketplace, food delivery |
| [18-blog-overview.md](18-blog-overview.md) | "What is Numscript and Why is it Awesome?" blog post content |

## Official Resources

- Documentation: <https://docs.formance.com/modules/numscript/introduction>
- GitHub: <https://github.com/formancehq/numscript>
- Playground: <https://playground.numscript.org/>
- VS Code Extension: Available on the VS Code Marketplace
- Hacker News discussion: <https://news.ycombinator.com/item?id=41593216>
