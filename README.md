# ULTRA

A visitor picks a secret key and encrypts a message on their phone with an **Enigma I**.
Only the encrypted letters travel to the laptop, and the laptop breaks the message live with the
**Turing–Welchman attack** (the Bombe), without ever seeing the key.

## How the demo works

```
 Phone                       Mailbox                       Laptop (screen)
 web/celular.html   ──►  Supabase: mensajes table   ──►  web/pantalla.html
 picks the key,           (cloud mode)                    queues the messages,
 types and encrypts       or servidor/servidor.js         runs the Bombe (Web Worker)
                          (local mode)                    and shows the key it found
```

1. **Phone** (`web/celular.html`). The visitor turns the wheels to choose 3 rotors (I to V)
   and their starting position, and connects up to 4 letter pairs on the plugboard (or taps *Al azar* / random).
   Then they type their message on the keyboard: the lamps light up, just like on the real machine.
   When they tap *Transmitir* (transmit), the phone sends **only the ciphertext**. Before the message, the phone
   always encrypts the same official header, the *crib* `FERIABEADRAGONENLAUSFQ` (defined in
   `web/config.js`). The key never leaves the phone.
2. **Mailbox**. It only stores the ciphertext; it doesn't break anything. There are two modes, and the pages choose
   on their own (`modo: "auto"` in `web/config.js`):
   - **Cloud**: the phone inserts the ciphertext into the Supabase `mensajes` table (`web/nube.js`).
   - **Local**: if the pages are served by `node servidor/servidor.js`, the phone sends
     `POST /api/interceptar` and the server keeps the messages in memory (up to 500).
   The page finds out which mode it's in by requesting `/api/config`: if it answers with JSON, there is a
   local server; otherwise, it's in the cloud.
3. **Screen** (`web/pantalla.html`). Every second it reads new messages from the mailbox
   (cloud: `mensajesDesde` in `web/nube.js`; local: `GET /api/mensajes?desde=N`) and puts them in a
   **queue**: all of them are handled, one by one, in order of arrival. When the page opens it doesn't replay
   old messages. For each message it shows the "listening station" (the letters arrive one
   by one), highlights the first letters that hide the crib, draws the message's Turing menu
   and animates the search. Finally it shows the decrypted message, the rotors, the position
   and the plugboard cables it found.
   - If others are waiting, each result stays on screen for at least 15 s (`RESULTADO_MIN_MS`).
     "En cola: N mensajes" (N messages queued) appears in red at the top right.
   - If nobody transmits for 45 s after a result (`VOLVER_A_ESPERA_MS`),
     the QR scene comes back.
4. **Bombe** (`bombe/bombe2.js`). In both modes it runs in the laptop's browser, in a
   **Web Worker** (`web/bombe-web.js`) so the screen doesn't freeze. Each message gets
   its own Worker: a new message never interrupts the one being broken.

## How the Bombe works

The problem: there are **60** ways to choose and order 3 rotors out of 5, and **17,576** starting
positions (26 × 26 × 26). That's **1,054,560 rotor configurations**. On top of that there's the
plugboard, with a huge number of cable combinations, so brute-forcing everything is not an option.

What we do know is that every message starts with the crib. With that, `bombe/bombe2.js` does the following:

1. **Crib.** We place the crib under the first encrypted letters. Each column tells us
   "at position *i*, crib letter X became ciphertext letter Y".
2. **Menu.** With those pairs we build a **graph**: each letter is a node and each column
   *i* is an edge between X and Y labeled *i*. That drawing is the one shown on screen.
3. **Cycles.** Sometimes the edges form a cycle, for example A → M → T → A. For any
   rotor configuration, we can compute what the rotors and reflector do at each
   position *i* (without the plugboard). If we walk the whole cycle, the letter must come back
   to itself. If it doesn't, that configuration is impossible.
4. **Plugboard hypothesis.** We don't know the plugboard, so the Bombe takes a
   **test letter** (the start of the shortest cycle) and guesses which letter it's connected to:
   there are 26 possibilities (including itself, meaning "no cable"). Each guess
   is sent around the cycle; most don't come back and are discarded.
5. **Propagation with the diagonal board.** If a guess survives, the remaining cables are deduced
   by following the menu's edges. Cables work both ways (if A is connected to M,
   M is connected to A); Welchman called this the diagonal board, and it helps find
   contradictions much earlier. If a letter ends up with two different partners, or more
   than 4 cables appear, the guess dies.
6. **Final check.** With the deduced rotors, position and cables, the crib is encrypted
   with a real Enigma (`web/enigma.js`). If it produces exactly the start of the ciphertext,
   we've found the key and decrypt the full message.

### The funnel

A real example with key `II-V-I`, position `KDQ`, cables `AM FT RX` and the message `NOSVEMOSENLAFERIA`:

| Step | How many remain |
|---|---|
| Rotor configurations | 1,054,560 |
| × 26 hypotheses for the test letter | 27,418,560 |
| Pass the first cycle | 1,097,574 |
| No contradictions after propagation | 1 |
| Final key (verified with the Enigma) | **1** |

On this laptop that took about 4 seconds. The screen shows these same numbers at the end of each message.

## Repo structure

| Path | What it is |
|---|---|
| `index.html` | Redirects to `web/celular.html` (it's what the QR opens in the cloud). |
| `web/celular.html` | The visitor's Enigma: choose the key, type, transmit. |
| `web/pantalla.html` | The laptop screen: waiting QR, message queue, interception, graph, result. |
| `web/explicacion.js` | "How an Enigma works" plate (key E): photo with its parts and the path of the current. |
| `web/img/enigma.jpg` | Photo of an Enigma used by the plate. |
| `web/enigma.js` | Enigma I/M3 engine in JavaScript (rotors I–VIII, reflectors A–C, rings, plugboard, double stepping). Used by the phone, the Bombe and the tests. |
| `web/config.js` | The only file you edit: mode (`auto`/`nube`/`local`), Supabase URL and public key, `urlPublica` for the QR, and the crib. |
| `web/nube.js` | Talks to Supabase through the REST API (no libraries): send, read the latest, and read new ones (`mensajesDesde`). |
| `web/bombe-web.js` | Web Worker that runs the Bombe in the screen's browser (in both modes). |
| `web/temas/` | Screen themes (`a-enigma.css`, `b-bletchley.css`) and their fonts. |
| `web/vendor/qrcode.js` | Library to generate the QR without internet. |
| `bombe/bombe2.js` | Bombe v2: Turing–Welchman attack with plugboard (up to 4 cables). |
| `bombe/bombe.js` | Bombe v1, no plugboard (brute force over the 1,054,560 configurations). Exports the `CRIB`. |
| `bombe/prueba.js`, `bombe/prueba2.js` | Tests for Bombe v1 and v2. |
| `servidor/servidor.js` | Dependency-free local server (Node only), used just as a mailbox: serves `web/` (plus `/web/*`, `/bombe/*` for the Worker) and the API (`/api/interceptar`, `/api/mensajes`, `/api/config`). |
| `herramientas/simular-visitantes.mjs` | Simulates several visitors transmitting back to back, to test the queue. |
| `supabase/esquema.sql` | The `mensajes` table and its security policies (RLS). |
| `simulador/enigma.cpp` | The original Enigma in C++, with its tests. |
| `contrato.md` | Skeleton of the message format between the pieces (unfinished). |

## How to run it

### Cloud mode (the normal one at the fair)

- Phone (this is the QR link): https://ultra-enigma.pages.dev
- Laptop screen: https://ultra-enigma.pages.dev/web/pantalla.html

The screen generates the QR on its own from its own address (or from `urlPublica` in `web/config.js`
if it's filled in). The laptop needs internet.

### Local mode (backup, no internet)

```
node servidor/servidor.js
```

- Listens on port **8080** (can be changed with the `PORT` variable).
- Connect the laptop and the phones to the same hotspot.
- Open the screen at `http://localhost:8080/pantalla.html`. The console also shows
  the addresses for the phone (`http://<laptop-IP>:8080`).
- The screen's QR is generated automatically with the Wi-Fi IP.

### Screen keys

- **T** switches theme (A → B → original). It can also be chosen in the link:
  `?tema=a` (Enigma I, dark room) or `?tema=b` (Bletchley Park, daylight, the default).
  The last chosen theme is remembered in that browser.
- **E** opens the "How an Enigma works" plate: the photo with its parts; another **E** moves on to
  the path of the current, and another **E** (or **Esc**) closes it. If a message arrives, it closes on its own.

## Tests

From the repo root:

```
node web/enigma.js --prueba     # 5 Enigma tests (double stepping, reciprocity, period 16,900…)
node bombe/prueba.js            # Bombe v1: breaks a message without cables
node bombe/prueba2.js 10        # Bombe v2: 10 random keys with 0 to 4 cables
```

`prueba2.js` ends with a summary like `Clave correcta: 10/10 · mensaje exacto: …` (correct key / exact message).
It accepts a second argument with a different crib (by default it uses `FERIABEADRAGON`).

The C++ simulator: `g++ -std=c++17 -O2 -Wall -o enigma simulador/enigma.cpp` and then `./enigma --prueba`.

To test the screen's queue, with the screen open:

```
node herramientas/simular-visitantes.mjs local 3 3   # 3 visitors to the local server, one every 3 s
node herramientas/simular-visitantes.mjs nube 3 3    # the same, to Supabase
```

Each visitor uses a random key (0 to 4 cables) and a different message (`VISITANTEUNO`,
`VISITANTEDOS`, …). The script prints the key and the message that should appear on the screen.

## How to publish changes

Cloudflare Pages publishes the `dist/` folder (it's in `.gitignore`). It only contains what the
browser needs; `servidor/` and `simulador/` are not published. In PowerShell:

```powershell
Remove-Item dist -Recurse -Force
New-Item -ItemType Directory dist
Copy-Item index.html dist/
Copy-Item web dist/web -Recurse
Copy-Item bombe dist/bombe -Recurse
npx wrangler pages deploy dist --project-name ultra-enigma
```

The database is described in `supabase/esquema.sql`. To create it in a new project, paste it
into Supabase → SQL Editor → Run. It only accepts texts of 10 to 300 letters A–Z, anyone can
insert, and only messages from the last 15 minutes can be read. `web/config.js` holds
**only the public key** (anon / publishable), never the `service_role` key.

## Known limitations

- The rings (Ringstellung) are fixed at A, both on the phone and in the Bombe.
- Only reflector B and rotors I to V.
- The Bombe searches for up to 4 plugboard cables.
- If a cable connects two letters that don't appear in the menu, the Bombe can't deduce it:
  the key comes out right, but 1 or 2 letters of the message may be wrong.
- Because of the middle rotor's double stepping, sometimes two different starting positions encrypt the same way
  (for example `RVY` and `SWY` with `II-III-V`). The Bombe finds both, sorts them with the
  "normal" one first, and the screen shows that one; the decrypted message is the same.
- In cloud mode, the laptop needs internet. Without internet, use local mode.

## Credits

Made by Vanessa Almagro for the USFQ **Be a Dragon** fair.
