# PB FPS Online — Standalone Multiplayer FPS

Browser-based multiplayer FPS (Point Blank-style), standalone Node.js + WebSocket server with a Three.js client. Deployable to Render free tier.

## Play

- Open the deployed URL, enter a soldier name, then **Create Room** (pick map, mode, team, character, weapon) or **Join Room** with a 4-letter code.
- Share the room code with friends — they join from the same URL.
- Desktop: WASD move, mouse aim (pointer lock), click shoot, R reload, Shift run, Tab scoreboard, Enter chat.
- Mobile: left joystick, drag right side to look, FIRE / JUMP / RLD buttons. Auto-landscape in rooms (MLBB-style forced landscape).

## Features

- Rooms with 4-letter codes; Team Deathmatch (8v8, first to 50) and FFA (first to 15); 5-minute matches.
- 6 maps: Downtown, Luxville, Midtown, Burning Hall, Safe House, Crackdown.
- 8 characters (CT-Force / Free Rebels), 6 weapons (K-5, AK-47, M4A1, MP5, AWP, M1887) with server-side damage.
- Bots fill teams to 8v8; text chat; WebRTC voice chat; ID/EN language toggle.

## Run locally

```bash
npm install
node server.js
# open http://localhost:3000
```

## Deploy to Render (free tier)

- Create a **Web Service** from this repo, environment **Node**, plan **Free**.
- Build command: `npm install`
- Start command: `node server.js`
- `render.yaml` in this repo declares the same settings.

## Protocol

See [PROTOCOL.md](PROTOCOL.md) for the WebSocket message protocol, maps, weapons and bot rules.
