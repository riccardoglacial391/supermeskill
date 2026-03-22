# Superme Skill for Claude Code

Online supermarket automation skill for [Claude Code](https://claude.ai/claude-code). Currently supports **Shufersal** (Israel's largest supermarket chain).

## Features

- **Login** — Authenticate to Shufersal Online
- **Search** — Find products with prices, transliterated for terminal display
- **Add to List** — Add search results to a session wishlist
- **Magicorder** — Consolidate your last 5 orders into one wishlist (deduped)
- **Lists** — View your Superme wishlists

## Installation

Copy `SKILL.md` to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/superme
cp SKILL.md ~/.claude/skills/superme/
```

### Prerequisites

- [browser-use](https://github.com/browser-use/browser-use) CLI: `uv tool install browser-use --python 3.12`
- A Shufersal Online account

## Usage

```
/superme login <email> <password>
/superme search <product>
/superme add <#>
/superme magicorder
/superme lists
/superme close
/superme help
```

## How it works

- Uses `browser-use` CLI to automate a Chromium browser session
- Interacts with Shufersal's Vue.js frontend and REST APIs
- Manages wishlists via the internal `/wishlist/*` API endpoints
- Fetches order history via `/my-account/orders` API
- Products are added to wishlists programmatically through the Vue component tree

## Supported Vendors

| Vendor | Status |
|--------|--------|
| Shufersal | Supported |
| Rami Levy | Planned |
| Yochananof | Planned |
| Victory | Planned |

## License

MIT
