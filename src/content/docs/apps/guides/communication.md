---
title: Talk to Home Assistant and the Supervisor
description: Reach Home Assistant Core, the Supervisor, and other apps from your app, over the internal network and through the API proxies.
order: 5
---

Your app runs on an internal network with the Supervisor and the other apps, and it gets a token to call their APIs. This guide covers the ways to talk to each of them, and which permission you need for each.

## The internal network

Every app reaches the others by host name. The Supervisor registers each app in the internal DNS as its app ID with `_` replaced by `-`:

| Host name | What it is |
| --- | --- |
| `supervisor` | The Supervisor and its API, at `172.30.32.2` |
| `homeassistant` | Home Assistant Core |
| `core-mosquitto` | The official Mosquitto broker app (`core_mosquitto`) |
| `local-hello-world` | Your local app `hello_world` |
| `a0d7b954-adguard` | An app from the community repository |

The app ID starts with the repository ID: `local` for local apps, `core` for the official apps, and a hash of the URL for other repositories. [How apps work](/apps/how-apps-work/#names-and-ids) explains how it is made. The `GET /addons` endpoint of the Supervisor API lists the `slug` and `repository` of every installed app.

## Your token

The Supervisor gives every app its own token, in the environment variable `SUPERVISOR_TOKEN`. Use it for the Supervisor API and for the Home Assistant Core API proxy, in an `Authorization: Bearer` header. What the token may do depends on the permissions in your `config.yaml`.

Remember that scripts started by S6 overlay need `#!/command/with-contenv bashio` (or `with-contenv bash`) to see this variable. See [the container environment](/apps/guides/s6-overlay/#the-container-environment).

## Home Assistant Core

The easiest way to call Home Assistant is through the proxy of the Supervisor. You do not need to know the port, whether SSL is on, or a user's access token. Add `homeassistant_api: true` to `config.yaml`, and use your `SUPERVISOR_TOKEN`:

```bash
curl -sSL \
  -H "Authorization: Bearer ${SUPERVISOR_TOKEN}" \
  -H "Content-Type: application/json" \
  http://supervisor/core/api/config
```

`http://supervisor/core/api/` maps to the REST API of Home Assistant, `/api/` on Core. The event stream at `/core/api/stream` works the same way.

The WebSocket API is at `ws://supervisor/core/websocket`. Send your `SUPERVISOR_TOKEN` as `access_token` in the auth message, like you would a normal access token:

```json
{ "type": "auth", "access_token": "<your SUPERVISOR_TOKEN>" }
```

The proxy does not forward commands of the `supervisor/` and `hassio/` types, and blocks the `hassio` endpoints of the REST API. Those are for the Home Assistant frontend only.

> [!WARNING]
> Through the proxy, your app acts as an administrator of Home Assistant. Everything else an administrator can do, your app can do too, including managing users and their credentials. Only add `homeassistant_api: true` when your app needs Home Assistant, and treat your `SUPERVISOR_TOKEN` as an administrator credential.

You can also connect to Home Assistant directly at `homeassistant` on port 8123, but then you need a user's access token and you have to know how the user configured Home Assistant. The proxy is almost always the better choice.

## The Supervisor API

The Supervisor API is at `http://supervisor/`, with the same token:

```bash
curl -sSL \
  -H "Authorization: Bearer ${SUPERVISOR_TOKEN}" \
  http://supervisor/addons/self/info
```

Every app may call a few endpoints without any extra permission:

- `/addons/self/...`: information about your own app, its options, logs, and stats. You can also start, stop, and restart it. Changing its security settings or updating it is not part of this: that needs `hassio_api` and a role that allows it.
- `/info`: general information about the system.
- `/services` and `/discovery`: [services and discovery](/apps/guides/services-discovery/).
- `/auth`: user authentication, with `auth_api: true`.
- `/supervisor/ping`: needs no token at all.

For anything else, add `hassio_api: true`. What you may call then depends on `hassio_role`: the `default` role can read every `info` endpoint, and the other roles add more. The roles are in [Secure your app](/apps/guides/security/#api-roles), the endpoints in the [Supervisor API reference](/apps/supervisor-api/).

With Bashio, most of these calls are a single function, and Bashio adds the token for you:

```bash
#!/command/with-contenv bashio

hostname=$(bashio::info.hostname)
port=$(bashio::app.ingress_port)

# An endpoint without a function of its own. Bashio returns the "data" part.
info=$(bashio::api.supervisor GET /addons/self/info)
```

Every function is in the [Bashio reference](/apps/bashio/).

## Other apps

Apps talk to each other over the internal network like any other network service: use the host name of the other app and the port it listens on. There is no permission involved; whatever the other app exposes inside the network is reachable.

For a few common services, the Supervisor can hand you the connection details, so the user does not need to type them. See [Share services and discover integrations](/apps/guides/services-discovery/).

## Send input to your app

With `stdin: true` in `config.yaml`, Home Assistant can write to the standard input of your app with the `hassio.app_stdin` action:

```yaml
actions:
  - action: hassio.app_stdin
    data:
      app: local_hello_world
      input: "restart"
```

Home Assistant encodes `input` as JSON before it sends it, so the text `restart` arrives as `"restart"`, quotes included, and a mapping arrives as a JSON object. The Supervisor writes the data to the standard input of your app as it is.
