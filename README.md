# Donations — NonprofitCRM plugin

The Donations domain vertical for
[NonprofitCRM](https://github.com/abeuscher/npc-beta): public donation
checkout (via the Payments capability), funds, soft credits, receipts and
acknowledgments, the donation and fund admin resources, the Giving Summary
page, the donations importer, and the two donation widgets (DonationForm,
RecentDonations). Extracted to its own repository in session 391 (Plugin
Architecture arc, Stage D — the second extracted migration-owning,
testsuite-carrying plugin).

**Package:** `nonprofitcrm/donations`
**Implements plugin contract:** `0.15.0` (`docs/plugin-contract.md` in the core repo)

## How this package is consumed

The core repo's `composer.json` carries a `vcs` repository entry pointing at
this repo and a bound version requirement. Composer resolves **git tags** as
versions — this package deliberately declares no `version` field in its
`composer.json` (a retained field that disagrees with the tag is a composer
install error; under a VCS repository the tag is canonical).

Activation stays with the core repo: `config/plugins.php` (generated from
`distribution.json`) lists `Plugins\Donations\DonationsServiceProvider`, and
the per-install `PLUGINS_DISABLED=donations` flag vanishes the plugin at
runtime. Nothing about activation is keyed to where this code lives.

This package owns its schema (contract surface 5): `database/migrations/`
creates the four donation tables (`funds`, `donations`, `donation_credits`,
`donation_receipts` — FK creation order). Install order is core schema dump →
enabled plugins' migrations → seeders; disabling the plugin never drops its
tables or data (disabled ≠ uninstalled). The `donation_receipts.contact_id`
FK is `ON DELETE RESTRICT` by design — that constraint *is* the
blocks-contact-force-delete-while-receipts-exist behavior, and it travels
with the table.

Payments is an **optional** capability, never a composer dependency
(contract surface 13): the checkout controller asks
`CapabilityRegistry::enabled('payments')` and returns its not-configured
response when it's absent. Inbound settlement arrives as core events
(`CheckoutSettled`, `InvoiceSettled`, `InvoicePaymentFailed`) dispatched by
the Payments plugin; this plugin's listeners own the donation fulfillment
(contract surface 10).

Front-end assets are declared vendor-relative
(`vendor/nonprofitcrm/donations/Widgets/…`) and compiled by the core repo's
`build:public` pipeline; widget thumbnails ship in this repo (the core
thumbnail generator cannot reach an extracted plugin — regenerate here when
visuals change).

## Tests

The `tests/` directory ships with this package and is the phpunit `Donations`
testsuite in the core repo, which runs it from
`vendor/nonprofitcrm/donations/tests` — the central build runs the
distribution's plugin suites (plugin contract, normative rule 5). No CI here
yet (per-plugin CI is a scheduled later arc position).

## Release discipline (mirrors the CRM ↔ Fleet Manager relationship)

- **Tags are releases.** Every consumable state of this repo is a tag
  (`v0.1.0`, `v0.1.1`, …). Core pins a bound constraint (`^0.1`) and bumps its
  lockfile deliberately via `composer update nonprofitcrm/donations`.
- **Contract-first.** A change that touches the plugin-contract surface lands
  in the core repo's `docs/plugin-contract.md` first; this repo then implements
  against the bumped version and records it in the header above.
