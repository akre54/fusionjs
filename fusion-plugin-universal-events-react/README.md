# fusion-plugin-universal-events-react

[![Build status](https://badge.buildkite.com/7a82192275779f6a8ba81f7d4a1b0d294256838faa1dfdf080.svg?branch=master)](https://buildkite.com/uberopensource/fusionjs)

The `fusion-plugin-universal-events-react` is a drop-in replacement for `UniversalEventsToken` from the [`fusion-plugin-universal-events`](https://github.com/fusionjs/fusionjs/tree/master/fusion-plugin-universal-events). It can be used as a dependency by other Fusion.js plugins and works as an event emitter for data such as statistics and analytics. This plugin captures events emitted from the client, sends them in batches to the server periodically, and allows the server to handle them. Note that due to the batched and fire-and-forget nature of the client-to-server event emission, this library is not suitable for timing-sensitive requests such as error logging or RPC calls.

This plugin also installs a React provider component at the top of the React tree and exports a higher order component that exposes the event emitter to component props.

This plugin is useful for when you want to collect data about user actions or other metrics, and send them in bulk to the server to minimize the number of HTTP requests.

For convenience, this plugin automatically flushes its queue on page unload.

---

### Table of contents

* [Installation](#installation)
* [Usage](#usage)
* [Setup](#setup)
* [API](#api)
  * [Registration API](#registration-api)
    * [`UniversalEvents`](#universalevents)
  * [Dependencies](#dependencies)
    * [`FetchToken`](#fetchtoken)
  * [Service API](#service-api)
* [Other examples](#other-examples)

---

### Installation

```sh
yarn add fusion-plugin-universal-events-react
```

---

### Usage

```js
import {withBatchEvents} from 'fusion-plugin-universal-events-react';

const Component = ({universalEvents}) => {
  universalEvents.on('foo', payload => {
    console.log(payload);
  });
};

export default withBatchEvents(Component);
```

---

### Setup

```js
// main.js
import React from 'react';
import App from 'fusion-react';
import UniversalEvents, {
  UniversalEventsToken,
} from 'fusion-plugin-universal-events-react';
import {FetchToken} from 'fusion-tokens';

export default function() {
  const app = new App();
  app.register(UniversalEventsToken, UniversalEvents);
  __BROWSER__ && app.register(FetchToken, window.fetch);
  return app;
}
```

---
### API

#### Registration API

##### `UniversalEvents`

```js
import UniversalEvents from 'fusion-plugin-universal-events';
```

The plugin. Provides the [service API](#service-api). Typically should be registered to `UniversalEventsToken`.

#### Dependencies

##### `FetchToken`

```js
import {FetchToken} from 'fusion-tokens';
// ...
__BROWSER__ && app.register(FetchToken, window.fetch);
```

**Required. Browser-only.** See [https://github.com/fusionjs/fusionjs/tree/master/fusion-tokens#fetchtoken](https://github.com/fusionjs/fusionjs/tree/master/fusion-tokens#fetchtoken)

---

#### Service API

##### `events.on`

Registers a callback to be called when an event of a type is emitted. Note that the callback will not be called if the event is emitted before the callback is registered.

```js
events.on((type: string), (callback: (payload: Object, ctx: ?Context) => void));
```

* `type: string` - Required. The type of event to listen on. The type `*` denotes all events.
* `callback: (mappedPayload: Object, ctx: ?Context) => void` - Required. Runs when an event of matching type occurs. Receives the `payload` after it has been transformed by [mapper functions](#eventsmap), as well an optional ctx object.

##### `events.emit`

```js
events.emit((type: string), (payload: Object));
```

* `type: string` - Required. The type of event to emit. The type `*` denotes all events.
* `payload: Object` - Optional. Data to be passed to event handlers

##### `events.map`

Mutates the payload. Useful if you need to modify the payload to include metrics or other meta data.

```js
events.map(
  (type: string),
  (callback: (payload: Object, ctx: ?Context) => Object)
);
```

* `type: string` - Required. The type of event to listen on. The type `*` denotes all events.
* `callback: (payload: Object, ctx: ?Context) => Object` - Required. Runs when an event of matching type occurs. Should return a modified `payload`

##### `events.flush`

Flushes the data queue to the server immediately. Does not affect flush frequency

```js
events.flush();
```

##### `events.setFrequency`

```js
events.setFrequency((frequency: number));
```

Sets the frequency at which data is flushed to the server. Resets the interval timer.

* `frequency: number` - Required.

##### `events.teardown`

```js
events.teardown();
```

Stops the interval timer, clears the data queue and prevents any further data from being flushed to the server. Useful for testing

##### `events.from`

```js
const scoped = events.from((ctx: Context));
```

Returns a scoped version of the events api.

* `ctx: Context` - A [Fusion.js context](https://github.com/fusionjs/fusionjs/tree/master/fusion-core#context)

#### `withBatchEvents`

```js
import {withBatchEvents} from 'fusion-plugin-universal-events-react';

const Component = ({universalEvents}) => {
  universalEvents.on('foo', payload => {
    console.log(payload);
  });
};

export default withBatchEvents(Component);
```

---

### Other examples

#### Event transformation

It's possible to transform event data with a mapping function, for example to attach a timestamp to all actions of a type.

```js
events.map('user-action', payload => {
  return {...payload, time: new Date().getTime()};
});

events.on('user-action', payload => {
  console.log(payload); // logs {type: 'click', time: someTimestamp}
});

events.emit('user-action', {type: 'click'});
```

#### Accessing `ctx`

Event mappers and handlers take an optional second parameter `ctx`. For example:

```js
events.on('type', (payload, ctx) => {
  //...
});
```

This parameter will be present when events are emitted from the `ctx` scoped EventsEmitter instance. For example:

```js
app.middleware({events: UniversalEventsToken}, ({events}) => {
  events.on('some-scoped-event', (payload, ctx) => {});
  return (ctx, next) => {
    const scoped = events.from(ctx);
    scoped.emit('some-scoped-event', {some: 'payload'});
  };
});
```

#### \* event type

`*` is a special event type which denotes all events. This allows you to add a mapper or handler to all events. For example:

```js
events.map('*', payload => {
  //
});
```
