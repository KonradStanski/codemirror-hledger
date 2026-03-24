# hledger Journal Syntax Reference

This document describes the hledger journal file format as supported by the `codemirror-lang-hledger` grammar. It serves as both a language reference and a guide to the AST node types produced by the parser.

## File Structure

A journal file is a sequence of top-level items separated by blank lines:

- **Transactions** — dated financial entries with postings
- **Periodic transactions** — templates for recurring entries
- **Auto postings** — rules that automatically add postings
- **Directives** — configuration and declarations
- **Comments** — line and block comments
- **Blank lines** — separators between items

---

## Transactions

A transaction starts with a date on an unindented line, followed by indented postings and comments.

```
DATE [STATUS] [DESCRIPTION] [; COMMENT]
    [STATUS] ACCOUNT  [AMOUNT] [COST] [BALANCE_ASSERTION] [; COMMENT]
    [STATUS] ACCOUNT  [AMOUNT] [COST] [BALANCE_ASSERTION] [; COMMENT]
    [; COMMENT]
```

### AST: `Transaction > TxnHeader, Posting*, IndentedComment*`

`TxnHeader` is split into sub-nodes:
- **`TxnDate`** — the date (including optional secondary date with `=`)
- **`TxnDescription`** — the status/code/description text before any same-line `;` comment
- **`InlineComment`** — an optional same-line transaction comment

### Dates

Supported separators: `-`, `/`, `.` (must be consistent within a date). Secondary dates use `=`.

```
2024-01-15 Grocery store
2024/01/15 Grocery store
2024.01.15 Grocery store
2024-01-15=2024-01-16 Transfer     ; secondary date
1/15 Short date
```

### AST: `TxnDate`, `TxnDescription`, `InlineComment`

### Status Markers

Optional, after the date (transactions) or indentation (postings):

| Marker | Meaning   |
|--------|-----------|
| `*`    | Cleared   |
| `!`    | Pending   |
| _(none)_ | Unmarked |

```
2024-01-15 * Cleared transaction
    ! expenses:food  $50
    assets:bank
```

### AST: `Status`

---

## Postings

Each posting is an indented line within a transaction (or periodic/auto posting block).

```
    ACCOUNT  AMOUNT
    ACCOUNT                     ; elided amount (inferred)
    * ACCOUNT  AMOUNT           ; cleared posting
    (virtual:account)  AMOUNT   ; unbalanced virtual posting
    [virtual:account]  AMOUNT   ; balanced virtual posting
```

### AST: `Posting > PostingIndent, Status?, AccountName, Amount?, PostingAnnotation*, InlineComment?`

### Account Names

Hierarchical, colon-separated. May contain single spaces within segments. In journal syntax they may also contain semicolons. Account names are terminated by double-space, tab, or end of line.

```
    assets:bank:td:checking_4506  $100
    expenses:alcohol & bars       $25
```

### AST: `AccountName`

### Amounts

Format: `[SIGN] [COMMODITY] [SIGN] NUMBER [COMMODITY]`

The sign (`+` or `-`) may appear before or after the commodity symbol.

```
    assets:bank  $100           ; prefix commodity
    assets:bank  100 EUR        ; suffix commodity
    assets:bank  -$50           ; sign before commodity
    assets:bank  $-50           ; sign after commodity
    assets:bank  $1,000.50      ; digit grouping
    assets:bank  10 "AAPL"      ; quoted commodity
    assets:bank  €100           ; unicode currency symbol
```

Supported currency symbols: `$`, `€`, `£`, `¥`, `₹`, `₽`, `₿`, `₩`, `₪`, `₺`, `₴`, `₦`, `₡`, `₣`, `₤`, `₧`, `₨`, and uppercase-letter commodities like `USD`, `EUR`, `BTC`.

### AST: `Amount > Sign?, Commodity?, Sign?, Number, Commodity?`

### Cost Annotations

Unit cost (`@`) or total cost (`@@`):

```
    assets:cash  -20 EUR @ 7.53 HRK       ; unit cost
    assets:cash  -20 EUR @@ 150.60 HRK    ; total cost
```

### AST: `CostAnnotation > CostOp, Sign?, Commodity?, Sign?, Number, Commodity?`

### Balance Assertions

Assert the account balance after a posting:

| Operator | Meaning                           |
|----------|-----------------------------------|
| `=`      | Assert single-commodity balance   |
| `==`     | Assert total (all commodities)    |
| `=*`     | Assert inclusive of subaccounts   |
| `==*`    | Assert total inclusive            |

```
    assets:bank  $100 = $1000
    income       ==* 0
```

### AST: `BalanceAssertion > BalanceOp, Sign?, Commodity?, Sign?, Number, Commodity?`

### Lot Prices

Lot price annotations may be unit-cost (`{...}`) or total-cost (`{{...}}`) and can be fixed with `=`.

```
    assets:stock  1 A {2 B}
    assets:stock  1 A {=2 B}
    assets:stock  1 A {{2 B}}
    assets:stock  1 A {{ = 2 B }}
```

### AST: `LotPrice > "{"/"{{", LotPriceFixed?, Sign?, Commodity?, Sign?, Number, Commodity?, "}"/"}}" `

### Lot Dates

Lot dates are square-bracket annotations attached to postings:

```
    assets:stock  1 A [2000-01-01]
```

### AST: `LotDate > "[", LotDateBody, "]"`

---

## Periodic Transactions

Templates for recurring entries, starting with `~`:

```
~ monthly from 2024-01
    expenses:rent  $1,200
    assets:bank
```

### AST: `PeriodicTransaction > PeriodicHeader, Posting*, IndentedComment*`

`PeriodicHeader` is split into sub-nodes:
- **`PeriodicMark`** — the `~` character
- **`PeriodicExpression`** — the period expression (e.g. `monthly from 2024-01`)

---

## Auto Postings (Transaction Modifiers)

Rules that automatically add postings to matching transactions, starting with `=`:

```
= expenses:food
    budget:food  -1
```

### AST: `AutoPosting > AutoHeader, Posting*, IndentedComment*`

`AutoHeader` is split into sub-nodes:
- **`AutoMark`** — the `=` character
- **`AutoQuery`** — the query expression (e.g. `expenses:food`)

---

## Directives

All directives start at the beginning of a line (no indentation). Each directive has a keyword node and an argument node.
Legacy `!` / `@` directive prefixes are accepted and are styled as part of the directive keyword token.

### Account Declaration

Declares an account name. May have a same-line `;` comment and indented sub-comments for metadata.

```
account assets:bank:checking
account assets:cash  ; type:A
    ; type: Asset
    ; description: Main checking account
```

**AST:** `AccountDirective > AccountKeyword, DirectiveAccountName, InlineComment?, Newline, IndentedComment*`

### Commodity Declaration

Declares a commodity with display format. May have a same-line `;` comment and `format` subdirectives.

```
commodity $1,000.00
commodity $1.00 ; display format
commodity EUR
    format EUR 1.000,00
```

**AST:** `CommodityDirective > CommodityKeyword, DirectiveArgument, InlineComment?, Newline, (CommodityFormatDirective | IndentedComment)*`
**AST:** `CommodityFormatDirective > PostingIndent, FormatKeyword, DirectiveArgument, InlineComment?, Newline`

### Include

Includes another journal file. Supports glob patterns.

```
include ./accounts.journal
include transactions/*.journal
include **/*.journal
!include legacy.journal
```

**AST:** `IncludeDirective > IncludeKeyword, IncludePath, InlineComment?, Newline`

### Alias

Defines account name aliases (literal or regex-based).

```
alias expenses = equity:draw:personal
```

**AST:** `AliasDirective > AliasKeyword, DirectiveArgument, Newline`

### Payee

Declares a payee name.

```
payee Amazon
payee Amazon ; merchant note
```

**AST:** `PayeeDirective > PayeeKeyword, DirectiveArgument, InlineComment?, Newline, IndentedComment*`

### Tag

Declares a tag. May have indented sub-comments.

```
tag project
    ; description: Project tracking tag
```

**AST:** `TagDirective > TagKeyword, DirectiveArgument, InlineComment?, Newline, IndentedComment*`

### Price (P)

Declares a market price for a commodity.

```
P 2024-01-01 EUR $1.10
```

**AST:** `PriceDirective > PriceKeyword, DirectiveArgument, Newline`

### Default Commodity (D)

Sets the default commodity for amounts without one.

```
D $1,000.00
```

**AST:** `DefaultCommodityDirective > DefaultCommodityKeyword, DirectiveArgument, Newline`

### Year (Y / year)

Sets the default year for partial dates.

```
Y 2024
year 2025
```

**AST:** `YearDirective > YearKeyword, DirectiveArgument, Newline`

### Decimal Mark

Sets the decimal separator character (period or comma).

```
decimal-mark .
decimal-mark ,
```

**AST:** `DecimalMarkDirective > DecimalMarkKeyword, DirectiveArgument, Newline`

### Commodity Conversion (C)

Declares a commodity conversion rate.

```
C 1h = $50.00
```

**AST:** `CommodityConversionDirective > CommodityConversionKeyword, DirectiveArgument, Newline`

### Bucket (A / bucket)

Sets the default balancing account.

```
A assets:bank:checking
bucket expenses:misc
```

**AST:** `BucketDirective > BucketKeyword, DirectiveArgument, Newline`

### Ignored Price Commodity (N)

Excludes a commodity from price calculations.

```
N EUR
```

**AST:** `IgnoredPriceDirective > IgnoredPriceKeyword, DirectiveArgument, Newline`

### Apply Directives

Temporarily apply settings to subsequent entries.

```
apply account assets:bank
apply year 2024
apply tag project
apply fixed EUR $1.10
```

**AST:** `ApplyAccountDirective`, `ApplyYearDirective`, `ApplyTagDirective`, `ApplyFixedDirective`
Each: `*Keyword, DirectiveArgument, InlineComment?, Newline`

### Generic One-Line Directives

These directives are parsed as a keyword plus the remaining text on the line:

```
assert balance
capture invoice
check assertions
define foo ; semicolons remain part of the line here
expr something
value cost
eval print(1)
```

**AST:** `AssertDirective`, `CaptureDirective`, `CheckDirective`, `DefineDirective`, `ExprDirective`, `ValueDirective`, `EvalDirective`
Each: `*Keyword, DirectiveRest?, Newline`

### Command Flag Directives

Top-level `--flag` directives are parsed as journal directives rather than comments:

```
--strict
--infer-costs
```

**AST:** `CommandFlagDirective > CommandFlagKeyword, DirectiveRest?, Newline`

### End Directives

Close an apply block or alias scope.

```
end apply account
end apply year
end apply tag
end apply fixed
end aliases
end tag
```

**AST:** `EndDirective > EndKeyword, DirectiveArgument?, InlineComment?, Newline`

---

## Comments

### Line Comments

Start with `;`, `#`, or `*` at the beginning of a line. Top-level file comments may also be indented:

```
; This is a comment
# This is also a comment
* This is a comment too
  ; This is also a top-level file comment
```

**AST:** `LineComment`

### Block Comments

Multi-line comments enclosed between `comment` and `end comment`:

```
comment
This entire block
is a comment.
end comment
```

**AST:** `BlockComment`

### Indented Comments

Inside transactions or directives, indented lines starting with `;`:

```
2024-01-15 Grocery
    ; This is a transaction comment
    expenses:food  $50  ; This is an inline posting comment
    assets:bank
```

**AST:** `IndentedComment > CommentIndent, CommentMark, CommentBody?, Newline`
**AST:** `InlineComment > CommentMark, CommentBody?`

---

## AST Node Summary

| Node | Style Tag | Description |
|------|-----------|-------------|
| `TxnHeader` | _(container)_ | Transaction header line (date + description) |
| `TxnDate` | `meta` | Transaction date (including secondary date) |
| `TxnDescription` | `string` | Transaction description (status, code, text) |
| `PeriodicHeader` | _(container)_ | Periodic transaction header |
| `PeriodicMark` | `meta` | The `~` character |
| `PeriodicExpression` | `string` | Period expression text |
| `AutoHeader` | _(container)_ | Auto posting header |
| `AutoMark` | `meta` | The `=` character |
| `AutoQuery` | `string` | Auto posting query text |
| `CommodityFormatDirective` | _(container)_ | Indented commodity `format` subdirective |
| `AccountKeyword` | `keyword` | The word `account` |
| `CommodityKeyword` | `keyword` | The word `commodity` |
| `FormatKeyword` | `keyword` | The word `format` |
| `IncludeKeyword` | `keyword` | The word `include` |
| `AliasKeyword` | `keyword` | The word `alias` |
| `PayeeKeyword` | `keyword` | The word `payee` |
| `TagKeyword` | `keyword` | The word `tag` |
| `PriceKeyword` | `keyword` | The letter `P` |
| `DefaultCommodityKeyword` | `keyword` | The letter `D` |
| `YearKeyword` | `keyword` | `Y` or `year` |
| `DecimalMarkKeyword` | `keyword` | `decimal-mark` |
| `ApplyAccountKeyword` | `keyword` | `apply account` |
| `ApplyYearKeyword` | `keyword` | `apply year` |
| `ApplyTagKeyword` | `keyword` | `apply tag` |
| `ApplyFixedKeyword` | `keyword` | `apply fixed` |
| `AssertKeyword` | `keyword` | `assert` |
| `CaptureKeyword` | `keyword` | `capture` |
| `CheckKeyword` | `keyword` | `check` |
| `DefineKeyword` | `keyword` | `define` |
| `ExprKeyword` | `keyword` | `expr` |
| `ValueKeyword` | `keyword` | `value` |
| `EvalKeyword` | `keyword` | `eval` |
| `CommodityConversionKeyword` | `keyword` | The letter `C` |
| `BucketKeyword` | `keyword` | `A` or `bucket` |
| `IgnoredPriceKeyword` | `keyword` | The letter `N` |
| `CommandFlagKeyword` | `keyword` | The `--` directive prefix |
| `EndKeyword` | `keyword` | `end ...` variants |
| `DirectiveAccountName` | `variableName` | Account name in account directive |
| `DirectiveArgument` | `string` | Generic directive argument text |
| `DirectiveRest` | `string` | Rest-of-line directive text without comment splitting |
| `IncludePath` | `string` | File path/glob in include directive |
| `AccountName` | `variableName` | Account name in postings |
| `Amount` | _(container)_ | Amount with commodity and number |
| `Number` | `number` | Numeric value |
| `Sign` | `operator` | `+` or `-` |
| `Commodity` | `unit` | Currency/commodity symbol |
| `CostOp` | `operator` | `@` or `@@` |
| `BalanceOp` | `operator` | `=`, `==`, `=*`, `==*` |
| `LotPrice` | _(container)_ | Lot-price annotation (`{...}` / `{{...}}`) |
| `LotDate` | _(container)_ | Lot-date annotation (`[DATE]`) |
| `LotPriceFixed` | `operator` | `=` inside a fixed lot price |
| `LotDateBody` | `meta` | Text inside `[DATE]` lot-date annotations |
| `Status` | `keyword` | `*` or `!` |
| `CommentMark` | `lineComment` | `;` in inline comments |
| `CommentBody` | `lineComment` | Comment text content |
| `InlineComment` | _(container)_ | Same-line transaction or posting comment |
| `LineComment` | `lineComment` | Full line comment |
| `BlockComment` | `blockComment` | Block comment |
| `BlankLine` | _(none)_ | Empty line |

---

## Known Limitations

- **Auto posting multiplier prefix** (`*0.5`, `*-1`): The `*` multiplier in auto posting rules conflicts with the `Status` marker. The grammar parses `*` as a status marker rather than a multiplier.
- **Secondary dates** (`DATE=DATE2`): Captured within `TxnDate` as a single token (not split into primary/secondary).
- **Transaction codes** (`(CODE)`): Captured within `TxnDescription` but not as a separate sub-node.
- **Comment tags and posting date tags** (`name: value`, `date:`, `date2:`, `[DATE=DATE2]` inside comments): These are semantic data in hledger, but they are not represented as dedicated syntax nodes by this grammar.
- **Posting modifier ordering**: The grammar accepts posting annotations more permissively than hledger; some invalid orderings still require higher-level validation.
- **Directive semantics**: Rules like commodity/`format` symbol matching and account-directive bracket rejection are not enforced syntactically.
- **Python directives**: Multi-line `python` directives are not modeled yet.
