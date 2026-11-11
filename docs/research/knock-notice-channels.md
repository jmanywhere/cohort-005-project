# Can Knock cover the bell, the live toast, and browser push?

Sources are Knock's own docs (docs.knock.app). Each claim cites the page that owns it.

Verdict: Knock covers all three for a web app. It does not miss a surface. The bell and the list are Knock components. The live toast is the same in-app feed, painted by the app. Browser push is not a first-party Knock channel; Knock sends it through Firebase Cloud Messaging, and the browser shows it when the page is in the background.

The bell and the toast share one in-app feed channel. Browser push is a separate push channel. One workflow trigger can run both channel steps. [Channels](https://docs.knock.app/concepts/channels), [Designing workflows](https://docs.knock.app/designing-workflows/overview).

## Bell

Covered. Knock renders the bell and the list.

Knock's React SDK ships `NotificationIconButton` (a bell whose badge counts unseen, unread, or all items), `NotificationFeedPopover` (the list in a floating popover), and `NotificationFeed` (the same list full page, with a header and status filter, defaulting each row to `NotificationCell`). [React overview](https://docs.knock.app/in-app-ui/react/overview), [NotificationIconButton](https://docs.knock.app/in-app-ui/react/sdk/components/notification-icon-button), [NotificationFeedPopover](https://docs.knock.app/in-app-ui/react/sdk/components/notification-feed-popover), [NotificationFeed](https://docs.knock.app/in-app-ui/react/sdk/components/notification-feed).

The documented wiring is local React state: the bell's `onClick` toggles `isVisible` on the popover. Opening the list does not navigate or reload. [Feed](https://docs.knock.app/in-app-ui/react/feed).

The feed is a reverse-chronological list of messages Knock already generated for that recipient when a workflow (or broadcast) ran. The client fetches them and can mark them seen, read, interacted, or archived. [Feeds overview](https://docs.knock.app/in-app-ui/feeds/overview). Clients also connect to Knock's websocket and receive new feed messages while the page is open, so the list is real-time, not only the snapshot from first load. [Feeds overview](https://docs.knock.app/in-app-ui/feeds/overview), [Socket behavior](https://docs.knock.app/in-app-ui/feeds/socket-behavior).

What Knock renders inside a cell comes from the in-app template on that channel step:

- The template has a required `markdown_body`, plus an optional `action_url` and `action_buttons`. [Templates](https://docs.knock.app/mapi-reference/templates).
- Three variants: standard (body and optional action URL), single-action (one button), multi-action (primary and secondary buttons). [In-app feeds](https://docs.knock.app/integrations/in-app/feeds).
- The feed API returns that copy as `blocks` on each `FeedItem`: markdown rendered to HTML, plaintext, or a button set. [In-app API](https://docs.knock.app/in-app-ui/api-overview).
- Default feed behavior Knock's component already implements: badge counts unseen items; opening the popover marks items seen; clicking a cell goes to the action URL; "mark all as read" marks the feed read; an "x" archives an item and removes it from the list; a filter switches All / Unread. Button colors are not in the template. They are CSS on the component. [In-app feeds](https://docs.knock.app/integrations/in-app/feeds), [Feed](https://docs.knock.app/in-app-ui/react/feed).
- Feed message content cannot be edited after Knock generates it. Status can change. A correction is archive plus a new trigger. [In-app feeds](https://docs.knock.app/integrations/in-app/feeds).

What the app must build:

- Mount `KnockProvider` (public API key and the current user id) and `KnockFeedProvider` (the in-app feed channel id), then render the bell and the popover. Import `@knocklabs/react/dist/index.css` unless replacing the styles. [Feed](https://docs.knock.app/in-app-ui/react/feed).
- Place the bell in the app's own header. The documented example renders it inside the app layout. [Feed](https://docs.knock.app/in-app-ui/react/feed).
- Before production, enable enhanced security mode. Knock then rejects a public-key request that has no signed user token. The token is a JWT the app's backend signs, and the current React SDK takes it as `userToken` on `KnockProvider`. With enhanced security off, a request with the public key can act as any user. [Security and authentication](https://docs.knock.app/in-app-ui/security-and-authentication), [KnockProvider](https://docs.knock.app/in-app-ui/react/sdk/components/knock-provider). The feed guide still says to pass `userToken` to `KnockFeedProvider`; the provider reference and the security example put it on `KnockProvider`. [Feed](https://docs.knock.app/in-app-ui/react/feed).
- Trigger the workflow for each recipient. [Triggering workflows](https://docs.knock.app/send-notifications/triggering-workflows/api), [Designing workflows](https://docs.knock.app/designing-workflows/overview).
- Author the template in Knock. The app does not own the sentence copy unless it switches to a headless feed and renders `blocks` itself. Headless is supported and is not required for the bell. [React overview](https://docs.knock.app/in-app-ui/react/overview), [Feeds overview](https://docs.knock.app/in-app-ui/feeds/overview).

## Live toast

Covered. Knock does not render the toast. The app does.

Knock has no toast component in `@knocklabs/react`. The toast guide builds one on the feed primitives with `react-hot-toast`. [Toasts](https://docs.knock.app/in-app-ui/react/toasts). Toasts are a use of the in-app feed channel, the same channel type as the bell, not a separate channel and not an in-app guide (guides are banners and modals). [Channels](https://docs.knock.app/concepts/channels).

Delivery onto an already-open page, without a reload:

- The SDK keeps a websocket open. When a new message is sent to the feed channel, Knock pushes an update over that socket. The SDK manages connect, reconnect, and delivery. [Socket behavior](https://docs.knock.app/in-app-ui/feeds/socket-behavior).
- The low-level client starts that socket with `listenForUpdates()`. That call is separate from `fetch()`, which loads the existing page of items. [JavaScript quick start](https://docs.knock.app/in-app-ui/javascript/sdk/quick-start), [FeedClient](https://docs.knock.app/in-app-ui/javascript/sdk/feed-client).
- `items.received.realtime` fires for items that arrive on the socket. `items.received.page` fires for a fetch, including the initial fetch. The toast recipe binds only to `items.received.realtime`, so the backlog loaded with the page is not toasted. [FeedClient](https://docs.knock.app/in-app-ui/javascript/sdk/feed-client), [Toasts](https://docs.knock.app/in-app-ui/react/toasts).
- `useNotifications` owns the feed client's real-time lifecycle (init, updates, cleanup). The toast recipe uses `useKnockFeed` inside `KnockFeedProvider` and only adds the event listener. [useNotifications](https://docs.knock.app/in-app-ui/react/sdk/hooks/use-notifications), [Toasts](https://docs.knock.app/in-app-ui/react/toasts).
- The raw socket event is `new-message`. It does not include the message body. It includes badge-count metadata, and it might be throttled. The React toast handler still receives feed items whose `blocks[0].rendered` is the HTML. [In-app API](https://docs.knock.app/in-app-ui/api-overview), [Toasts](https://docs.knock.app/in-app-ui/react/toasts).
- Disconnecting inactive tabs is off unless `auto_manage_socket_connection` is set. When it is on, an inactive tab drops the socket after a delay (default 2s) and reconnects when the tab is active again. [Feed](https://docs.knock.app/in-app-ui/react/feed).

What Knock renders: the same feed-item HTML (and button blocks) it renders for the bell. [In-app API](https://docs.knock.app/in-app-ui/api-overview).

What the app must build:

- A toaster. The documented one is `react-hot-toast`'s `<Toaster />`, fed by `toast.custom` of `notification.blocks[0].rendered`. Knock does not specify duration or auto-dismiss. The app owns when the toast leaves and that it does not take over the page. [Toasts](https://docs.knock.app/in-app-ui/react/toasts).
- The same providers and user identity as the bell. Point the toaster at the same feed channel id if the toast and the bell are the same Notice. [Toasts](https://docs.knock.app/in-app-ui/react/toasts), [Feed](https://docs.knock.app/in-app-ui/react/feed).

Constraint: the toast recipe optionally calls `feedClient.markAsSeen(items)`. The default bell badge counts unseen items and clears when those items become seen. Marking the toast as seen drops the badge before the person opens the bell. Skip that call if the badge should wait for the list. [Toasts](https://docs.knock.app/in-app-ui/react/toasts), [In-app feeds](https://docs.knock.app/integrations/in-app/feeds).

## Browser push

Covered, through FCM. Knock does not have a first-party web push channel. [Web push with FCM](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm).

The push overview lists Firebase Cloud Messaging as the Android provider and does not list a browser provider. The FCM integration page is the one that says the same provider sends mobile or web push, and the tutorial is the one that owns the browser path. [Push overview](https://docs.knock.app/integrations/push/overview), [FCM](https://docs.knock.app/integrations/push/firebase), [Web push with FCM](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm).

What Knock renders: the push template, which requires a `title` and a `text_body`. Knock sends that as an FCM payload. By default the payload is a `notification` message (a top-level `notification` object, optional `data`). Knock does not paint the operating-system notification. [Push template](https://docs.knock.app/mapi-reference/templates/schemas/push_template), [Web push with FCM](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm).

When the page is not in front: for a `notification` message, the FCM SDK and the browser display it in the background. Knock's examples of background are "not in the current tab" and "window is minimized." A service worker is required. Creating `messaging` in `firebase-messaging-sw.js` is what automatically handles those background notifications. The documented case stops there: the page is in the background and the service worker is registered. [Web push with FCM](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm).

Foreground is the opposite path. A `notification` message is not shown by that automatic background handler while the page is in front. Showing one then is app code: Firebase `onMessage`, then `registration.showNotification`. Leave that handler empty and an open, focused Cadence tab does not get a browser notification from the default payload. [Web push with FCM](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm).

Knock will still attempt the push channel step while the in-app socket is connected. Documented skip reasons are step conditions, a missing channel configuration, a preference opt-out, missing recipient channel data, and an invalid batch, delay, or throttle window. An open socket is not one of them. [Debugging workflows](https://docs.knock.app/send-notifications/debugging-workflows). The background-versus-foreground display above is what lines the default payload up with "show the browser notification when the page is not in front." It does not stop Knock from sending the push.

What the app must build:

- An FCM channel in Knock (Firebase project id and service account JSON), per environment. [FCM](https://docs.knock.app/integrations/push/firebase).
- A service worker at a public path (`firebase-messaging-sw.js` in the tutorial) that initializes Firebase messaging, plus registration of that worker from the page. [Web push with FCM](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm).
- A consent control. The browser shows its own permission prompt only after the app calls `Notification.requestPermission`. Then `getToken` with the VAPID key and the service worker registration. [Web push with FCM](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm).
- Store the token on the Knock recipient as channel data `{ tokens: [...] }` for the FCM channel id (`knock.user.setChannelData`, or inline on the trigger). A push step is skipped when that channel data is missing. `setChannelData` overwrites the stored tokens. [Web push with FCM](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm), [FCM](https://docs.knock.app/integrations/push/firebase), [Setting channel data](https://docs.knock.app/managing-recipients/setting-channel-data).
- A push channel step on the workflow, with its own title and body. The in-app feed step does not send the browser notification. [Channel step](https://docs.knock.app/designing-workflows/channel-step), [Channels](https://docs.knock.app/concepts/channels).

Limits that do not remove the surface:

- FCM web push works in current major browsers, per Knock's tutorial FAQ. The tutorial's sample refuses an iPhone unless the site is installed to the Home Screen. [Web push with FCM](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm).
- A `data` message (payload override with `__strategy__: "replace"` and an empty `notification`) is not shown automatically. The service worker must call `showNotification` from `onBackgroundMessage`. The default `notification` payload is the one that matches "the browser shows it when the app is in the background." [Web push with FCM](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm).
- If background handling is wrong, the browser can show "site updated in the background" instead of the notification. [Web push with FCM](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm).
- Knock does not validate that a stored token still works. Opt-in token deregistration can drop a token after a bounce. Otherwise the app removes it. [Setting channel data](https://docs.knock.app/managing-recipients/setting-channel-data), [FCM](https://docs.knock.app/integrations/push/firebase).

## What this leaves for the notices map

Knock can deliver the bell, the live toast, and browser push. A wider vendor search is not forced by a missing surface.

Cadence still builds the mount, the toaster, and the FCM registration. Cadence still triggers the workflow. The toast and the bell should share one in-app feed channel, and the browser push should be its own FCM channel step using the default notification payload.
