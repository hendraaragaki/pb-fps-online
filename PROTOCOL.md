# PB FPS Online — Standalone Protocol & Shared Constants

Target: Node.js WebSocket server + Three.js browser client, deployable to Render free tier.
Transport: WebSocket, JSON messages. Server is authoritative for HP/damage/scores/bots.
All positions are on a flat ground plane: `x` and `z` (metres), `yaw` (radians, 0 = facing -Z).
Player HP = 100. Respawn delay = 3s. Match length = 300s (5 min).

## Maps (deterministic obstacles, server & client MUST agree)
Maps: Downtown, Luxville, Midtown, Burning Hall, Safe House, Crackdown.
Map size: 64 x 64 (x,z in [-32, 32]). Spawn CT at (-26, 0), FR at (26, 0); FFA spawns spread on a circle radius 24.
Obstacles: seeded PRNG (mulberry32) seeded by sum of char codes of the map name.
Generate 26 axis-aligned boxes: for i in 0..25: w = 2 + rnd()*4, d = 2 + rnd()*4, x = -28 + rnd()*56, z = -28 + rnd()*56. Skip if within 6 of either spawn. Height = 2 + rnd()*3.
Box = {x, z, w, d, h}. Client renders them as textured boxes; server uses them for movement collision (circle-vs-box push-out, radius 0.6) and line-of-sight checks (2D segment vs box).
Map theme (client only): Downtown {sky 0x87b5d6, ground 0x6b6f6b}, Luxville {sky 0xffd9a0, ground 0x8a7a5a}, Midtown {sky 0x9fc3e8, ground 0x5f6b7a}, Burning Hall {sky 0x3a2a2a, ground 0x4a3a32}, Safe House {sky 0x7aa08a, ground 0x5a6b52}, Crackdown {sky 0x2a2f3a, ground 0x3a3f4a}.

## Modes
- `tdm`: Team Deathmatch. Target 50 team kills, 5 min. Teams CT vs FR.
- `ffa`: Deathmatch FFA. Target 15 kills, 5 min. Everyone solo.

## Teams & characters (cosmetic; character also picks a colour)
- CT-Force: Keen Eyes, Acid Pol, Hide, Leopard.
- Free Rebels: Rica, Tarantula, D-Fox, Viper Red.
- In FFA any character allowed. Character colour: Keen Eyes #3a7bd5, Acid Pol #2ecc71, Hide #95a5a6, Leopard #e67e22, Rica #e74c3c, Tarantula #8e44ad, D-Fox #f1c40f, Viper Red #c0392b.
- Team colours: CT #2f7bff, FR #ff3b30.

## Weapons (server-authoritative damage)
- K-5: dmg 25, rpm 600, range 40, auto, mag 30, reload 1.6s
- AK-47: dmg 34, rpm 600, range 60, auto, mag 30, reload 2.2s
- M4A1: dmg 30, rpm 700, range 55, auto, mag 30, reload 2.0s
- MP5: dmg 22, rpm 800, range 35, auto, mag 30, reload 1.8s
- AWP: dmg 100, rpm 45, range 100, bolt, mag 5, reload 3.0s
- M1887: dmg 12 (x8 pellets, close range only), rpm 70, range 15, pump, mag 6, reload 2.8s
Weapon ids (lowercase): k5, ak47, m4a1, mp5, awp, m1887. Default: k5.

## Bots
- Fill each team to 8 in TDM (8v8 max) and to 8 total in FFA. Bots join when a match starts (host's room gets bots) and leave when humans leave (keep total per team at 8 while match is running; never exceed 8/team).
- Bot names: CT bots: Bravo, Charlie, Delta, Echo, Foxtrot, Golf, Hotel. FR bots: Viper, Cobra, Raven, Hawk, Falcon, Wolf, Ghost.
- Bot AI (server tick, 10/s): pick nearest living enemy; move toward them (speed 4 m/s) with simple obstacle push-out; if dist < weapon range and roughly facing (dot > 0.6), shoot with hit chance 0.35 (medium); if no enemy visible, wander to a random waypoint. Bots use weapon k5 or ak47. Respawn 3s like players.
- Bots are in `state` as {id:'bot-<n>', name, team, character, weapon, x, z, yaw, hp, alive, kills, deaths, isBot:true}.

## Client -> Server
- {type:'createRoom', name, map, mode, team, character, weapon}
- {type:'joinRoom', code, name, team, character, weapon}
- {type:'leaveRoom'}
- {type:'startMatch'} (host only)
- {type:'move', x, z, yaw} (throttled by client to ~15/s; server stores it)
- {type:'shoot', targetId} (targetId = player id or bot id, or null for a miss shot). Server validates and broadcasts 'hit'.
- {type:'chat', text}
- {type:'voiceSignal', to, signal} (to = player id, or '*' for all; server relays)
- {type:'updateWeapon', weapon} (lobby or anytime; server stores)
- {type:'updateCharacter', character}
- {type:'updateTeam', team} (lobby only)
- {type:'respawn'}

## Server -> Client
- {type:'roomCreated', code, room} ; {type:'roomJoined', room}
  room = {code, map, mode, hostId, state:'lobby'|'playing'|'ended', timeLeft, scores:{CT,FR}, players:[...], bots:[...]}
  player/bot entry = {id, name, team, character, weapon, x, z, yaw, hp, alive, kills, deaths, isBot}
- {type:'playerJoined', player} ; {type:'playerLeft', id}
- {type:'matchStarted', timeLeft}
- {type:'state', players, bots, timeLeft, scores} (10/s while playing; also sent in lobby at 2/s)
- {type:'hit', shooterId, targetId, targetIsBot, damage, hpLeft, killed}
- {type:'killfeed', killer, victim, weapon, headshot} (headshot=false; server does not track headshots in v1)
- {type:'chat', name, team, text, time}
- {type:'voiceSignal', from, signal}
- {type:'matchEnded', winner, scores} (winner = 'CT' | 'FR' | player name in FFA | 'draw')
- {type:'respawned', id, x, z}
- {type:'error', message}

## Server rules
- Room code: 4 uppercase letters, random, unique among active rooms.
- Host = creator. Only host can startMatch. Match starts only in lobby with >= 1 human.
- On startMatch: fill bots, set state='playing', timeLeft=300, reset kills/scores/hp, place spawns.
- Hit validation: shooter alive, target alive, target within weapon range, not friendly fire (TDM same team), shooter roughly facing target (dot > 0.35). If valid, hp -= damage; if hp<=0, killed=true, killer kills++, victim deaths++, respawn in 3s at team spawn, broadcast killfeed. If target already dead or invalid, ignore (no broadcast).
- Match end: timeLeft<=0 or a team reaches 50 (TDM) / a player reaches 15 (FFA). Broadcast matchEnded, set state='ended'. Room stays for 60s then closes if empty.
- Chat: max 200 chars, broadcast to room with sender name/team.
- Voice signaling: relay only within the same room; 'to':'*' broadcasts to all other humans in room.
- Empty room: delete after 5 min with no humans.
- Movement: server clamps x,z to [-31, 31] and applies obstacle push-out (radius 0.6).

## Render deployment
- package.json: main server.js, script "start": "node server.js". No build step.
- Server listens on process.env.PORT || 3000, serves ./public statically, WebSocket on same port.
- Add render.yaml (service type web, env node, plan free, buildCommand "npm install", startCommand "node server.js"). Keep it minimal.
