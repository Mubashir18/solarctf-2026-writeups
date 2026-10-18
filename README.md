# 🛡️ Solar CTF 2026 — Web Writeups

> My solutions for the **Web** challenges from [Solar CTF](https://solarctf.ru/).
> Three challenges, three very different web bugs — a wildcard auth bypass, an SSRF, and an OAuth logic flaw told as three short stories.

<p align="center">
  <img src="https://img.shields.io/badge/CTF-SolarCTF%202026-F9A825?style=flat-square" />
  <img src="https://img.shields.io/badge/Category-Web-1565C0?style=flat-square" />
  <img src="https://img.shields.io/badge/Solved-3%2F3-2E7D32?style=flat-square" />
</p>

---

## 📋 Challenges

| # | Challenge | Bug class | Flag |
|---|-----------|-----------|------|
| 1 | [Vintage Bank](web/vintage-bank/) | Wildcard / broken authorization | `SOLARCTF{W1ldc4rd_DN3_R00m}` |
| 2 | [KuMir Behind the Wall](web/kumir-behind-the-wall/) | SSRF → internal info leak | `SOLARCTF{n3_s0tv0r1_s3b3_kum1r4}` |
| 3 | [Online School](web/online-school/) | OAuth `state` abuse + bot oracle | `SOLARCTF{st4te_1s_n0t_a_r0ut3r}` |

---

## 🧠 The one-line lesson from each

- **Vintage Bank** — a key that proves *who you are* but never checks *which door you're allowed to open*. One key, every door.
- **KuMir Behind the Wall** — when a server will fetch any URL you hand it, point it inward and let it read the secrets it can see but you can't.
- **Online School** — the `state` parameter is for CSRF, full stop. The moment it starts steering a redirect, it becomes an open door — and a helpful bot handed me the key.

---

## 🗂️ Repo layout

```
solarctf-2026-writeups/
├── README.md                     ← you are here
└── web/
    ├── vintage-bank/
    │   └── README.md
    ├── kumir-behind-the-wall/
    │   └── README.md
    └── online-school/
        └── README.md
```

Each folder is a self-contained story: the setup, the moment the bug reveals itself, the exploit, and the fix.

---

## ⚠️ Disclaimer

Everything here was done against the official Solar CTF practice targets as part of the event. Shared for learning only — don't point these techniques at systems you don't have permission to test.

---

*Author: Khanzada Mubashir Hassan*
