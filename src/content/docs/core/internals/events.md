---
title: The event bus
description: How the event bus works, how to fire your own events, and how to listen for events without leaking listeners.
sidebarGroup: The core
order: 2
sidebarLabel: Events
---

This page explains how the event bus works underneath. To fire events from your integration, or to react to what happens in Home Assistant with the event helpers, see [events in integrations](/core/integrations/events/).

Home Assistant is driven by events. A state changes: that is an event. An action is called: event. Home Assistant starts or stops: event. The event bus on `hass.bus` carries all of them, and any code can listen.

In practice, you rarely listen to the raw bus. Home Assistant has [helpers](/core/integrations/events/#listen-to-home-assistant) for the common cases, like tracking a state change or running something every minute. Use those when they fit. This page explains what they are built on.

## What an event is

An event is an `Event` object from `homeassistant.core` with:

- `event_type`: a string, at most 64 characters. Custom events are prefixed with the domain of your integration, like `my_integration_button_pressed`.
- `data`: a dictionary. Keep it JSON serializable, because events are stored by the recorder and sent over the WebSocket API.
- `context`: the `Context` that caused the event. It links the event to the user, automation, or action that started it.
- `origin`: `EventOrigin.local` for events from this instance, `EventOrigin.remote` for events that came in from outside, like through the REST API.
- `time_fired`: when the event was fired.

Home Assistant itself fires events like `state_changed`, `call_service`, `homeassistant_start`, `homeassistant_started`, and `homeassistant_stop`. The constants live in `homeassistant/const.py`, like `EVENT_STATE_CHANGED`. The [event reference on home-assistant.io](https://www.home-assistant.io/docs/configuration/events/) lists the events users see.

> [!IMPORTANT]
> Events are recorded in the database. Think about the size and shape of your event data before you fire events often. The [data science portal](https://data.home-assistant.io/docs/events/) explains how events are stored.

## Fire an event

From the event loop, use `async_fire`:

```python
from homeassistant.core import HomeAssistant

DOMAIN = "my_integration"


async def async_press(hass: HomeAssistant, button_id: str) -> None:
    """Tell the rest of Home Assistant a button was pressed."""
    hass.bus.async_fire(
        f"{DOMAIN}_button_pressed",
        {"button_id": button_id},
    )
```

From another thread, use `hass.bus.fire` instead. It schedules the event on the event loop for you. See [thread safety](/core/internals/thread-safety/).

When you fire an event because a user did something, pass their `context`, so the logbook and the permissions system can attribute it. Inside an action handler, that is `call.context`.

Integrations that represent a physical button or remote often want an [event entity](/core/entities/event/) instead of a raw event. See [make device events available](/core/integrations/events/#make-device-events-available).

## Listen for events

`async_listen` registers a listener and returns a function that removes it. Always keep that function and call it when you are done, or your listener lives forever.

```python
from homeassistant.config_entries import ConfigEntry
from homeassistant.const import EVENT_HOMEASSISTANT_STOP
from homeassistant.core import Event, HomeAssistant, callback


async def async_setup_entry(
    hass: HomeAssistant, entry: ConfigEntry
) -> None:
    """Set up my integration."""

    @callback
    def handle_stop(event: Event) -> None:
        """Close the connection when Home Assistant stops."""
        entry.runtime_data.close()

    entry.async_on_unload(
        hass.bus.async_listen_once(EVENT_HOMEASSISTANT_STOP, handle_stop)
    )
```

`entry.async_on_unload` calls the remove function when the config entry is unloaded. In an entity, use `self.async_on_remove` the same way.

A few things to know:

- **How your listener runs depends on what it is.** A function decorated with `@callback` runs directly in the event loop. A coroutine function runs as a task in the event loop. A plain function without `@callback` runs in a thread of the executor. Decorate small, non-blocking listeners with `@callback`: it is the fastest option.
- **Listen to one event type.** You can pass `MATCH_ALL` to get every event, but that is expensive and rarely what you want.
- **Filter early.** `async_listen` takes an optional `event_filter`: a `@callback` function that gets the event data and returns `True` if the listener should run. It runs before your listener is scheduled, which saves work on a busy bus.
- **`async_listen_once`** removes the listener after the first event.

To type the event data, core uses `EventType` from `homeassistant.util.event_type`. `EVENT_STATE_CHANGED` is an `EventType[EventStateChangedData]`, so a listener gets `Event[EventStateChangedData]` and a type checker knows what is in `event.data`.

## Helpers for common events

`homeassistant.helpers.event` builds on `async_listen` with helpers for what you usually want, like following the state of a few entities or running something at a time of day. They filter for you, and return a function to remove the listener, just like `async_listen`. The full list, with examples, is on [events in integrations](/core/integrations/events/#listen-to-home-assistant).

## Events or the dispatcher

Events go over the bus, get recorded, and can be seen by anyone, including automations and the WebSocket API. For signals that only matter inside your integration, like "new data arrived for device 12", use the dispatcher in `homeassistant.helpers.dispatcher` instead. `async_dispatcher_send` and `async_dispatcher_connect` work like events, but stay in memory and are not recorded.
