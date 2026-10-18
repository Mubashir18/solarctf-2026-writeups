# 🏦 Vintage Bank

**Category:** Web
**Target:** `http://vintage-bank.solarctf.ru/`
**Flag:** `SOLARCTF{W1ldc4rd_DN3_R00m}`

> *"I became a depositor, quite by chance, at an exclusive, long-established bank that caters to the ultra-wealthy. But it turned out that the bank's security system falls far short of its century-old, rock-solid reputation..."*

---

## The story

The challenge opens not with a login box, but with a little 2D game — the interior of an old bank you can walk around in. Marble floors, a teller's window, locked doors down a hallway. Each depositor gets their own private room, and the only way in is to prove the room is yours.

I had a room. Mine was `mubashir207`, and I had the credentials for it. But the flag wasn't in my room — it was sitting in a room called **`upravdom`**, and nobody had handed me the keys to that one. So the whole challenge came down to a single question: *can I walk into someone else's room using only my own key?*

---

## Walking up to the counter

First I played it straight. I went to the **КАССА** — the bank counter — and asked for my card. The game handed me a password and logged me in, same as any depositor would. Nothing suspicious on the surface.

But I had the developer tools open the whole time (F12 → Network), watching what the game said to the server behind the scenes. That's where the real story was.

---

## The key in the cookie

The moment I submitted my password to `/enter`, the server answered with a **302 redirect** and quietly set a cookie. Its name was `klyuch` — Russian for *"key."* Fitting.

```
klyuch = eyJyIjoibXViYXNoaXIyMDcifQ.atN8mg.f8X3HGlNbgRu8_mxzXDx29zvSvo
```

That shape is unmistakable — it's a signed token. I decoded the first part from Base64 and found exactly what I expected:

```json
{ "r": "mubashir207" }
```

So the server identifies me by a single field: `r`, the room name. My first instinct was the obvious one — just change `mubashir207` to `upravdom` and walk in. But the token had a **signature** glued to the end (`.atN8mg.f8X...`). Tamper with the payload and the signature no longer matches; the server would throw my forged key straight back at me.

I needed a different way in.

---

## The notice on the wall

So I went exploring. Walking through the bank, I found a **notice board** — the building's house register — pinned to a wall. And on it was a message that read almost like a confession:

> *"The apartment repair shop has issued a single key for all doors — until the renovation is complete."*
> `Key: SOLARCTF{W1ldc4rd_DN3_R00m}`

*A single key for all doors.*

That one line gave the whole game away. The bank wasn't checking **which** door your key opened — during the "renovation," every valid key opened every room. My key was valid. I just had to point it at the right door.

---

## One key, every door

Here's the detail that made it work. Each room isn't a path on one server — it's its own **subdomain**, served through wildcard DNS (`nip.io`):

| | URL |
|---|---|
| My room | `http://mubashir207.188.243.222.16.nip.io:30101/` |
| The target | `http://upravdom.188.243.222.16.nip.io:30101/` |

Since the server honored "one key for all doors," I simply took my **valid session** — the `klyuch` cookie the bank had given me for `mubashir207` — and walked up to the `upravdom` URL with it. No new password. No forged token. Just my real key at a different door.

The bank didn't blink. It logged me straight into `upravdom` as if I'd always belonged there. I found the room's notice board, and the flag was waiting on it.

```
SOLARCTF{W1ldc4rd_DN3_R00m}
```

---

## Why the vault opened

The bank confused two very different questions:

- **"Is this key real?"** — it checked this carefully (the signature was valid).
- **"Is this key allowed *here*?"** — it never asked at all.

A token that says *"I am mubashir207"* was treated as permission to enter any room, because the server never compared the room in the token against the room being requested. That's broken authorization — an IDOR / BOLA living at the level of entire rooms.

---

## 🛡️ Fix

1. **Check authorization, not just authenticity.** On every request, confirm the logged-in session is actually allowed to reach the room it's asking for (`upravdom` vs `mubashir207`), not merely that its token is signed correctly.
2. **Bind the session to the resource.** Cross-check the token's `r` claim against the requested host/room; on mismatch, reject.
3. **Mind wildcard tenancy.** In subdomain-based multi-tenancy, always cross-verify the Host header against the session token — one valid key must never open every door.
