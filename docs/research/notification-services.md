# Notification services for Cadence

**Date:** 2026-11-13

**Question:** Which notification services fit Cadence? Cadence is a server-rendered React Router 8 app (framework mode, Node server) using SQLite through better-sqlite3 and Drizzle. It has no native app, no service worker, and no job runner today. Students and Instructors need:

- an in-app inbox or feed
- push that reaches phones, most likely Web Push from a PWA (on iOS this needs a home-screen-installed PWA on iOS 16.4+)
- possibly email
- scheduled re-engagement nudges, digests or batching, and user preference controls

Sources are primary only: vendor pricing pages, vendor docs, GitHub, and npm. Claims marked "unverified" come from memory or could not be confirmed.

## Summary

- **Web Push is the main gap for the hosted vendors.** Knock, Novu, and Courier do not send VAPID Web Push themselves. They route it through FCM or OneSignal, so Cadence would ship a service worker and a second vendor's SDK either way. MagicBell is the hosted option that documents a native Web Push channel. OneSignal and FCM are push-first but have no web inbox.
- **Knock and Novu cover the most of the brief out of the box:** feed, digests, delays, preferences, and email. Each has a free tier of about 10k messages or workflow runs a month. The next paid step is a big jump for Knock ($250/mo) and a small one for Novu ($30/mo).
- **Novu's free plan caps delay and digest windows at 1 day.** That limits multi-day re-engagement nudges unless Cadence pays for Pro (7 days) or Team (30 days).
- **DIY is viable.** `web-push` released v4.0.0 on 2026-11-06 and is actively maintained. A DIY build needs Cadence to own the scheduler and job table, the inbox table, the preference UI, and the digest logic.
- **Every option needs a service worker and a manifest for push.** That work is the same whichever option is chosen.

## Comparison

| Option | Free tier | First paid tier | In-app feed UI | Web Push | Email / SMS | Digest / batch | Preferences UI | Delay / schedule | Self-host | Fit with React Router SSR + SQLite | Signup needed to judge? |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **DIY** (`web-push` + SQLite + Resend) | Library free. Resend: 3k emails/mo, 100/day | Resend Pro $20/mo for 50k emails | Build it | Native VAPID (`web-push` 4.0.0) | Through an email provider. SMS separate | Build it | Build it | Build it (job table plus a loop) | Yes, it is our code | Best fit. Data stays in SQLite and rendering is fully SSR | No |
| **Knock** | 10k messages/mo | Starter $250/mo with 50k included, then $0.005 each | Yes. `@knocklabs/react` feed, popover, and bell | Only through FCM (or OneSignal) | Yes, unlimited channels | Yes, batch function | Hooks and a guide. Not documented as a drop-in component | Delay function. Schedules (recurring, timezone-aware) | No | Client SDK. Users are identified in Knock. Signed JWT in production. SSR not documented | Partly. Behavior needs a free account |
| **Novu Cloud** | 10k workflow runs/mo. Delay and digest capped at 1 day | Pro from $30/mo for 30k runs, $1.20 per 1k extra. Delay and digest up to 7 days | Yes. `@novu/react` Inbox, Bell, Notifications, Preferences | Through FCM, OneSignal, Pusher Beams, or Pushpad. Native VAPID not documented | Yes, all channels on Free | Yes (regular, backoff, timed) | Yes, `<Preferences>` component | Delay: fixed, scheduled, dynamic | n/a | Client SDK. Subscriber created on first session. HMAC for production. SSR not documented | Free tier is enough to judge |
| **Novu self-hosted (Community)** | Free, MIT (except `/enterprise`) | Enterprise self-host: price through sales | Yes, same components except Snooze | Same as Cloud | Yes | Yes | Yes | Yes | Yes: API, Worker, WS, Dashboard, MongoDB, Redis (AOF), S3 | Heavy extra infrastructure next to a SQLite app. 1 team member | No signup, but needs a trial deployment |
| **OneSignal** | Web push free (10k subscribers per send), 10k emails/mo | Growth from $19/mo plus $0.004 per web-push subscriber, $1.50 per 1k emails | No web inbox. In-app messages are mobile-only | Yes, native. Strong iOS PWA docs | Email yes. SMS on paid tiers (price not listed) | Not found | Not found as a component | Time-delayed sends look paid-only. Journeys capped at 3 on Free | No | Web SDK and its own service worker. A feed would be DIY | Pricing yes, features partly |
| **Courier** | 10k sends/mo | Business $0.005 per send, every send billed | Yes. `@trycourier/courier-react` Inbox and Toast | Only through a provider (FCM or OneSignal) | Yes | Yes (batch and digest steps). Pricing page lists digest as Enterprise | Yes, hosted page and component | Delay step. Digest schedules | No | Client SDK with a server-minted JWT. No user pre-sync | Yes. Plan gating of digests is unclear |
| **FCM** | Free (no-cost product) | n/a | No | Yes, through the FCM web SDK and `firebase-messaging-sw.js` | No | No | No | No | No | Transport only. Adds the Firebase SDK and a Google project | No |
| **MagicBell** | 1k deliveries/mo | Startup $249/mo for 50k, then $0.0025 each | Yes. React `Inbox` and `FloatingInbox` | Yes, native VAPID channel | Yes (per its docs. Plan mapping not listed) | Not found in docs | Claimed on its homepage | Workflows API. Delays not confirmed | No | Client SDK | Yes. Plan features are not listed |
| **SuprSend** | 10k notifications/mo, branded feed | Essentials $100/mo for 50k. Digest and preferences need Business at $250/mo | Yes | Listed under "unlimited channels" (not verified in detail) | Yes | Business tier and up | Business tier and up | Not explicit | Enterprise only | Not checked in depth | Yes |

## Per-option details

### DIY: `web-push`, SQLite inbox, own scheduler, email provider

**`web-push` package**

- The latest version is **4.0.0**, published 2026-11-06. The previous release was 3.6.7 on 2024-01-16. Source: `npm view web-push`; tag `v4.0.0` on [GitHub](https://github.com/web-push-libs/web-push).
- It requires Node `>= 16` and is licensed MPL-2.0. It has 4 runtime dependencies: `jws`, `asn1.js`, `http_ece`, and `https-proxy-agent`. Source: npm registry metadata for `web-push@4.0.0`.
- It is actively maintained. The repo had commits on 2026-11-10, and the npm maintainer is the Mozilla-affiliated `marco-c`. Source: GitHub commits API on `web-push-libs/web-push`.
- The repo has 37 open issues, 7 open PRs, and 3.6k stars. Source: [GitHub](https://github.com/web-push-libs/web-push).
- No changelog for 4.0.0 was found on the repo page, so the breaking changes are **unknown**.
- The README lists Safari 16+ support with VAPID.
- **Apple gotcha:** use a real `mailto:` or `https:` VAPID subject. Apple rejects `https://localhost` subjects with `403 BadJwtToken`. Source: [GitHub README](https://github.com/web-push-libs/web-push).
- The API is `generateVAPIDKeys()`, `setVapidDetails()`, and `sendNotification()`. Source: [GitHub README](https://github.com/web-push-libs/web-push).

**iOS requirements** (from OneSignal's docs, which apply to any Web Push setup):

- iOS 16.4 or later, served over HTTPS.
- A manifest with `display: standalone`.
- The user adds the site to the Home Screen.
- The permission prompt runs from a user tap inside the installed app.
- If the user taps "Don't Allow", the prompt is never shown again for that install.
- Safari revokes permission if a push arrives and no notification is shown.

Source: [OneSignal: Web push for iOS](https://documentation.onesignal.com/docs/web-push-for-ios).

**What Cadence builds**

- A `push_subscriptions` table, a `notifications` inbox table, and a preferences table.
- A job table polled by an in-process loop for delays, nudges, and digests.
- The service worker and manifest.
- An email template plus a provider.

**Email**

- Resend's free tier is 3,000 emails/mo, capped at 100/day.
- Resend Pro is $20/mo for 50,000 emails/mo.
- Source: [Resend pricing](https://resend.com/pricing). Postmark and SES were not checked.

**Fit:** Best SSR and SQLite fit. There is no third-party user sync and no client SDK. The cost is engineering time to build retries, cleanup of expired subscriptions (push services return 410 for those), and digest logic.

### Knock

**Pricing** ([knock.app/pricing](https://knock.app/pricing))

- **Free:** 10,000 messages/mo. There is no hard cap; Knock contacts you to set up billing.
- **Starter:** $250/mo with 50,000 messages, then $0.005 each.
- Batching, digests, preferences, and the in-app feed are on all tiers.
- **Enterprise only:** per-customer branding and preferences, and i18n.

**Channels**

- **Push:** APNs, FCM, Expo, Amazon SNS, and OneSignal ([push overview](https://docs.knock.app/integrations/push/overview)).
- **Web Push:** Knock says it has no first-party web push channel. Web delivery goes through FCM: the FCM web SDK gets a token, which is stored as the user's FCM channel data ([tutorial](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm)).

**Digests**

- The batch function supports fixed, dynamic, relative, and sliding windows.
- A batch stays open for at most 90 days, with an optional activity cap.
- Per-recipient windows and timezones are "not currently supported" (per the FAQ).
- Source: [batch function](https://docs.knock.app/designing-workflows/batch-function).

**Delays and scheduling**

- The delay function supports a fixed interval, a dynamic period, or a relative timestamp ([delay function](https://docs.knock.app/designing-workflows/delay-function)).
- **Schedules** are "a managed, recipient-timezone-aware cron job", recurring or one-off ([schedules](https://docs.knock.app/concepts/schedules)). This covers re-engagement nudges without a Cadence job runner.

**Preferences**

- A preferences API, with a React guide and a `usePreferences` hook.
- Rows and labels can be configured in the dashboard.
- No drop-in preference component is documented on the React overview.
- Sources: [React preferences](https://docs.knock.app/in-app-ui/react/preferences), [custom preference center](https://docs.knock.app/preferences/custom-preference-center).

**SDK fit**

- Components: `@knocklabs/react` provides `NotificationFeedPopover`, `NotificationFeed`, `NotificationIconButton`, and headless hooks ([React overview](https://docs.knock.app/in-app-ui/react/overview)).
- Auth: the client authenticates with a public key and the user ID. Production should enable **enhanced security**, where the backend signs an RS256 JWT with `sub` set to the user ID ([security](https://docs.knock.app/in-app-ui/security-and-authentication)).
- Users are addressed by ID at `/v1/users/:id/feeds/:channel_id`, so Cadence users must be identified in Knock.
- No React Router or SSR guidance was found in the docs.
- **Unverified:** the component is likely client-rendered after hydration.

**Self-host:** No.

### Novu Cloud and Novu self-hosted

**Pricing** ([novu.co/pricing](https://novu.co/pricing))

- **Free:** 10,000 workflow runs/mo, 5 workflows, 3 team members, all channels.
- **Pro:** from $30/mo for 30,000 runs, then $1.20 per 1,000 extra.
- **Team:** from $250/mo for 250,000 runs.
- **Enterprise self-host:** through sales only, with no published price.

**Limits** ([limits](https://docs.novu.co/platform/developer/limits))

| | Free | Pro | Team |
|---|---|---|---|
| Max delay and digest duration | 1 day | 7 days | 30 days |
| Inbox feed retention | 30 days | 90 days | 90 days |

**Channels**

- Email, in-app, SMS, chat, and push.
- Push providers: FCM, Expo, APNs, OneSignal, Push Webhook, Pusher Beams, and Pushpad.
- Native VAPID Web Push is not documented ([push](https://docs.novu.co/platform/integrations/push)).

**Workflow steps**

- **Delay:** fixed, scheduled (in the subscriber's timezone), or dynamic ([delay](https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/delay)).
- **Digest:** regular, backoff, and timed modes ([digest](https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/digest)).

**SDK fit** ([Inbox overview](https://docs.novu.co/platform/inbox/overview))

- `@novu/react` needs React 18+ and provides `<Inbox>`, `<Bell>`, `<Notifications>`, `<InboxContent>`, and `<Preferences>`.
- `@novu/js` is the framework-agnostic option.
- "New subscribers are created on the first session", so Cadence does not need to sync users up front.
- Production uses HMAC `subscriberHash`.
- SSR is not documented.
- Latest release: `@novu/react` v3.19.2 on 2026-09-07 (GitHub releases).

**Self-hosted (Community) contents** ([self-hosted vs Cloud](https://docs.novu.co/community/self-hosted-and-novu-cloud))

- **Included:** all channels, digest, GUI and code workflows, and the Inbox, Bell, and Preferences components.
- **Missing compared with Cloud:**
  - Snooze
  - branding removal
  - email delivery tracking
  - custom environments and more than 2 environments
  - translations
  - social login, SSO, MFA, and RBAC
- Max 1 team member.

**License:** "Open Core". The core is MIT (`LICENSE-MIT`). The `/enterprise` folder is under a proprietary license that requires prior written approval ([README](https://github.com/novuhq/novu)).

**Self-host infrastructure** ([self-hosting overview](https://docs.novu.co/community/self-hosting-novu/overview))

- **Single-VM:** 4 vCPU and 8 GB RAM for all services, plus MongoDB (2 GB RAM), Redis with AOF, and at least 10 GB of S3-compatible storage.
- **Recommended multi-VM:** separate API, Worker, WS, and Dashboard VMs, 2 Redis clusters, and MongoDB Atlas M20+.
- That is far heavier than Cadence's current single Node process with SQLite.

### OneSignal

**Pricing** ([onesignal.com/pricing](https://onesignal.com/pricing))

- **Free:** web push (comparison table: "Max 10,000 subscribers per send"), 10,000 emails/mo, and mobile push and in-app for up to 1,000 MAU.
- **Growth:** from $19/mo, plus $0.004 per web-push subscriber, $0.012 per mobile MAU, and $1.50 per 1,000 emails beyond 20,000.
- Time-delayed sends and Intelligent Delivery show "—" on Free.
- Active journeys: Free 3, Growth 5.
- No preference center is listed.

**Channels**

- Web Push is native, with the most thorough iOS PWA docs found ([iOS web push](https://documentation.onesignal.com/docs/web-push-for-ios)).
- It requires OneSignal's own `OneSignalSDKWorker.js`. Only one service worker can control a scope, so Cadence's service worker would have to coexist with or import OneSignal's.
- **No web inbox:** in-app messages are mobile-SDK only. OneSignal suggests building a feed from its REST API or Event Streams ([in-app overview](https://documentation.onesignal.com/docs/en/in-app-messages-setup)).

**Digests and preference UI:** Not found.

**Fit:** Strong for the push leg, weak for the inbox and digests.

### Courier

**Pricing** ([courier.com/pricing](https://www.courier.com/pricing))

- **Developer:** free, 10,000 sends/mo.
- **Business:** $0.005 per send, and every send is billed, including the first 10k.
- One template to one recipient counts as one send, whatever the number of channels.
- The pricing table lists "Digest and batching" under Enterprise and does not list it for the other tiers. That conflicts with the docs below, so plan gating is **unclear**.

**Digests and delays**

- Digests use schedules defined in the Preferences Editor, and users can pick schedules on the hosted preference page ([digest](https://www.courier.com/docs/platform/automations/digest)).
- The batch step supports `wait_period`, `max_wait_period` (up to 60 days), and `max_items`. A delay step waits for a duration or until a timestamp ([steps](https://courier.com/docs/platform/automations/steps)).
- Automations is now labelled "legacy", with "Journeys" replacing it ([automations overview](https://www.courier.com/docs/platform/automations/automations-overview)).

**Preferences:** A hosted preference page plus a component ([preferences editor](https://www.courier.com/docs/recipients/preferences/preferences-editor)).

**Inbox and auth**

- `@trycourier/courier-react` v9 provides `CourierInbox` and `useCourier` ([inbox overview](https://www.courier.com/docs/platform/inbox/inbox-overview)).
- The backend mints a JWT through `POST /auth/issue-token` with scopes. Tokens are not auto-refreshed.
- "Nothing has to exist in Courier first, since signing in registers the user" ([inbox auth](https://www.courier.com/docs/platform/inbox/authentication)).

**Web Push:** The inbox docs mention mobile push only (APNs and FCM). Browser push goes through a provider such as OneSignal ([Courier WordPress guide](https://www.courier.com/guides/wordpress-notifications)). Courier's own blog content was not used as evidence.

### Firebase Cloud Messaging

- **Pricing:** listed as a "No-cost" product ([Firebase pricing](https://firebase.google.com/pricing)).
- **Transport only:** no inbox, digests, preferences, or scheduling.
- **Web setup** ([FCM JS client](https://firebase.google.com/docs/cloud-messaging/js/client)):
  - a `firebase-messaging-sw.js` service worker at the root
  - a VAPID key pair created in the Firebase console
  - HTTPS
  - the Firebase JS SDK on the client
  - a server that stores tokens and sends through the FCM API
- Knock and Novu both use FCM as their web-push path, so FCM may end up in the stack regardless.
- **Unverified:** iOS PWA support through FCM web. The FCM page does not address Safari or iOS specifically.

### MagicBell (extra contender 1)

- **Pricing** ([magicbell.com/pricing](https://www.magicbell.com/pricing)):
  - Builder: free, 1,000 deliveries/mo.
  - Startup: $249/mo for 50,000, then $0.0025 per delivery.
  - The page does not map features to plans.
- **Native Web Push:** you upload your VAPID keys and MagicBell stores the endpoint and keys (p256dh, auth) ([web push channel](https://www.magicbell.com/docs/channels/web_push)).
- **In-app inbox:** React `Inbox` and `FloatingInbox` ([React components](https://magicbell.com/docs/libraries/react), [in-app channel](https://www.magicbell.com/docs/channels/in_app)).
- **Workflows:** a Workflows API exists ([API v2](https://www.magicbell.com/docs/api/reference)).
- **Digests and delays:** no first-party documentation found.
- **Preferences:** claimed on the homepage only (unverified).
- **Fit:** the free tier is small, at 1k deliveries.

### SuprSend (extra contender 2)

- **Pricing** ([suprsend.com/pricing](https://www.suprsend.com/pricing)):
  - Free: 10k notifications/mo, with a branded in-app feed.
  - Essentials: $100/mo for 50k.
  - Business: $250/mo for 50k, then $5 per 1k extra. **Batching, digests, and user preferences start at Business.**
  - Self-hosting is Enterprise only.
- Includes a one-month sandbox for paid features and a 30-day premium trial.
- Web Push details and the SDK were not verified.

## Leading candidates

These are front-runners only. A later ticket makes the decision.

1. **DIY (`web-push` + SQLite + Resend)**
   - **For:** best fit with SSR and SQLite. No vendor user sync, no client SDK, native VAPID Web Push, and near-zero cost.
   - **Against:** Cadence must build and own the job table and scheduler, digests, the preference UI, the inbox UI, retries, and subscription cleanup. No dashboard for non-engineers.

2. **Novu (Cloud, with self-hosting as an exit)**
   - **For:** the most complete drop-in UI (`<Inbox>` plus `<Preferences>`) and digest and delay steps. Subscribers are created lazily, so no user sync. Cheapest paid step ($30/mo). An MIT core gives a self-host escape hatch.
   - **Against:** the free plan's 1-day delay and digest cap blocks multi-day nudges. Web Push only through FCM or OneSignal. Self-hosting needs MongoDB, Redis, and S3, which is heavy for this stack.

3. **Knock**
   - **For:** the most mature workflow engine. The batch function supports windows up to 90 days, and recipient-timezone-aware **schedules** suit re-engagement nudges with no job runner. The free tier is generous at 10k messages.
   - **Against:** a $250/mo jump at the first paid tier. Web Push only through FCM. Users must be identified in Knock, with signed JWTs in production. The preference UI is hook-based, not drop-in. SSR is undocumented.

A hybrid is also plausible: a hosted inbox and workflow vendor, with push through FCM or OneSignal as the provider. Either way, Cadence ships a service worker and manifest.

## Needs signup/trial to judge?

- **DIY, FCM:** No. The docs and npm are enough.
- **Novu:** The free Cloud tier (no card) is enough to test the Inbox under React Router SSR, FCM web push, and digests. Self-host can be tested locally with Docker, with no signup.
- **Knock:** Pricing is public. A free account is needed to confirm three things: SSR behavior of `@knocklabs/react` in React Router, FCM web push on an iOS PWA, and whether delays and schedules have plan limits (not stated on the pricing page).
- **Courier:** Yes, to resolve whether digests are Enterprise-only (pricing table) or available generally (docs).
- **OneSignal:** Partly. Some cells on the pricing feature table did not render (scheduling and frequency capping per tier).
- **MagicBell, SuprSend:** Yes. MagicBell's pricing page does not map features to plans. SuprSend gates digests and preferences behind its $250/mo Business tier, which needs confirming in its sandbox.
- **Novu Enterprise self-host:** pricing only through sales.

## Sources

- https://www.npmjs.com/package/web-push (npm registry metadata, via `npm view`)
- https://github.com/web-push-libs/web-push
- https://documentation.onesignal.com/docs/web-push-for-ios
- https://resend.com/pricing
- https://knock.app/pricing
- https://docs.knock.app/integrations/push/overview
- https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm
- https://docs.knock.app/designing-workflows/batch-function
- https://docs.knock.app/designing-workflows/delay-function
- https://docs.knock.app/concepts/schedules
- https://docs.knock.app/in-app-ui/react/overview
- https://docs.knock.app/in-app-ui/react/preferences
- https://docs.knock.app/preferences/custom-preference-center
- https://docs.knock.app/in-app-ui/security-and-authentication
- https://novu.co/pricing
- https://docs.novu.co/platform/developer/limits
- https://docs.novu.co/platform/integrations/push
- https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/delay
- https://docs.novu.co/platform/workflow/add-and-configure-steps/configure-action-steps/digest
- https://docs.novu.co/platform/inbox/overview
- https://docs.novu.co/community/self-hosted-and-novu-cloud
- https://docs.novu.co/community/self-hosting-novu/overview
- https://github.com/novuhq/novu (README license section, LICENSE-MIT, EE-PACKAGES-LICENSE, releases)
- https://onesignal.com/pricing
- https://documentation.onesignal.com/docs/en/in-app-messages-setup
- https://www.courier.com/pricing
- https://www.courier.com/docs/platform/inbox/inbox-overview
- https://www.courier.com/docs/platform/inbox/authentication
- https://www.courier.com/docs/platform/automations/digest
- https://courier.com/docs/platform/automations/steps
- https://www.courier.com/docs/platform/automations/automations-overview
- https://www.courier.com/docs/recipients/preferences/preferences-editor
- https://www.courier.com/guides/wordpress-notifications
- https://firebase.google.com/pricing
- https://firebase.google.com/docs/cloud-messaging/js/client
- https://www.magicbell.com/pricing
- https://www.magicbell.com/docs/channels/web_push
- https://www.magicbell.com/docs/channels/in_app
- https://magicbell.com/docs/libraries/react
- https://www.magicbell.com/docs/api/reference
- https://www.suprsend.com/pricing

**Could not fetch (404):**

- `docs.novu.co/community/self-hosting-novu/community-vs-cloud`. The current page `self-hosted-and-novu-cloud` was used instead.
- `courier.com/docs/platform/sending/batching`. The automations digest and steps pages were used instead.
