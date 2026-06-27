# Payments

Tokens are the unit of in-app spend (gifts, VIP access, gigs, and other paid features). They are bought through **Niilox-hosted checkout** and credited after a verified webhook.

> **Public docs note:** Step-by-step transaction flows, paywall semantics, and revenue-split implementation are withheld from this repository. Registered developers receive the full integration guide — contact **dev@niilox.com**.

## Checkout channels

Integrators and clients choose a **channel**, not a vendor:

| Channel | Typical use |
|---------|-------------|
| `card` | Global card checkout (web) |
| `bank` | Local bank rails (web, where enabled) |
| `mobile_iap` | App Store / Google Play (native apps only) |

Optional `provider` field on checkout requests accepts `card` or `bank` (legacy values are mapped server-side).

## Mobile in-app purchases

Native apps must use **`mobile_iap`** — not web checkout URLs. See [MOBILE_PAYMENTS.md](./MOBILE_PAYMENTS.md).

## Token packs

Per-tenant token pack catalog is configurable in the developer portal. Packs are listed via the tokens API and purchased through hosted checkout.

## In-app commerce

Niilox supports token spend for gifts, capped VIP events, gigs, and host payouts. Revenue split and seat capacity are enforced server-side per tenant.

Full endpoint reference, paywall responses, and webhook handling are provided to registered developers.

## Checkout flow (overview)

1. Client requests checkout for a pack id (optional `provider`: `card` | `bank`).
2. API creates a pending purchase row and returns a hosted `checkout_url`.
3. User completes payment on the hosted page.
4. Niilox webhook confirms the purchase; tokens are credited idempotently.

## Connect account (card revenue)

Per-tenant **payment connect accounts** are configured in the developer portal (`acct_…` style id). Card charges run on your connected account; Niilox takes the platform share per your billing settings.

## Provisioning

Payment rails are **enabled by Niilox operations** for each tenant. Integrators do not configure upstream gateway credentials.

Check `GET /api/v1/tokens/packs` for `fiat_checkout`, `token_checkout`, and `payment_connect_account`.

## Withdrawals

Withdrawals are a request queue — users withdraw earned tokens; an admin or back-office process pays them out off-platform.

Contact **dev@niilox.com** for withdrawal rules and admin tooling.

## Common questions

**Q: What happens if checkout completes but the webhook never arrives?**  Webhooks are retried for several days. The purchase row stays pending until confirmation lands.

**Q: Can I offer both card and bank on the same tenant?**  Pass `provider: "card"` or `provider: "bank"` on checkout; tenant default applies when omitted.

**Q: How do refunds work?**  Manual refund via your admin tool; mark the purchase refunded and adjust balance accordingly.
