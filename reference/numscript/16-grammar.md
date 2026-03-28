# Numscript ANTLR Grammar

> Source: <https://github.com/formancehq/numscript>

The Numscript language is formally defined using ANTLR grammar files. These define the
complete syntax of the language.

## Lexer Grammar (Lexer.g4)

```antlr
lexer grammar Lexer;

WS: [ \t\r\n]+ -> skip;

NEWLINE: [\r\n]+;

MULTILINE_COMMENT: '/*' (MULTILINE_COMMENT | .)* '*/' -> skip;

LINE_COMMENT: '//' .*? NEWLINE -> channel(HIDDEN);

VARS: 'vars';
MAX: 'max';
SOURCE: 'source';
DESTINATION: 'destination';
SEND: 'send';
FROM: 'from';
UP: 'up';
TO: 'to';
REMAINING: 'remaining';
ALLOWING: 'allowing';
UNBOUNDED: 'unbounded';
OVERDRAFT: 'overdraft';
ONEOF: 'oneof';
KEPT: 'kept';
SAVE: 'save';

LPARENS: '(';
RPARENS: ')';
LBRACKET: '[';
RBRACKET: ']';
LBRACE: '{';
RBRACE: '}';
COMMA: ',';
EQ: '=';
STAR: '*';
PLUS: '+';
MINUS: '-';
DIV: '/';
RESTRICT: '\\';
WITH: 'with';
SCALING: 'scaling';
THROUGH: 'through';
FEATURE: 'feature';
HASH_BANG: '#!';

PERCENTAGE_PORTION_LITERAL: [0-9]+ ('.' [0-9]+)? '%';

STRING: '"' ('\\"' | ~[\r\n"])* '"';

IDENTIFIER: [a-z]+ [a-z_]*;
NUMBER: [0-9]+ ('_' [0-9]+)*;
ASSET: [A-Z][A-Z0-9]* ('/' [0-9]+)?;

ACCOUNT_START: '@' -> pushMode(ACCOUNT_MODE);
COLON: ':' -> pushMode(ACCOUNT_MODE);

fragment VARIABLE_NAME_FRAGMENT: '$' [a-z_]+ [a-z0-9_]*;

mode ACCOUNT_MODE;
ACCOUNT_TEXT: [a-zA-Z0-9_-]+ -> popMode;
VARIABLE_NAME_ACC: VARIABLE_NAME_FRAGMENT -> popMode;

mode DEFAULT_MODE;
VARIABLE_NAME: VARIABLE_NAME_FRAGMENT;
```

## Parser Grammar (Numscript.g4)

```antlr
grammar Numscript;

options {
    tokenVocab = 'Lexer';
}

monetaryLit:
    LBRACKET (asset = valueExpr) (amt = valueExpr) RBRACKET;

accountLiteralPart:
    ACCOUNT_TEXT                                     # accountTextPart
    | VARIABLE_NAME_ACC                              # accountVarPart;

valueExpr:
    VARIABLE_NAME                                    # variableExpr
    | ASSET                                          # assetLiteral
    | STRING                                         # stringLiteral
    | ACCOUNT_START accountLiteralPart
      (COLON accountLiteralPart)*                    # accountLiteral
    | NUMBER                                         # numberLiteral
    | PERCENTAGE_PORTION_LITERAL                     # percentagePortionLiteral
    | monetaryLit                                    # monetaryLiteral
    | op=MINUS valueExpr                             # prefixExpr
    | left=valueExpr op=DIV right=valueExpr          # infixExpr
    | left=valueExpr op=(PLUS | MINUS) right=valueExpr # infixExpr
    | LPARENS valueExpr RPARENS                      # parenthesizedExpr
    | functionCall                                   # application;

functionCallArgs: valueExpr (COMMA valueExpr)*;
functionCall:
    fnName=(OVERDRAFT | IDENTIFIER) LPARENS functionCallArgs? RPARENS;

varOrigin: EQ valueExpr;
varDeclaration:
    type_=IDENTIFIER name=VARIABLE_NAME varOrigin?;
varsDeclaration: VARS LBRACE varDeclaration* RBRACE;

featureDecl:
    HASH_BANG LBRACKET FEATURE LPARENS
    (flag+=STRING (COMMA flag+=STRING)*)? RPARENS RBRACKET;

program: featureDecl? varsDeclaration? statement* EOF;

sentAllLit: LBRACKET (asset = valueExpr) STAR RBRACKET;

allotment:
    valueExpr                                        # portionedAllotment
    | REMAINING                                      # remainingAllotment;

colorConstraint: RESTRICT valueExpr;

source:
    address=valueExpr colorConstraint?
      ALLOWING UNBOUNDED OVERDRAFT                   # srcAccountUnboundedOverdraft
    | address=valueExpr colorConstraint?
      ALLOWING OVERDRAFT UP TO
      maxOvedraft=valueExpr                          # srcAccountBoundedOverdraft
    | address=valueExpr WITH SCALING
      THROUGH swap=valueExpr                         # srcAccountWithScaling
    | valueExpr colorConstraint?                     # srcAccount
    | LBRACE allotmentClauseSrc+ RBRACE              # srcAllotment
    | LBRACE source* RBRACE                          # srcInorder
    | ONEOF LBRACE source+ RBRACE                    # srcOneof
    | MAX cap=valueExpr FROM source                  # srcCapped;

allotmentClauseSrc: allotment FROM source;

keptOrDestination:
    TO destination                                   # destinationTo
    | KEPT                                           # destinationKept;

destinationInOrderClause: MAX valueExpr keptOrDestination;

destination:
    valueExpr                                        # destAccount
    | LBRACE allotmentClauseDest+ RBRACE             # destAllotment
    | LBRACE destinationInOrderClause*
      REMAINING keptOrDestination RBRACE             # destInorder
    | ONEOF LBRACE destinationInOrderClause*
      REMAINING keptOrDestination RBRACE             # destOneof;

allotmentClauseDest: allotment keptOrDestination;

sentValue:
    valueExpr                                        # sentLiteral
    | sentAllLit                                     # sentAll;

statement:
    SEND sentValue LPARENS SOURCE EQ source
      DESTINATION EQ destination RPARENS             # sendStatement
    | SAVE sentValue FROM valueExpr                  # saveStatement
    | functionCall                                   # fnCallStatement;
```

## Keywords Reference

| Keyword | Purpose |
|---------|---------|
| `vars` | Declare variable block |
| `send` | Transfer assets between accounts |
| `save` | Set minimum balance protection |
| `source` | Specify source account(s) |
| `destination` | Specify destination account(s) |
| `from` | Used in source/allotment clauses |
| `to` | Used in destination clauses |
| `max` | Cap amount from a source or to a destination |
| `remaining` | Capture remaining balance in allotments |
| `allowing` | Begin overdraft clause |
| `unbounded` | Unlimited overdraft |
| `overdraft` | Overdraft keyword / function |
| `up` | Used with `to` for bounded overdraft |
| `oneof` | Select one from multiple options |
| `kept` | Keep funds in source account |
| `with` | Used with scaling |
| `scaling` | Currency scaling operation |
| `through` | Used with scaling |
| `feature` | Feature flag declaration |

## Operators

| Operator | Purpose |
|----------|---------|
| `+` | Addition |
| `-` | Subtraction / negation |
| `/` | Division (used in fractions and asset scales) |
| `*` | Wildcard (all balance) |
| `%` | Percentage portion |
| `@` | Account prefix |
| `$` | Variable prefix |
| `:` | Account segment separator |
| `\` | Color/restrict constraint |

## Literal Types

| Type | Examples |
|------|---------|
| Account | `@world`, `@users:001`, `@payments:stripe` |
| Asset | `USD/2`, `EUR/2`, `BTC/8`, `COIN`, `GEM` |
| Monetary | `[USD/2 100]`, `[COIN 50]`, `[BTC/8 *]` |
| Number | `100`, `42`, `1_000_000` |
| String | `"hello"`, `"order-123"` |
| Percentage | `50%`, `10.5%`, `0.6%` |
| Fraction | `1/5`, `85/100`, `7/1999` |
| Variable | `$price`, `$user_id` |
