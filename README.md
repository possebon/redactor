# Redactor

> A single-file, fully-offline tool that strips secrets and PII from text before you paste it into an LLM — and puts the originals back when the model replies.

![status: 100% local, no network](https://img.shields.io/badge/100%25%20local-no%20network-10b981) ![size: ~50 KB](https://img.shields.io/badge/size-~50%20KB-666) ![deps: none](https://img.shields.io/badge/dependencies-none-666)

## Why

LLMs are great rubber ducks, but pasting prod logs, connection strings, customer records, or AWS keys into a chat window is a compliance landmine. Most teams either (a) self-censor by hand, which is slow and error-prone, or (b) shrug and hope nothing goes wrong.

Redactor sits between you and the model. Paste anything in — it returns the same text with sensitive values swapped for stable tokens like `[EMAIL_001]`, `[CPF_001]`, `[AWSKEY_001]`. When the LLM replies referencing those tokens, paste the reply back into **Restore** mode and get the original values back. The mapping never leaves your browser tab.

## Quick start

1. Save `redactor.html` anywhere on your computer.
2. Double-click to open it in any modern browser. No installer, no server, no Node, no Python.
3. Paste text into the **Original** pane. The **Sanitized** pane updates as you type.
4. Click **Copy all**, then paste into your LLM of choice.
5. When the LLM replies, switch to the **Restore** tab, paste the reply, get the original values back.

That's it. The first byte of network traffic this page generates is whatever you do *outside* the page.

## Features

- **42 detection patterns** across credentials, network identifiers, and PII for Brazil, US, and Europe/UK — plus universal patterns (email, credit card, IBAN, E.164 phone).
- **Reversible.** Every redaction is recorded in an in-memory map. Use **Restore** to reverse the process — paste the LLM reply with tokens, get the original values back.
- **Stable tokens.** The same value always gets the same token within a session, so the LLM can reason about "the user" or "the database" coherently across the document.
- **Per-pattern toggles.** Turn off categories you don't need (e.g. disable EU patterns when working with Brazilian-only data).
- **Map export.** Download the token-to-original map as JSON if you need to audit, share with a teammate, or restore in another session.
- **Zero network calls.** No fonts, no analytics, no CDNs. Open the file in your editor and search for `http` — the only matches are SVG namespace declarations.

## What gets redacted

| Group | Patterns |
| --- | --- |
| **Credentials & secrets** | PEM private keys, JWTs, AWS access keys, GitHub / Slack / Stripe / Google API tokens, Bearer / Basic auth headers, SSH public keys, SQL Server & libpq / JDBC URL connection strings, `password=…` / `secret=…` / `apikey=…` assignments, **DB connection fields** (`user=`, `dbname=`, `host=`, `database=`, `schema=`, `account=`, …) |
| **Network & infrastructure** | HTTPS URLs, IPv4 & IPv6, MAC addresses, FQDN / hostnames, Windows AD `DOMAIN\user`, `/home/x` & `/Users/x` & `C:\Users\x` paths, `user@host` SSH-style |
| **Personal — universal** | Email addresses, credit cards (Luhn-validated), IBAN, E.164 international phone |
| **Personal — Brazil 🇧🇷** | CPF, CNPJ, CEP, Brazilian phone, PIS / PASEP, Título de Eleitor *(off by default)* |
| **Personal — United States 🇺🇸** | SSN, EIN, US phone, ZIP code *(off by default)* |
| **Personal — Europe / UK 🇪🇺** | UK National Insurance number, UK postcode, EU VAT, Spanish DNI / NIE, Italian Codice Fiscale, French INSEE / Sécu social, Portuguese NIF *(off by default)* |

Patterns marked *off by default* are high-false-positive on bare digit strings (a 9-digit number isn't necessarily a Portuguese NIF). Enable them per session via the **Patterns** drawer when you're working with that data.

## Token format

Tokens look like `[CATEGORY_NNN]`:

- `CATEGORY` — one of the pattern IDs (`EMAIL`, `IPV4`, `CPF`, `AWSKEY`, `DB_FIELD`, …).
- `NNN` — a zero-padded counter scoped to that category for the current session.

Tokens are **stable per value**: the same email gets the same `[EMAIL_001]` everywhere it appears, so the LLM can correctly say "EMAIL_001 logged in three times" or "send the response to EMAIL_001 and EMAIL_002".

## Two modes

### Redact
The default mode. Paste original text on the left, see sanitized output on the right. The stats strip at the top shows per-pattern hit counts. The **Token map** drawer at the bottom lists every redaction with its category.

### Restore
For when the LLM replies and refers to your tokens. Paste the LLM reply into the left pane and the right pane shows the reply with tokens swapped back to original values. Useful when you ask the model to "rewrite this email" or "summarize these logs" — you get back a polished version that already contains your real data.

The mapping lives in browser memory only. Closing the tab discards it. If you need to keep it across sessions, export the JSON via the **Token map** drawer.

## Privacy & security

- **No network calls.** The HTML file references no external scripts, stylesheets, fonts, or images. All detection is regex JavaScript running in your browser.
- **No persistent storage.** No cookies, no `localStorage`, no `IndexedDB`. State exists only as long as the tab is open.
- **No telemetry.** Nothing phones home. Verify with browser DevTools → Network tab — it stays empty no matter how much you redact.
- **Auditable.** ~50 KB of vanilla HTML / CSS / JS in a single file. Open it in any editor — every line is human-readable, no minification, no transpilation.
- **Skips structural noise.** Values that are clearly variable references (`$RDSHOST`), templates (`{db_url}`), or placeholders (`<your-password>`) are deliberately *not* tokenized — they're not real secrets, and tokenizing them only makes the output noisier.

## Self-hosting (optional)

If your team prefers a shared URL over passing the file around in chat, host it as a static asset behind any reverse proxy. Example for Docker Swarm + Traefik:

```yaml
services:
  redactor:
    image: nginx:alpine
    volumes:
      - ./redactor.html:/usr/share/nginx/html/index.html:ro
    deploy:
      labels:
        - traefik.enable=true
        - traefik.http.routers.redactor.rule=Host(`redactor.internal.example.com`)
        - traefik.http.routers.redactor.tls=true
        - traefik.http.routers.redactor.tls.certresolver=letsencrypt
        - traefik.http.services.redactor.loadbalancer.server.port=80
    networks:
      - traefik-public

networks:
  traefik-public:
    external: true
```

That's the entire deployment — no backend, no database, no auth required (since nothing sensitive is ever stored server-side; the tool is client-only). The same `redactor.html` file works behind nginx, Caddy, GitHub Pages, S3 static hosting, or just a `python3 -m http.server`.

## Customizing patterns

All patterns live in the `PATTERNS` array inside the embedded `<script>` block, ordered most-specific to least-specific. A pattern looks like this:

```javascript
{
  id: 'MYPATTERN',                          // appears in tokens as [MYPATTERN_001]
  label: 'My custom thing',                 // shown in the settings drawer
  group: 'cred',                            // 'cred' | 'net' | 'pii_univ' | 'pii_br' | 'pii_us' | 'pii_eu'
  def: true,                                // enabled by default?
  regex: /\bMY[A-Z0-9]{10}\b/g,             // MUST include the 'g' flag
  validator: (match) => /* optional */,     // returns true to redact, false to skip (e.g. Luhn check)
  valueGroup: 3,                            // optional — for key=value patterns, only replace capture group N
}
```

**Order matters.** More specific patterns must come first, otherwise a less specific one will swallow part of the input. For example, `EMAIL` must run before `HOSTNAME`, otherwise the domain gets tokenized first and the local-part of the email is left exposed.

### Adding a key to `DB_FIELD`

To capture more connection-field keys (say `tenant_id`), edit its regex directly:

```javascript
regex: /\b(database|dbname|...|tenant[_-]?id)(\s*[:=]\s*['"]?)([^\s'",;}\n)]+)/gi
```

The `valueGroup: 3` line tells the engine to replace only capture group 3 (the value), leaving the key intact for context.

### Disabling a pattern permanently

Either set `def: false` on its definition, or just delete the object from the array. The settings drawer will reflect the change automatically.

## Known limitations

- **Bare names aren't detected.** "Maria Silva" or "John Smith" remain visible — the tool relies on patterns, not NER. If you need to scrub names, add a pattern for them or do a manual find-replace before pasting.
- **Free-text addresses** (street addresses without an adjacent postcode pattern) aren't caught.
- **`port=`** is intentionally ignored. Database ports are usually defaults (3306, 5432) and rarely sensitive on their own. Add `port` to `DB_FIELD` if your threat model says otherwise.
- **Token collision** is theoretical: if your input *already* contains literal text like `[EMAIL_001]`, restoration would replace it. The bracket format makes this nearly impossible in practice.
- **Pattern coverage** is opinionated, not exhaustive. If a sensitive value type isn't covered, add a pattern — see *Customizing patterns* above.

## How it works (technical)

1. The input string is processed sequentially through every enabled pattern, in array order.
2. Each match is replaced with a token. If the same value appears multiple times, it always gets the same token (deduplication via a `Map<original, token>`).
3. For `valueGroup` patterns (key=value style), only the captured value group is replaced — the key and separator stay intact for readability.
4. Tokens use the format `[CATEGORY_NNN]` with brackets that can't appear inside any pattern's regex by design, preventing recursive redaction.
5. **Restore** runs the inverse: it sorts the mapping by descending token length (so `[EMAIL_0010]` is processed before `[EMAIL_001]` to avoid prefix collisions) and does a literal `split`/`join` on each token.

The whole pipeline is well under 1000 lines of JavaScript. Read the source.

## License

Use it, modify it, share it. Treat it as MIT-licensed unless your organization needs something more formal.

## Contributing

This is a single-file project by design — no build step, no `node_modules`, no PR pipeline. If a pattern is missing for your locale or stack, add it to the `PATTERNS` array and pass the edited file around. The regex section is under 100 lines.

If you find a bug or false-positive, the fastest fix is usually to refine the offending pattern's regex in place. Feel free to fork.
