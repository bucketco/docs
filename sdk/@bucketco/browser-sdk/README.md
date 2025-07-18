---
layout:
  visible: true
title:
  visible: true
description:
  visible: false
tableOfContents:
  visible: true
outline:
  visible: true
pagination:
  visible: true
---

# Bucket Browser SDK

Basic client for Bucket.co. If you're using React, you'll be better off with the Bucket React SDK.

Bucket supports feature toggling, tracking feature usage, [collecting feedback](#qualitative-feedback) on features, and [remotely configuring features](#remote-config-beta).

## Install

First find your `publishableKey` under [environment settings](https://app.bucket.co/envs/current/settings/app-environments) in Bucket.

The package can be imported or used directly in a HTML script tag:

A. Import module:

```typescript
import { BucketClient } from "@bucketco/browser-sdk";

const user = {
  id: 42,
  role: "manager",
};

const company = {
  id: 99,
  plan: "enterprise",
};

const bucketClient = new BucketClient({ publishableKey, user, company });

await bucketClient.initialize();

const {
  isEnabled,
  config: { payload: question },
  track,
  requestFeedback,
} = bucketClient.getFeature("huddle");

if (isEnabled) {
  // Show feature. When retrieving `isEnabled` the client automatically
  // sends a "check" event for the "huddle" feature which is shown in the
  // Bucket UI.

  // On usage, call `track` to let Bucket know that a user interacted with the feature
  track();

  // The `payload` is a user-supplied JSON in Bucket that is dynamically picked
  // out depending on the user/company.
  const question = payload?.question ?? "Tell us what you think of Huddles";

  // Use `requestFeedback` to create "Send feedback" buttons easily for specific
  // features. This is not related to `track` and you can call them individually.
  requestFeedback({ title: question });
}

// `track` just calls `bucketClient.track(<featureKey>)` to send an event using the same feature key
// You can also use `track` on the client directly to send any custom event.
bucketClient.track("huddle");

// similarly, `requestFeedback` just calls `bucketClient.requestFeedback({featureKey: <featureKey>})`
// which you can also call directly:
bucketClient.requestFeedback({ featureKey: "huddle" });
```

B. Script tag (client-side directly in html)

See [example/browser.html](https://github.com/bucketco/bucket-javascript-sdk/tree/main/packages/browser-sdk/example/browser.html) for a working example:

```html
<script src="https://cdn.jsdelivr.net/npm/@bucketco/browser-sdk@2"></script>
<script>
  const bucket = new BucketBrowserSDK.BucketClient({
    publishableKey: "publishableKey",
    user: { id: "42" },
    company: { id: "1" },
  });

  bucket.initialize().then(() => {
    console.log("Bucket initialized");
    document.getElementById("loading").style.display = "none";
    document.getElementById("start-huddle").style.display = "block";
  });
</script>
<span id="loading">Loading...</span>
<button
  id="start-huddle"
  style="display: none"
  onClick="bucket.track('Started huddle')"
>
  Click me
</button>
```

### Init options

Supply these to the constructor call:

```typescript
type Configuration = {
  logger: console; // by default only logs warn/error, by passing `console` you'll log everything
  apiBaseUrl?: "https://front.bucket.co";
  sseBaseUrl?: "https://livemessaging.bucket.co";
  feedback?: undefined; // See FEEDBACK.md
  enableTracking?: true; // set to `false` to stop sending track events and user/company updates to Bucket servers. Useful when you're impersonating a user
  fallbackFeatures?:
    | string[]
    | Record<string, { key: string; payload: any } | true>; // Enable these features if unable to contact bucket.co. Can be a list of feature keys or a record with configuration values
  timeoutMs?: number; // Timeout for fetching features (default: 5000ms)
  staleWhileRevalidate?: boolean; // Revalidate in the background when cached features turn stale to avoid latency in the UI (default: false)
  staleTimeMs?: number; // at initialization time features are loaded from the cache unless they have gone stale. Defaults to 0 which means the cache is disabled. Increase this in the case of a non-SPA
  expireTimeMs?: number; // In case we're unable to fetch features from Bucket, cached/stale features will be used instead until they expire after `expireTimeMs`. Default is 30 days
  offline?: boolean; // Use the SDK in offline mode. Offline mode is useful during testing and local development
};
```

## Feature toggles

Bucket determines which features are active for a given user/company. The user/company is given in the BucketClient constructor.

If you supply `user` or `company` objects, they must include at least the `id` property otherwise they will be ignored in their entirety.
In addition to the `id`, you must also supply anything additional that you want to be able to evaluate feature targeting rules against.

Attributes cannot be nested (multiple levels) and must be either strings, integers or booleans.
Some attributes are special and used in Bucket UI:

- `name` -- display name for `user`/`company`,
- `email` -- is accepted for `user`s and will be highlighted in the Bucket UI if available,
- `avatar` -- can be provided for both `user` and `company` and should be an URL to an image.

```ts
const bucketClient = new BucketClient({
  publishableKey,
  user: {
    id: "user_123",
    name: "John Doe",
    email: "john@acme.com"
    avatar: "https://example.com/images/udsy6363"
  },
  company: {
    id: "company_123",
    name: "Acme, Inc",
    avatar: "https://example.com/images/31232ds"
  },
});
```

To retrieve features along with their targeting information, use `getFeature(key: string)`:

```ts
const huddle = bucketClient.getFeature("huddle");
// {
//   isEnabled: true,
//   config: { key: "zoom", payload: { ... } },
//   track: () => Promise<Response>
//   requestFeedback: (options: RequestFeedbackData) => void
// }
```

You can use `getFeatures()` to retrieve all enabled features currently.

```ts
const features = bucketClient.getFeatures();
// {
//   huddle: {
//     isEnabled: true,
//     targetingVersion: 42,
//     config: ...
//   }
// }
```

`getFeatures()` is meant to be more low-level than `getFeature()` and it typically used
by down-stream clients, like the React SDK.

Note that accessing `isEnabled` on the object returned by `getFeatures` does not automatically
generate a `check` event, contrary to the `isEnabled` property on the object returned by `getFeature`.

## Remote config

Remote config is a dynamic and flexible approach to configuring feature behavior outside of your app – without needing to re-deploy it.

Similar to `isEnabled`, each feature has a `config` property. This configuration is managed from within Bucket.
It is managed similar to the way access to features is managed, but instead of the binary `isEnabled` you can have
multiple configuration values which are given to different user/companies.

```ts
const features = bucketClient.getFeatures();
// {
//   huddle: {
//     isEnabled: true,
//     targetingVersion: 42,
//     config: {
//       key: "gpt-3.5",
//       payload: { maxTokens: 10000, model: "gpt-3.5-beta1" }
//     }
//   }
// }
```

`key` is mandatory for a config, but if a feature has no config or no config value was matched against the context, the `key` will be `undefined`. Make sure to check against this case when trying to use the configuration in your application. `payload` is an optional JSON value for arbitrary configuration needs.

Just as `isEnabled`, accessing `config` on the object returned by `getFeatures` does not automatically
generate a `check` event, contrary to the `config` property on the object returned by `getFeature`.

## Updating user/company/other context

Attributes given for the user/company/other context in the BucketClient constructor can be updated for use in feature targeting evaluation with the `updateUser()`, `updateCompany()` and `updateOtherContext()` methods.
They return a promise which resolves once the features have been re-evaluated follow the update of the attributes.

The following shows how to let users self-opt-in for a new feature. The feature must have the rule `voiceHuddleOptIn IS true` set in the Bucket UI.

```ts
// toggle opt-in for the voiceHuddle feature:
const { isEnabled } = bucketClient.getFeature("voiceHuddle");
// this toggles the feature on/off. The promise returns once feature targeting has been
// re-evaluated.
await bucketClient.updateUser({ voiceHuddleOptIn: (!isEnabled).toString() });
```

{% hint style="info" %}
{% endhint %}

## Toolbar

The Bucket Toolbar is great for toggling features on/off for yourself to ensure that everything works both when a feature is on and when it's off.

<img width="352" alt="Toolbar screenshot" src="https://github.com/user-attachments/assets/c223df5a-4bd8-49a1-8b4a-ad7001357693" />

The toolbar will automatically appear on `localhost`. However, it can also be incredibly useful in production.
You have full control over when it appears through the `toolbar` configuration option passed to the `BucketClient`.

You can pass a simple boolean to force the toolbar to appear/disappear:

```typescript
const client = new BucketClient({
  // show the toolbar even in production if the user is an internal/admin user
  toolbar: user?.isInternal,
  ...
});
```

You can also configure the position of the toolbar on the screen:

```typescript
const client = new BucketClient({
  toolbar: {
    show: true;
    position: {
      placement: "bottom-left",
      offset: {x: "1rem", y: "1rem"}
    }
  }
  ...
})
```

See [the reference](https://docs.bucket.co/supported-languages/browser-sdk/globals#toolbaroptions) for details.

## Qualitative feedback on beta features

Bucket can collect qualitative feedback from your users in the form of a [Customer Satisfaction Score](https://en.wikipedia.org/wiki/Customer_satisfaction) and a comment.

### Automated feedback collection

The Bucket Browser SDK comes with automated feedback collection mode enabled by default, which lets the Bucket service ask your users for feedback for relevant features just after they've used them.

{% hint style="info" %}
To get started with automatic feedback collection, make sure you've set `user` in the `BucketClient` constructor.
{% endhint %}

Automated feedback surveys work even if you're not using the SDK to send events to Bucket.
It works because the Bucket Browser SDK maintains a live connection to Bucket's servers and can automatically show a feedback prompt whenever the Bucket servers determines that an event should trigger a prompt - regardless of how this event is sent to Bucket.

You can find all the options to make changes to the default behavior in the [Bucket feedback documentation](../../documents/browser-sdk/FEEDBACK.md).

### Bucket feedback UI

Bucket can assist you with collecting your user's feedback by offering a pre-built UI, allowing you to get started with minimal code and effort.

[Read the Bucket feedback UI documentation](../../documents/browser-sdk/FEEDBACK.md)

### Bucket feedback SDK

Feedback can be submitted to Bucket using the SDK:

```ts
bucketClient.feedback({
  featureKey: "my-feature-key", // String (required), copy from Feature feedback tab
  score: 5, // Number: 1-5 (optional)
  comment: "Absolutely stellar work!", // String (optional)
});
```

### Bucket feedback API

If you are not using the Bucket Browser SDK, you can still submit feedback using the HTTP API.

See details in [Feedback HTTP API](https://docs.bucket.co/api/http-api#post-feedback)

## Tracking feature usage

The `track` function lets you send events to Bucket to denote feature usage.
By default Bucket expects event names to align with the feature keys, but
you can customize it as you wish.

```ts
bucketClient.track("huddle", { voiceHuddle: true });
```

## Event listeners

Event listeners allow for capturing various events occurring in the `BucketClient`. This is useful to build integrations with other system or for various debugging purposes. There are 5 kinds of events:

- `check`: Your code used `isEnabled` or `config` for a feature
- `featuresUpdated`: Features were updated. Either because they were loaded as part of initialization or because the user/company updated
- `user`: User information updated (similar to the `identify` call used in tracking terminology)
- `company`: Company information updated (sometimes to the `group` call used in tracking terminology)
- `track`: Track event occurred.

Use the `on()` method to add an event listener to respond to certain events. See the API reference for details on each hook.

```ts
import { BucketClient, CheckEvent, RawFeatures } from "@bucketco/browser-sdk";

const client = new BucketClient({
  // options
});

// or add the hooks after construction:
const unsub = client.on("check", (check: CheckEvent) =>
  console.log(`Check event ${check}`),
);
// use the returned function to unsubscribe, or call `off()` with the same arguments again
unsub();
```

## Zero PII

The Bucket Browser SDK doesn't collect any metadata and HTTP IP addresses are _not_ being stored.

For tracking individual users, we recommend using something like database ID as userId, as it's unique and doesn't include any PII (personal identifiable information). If, however, you're using e.g. email address as userId, but prefer not to send any PII to Bucket, you can hash the sensitive data before sending it to Bucket:

```ts
import bucket from "@bucketco/browser-sdk";
import { sha256 } from "crypto-hash";

bucket.user(await sha256("john_doe"));
```

## Use of cookies

The Bucket Browser SDK uses a couple of cookies to support automated feedback surveys. These cookies are not used for tracking purposes and thus should not need to appear in cookie consent forms.

The two cookies are:

- `bucket-prompt-${userId}`: store the last automated feedback prompt message ID received to avoid repeating surveys
- `bucket-token-${userId}`: caching a token used to connect to Bucket's live messaging infrastructure that is used to deliver automated feedback surveys in real time.

## Upgrading to 3.0 from 2.x

Breaking changes:

- `client.onFeaturesUpdated()` is now replaced by [event listeners](#event-listeners)
- Arguments to the `BucketClient` constructor which were previously under `featureOptions` are now supplied directly in the root.

## TypeScript

Types are bundled together with the library and exposed automatically when importing through a package manager.

## Content Security Policy (CSP)

If you are running with strict Content Security Policies active on your website, you will need to enable these directives in order to use the SDK:

| Directive   | Values                                                             | Reason                                                                                                                                   |
| ----------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| connect-src | [https://front.bucket.co](https://front.bucket.co)                 | Basic functionality`                                                                                                                     |
| connect-src | [https://livemessaging.bucket.co](https://livemessaging.bucket.co) | Server sent events for use in automated feedback surveys, which allows for automatically collecting feedback when a user used a feature. |
| style-src   | 'unsafe-inline'                                                    | The feedback UI is styled with inline styles. Not having this directive results unstyled HTML elements.                                  |

If you are including the Bucket tracking SDK with a `<script>`-tag from `jsdelivr.net` you will also need:

| Directive       | Values                                               | Reason                          |
| --------------- | ---------------------------------------------------- | ------------------------------- |
| script-src-elem | [https://cdn.jsdelivr.net](https://cdn.jsdelivr.net) | Loads the Bucket SDK from a CDN |

## License

> MIT License
> Copyright (c) 2025 Bucket ApS
