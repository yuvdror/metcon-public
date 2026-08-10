# metcon-public

Every public static page for the **Metcon** app. Deployed on Vercel.

| Path | What it is |
| --- | --- |
| `index.html` | Marketing landing page |
| `privacy.html` | Privacy policy — also served at `/privacy` |
| `support.html` | Support page — required by both stores |
| `delete-account.html` | Out-of-app deletion request — required by Play |
| `auth-callback.html` | Password-recovery / email-confirmation landing page |
| `style.css` | Shared stylesheet |
| `vercel.json` | Routing + security headers |

Vercel serves the directory as-is. There is no build step.

## Why this is a separate repo from the app

The app monorepo is private and holds signing config, store credentials and the
full schema. This repo is public because it has to be: the privacy policy URL
goes in both store listings, and `auth-callback.html` has to be reachable by
anyone who clicks a link in a password-reset email.

Keeping them apart means the public surface can be deployed, cached and indexed
without exposing anything else, and a marketing copy change does not touch a repo
that can ship a binary.

## `auth-callback.html`

Supabase's Site URL and Redirect URLs point here.

The recovery token arrives in the URL **fragment**, which browsers never send to
a server — so the host never sees it. The embedded project URL and publishable
key are public by design; the same pair ships inside the apps.

**If the project ref or publishable key change, update them in TWO places:** the
`CFG` object in `auth-callback.html`, and the `connect-src` directive in
`vercel.json`. Miss the second and password recovery fails silently, because the
browser blocks the request with no visible error.

**`cleanUrls` is off deliberately.** With it on, Vercel 308-redirects
`/auth-callback.html` to `/auth-callback` — and that exact URL is configured in
Supabase as the Site URL. The redirect would have to be mirrored there or
recovery breaks. Not worth it for cosmetics; `/privacy` gets an explicit rewrite
instead.

**`form-action` is `'self'`, not `'none'`.** The reset form's submit handler
calls `preventDefault()`, so no navigation should ever happen — but if the script
fails to attach, `'self'` degrades to a harmless page reload where `'none'` would
be a blocked submit with no feedback, and password reset is an App Store 5.1.1
dependency. `'self'` still blocks posting the password to any other origin, which
is the actual threat.

## Deploying

Add New → Project → import `yuvdror/metcon-public`. Framework preset **Other**,
build command **empty**, output directory **`.`**. Every push to the default
branch deploys; pull requests get preview URLs.

Then attach `metcon.fit` in **Project → Settings → Domains** and create the DNS
records Vercel shows you. Do not move the nameservers if the domain carries email
sending records — add only the A/CNAME records asked for.

## Before this goes live

- [ ] Replace `YOUR_PROJECT_REF` in `vercel.json` and `auth-callback.html` with
      the real Supabase project ref. Recovery fails silently until you do.
- [ ] Set Supabase → Authentication → URL Configuration → **Site URL** to
      `https://metcon.fit/auth-callback.html` and add it to **Redirect URLs**.
- [ ] Test a real password reset end to end, triggered **from the app**. Opening
      the page by hand proves nothing — with no URL fragment it correctly shows
      its "link has expired" state.
- [ ] Point the store badges in `index.html` at the real listings.
- [ ] **Localisation.** The app is English-only today and so is this site. The
      pattern to follow when that changes is kinsense-public's: one content file,
      a generator, and reciprocal `hreflang` across every page. Do not hand-write
      the translations into separate HTML files — they go stale within one release
      and `hreflang` only works if every page links to every other one.
