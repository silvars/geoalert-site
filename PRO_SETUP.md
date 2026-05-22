# PRO subscription — setup guide

End-to-end checklist to turn the **GeoAlert PRO** flow on in production.
Code is already wired (RevenueCat SDK + Cloud Function webhook + paywall UI).
What remains is **store/dashboard config** — no further code changes needed.

---

## 1. Native rebuild

`react-native-purchases` has a native module. After install:

```bash
cd ios && pod install && cd ..
npx expo prebuild --clean      # only if your native folders are dirty
# Build to device:
npx expo run:ios --device      # or :android
```

Verify the SDK at first boot — Metro log should show
`[Purchases] SDK configurado` (only appears when `EXPO_PUBLIC_REVENUECAT_*_KEY`
is set in `.env`).

---

## 2. RevenueCat dashboard

1. Create project at <https://app.revenuecat.com>.
2. **Apps** → add 2 apps:
   - iOS bundle `com.silvars.geoalert`
   - Android package `com.geoalert.app`
3. **API keys** (Project Settings → API keys → *Public*):
   - Copy iOS `appl_...` → `.env` as `EXPO_PUBLIC_REVENUECAT_IOS_KEY`
   - Copy Android `goog_...` → `.env` as `EXPO_PUBLIC_REVENUECAT_ANDROID_KEY`
4. **Entitlement** → create one called **`geoalert-pro`**.
5. **Products** → create 2 products and attach them to the `geoalert-pro` entitlement:
   - `geoalert_pro_monthly` — Auto-renewable subscription, R$ 9,90 / mês
   - `geoalert_pro_annual`  — Auto-renewable subscription, R$ 79,90 / ano
6. **Offerings** → create offering `default` with packages:
   - Monthly  → `geoalert_pro_monthly`
   - Annual   → `geoalert_pro_annual`
7. **Integrations → Webhooks**:
   - URL: `https://southamerica-east1-geo-alert-6c2d4.cloudfunctions.net/revenuecatWebhook`
   - Header `Authorization: Bearer <SECRET>` — same value as the
     `REVENUECAT_WEBHOOK_SECRET` defined in step 4 below.

---

## 3. App Store Connect

1. App `GeoAlert` → **App Store → Subscriptions**.
2. Create a **Subscription Group** named `geo_alert_pro` (any localized name).
3. Inside the group create 2 auto-renewable subscriptions:
   - Product ID `geoalert_pro_monthly` — 1 month — Price tier closest to BRL 9,90 / USD 1,99.
   - Product ID `geoalert_pro_annual`  — 1 year  — Price tier closest to BRL 79,90 / USD 14,99.
4. Localize display name + description (PT/EN).
5. **Do not** enable an Introductory Offer / free trial — by product decision.
6. Wait for `Ready to Submit` state for both before testing with sandbox accounts.
7. Connect Apple to RevenueCat (RevenueCat → Apps → iOS → enter shared secret
   from App Store Connect → App Information → App-Specific Shared Secret).

---

## 4. Google Play Console

1. App `GeoAlert` → **Monetize → Subscriptions**.
2. Create subscription product `geoalert_pro` (single product).
3. Add 2 **base plans**:
   - `monthly` — auto-renew, R$ 9,90.
   - `annual`  — auto-renew, R$ 79,90.
4. Tag each base plan with the same offer/tag RevenueCat expects
   (RevenueCat **Products** screen shows the exact base-plan-id mapping).
5. Connect Play to RevenueCat (RevenueCat → Apps → Android → service-account
   JSON with Pub/Sub + AndroidPublisher API enabled).

> Note: RevenueCat product IDs on Android map to `subscriptionId:basePlanId`
> when a single product has multiple base plans. Use whatever IDs the
> RevenueCat "Products" page expects — it tells you exactly.

---

## 5. Cloud Function webhook deploy

```bash
# Generate a random shared secret (one-time)
openssl rand -hex 32

# Save it into Functions secret manager
firebase functions:secrets:set REVENUECAT_WEBHOOK_SECRET
# (paste the value from the previous command)

# Deploy
firebase deploy --only functions:revenuecatWebhook
```

Logs:
```bash
firebase functions:log --only revenuecatWebhook
```

Sanity check: in RevenueCat → Integrations → Webhooks click **Send test**.
You should see `[rcWebhook] TEST recebido para …` in the logs and a `200 OK`
on the RevenueCat side.

---

## 6. End-to-end smoke test (sandbox)

1. **iOS**: sign out of the real Apple ID on the device, install build via
   `npx expo run:ios --device`. iOS will ask for a sandbox tester at purchase
   time. Configure testers under App Store Connect → Users and Access → Sandbox.
2. **Android**: upload a build to an internal test track and license-test users
   in Play Console → Settings → License testing.
3. Log into the app with a real Google/Apple/email account (required —
   PRO is gated to non-anonymous users).
4. Tap "Assinar PRO" in Settings, pick a plan, complete the sandbox purchase.
5. Expected:
   - `Purchases.purchasePackage` resolves successfully.
   - `onProChange` listener fires → SettingsScreen flips to **PRO ativo**.
   - Cloud Function receives `INITIAL_PURCHASE` → `users/{uid}` document gets
     `isPro=true` + `proExpiresAt` + `proSince`.
   - `LIMITS` switch from `free` to `pro` → can add 25 places, 50 friends, etc.
6. Force-quit + reopen → status persists (read from Firestore cache + RC SDK).
7. Test restore: log out, log back in, tap "Restaurar compra" → status returns.

---

## 7. Production rollout checklist

- [ ] `EXPO_PUBLIC_REVENUECAT_IOS_KEY` and `EXPO_PUBLIC_REVENUECAT_ANDROID_KEY`
      set in production `.env` (and CI secrets if used).
- [ ] `REVENUECAT_WEBHOOK_SECRET` defined in Firebase Functions secrets.
- [ ] Webhook deployed and showing 200 OK on RC test event.
- [ ] Both products `Ready to Submit` on App Store and Active on Play.
- [ ] Tested at least one full purchase + restore + cancellation cycle in
      sandbox on each platform.
- [ ] App Store Connect: subscription privacy policy URL + EULA + paid
      agreements signed.
- [ ] `firestore.rules` deployed (already hardened — clients cannot set
      `isPro=true` from the app).
