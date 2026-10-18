# 🧱 KuMir Behind the Wall

**Category:** Web
**Target:** `https://kumir-behind-the-wall.solarctf.ru/`
**Flag:** `SOLARCTF{n3_s0tv0r1_s3b3_kum1r4}`

---

## The story

The site looked like a relic — a retro KuMir programming forum called *"Клеточка,"* all boxy borders and old-school charm. But the interesting part wasn't the forum at all. It was something the forum could talk to that I couldn't: an internal grading service named `kumir-checker`, running on port `8081`, tucked away on a private Docker network with no door to the outside world.

That checker held the answer key. Literally — the flag lived inside it as the "expected answer" to a task. I could see the forum. The forum could see the checker. I couldn't see the checker. So the whole challenge became an exercise in persuasion: *convince the forum to go ask the checker on my behalf, and bring back what it says.*

---

## The lay of the land

The `docker-compose.yml` laid out the geography plainly:

```
[Internet] → [web :8000] → [govnonet] → [kumir-checker :8081]
```

- `kumir-checker` sat only on `govnonet`. No internet access, no way to reach it directly.
- The `web` app (a Flask service) could reach it by its internal hostname.

That `web` service was my bridge. If I could make it fetch a URL of my choosing, I could make it fetch *the checker.*

---

## The forum offers to fetch things for me

And it turned out the forum was only too happy to oblige. The `/profile` page had an innocent-looking feature: set your avatar by URL.

```html
<form method="post" action="/profile/avatar">
  <input name="avatar_url" type="text">
</form>
```

Here's the catch — the server doesn't just *store* that URL. It **fetches it itself**, server-side, and serves whatever comes back at `/api/avatar/me`. That's a textbook **SSRF** (Server-Side Request Forgery). The server will reach out to any address I give it, including addresses only *it* can see.

So I stopped feeding it cat pictures and started feeding it the checker.

---

## Knocking on the checker's door

**Step 1 — Ask the forum what the checker looks like.** I set my avatar URL to the checker's root and then read back the "avatar":

```bash
curl -X POST https://kumir-behind-the-wall.solarctf.ru/profile/avatar \
  --data-urlencode "avatar_url=http://kumir-checker:8081/"
curl https://kumir-behind-the-wall.solarctf.ru/api/avatar/me
```

The checker introduced itself:

```json
{ "diagnostic": "Kumir 2 teacher checker\nGET /teacher/check?task=robot-final&answer=<one-line answer>\n..." }
```

Now I knew the exact endpoint it graded answers on.

---

## The checker that says too much

**Step 2 — Give it a wrong answer on purpose.** This is where the checker betrayed itself. I pointed the SSRF at its grading endpoint with a deliberately silly answer:

```bash
curl -X POST https://kumir-behind-the-wall.solarctf.ru/profile/avatar \
  --data-urlencode "avatar_url=http://kumir-checker:8081/teacher/check?task=robot-final&answer=hello"
curl https://kumir-behind-the-wall.solarctf.ru/api/avatar/me
```

And the checker, trying to be a *helpful teacher,* returned a full report explaining what I got wrong:

```json
{ "verdict": "WA", "expected": "SOLARCTF{n3_s0tv0r1_s3b3_kum1r4}", "received": "hello" }
```

There it was. In its eagerness to show the correct answer next to mine, it printed the flag in plaintext, right in the `expected` field.

```
SOLARCTF{n3_s0tv0r1_s3b3_kum1r4}
```

---

## The one detail that nearly tripped me

There's a subtle trap in step 2. The checker URL contains an `&` (between `task` and `answer`), and that `&` had to be **URL-encoded** when I sent it through the avatar form — which is exactly why I used `--data-urlencode`.

Send it raw, and the `&answer=hello` gets swallowed by the *outer* form as a second field. The server then forwards only `?task=robot-final` to the checker, with no answer at all — and instead of the juicy "wrong answer" report, you get a boring validation error. Encoding the `&` keeps the whole thing glued together as one URL meant for the checker.

---

## 🛡️ Fix

1. **Lock down what the avatar fetcher can reach.** Only allow external, whitelisted domains — block internal hostnames (`kumir-checker`), `localhost`, link-local, and private IP ranges.
2. **Don't let the grader leak the answer.** The checker's teacher report should never put the expected answer in its response body — a pass/fail verdict is all a client needs.
