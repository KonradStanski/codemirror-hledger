# codemirror-lang-hledger

[hledger](https://hledger.org/) journal format language support for [CodeMirror 6](https://codemirror.net/).

## Features

- Syntax highlighting for transactions, postings, amounts, account names, dates, comments, directives, and more
- Code folding for transactions and multi-line directives
- Auto-indentation for postings under transaction headers
- Autocompletion for directive keywords and common account prefixes
- Toggle line comments with `;`

## Installation

```
npm install codemirror-lang-hledger
```

## Usage

```typescript
import {EditorView, basicSetup} from "codemirror"
import {hledger} from "codemirror-lang-hledger"

new EditorView({
  extensions: [basicSetup, hledger()],
  parent: document.body
})
```

If you only need the language without autocompletion and folding:

```typescript
import {hledgerLanguage} from "codemirror-lang-hledger"
```

## Supported syntax

- **Transactions**: dates (including `DATE=DATE2`), status markers (`*`/`!`), codes, payee|description text, same-line `;` comments, indented transaction comments
- **Postings**: account names (with single-space support and journal-style semicolons), amounts, commodity symbols, cost (`@`/`@@`), balance assertions (`=`/`==`/`=*`/`==*`), lot prices (`{...}` / `{{...}}`), lot dates (`[DATE]`), inline comments
- **Virtual postings**: `(account)` and `[account]` syntax
- **Periodic transactions**: `~ monthly`, `~ weekly`, etc.
- **Auto posting rules**: `= query`
- **Directives**: `account`, `commodity`, `payee`, `tag`, `include`, `alias`, `end aliases`, `decimal-mark`, `apply account`, `apply year`, `apply tag`, `apply fixed`, `end ...`, `assert`, `capture`, `check`, `define`, `expr`, `value`, `eval`, `P` (market price), `D` (default commodity), `Y`/`year`, `C`, `A`/`bucket`, `N`, and command-line `--flag` directives
- **Directive details**: same-line `;` comments on account/commodity/include/payee/tag directives, commodity `format` subdirectives, and legacy `!` / `@` directive prefixes
- **Comments**: line comments (`;`, `#`, `*`), indented top-level file comments, and block comments (`comment`...`end comment`)
- **Date formats**: `YYYY-MM-DD`, `YYYY/MM/DD`, `YYYY.MM.DD`

## Notes

- The grammar models journal syntax, not all of hledger's semantic validation rules.
- Transaction codes are still part of `TxnDescription`, not a separate AST node.
- Tags and posting dates embedded inside comment text are still extracted by a higher-level parser layer, not by the Lezer grammar itself.
- Multi-line `python` directives are not modeled yet.
