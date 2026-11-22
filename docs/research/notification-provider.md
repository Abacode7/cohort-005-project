# Research: hosted provider for the Cadence Notification Feed

- **Question:** Which hosted notification provider should power Cadence's in-app Notification Feed? Cadence uses React Router 8 (framework mode, Node server), TypeScript, SQLite/Drizzle, Tailwind and shadcn/ui.
- **Scope:** Only the in-app feed channel. Email, push, SMS, preferences and digests are out of scope.
- **Hard constraints:** (1) a free tier big enough for a learning project, (2) a Node SDK that works server-side, (3) a prebuilt React in-app feed component.
- **Checked on:** 2026-11-22. Vendor pages change often, so check the pricing pages again before you sign up.

Every claim links to the primary source it came from. "Not verified" means the official page did not state the fact when it was checked.

---

## 1. Knock

**Free tier.** The Free plan costs $0/month and includes "up to 10,000" messages sent per month. No credit card is required. If you go over the limit, Knock does not cut you off; it contacts you about billing the next month. ([knock.app/pricing](https://knock.app/pricing))
- A message is "counted each time a notification is successfully delivered to a single user on a single channel" ([docs.knock.app/manage-your-account/knock-plans](https://docs.knock.app/manage-your-account/knock-plans)). One in-app feed item for one user is therefore about one message.
- Feed retention on the Free plan is not verified.

**Node SDK.** The package is `@knocklabs/node` ([github.com/knocklabs/knock-node](https://github.com/knocklabs/knock-node)).
```ts
import Knock from "@knocklabs/node";
const knock = new Knock({ apiKey: process.env.KNOCK_API_KEY });
await knock.workflows.trigger("lesson-published", {
  recipients: ["user_123"],          // or inline objects, see below
  data: { lessonTitle: "Intro" },
});
```
You create the workflow (for example `lesson-published`, with an in-app feed step) in the Knock dashboard.

**User identification.** You can identify users inline when you trigger. Pass a recipient object such as `{ id, name, email }` and Knock guarantees the recipient is "identified **before** the workflow is triggered". Only `id` is required. ([docs.knock.app/send-notifications/triggering-workflows/api](https://docs.knock.app/send-notifications/triggering-workflows/api)) You do not need a separate identify step.

**Browser feed auth.**
- The client sends the public API key plus the user ID.
- In **enhanced security mode** it must also send a signed user token:
  - signed with RS256 using the environment's private signing key (`KNOCK_SIGNING_KEY`, from Platform > API keys);
  - `sub` claim is required, and `iat`/`exp` are recommended;
  - created on the server with `signUserToken` from `@knocklabs/node/lib/tokenSigner` (`expiresInSeconds` defaults to 3600);
  - passed to `<KnockProvider userToken=...>`.
- The docs say "Production environments should enable enhanced security mode". Without it, "a request with your public key can act as any user".
- Source: [docs.knock.app/in-app-ui/security-and-authentication](https://docs.knock.app/in-app-ui/security-and-authentication)

**Real-time.** Yes. The feed is "a real-time feed component" that uses websockets ([docs.knock.app/in-app-ui/react/feed](https://docs.knock.app/in-app-ui/react/feed)).
- `feedClient.listenForUpdates()` connects to the real-time socket, and new items arrive as `items.received.realtime` events.
- `auto_manage_socket_connection` disconnects sockets in inactive tabs.
- Source: [@knocklabs/client README](https://github.com/knocklabs/javascript/tree/main/packages/client)

**Customisability.**
- **Styled components** in `@knocklabs/react`: `KnockProvider`, `KnockFeedProvider`, `NotificationIconButton`, `NotificationFeedPopover`, `NotificationFeed` (which takes a `renderItem` prop).
- **Styling:** CSS variables prefixed `--rnf-`, `rnf-` classes, or replace the Knock CSS entirely. `colorMode="dark"` switches to dark mode. ([docs.knock.app/in-app-ui/react/feed](https://docs.knock.app/in-app-ui/react/feed))
- **Headless:** `useKnockClient`, `useNotifications` and `useNotificationStore` give you `items` and `metadata` to render with your own markup, such as shadcn components ([docs.knock.app/in-app-ui/react/headless/feed](https://docs.knock.app/in-app-ui/react/headless/feed)).

**Read/unread/seen.**
- The feed client has `markAsRead`, `markAsSeen` and `markAsArchived`, plus the reverse `markAsUnread`, `markAsUnseen` and `markAsUnarchived` ([client README](https://github.com/knocklabs/javascript/tree/main/packages/client)).
- The popover marks everything as **seen** when it opens. Set `badgeCountType="unread"` to count unread items in the badge instead ([feed docs](https://docs.knock.app/in-app-ui/react/feed)).
- Headless hooks do **not** mark items seen automatically. You call `markAllAsSeen()` or `markAsSeen(item)` yourself ([headless docs](https://docs.knock.app/in-app-ui/react/headless/feed)).

**Failure at trigger time.**
- The SDK automatically retries connection errors, 408, 409, 429 and 5xx responses **2 times** with exponential backoff. `maxRetries` is configurable. The default timeout is 1 minute and throws `APIConnectionTimeoutError`. ([knock-node README](https://github.com/knocklabs/knock-node))
- The trigger endpoint accepts an `Idempotency-Key` header. Knock keeps it for 24 hours, and a repeated request returns the original response ([triggering via API](https://docs.knock.app/send-notifications/triggering-workflows/api)).
- The README does not say whether the SDK sets the idempotency key for you (not verified).
- Anything beyond those retries, such as an outbox, logging, or swallowing the error, is up to the caller.

**Testing.**
- **Environments:** Knock has separate Development and Production environments with isolated data ([docs.knock.app/version-control/environments](https://docs.knock.app/version-control/environments)).
- **Sandbox mode:** channels have a sandbox mode, but "the in-app channel doesn't support sandbox mode" ([knock-plans](https://docs.knock.app/manage-your-account/knock-plans)).
- **Mocking:** there is no official mock mode and no unit-test guidance in the SDK README. The README only notes Jest 28+ with the `node` environment ([knock-node](https://github.com/knocklabs/knock-node)). Plan to stub the client yourself.

---

## 2. Novu

**Free tier.** 10,000 workflow runs per month, no credit card required ([novu.co/pricing](https://novu.co/pricing)). The plan also includes:
- up to 5 workflows;
- 2 environments (Dev and Prod);
- up to 3 team members;
- **24-hour activity feed retention**;
- all channels, including In-App.

The free plan cannot remove Novu branding from the Inbox ([prepare for production](https://docs.novu.co/platform/inbox/prepare-for-production)).

**Node SDK.** The package is `@novu/api` ([github.com/novuhq/novu-ts](https://github.com/novuhq/novu-ts)).
```ts
import { Novu } from "@novu/api";
const novu = new Novu({ secretKey: process.env.NOVU_SECRET_KEY });
await novu.trigger({ workflowId: "lesson-published", to: "user_123", payload: { lessonTitle: "Intro" } }, "idempotency-key");
```
The README says the SDK is **beta**: "there may be breaking changes between versions without a major version update". Pin the version.

**User identification.** Subscribers are created just-in-time from the trigger's `to` field, and Novu "upserts subscriber details on every trigger". You can also create them ahead of time through the API. ([docs.novu.co/platform/concepts/subscribers](https://docs.novu.co/platform/concepts/subscribers)) If `<Inbox subscriberId>` gets an unknown ID, Novu creates the subscriber ([React quickstart](https://docs.novu.co/platform/quickstart/react)).

**Browser feed auth.**
- `<Inbox applicationIdentifier subscriberId />` with the public application identifier.
- For production, enable "Security HMAC encryption" on the Novu In-App integration. The server then computes `HMAC-SHA256(NOVU_SECRET_KEY, subscriberId)` as lowercase hex and passes it as `subscriberHash`.
- With HMAC on and no hash, "the Inbox will not load". Without HMAC, anyone who can guess a `subscriberId` can read that user's feed.
- Source: [docs.novu.co/platform/inbox/prepare-for-production](https://docs.novu.co/platform/inbox/prepare-for-production)
- The server side is a single `createHmac` call with no JWT library and no expiry.

**Real-time.** Yes, over websockets. The production docs list a `socketUrl` (`wss://...socket.novu.co`). The `useNotifications` and `useCounts` hooks "automatically listen for real-time events", and you can subscribe yourself with `novu.on("notifications.notification_received")`. ([headless mode](https://docs.novu.co/platform/inbox/headless-mode))

**Customisability.**
- The `appearance` prop takes `variables` and `elements`. Elements accept "a style object or a set of CSS classes", and the docs say outright that you "can pass Tailwind CSS classes to specific elements". Callbacks are supported for dynamic classes. ([styling](https://docs.novu.co/platform/inbox/configuration/styling))
- `renderNotification`, `renderSubject` and `renderBody` let you replace parts of an item or the whole item ([customize notification items](https://docs.novu.co/platform/inbox/advanced-customization/customize-notification-items)).
- **Headless:** `useNotifications`, `useCounts` and `useNovu` ([headless mode](https://docs.novu.co/platform/inbox/headless-mode)).
- This is the best fit for Tailwind/shadcn of all the candidates.

**Read/unread.** `novu.notifications.read({ notificationId })` and `.archive(...)`. Mark-as-unread is mentioned, but no code is shown. The hooks refetch after these actions. ([headless mode](https://docs.novu.co/platform/inbox/headless-mode))

**Failure at trigger time.**
- **Retries:** the SDK supports a configurable `retryConfig` (backoff), but the README does not document the defaults ([novu-ts](https://github.com/novuhq/novu-ts)).
- **Idempotency:** the `Idempotency-Key` is passed as the second argument to `trigger`.
  - Responses are cached for 24 hours; in-progress requests are tracked for 5 minutes.
  - A key still being processed returns 409 with `Retry-After`; a key reused with a different body returns 422.
  - The docs warn that "idempotency is not enabled for all organizations".
  - Source: [docs.novu.co/api-reference/idempotency](https://docs.novu.co/api-reference/idempotency)

**Testing.**
- **Environments:** separate Dev and Prod ([pricing](https://novu.co/pricing)).
- **Mocking:** `HTTPClient` accepts a custom `fetcher`, which the README calls useful "when writing tests to mock out the HTTP client and feed in fixtures" ([novu-ts](https://github.com/novuhq/novu-ts)).

---

## 3. MagicBell

**Free tier.** The "Builder" plan is $0 with 1,000 deliveries per month, counted across channels. It includes 1 project and unlimited users. Extra deliveries are billed per use. The page does not say whether a credit card is required (not verified). ([magicbell.com/pricing](https://www.magicbell.com/pricing))

**Node SDK.** The package is `magicbell-js` (`magicbell-js/project-client`). `client.broadcasts.createBroadcast({ title, content, recipients: [{ externalId }] })` sends to recipients. ([docs/libraries/magicbell-js](https://www.magicbell.com/docs/libraries/magicbell-js)) The page does not document retries or idempotency (not verified).

**User identification and auth.**
- Recipients are identified by `externalId` or `email` ([magicbell-js](https://www.magicbell.com/docs/libraries/magicbell-js)).
- The browser uses a **User JWT**: HS256, signed with the project Secret Key, with claims `api_key` plus `user_external_id` and/or `user_email`. You mint it on your server. ([docs/api/authentication/user](https://www.magicbell.com/docs/api/authentication/user))
- There is no unauthenticated dev mode: `<Provider token>` always needs a JWT ([magicbell-react](https://www.magicbell.com/docs/libraries/magicbell-react)).

**Real-time.** MagicBell "maintains a persistent connection to deliver notifications in real-time". The transport is not named. ([docs/channels/in_app](https://www.magicbell.com/docs/channels/in_app))

**Customisability.** `@magicbell/react` provides `Provider`, `Inbox` and `FloatingInbox`. You can swap in your own components with `ItemComponent`, `ButtonComponent` and `InboxComponent`. Styling uses CSS files and `--magicbell-*` variables. The React page documents no headless hooks. ([magicbell-react](https://www.magicbell.com/docs/libraries/magicbell-react))

**Read/unread.** The UI supports mark read, archive and delete ([in_app](https://www.magicbell.com/docs/channels/in_app)). The pages checked do not name the API methods (not verified).

**Testing.** No test or mock guidance was found (not verified).

---

## 4. Courier

**Free tier.** The Developer plan includes 10,000 sends per month. Test environment sends count the same as production sends. ([courier.com/pricing](https://www.courier.com/pricing), [docs overview](https://www.courier.com/docs/overview)) The pricing page does not say whether Inbox is included or whether a card is needed. Courier's own 2026 blog pages say the free plan includes the inbox and needs no card, but that is secondary material, so treat it as not verified on the pricing page.

**Node SDK.** The package is `@trycourier/courier` and needs Node 20+ ([github.com/trycourier/courier-node](https://github.com/trycourier/courier-node)).
- You send with `client.send.message({ message: { to: { user_id }, content: { title, body } } })`.
- Courier advises creating the profile before sending to a `user_id`.
- The SDK retries **2 times** with exponential backoff on connection errors, 408, 409, 429 and 5xx. The timeout is 60 seconds.
- It supports idempotency keys, and "the SDK sets the `Idempotency-Key` header for you".
- Source: [courier.com/docs/sdk-libraries/node](https://www.courier.com/docs/sdk-libraries/node)

**Browser feed auth.**
- Your server calls `POST /auth/issue-token` (or `client.auth.issueToken`) with scope `user_id:<id> inbox:read:messages inbox:write:events` and an `expires_in` value. Tokens without `expires_in` never expire.
- The client calls `signIn({ userId, jwt })`.
- The docs warn: read the user ID "from your backend's own authenticated session, never from the request body".
- Source: [courier.com/docs/platform/inbox/authentication](https://www.courier.com/docs/platform/inbox/authentication)

**Real-time.** Websocket. `inbox.listenForUpdates()` "opens the WebSocket connection required for real-time updates" ([courier-react-web](https://www.courier.com/docs/sdk-libraries/courier-react-web)).

**Customisability.** `@trycourier/courier-react` v9 (React 18+) provides `<CourierInbox />` and `<CourierInboxPopupMenu />` ([inbox overview](https://www.courier.com/docs/platform/inbox/inbox-overview)).
- **Themes:** `CourierInboxTheme` objects.
- **Render props:** `renderListItem` and others.
- **Headless:** `useCourier()`.
- Sources: [v8 migration guide](https://www.courier.com/docs/sdk-libraries/courier-react-v8-migration-guide), [courier-react-web](https://www.courier.com/docs/sdk-libraries/courier-react-web)
- The checked pages do not say whether the components use Shadow DOM, which affects Tailwind (not verified).

**Read/unread.** The Inbox shows unread indicators and a badge. The `inbox:write:events` scope lets the user mark messages read, opened or archived. ([authentication](https://www.courier.com/docs/platform/inbox/authentication))

**Testing.** Each workspace has several environments with their own keys, and the docs advise starting in Test ([docs overview](https://www.courier.com/docs/overview)). No mock mode was found.

---

## 5. SuprSend

**Free tier.** $0 with 10,000 notifications per month, no credit card required ([suprsend.com/pricing](https://www.suprsend.com/pricing)).
- The in-app feed is included **with SuprSend branding**.
- Logs are kept for 30 days.
- Over the limit, SuprSend keeps processing and contacts you.

**Node SDK.** The package is `@suprsend/node-sdk` ([node trigger docs](https://docs.suprsend.com/docs/node-trigger-workflow-from-api)).
- You send with `new Suprsend(key, secret).workflows.trigger(new WorkflowTriggerRequest(body, { idempotency_key }))`. A successful trigger returns 202.
- Inline recipients (`distinct_id`) "will be automatically created in the SuprSend database". Set `$skip_create` to prevent that.
- Repeated idempotency keys are skipped for 24 hours. Keys can be up to 255 characters.
- Retries are not documented.

**Browser feed auth.**
- `SuprSendProvider` takes `publicApiKey`, `distinctId`, `userToken` and `refreshUserToken`.
- In Enhanced Security Mode, the server signs an **ES256** JWT with claims `entity_type: "subscriber"`, `entity_id: distinct_id`, `iat` and `exp`.
- The signing key is "shown once, never retrievable".
- Enhanced Security Mode is off in Sandbox, where "a malformed or wrongly signed token still loads the inbox".
- Sources: [inbox quick start](https://docs.suprsend.com/docs/inbox-quick-start), [react-sdk](https://docs.suprsend.com/docs/react-sdk)

**Real-time.** Socket-based. `initializeSocketConnection` is used, and reachability tracks socket status (`CONNECTING`, `DEGRADED`, `ONLINE`). ([headless inbox](https://docs.suprsend.com/docs/headless-inbox), [react-sdk](https://docs.suprsend.com/docs/react-sdk))

**Customisability.**
- **Drop-in:** `@suprsend/react` provides `Inbox` and `NotificationsFeed`, styled through a `theme` prop of `React.CSSProperties` objects ([customisation](https://docs.suprsend.com/docs/web-components-customisations)). This suits Tailwind less well, because you style with CSS objects, not class strings.
- **Headless:** `@suprsend/react-core` provides `useFeedData`, `useFeedClient` and `useFeed` ([headless inbox](https://docs.suprsend.com/docs/headless-inbox)).

**Read/unread.** `markAsRead(id)` and `markAsSeen(id)`. Other methods are on the JS feed reference. ([headless inbox](https://docs.suprsend.com/docs/headless-inbox))

**Testing.** There is a Sandbox workspace ([inbox quick start](https://docs.suprsend.com/docs/inbox-quick-start)). No mock mode was found.

---

## 6. Others considered

**Pingram (formerly NotificationAPI)**
- The React package is `@notificationapi/react` (`NotificationAPIProvider`, `NotificationPopup`), with a headless option in `@notificationapi/core` ([docs.notificationapi.com/components/inapp](https://docs.notificationapi.com/components/inapp)).
- `notificationapi.com/pricing` now 301-redirects to [pingram.io/pricing](https://www.pingram.io/pricing). That page lists 3,000 emails, 100 SMS and 100 call minutes free with no card, but **no in-app quota**.
- Pingram's own `llms.txt` says "3,000 emails/in-app".
- The free in-app allowance is ambiguous, so it is not verified. It drops out on constraint (1) uncertainty.

**Engagespot**
- The official pricing page (`engagespot.co/pricing`) returned 404 when checked.
- Third-party aggregators report 10k triggers/month free, 7-day feed retention and a real-time inbox. None of this could be verified from a primary source, so Engagespot is excluded.

---

## Comparison

| | Knock | Novu | MagicBell | Courier | SuprSend |
|---|---|---|---|---|---|
| Free volume / card | 10k msgs/mo, no card | 10k runs/mo, no card | 1k deliveries/mo, card not stated | 10k sends/mo, card not stated on pricing page | 10k/mo, no card |
| Free-tier caveats | soft overage | 5 workflows, 24h activity retention, branding | very low volume | test sends count | branding |
| Node package | `@knocklabs/node` | `@novu/api` (beta) | `magicbell-js` | `@trycourier/courier` | `@suprsend/node-sdk` |
| Inline identify on trigger | yes | yes (upsert) | externalId/email | create profile first (advised) | yes (auto-create) |
| Browser auth (prod) | RS256 JWT, `signUserToken` helper | HMAC-SHA256 of subscriberId | HS256 JWT | server-issued scoped JWT via API call | ES256 JWT |
| Unauthenticated dev mode | yes (enhanced security off) | yes (HMAC off) | no | no | yes (Sandbox) |
| Real-time | websocket | websocket | persistent connection (transport not named) | websocket | socket |
| Headless hooks | yes | yes | not documented | `useCourier` | yes (`react-core`) |
| Tailwind fit | CSS vars / headless | Tailwind classes in `appearance` + render props | CSS vars | theme objects / render props | CSSProperties theme / headless |
| Mark read API | read/seen/archived + reverses | read/archive (+unread) | UI only (API not verified) | via events scope | read/seen |
| SDK auto-retry | 2x, documented | configurable, defaults undocumented | not documented | 2x, documented | not documented |
| Idempotency key | header, 24h | 2nd arg, 24h (not all orgs) | not documented | SDK sets header | option, 24h |
| Test support | Dev/Prod envs; in-app has no sandbox | Dev/Prod; injectable `fetcher` | none found | Test env | Sandbox |

## Recommendation

**Pick Knock. Runner-up: Novu.**

Why Knock:
- **Constraints met:**
  - (1) 10k messages/month with no card and only a soft overage, which is ample for a course project.
  - (2) A mature, typed `@knocklabs/node` SDK with **documented** retries (2x, backoff) and a 24h `Idempotency-Key`. That is the strongest trigger-time failure story of the five.
  - (3) `@knocklabs/react` ships both styled feed components and headless hooks. You can start with the popover and later render the feed with shadcn `Popover`/`ScrollArea` markup.
- Users are identified inline on trigger, so there is no sync step from SQLite.
- `signUserToken` turns enhanced security into a one-line loader call.
- The read, seen and archived APIs are the most complete of the five.

Why Novu is the runner-up:
- It has the nicest Tailwind story: class strings in `appearance.elements`, plus `renderNotification`.
- Its feed auth is the simplest: one HMAC call.
- It loses on three points: the beta SDK ("breaking changes ... without a major version"), undocumented retry defaults, and idempotency that is "not enabled for all organizations". The 5-workflow cap and 24h activity retention make debugging harder.

## Implications for open questions

**Real-time vs refresh.**
- Every viable candidate pushes updates over a socket, so the feed can be real-time with no polling or loader revalidation.
- With Knock, the styled feed connects automatically. In headless mode, call `listenForUpdates()` / `fetch()` on the client.
- The feed should be client-only. Render it after hydration so the socket and SDK do not run during React Router SSR. This is a general SSR concern, not a documented Knock requirement.
- Badge count: decide between unseen (the default) and unread (`badgeCountType="unread"`).

**Failure handling at trigger time.**
- The SDK covers short blips: 2 retries on 5xx/429/connection errors, with a default 60s timeout. Lower `timeout` and `maxRetries` so a slow provider cannot hold up an action.
- Pass a deterministic `Idempotency-Key`, for example `lesson-published:<lessonId>:<userId>`, so retries never double-post (24h window).
- Decide whether a failed trigger should fail the user's action. The likely answer is no: catch, log and continue.
- For guaranteed delivery you would need an outbox table in SQLite, which is probably out of scope for a learning project.

**Identity mapping.**
- Use the Cadence user ID, stringified, as the Knock user `id`. Pass `{ id, name, email }` inline on trigger, so dev users seeded in SQLite get created in Knock the first time they are notified.
- In the browser, the root loader returns `{ knockUserId, knockToken }` for the **session** user. Mint the token with `signUserToken(userId)` on the server, never from a client-supplied ID; Courier's docs give the same warning.
- When the DevUI panel switches users, the session changes and the loader re-runs. Key `KnockProvider` on the user ID (`key={userId}`) so it tears down and reconnects the socket for the new user. (Knock documents that `logout()` tears down the socket.)
- Keep enhanced security off in the Development environment for convenience if you like, but turn it on in Production.

**Testing triggers in `npm run test`.**
- No provider offers an official mock mode, and Knock's in-app channel does not support sandbox mode. Tests must not call the real service.
- Put the SDK behind a small `notifications` module, for example `notify(event, recipients, data)`, and inject or `vi.mock` it. Then assert on the calls: workflow key, recipient IDs, payload and idempotency key.
- Test the failure path by making the fake throw, and assert that the user-facing action still succeeds.
- Use the Knock Development environment only for manual end-to-end checks in the running app.
