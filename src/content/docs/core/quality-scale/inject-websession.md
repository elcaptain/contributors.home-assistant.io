---
title: Pass in the web session
description: "Platinum rule inject-websession: the library accepts the web session of Home Assistant."
sidebarLabel: inject-websession
sidebarGroup: Platinum
order: 2
---

This is a Platinum rule. In `quality_scale.yaml`, it is called `inject-websession`.

The library of the integration must accept an HTTP session from the outside, and the integration must pass in the session that Home Assistant provides. Home Assistant has shared sessions for [aiohttp](https://docs.aiohttp.org/) and [HTTPX](https://www.python-httpx.org/), so the library uses one of those two.

## Why it matters

Many integrations talk HTTP. If each library opens its own session, Home Assistant ends up with many connection pools, each with their own connections and SSL setup. A shared session reuses connections between integrations, and uses less memory and fewer sockets.

## Example

Pass the shared aiohttp session to the client in `async_setup_entry`:

```python
from homeassistant.helpers.aiohttp_client import async_get_clientsession


async def async_setup_entry(
    hass: HomeAssistant, entry: MyConfigEntry
) -> None:
    """Set up My integration from a config entry."""
    session = async_get_clientsession(hass)
    client = MyClient(entry.data[CONF_HOST], session=session)
    ...
```

For HTTPX, use `get_async_client` from `homeassistant.helpers.httpx_client`.

Sometimes a shared session is not what you want, for example when the library uses cookies that should not leak to other integrations. Then create a session of your own that Home Assistant still manages: `async_create_clientsession` for aiohttp, or `create_async_httpx_client` for HTTPX.

## How it is checked

This rule is checked in review.

## Exceptions

If the integration makes no HTTP requests, for example because it talks over Modbus, Bluetooth, or a raw TCP socket, this rule does not apply.

## Related rules

- [async-dependency](/core/quality-scale/async-dependency/): use an async dependency.
