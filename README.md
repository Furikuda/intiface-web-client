# Intiface Web Client

This simple, entirely vibecoded Python server acts as a **client** for [Intiface Central](https://github.com/intiface/intiface-central) application, which is the thing that connects to your device, and listen for clients to provide the settings.

It acts as a browser "bridge", so a second person can drive the toys.

<img src="/doc/schema.png">

## How it works

1. You need a version of the Intiface Central app that implements the "Client Mode". Check out [my branch](https://github.com/Furikuda/intiface-central/tree/client-mode).
2. On the app, you setup your Bluetooth device
<img src="/doc/sc1.jpg" width="20%" height="20%">
4. Once it's done, you provide the URL of where your Intiface Web Client instance runs (I know it's confusing that this server is a client)
<img src="/doc/sc2.jpg" width="20%" height="20%">
3. If everything works, the App will generate a **session ID** (a 4-word passphrase), shows it, and lets you copy paste it for easy sharing:
<img src="/doc/sc3.jpg" width="20%" height="20%">
3. When someone opens the link you've just copied (or just goes to the base URL you've set up earlier), say `https://example.com/intiface/`, they need to specify **their name + the session ID**.
<img src="/doc/sc4.jpg" width="20%" height="20%">
4. And if everything goes well, they can now see sliders and remotely control your toys
<img src="/doc/sc5.jpg" width="20%" height="20%">
5. lol

### Known limitations (by design)

A Buttplug connection allows exactly one handshake and the app's engine stops if the connection drops (no auto-reconnect). So:

- **One controller per app session.**
- A **wrong/typo'd session ID consumes the handshake** — the app must restart Client  Mode. (The form is rate limited to blunt guessing.)

No real auth, no audit, no warranty — **test-only**.

## Setup

```bash
pip install -r requirements.txt
```

## Run

You can specify on which port to run in
```bash
WEB_PORT=8080 python3 webclient.py
```
A systemd service available [here]().

## Reverse proxy / base path (recommended)

The app's Client Mode takes a single **Server URL** like `https://domain.com/intiface` and derives both endpoints from it:

- engine websocket (to talk to the app) → `wss://domain.com/intiface/ws`
- control link → `https://domain.com/intiface/?session=<id>` (opens the form prefilled)

So put both listeners behind one host:port, split by path. The browser control socket lives at `<base>/socket/<id>` (deliberately **not** under `/ws`) so the proxy can route `<base>/ws` to the app websocket without catching it. 

Example nginx config:

```nginx
location = /intiface/ws {                 # app (engine) websocket -> :8765
  proxy_pass http://127.0.0.1:8765/;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
}
location /intiface/ {                      # web UI + control socket -> :80
  proxy_pass http://127.0.0.1:80/;         # trailing slash strips the base path
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;   # control socket at /intiface/socket/<id>
  proxy_set_header Connection "upgrade";
}
```

Because the app derives both endpoints from one host:port, the two listeners must share one — so even local testing wants a tiny proxy. A minimal Caddyfile (root base path) (untested):

```
:8443 {
  handle /ws        { reverse_proxy 127.0.0.1:8765 }   # app websocket
  handle            { reverse_proxy 127.0.0.1:8080 }   # web UI + /socket/<id>
}
```

Then set the app's Server URL to `http://localhost:8443` (engine → `ws://localhost:8443/ws`,
link → `http://localhost:8443/?session=<id>`).

## Tests

```bash
pip install -r requirements-dev.txt
cd webcient/ ; pytest
```

Unit tests live in `tests/` and cover the protocol builders, session registry,
the Buttplug bridge (handshake/verify, command relay, disconnect handling), and the web routes/control websocket.

## Files

- `client.py` — entrypoint, starts both listeners.
- `buttplug_bridge.py` — app-facing Buttplug client (deferred handshake, device relay).
- `buttplug_protocol.py` — minimal Buttplug v3 message builders.
- `sessions.py` — in-memory session registry.
- `web.py` — aiohttp routes + control websocket.
- `templates/` — the form and control pages.
