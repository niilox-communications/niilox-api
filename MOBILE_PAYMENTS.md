# Mobile token purchases

Consumer **web** buys tokens via hosted checkout (`POST /api/v1/tokens/checkout`).

Consumer **mobile** (React Native) must use **App Store / Google Play** through Niilox **`mobile_iap`** — not web checkout URLs (store policy).

## Architecture

```text
Mobile app                    Store billing              Niilox API
──────────                    ─────────────              ─────────
Configure IAP SDK with mobile_iap keys from /tokens/mobile-config
Purchase consumable pack ──► App Store / Play ──► Webhook (ops-configured)
                                                      └─► credit token_balance
```

1. User signs in → JWT `sub` = tenant `users.id`.
2. On app launch: bind the store SDK to the same user id.
3. Fetch catalog: `GET /api/v1/tokens/mobile-config` (public IAP SDK keys + packs with `store_product_id`).
4. Map store **offerings** to those product ids.
5. After purchase, Niilox webhook credits tokens (idempotent per transaction).

**Do not** call `POST /tokens/checkout` from mobile.

## Store product ids

Configured per tenant in the developer portal.

Create matching **consumable** IAP products in App Store Connect and Google Play Console, then attach them to your mobile IAP offering.

## API

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/tokens/packs` | — | Pack list (includes `store_product_id`; web ignores extra field) |
| GET | `/api/v1/tokens/mobile-config` | — | `{ provider: "mobile_iap", mobile_iap_api_key_apple, mobile_iap_api_key_google, packs }` |
| POST | `/webhooks/mobile-iap` | ops-configured | Credits tokens on purchase; clawback on refund (contact Niilox for URL) |

Mobile IAP webhooks and keys are provisioned by **Niilox operations** — not in integrator env files.

## React Native (Expo) sketch

```bash
npx expo install react-native-purchases
```

```typescript
import Purchases from 'react-native-purchases'
import { Platform } from 'react-native'

const cfg = await fetch(`${API}/tokens/mobile-config`).then(r => r.json())
await Purchases.configure({
  apiKey: Platform.OS === 'ios' ? cfg.mobile_iap_api_key_apple : cfg.mobile_iap_api_key_google,
  appUserID: user.id,
})

const offerings = await Purchases.getOfferings()
// Match offering packages to cfg.packs by store_product_id
await Purchases.purchasePackage(selectedPackage)
// Tokens arrive when webhook fires — refresh GET /users/me or poll balance
```

VIP private rooms still spend **in-app tokens** (same as web) after balance updates.

## Refunds

Store refund events run through the same clawback path as card disputes (`payment_disputes` audit + balance debit).
