# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2026-05-01

### Added

- **Vendor API key detection** — Anthropic (`sk-ant-…`), OpenAI (`sk-…` legacy and `sk-proj-…` / `sk-svcacct-…`), Twilio (`AC…` / `SK…`), SendGrid, Mailgun, npm tokens, GitLab PATs (`glpat-…`), Atlassian (`ATATT…`), DigitalOcean (`dop_v1_…`), Heroku (`HRKU-…`), GCP OAuth access tokens (`ya29.…`).
- **Webhook URL detection** — Slack (`https://hooks.slack.com/…`) and Discord webhook URLs are tokenized as their own categories instead of being swallowed by the generic URL pattern.
- **WebSocket URI detection** — `ws://` and `wss://` connection URIs (often carry tokens in query strings).
- **JDBC connection string support** — `URLCONN` now matches `jdbc:postgresql://…`, `jdbc:mysql://…`, `oracle:thin:@…`, ClickHouse, Cassandra, and SQL Server in addition to the previously supported native schemes.
- **Latin America PII group** — new pattern group with:
  - Argentine CUIT/CUIL with mod-11 checksum validation
  - Mexican CURP (18-char structure)
  - Mexican RFC (12/13-char structure)
  - Chilean RUT with mod-11 checksum validation
- **Reserved-value skip list** — well-known documentation IPs (RFC 5737: `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`), loopback (`127.0.0.1`, `::1`), broadcast (`255.255.255.255`), all-zero / all-ones MAC, and RFC 2606 example domains (`example.com`, `*.test`, `*.local`) are no longer tokenized.

### Changed

- **CPF detection** now requires a valid check-digit (mod-11) — eliminates false positives from arbitrary `XXX.XXX.XXX-XX`-shaped strings (test fixtures, serial numbers).
- **CNPJ detection** now requires a valid check-digit (mod-11) — same FP reduction.

### Notes

This release adds 17 new patterns and 2 new validators while remaining a
single-file, dependency-free, fully-offline tool.

## [0.1.0] - 2026-05-01

### Added

- Initial release with 42 detection patterns covering credentials, network
  identifiers, and PII for Brazil, US, and Europe/UK.
- Reversible tokenization with stable per-value tokens.
- Per-pattern toggles via settings drawer.
- Token map JSON export.
