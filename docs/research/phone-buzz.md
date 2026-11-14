# How can a web app make a phone buzz?

Research for issue #2 (child of map #1, "App notifications spec"). Researched 2026-11-13.

Cadence today: server-rendered React Router 8 + SQLite, no service worker, no web app manifest, no job runner, no native app.

## Answer

**Web Push (Push API + a service worker + a manifest) is the default choice.** It is free, needs no app store, and reaches:

- **Android:** every major browser, from a normal tab. No install is needed.
- **iPhone/iPad:** only after the user adds Cadence to the Home Screen and opens it from there (iOS/iPadOS 16.4+).

The iOS install step is the real gap. A Student who never installs Cadence on an iPhone cannot be buzzed by Web Push. Two options fill it:

- **SMS** reaches every phone with no install, but each message costs money and US sending needs carrier registration and opt-in consent.
- **A native wrapper (Capacitor)** gets a native APNs/FCM channel. It needs Apple and Google developer accounts, Firebase, store review (where a website wrapper risks rejection), and it is a second app to maintain.

Recommendation:

1. Ship Web Push with an iOS "Add to Home Screen" prompt. Consider Declarative Web Push for iOS/Safari.
2. Treat SMS as an optional, opt-in fallback for a few high-value events.
3. Don't build a wrapper for notifications alone.

## Key facts

| | Web Push (browser / PWA) | SMS (e.g. Twilio) | Native wrapper (Capacitor) |
|---|---|---|---|
| Android reach | Chrome 42+, Firefox for Android 48+, Samsung Internet, Opera. Works from a normal tab [1] | Any phone with a number | Any Android with Google Play services (FCM) [8] |
| iOS reach | iOS/iPadOS 16.4+, **only for web apps added to the Home Screen** [1][2][3] | Any phone | Any iOS with the app installed (APNs) [8] |
| Install needed | Android no. iOS yes (Home Screen) | No | Yes (app store) |
| Permission | Must be asked from a user gesture (tap). OS/browser prompt [2][3][5] | Explicit opt-in consent. US A2P 10DLC registration [7] | OS prompt. Android 13+ runtime permission [8] |
| Payload | 4 KB max (Apple returns `PayloadTooLarge`) [3]. Must always show a visible notification [3][5] | 160 GSM-7 chars per segment, billed per segment | ~4 KB (APNs/FCM) |
| Reliability | Push service stores the message while the device is offline, up to `TTL` (Apple: up to 30 days, limited count) [3]. Safari revokes permission if a push shows no notification [3]. Chrome rate-limits and auto-revokes for low-engagement, high-volume sites [9][10] | Carrier delivery. Filtering if unregistered | Best. Native OS channel |
| Money | Free. No Apple Developer Program needed [3] | ~$0.0083/segment + $0.0035–0.005 carrier fee. Number $1.15/mo. 10DLC: $4.50–46 brand + $15 campaign vetting + $1.50–10/mo [6][7] | Apple Developer Program $99/yr + Google Play $25 one-off, plus Firebase setup |
| Effort for Cadence | Medium: manifest, service worker, subscribe UI, subscriptions table, VAPID keys, server sender (e.g. `web-push` npm). No job runner needed for synchronous sends | Low code (one HTTP call), plus ops: registration, consent capture, STOP handling, phone number field | High: second codebase/build pipeline, store review risk (Guideline 4.2) [11], push plugin + FCM/APNs credentials |

## Web Push details

### How it works

- A service worker subscribes through `PushManager.subscribe()` with a VAPID public key. The server stores the returned endpoint and keys against the user, then POSTs an encrypted payload to that endpoint (RFC 8030/8291) [3][5].
- Each browser picks its own push service: FCM for Chrome, Mozilla's for Firefox, `*.push.apple.com` for Safari/iOS. Because the protocol is standard, one server implementation covers all of them [5].
- Requires HTTPS, an active service worker, and `userVisibleOnly: true`. Silent push is not allowed anywhere [1][3][5].
- Android WebView does **not** support the Push API [1]. A Capacitor/WebView wrapper cannot reuse Web Push and must use native FCM.

### iOS / iPadOS state as of November 2026

- **Since iOS 16.4 (March 2023):** Web Push and app badges work for Home Screen web apps. The site needs a manifest with `display: standalone` or `fullscreen`. Permission can only be requested from a direct user gesture. No Apple Developer account is needed. Any iOS browser that supports Add to Home Screen (Safari, Chrome, Edge, Firefox via the browser entitlement) can create the web app, and push then belongs to the Home Screen app, not the browser [2]. MDN compat confirms that iOS notifications work "in web apps saved to the home screen" only [1].
- **No push in a plain Safari tab on iOS.** macOS Safari 16+ (Ventura and later) supports push in normal tabs [1][3].
- **iOS 26 (Safari 26, September 2025):** every site added to the Home Screen now opens as a web app by default, even without a manifest. Users can turn off "Open as Web App" [12]. This lowers the bar for installing but does not remove it.
- **Declarative Web Push (iOS/iPadOS 18.4+, macOS Safari 18.5+):**
  - The server sends JSON with `"web_push": 8030` and a `notification` object (`title`, `navigate` URL, optional `app_badge`).
  - The browser shows the notification itself, with no service worker JavaScript needed. Subscriptions are made through `window.pushManager`.
  - It is more robust: no service worker code runs, so there is no risk of permission revocation from a failed `showNotification`.
  - It is WebKit-only for now and still going through standardisation. Chrome and Firefox still need the classic service worker path [4][13].
- **EU:** Apple's iOS 17.4 betas removed Home Screen web apps in the EU (DMA), and Apple reversed this before the iOS 17.4 release. EU users keep Home Screen web apps and Web Push, built on WebKit [14].
- I found no Apple announcement through Safari 26.x of push for non-installed iOS tabs.

### Android / Chrome limits

- Since Chrome 80/84, sites with low prompt-accept rates or abusive notifications get the quieter permission UI [15].
- **Auto-revoke (announced 10 October 2025):** Chrome removes notification permission on Android and desktop from sites with very low engagement and high notification volume. **Installed web apps are exempt** [10].
- **Push rate limits (rolling out from January 2026):** for sites judged disruptive (many pushes, little engagement), Chrome caps sending at about 1,000 messages per minute with HTTP 429, escalating from 1 to 7 to 14 days [9]. This won't affect normal course notifications.

### Practical caveats

- Subscriptions are per device and per browser. A user can have several. Delete them on HTTP 404/410 from the push service [3].
- If a device has no network, the message waits up to `TTL`. Use the `Urgency: high` header for time-sensitive messages and `Topic` to collapse duplicates [3].
- Users can silence notifications at the OS level (Focus on iOS, channel settings on Android), so delivery is not a guarantee the user saw it. Keep an in-app notification list as the source of truth.

## SMS details

- **Twilio US:** $0.0083 per outbound segment plus carrier fees ($0.0035 AT&T, $0.0045 T-Mobile, $0.005 Verizon). A long-code number is $1.15/month and a toll-free number $2.15/month [6].
- **US A2P 10DLC registration is mandatory:** brand $4.50 (Sole Prop / Low-Volume) or $46 (Standard, incl. vetting), $15 non-refundable campaign vetting, then $1.50–10 per month per campaign [7]. Toll-free numbers need toll-free verification instead.
- **Needs** a verified phone number per user, recorded opt-in consent, and STOP/HELP handling. Pricing differs per country. Canada, the EU, etc. were not researched here.
- **Good for:** critical, rare events (e.g. "your live session starts in 15 min"). Too costly and noisy for routine activity.

## Native wrapper (comparison only)

- **Capacitor Push Notifications plugin:** uses FCM on Android (needs a Firebase project and `google-services.json`) and APNs on iOS (needs the Push Notifications capability, an Apple Developer account and AppDelegate wiring). It needs the Android 13 runtime permission, and iOS silent push is not supported by the plugin [8].
- **App Store risk:** Guideline 4.2 rejects apps that are not "beyond a repackaged website", and 4.2.2 rejects "web clippings" [11]. A thin wrapper around Cadence is at risk.
- It adds a store release cycle, signing and a second build. That is out of scope per the issue, and only worth it if native features beyond notifications are wanted.

## New questions surfaced

1. How many Cadence Students and Instructors are on iOS? Will they install a Home Screen web app, and how do we prompt them to?
2. Which events actually warrant a phone buzz, and is any of them critical enough to justify paid SMS? Who pays?
3. Where do sends run? No job runner exists, so do we send inline in the request, or add a queue/cron for scheduled reminders such as "session starts soon"?
4. Do we adopt Declarative Web Push for Safari and the classic service worker path for Chrome/Firefox, or the classic path for all (iOS still supports it)?
5. Privacy and compliance: storing push subscriptions and phone numbers, consent records, and per-user notification preferences.

## Sources

1. MDN browser-compat-data, `api/PushManager.json` (main branch, read 2026-11-13): https://github.com/mdn/browser-compat-data/blob/main/api/PushManager.json
2. WebKit blog, "Web Push for Web Apps on iOS and iPadOS" (2023-02-16): https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
3. Apple Developer, "Sending web push notifications in web apps and browsers": https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers
4. WebKit blog, "Meet Declarative Web Push" (2025-03-27): https://webkit.org/blog/16535/meet-declarative-web-push/
5. web.dev, "Push notifications overview": https://web.dev/articles/push-notifications-overview
6. Twilio, SMS pricing (US): https://www.twilio.com/en-us/sms/pricing/us
7. Twilio, Pricing and fees for A2P 10DLC: https://support.twilio.com/hc/en-us/articles/1260803965530-Pricing-and-Fees-for-A2P-10DLC-Service
8. Capacitor, Push Notifications plugin: https://capacitorjs.com/docs/apis/push-notifications
9. Chrome for Developers, "Increasing web push notification value with rate limits" (2026-01-06): https://developer.chrome.com/blog/web-push-rate-limits
10. Chromium blog, "Automatic notification permission revocation" (2025-10-10): https://blog.google/chromium/automatic-notification-permission/
11. Apple, App Store Review Guidelines §4.2: https://developer.apple.com/app-store/review/guidelines/
12. WebKit blog, "News from WWDC25: WebKit in Safari 26 beta": https://webkit.org/blog/16993/news-from-wwdc25-web-technology-coming-this-fall-in-safari-26-beta/
13. WebKit blog, "WebKit Features in Safari 18.4": https://webkit.org/blog/16574/webkit-features-in-safari-18-4/
14. Apple Developer, "Update on apps distributed in the European Union" (Home Screen web apps FAQ): https://developer.apple.com/support/dma-and-apps-in-the-eu/
15. Chrome for Developers, "New in Chrome 84" (quieter notification UI for abusive sites): https://developer.chrome.com/blog/new-in-chrome-84
