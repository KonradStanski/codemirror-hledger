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

- **Transactions**: dates, status markers (`*`/`!`), codes, payee|description, inline comments
- **Postings**: account names (with single-space support), amounts, commodity symbols, cost (`@`/`@@`), balance assertions (`=`/`==`)
- **Virtual postings**: `(account)` and `[account]` syntax
- **Periodic transactions**: `~ monthly`, `~ weekly`, etc.
- **Auto posting rules**: `= query`
- **Directives**: `account`, `commodity`, `payee`, `tag`, `include`, `alias`, `end aliases`, `decimal-mark`, `apply account`, `end apply account`, `P` (market price), `D` (default commodity), `Y`/`year`
- **Comments**: line comments (`;`, `#`, `*`) and block comments (`comment`...`end comment`)
- **Date formats**: `YYYY-MM-DD`, `YYYY/MM/DD`, `YYYY.MM.DD`
