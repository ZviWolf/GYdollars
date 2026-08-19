# GYdollars

**$GY Counter** — a Gan Yisroel dollar tracker for CGI Toronto 5786.

Counselors keep a running tally of the $GY dollars each camper in their bunk has
earned. Head counselors see every bunk in camp; admins can edit any of them.

The whole app is a single static file, `index.html`, with Firebase Auth and
Cloud Firestore behind it. No build step, no dependencies to install, no server
to run.

---

## Roles

| Sign in as | Sees | Can do |
| --- | --- | --- |
| **Counselor** (email + password) | Their own bunk | Add and remove campers, adjust counts |
| **Head counselor** (username `headcounselor`) | Every bunk, read-only | Browse bunks and camper totals |
| **Admin** (username `admin`) | Every bunk | Everything above, plus edit and remove any camper, and delete whole bunks |

Head counselor and admin are shared accounts. Instead of an email address, type
`headcounselor` or `admin` into the email field — the app maps those to internal
addresses (`headcounselor@gydollars.internal` / `admin@gydollars.internal`) that
never receive mail. They skip email verification and have no bunk of their own.
Both are created through the normal Sign Up form, once, by whoever sets the app
up; the password you choose at that point is the shared password.

## Counselor flow

1. **Sign up** with an email, a password, and a bunk number. Each bunk number can
   only be claimed by one account — the app checks before creating the account.
2. **Verify your email.** Firebase sends a link; the app waits on a "Check your
   email" screen until you've clicked it. Use *Resend email* if it doesn't arrive.
3. **Add campers** with the ➕ button. Names have to be unique within your bunk
   (compared without regard to capitalization).
4. **Tap `+` or `−`** on a camper, then pick an amount: **1, 5, 10, or 20**.
   Counts never go below zero.
5. **Removing a camper** requires typing their name exactly, including
   capitalization. Same for an admin deleting a bunk — you type the bunk number.

Forgot your password? The link on the login screen sends a Firebase reset email.
It only works for real counselor accounts, not the shared `headcounselor` /
`admin` logins.

## Saving

Saving is automatic — the **Save** button is there for reassurance, not because
anything is waiting on it.

Amount changes are batched: rapid `+`/`−` taps are collapsed into a single write
about 0.7s after you stop tapping, rather than one write per tap. Adding or
removing a camper writes immediately. Any pending write is also flushed when you
press Save, log out, switch away from the app, or close the tab, so backgrounding
your phone mid-tally won't lose anything.

The header line under the title shows the current state: your camper count
normally, `saved` after a successful write, or `save failed — try again` if the
network dropped it. A failed write stays queued and retries on your next edit.

On the admin side, edits to a bunk are pooled the same way and committed as one
update per bunk owner, flushed on **Save**, **Back**, **Refresh**, or logout.

## Data model

One Firestore document per counselor account, in the `userData` collection, keyed
by Firebase Auth uid:

```
userData/{uid} = {
  bunkNumber: "7",                                  // string; unique across accounts
  campers: [                                        // array, rewritten as a whole
    { id: "9f3c…", name: "Avi", count: 23 },
    { id: "1a7e…", name: "Ben", count: 10 }
  ]
}
```

Bunks aren't documents of their own — a bunk is just the set of accounts sharing
a `bunkNumber`, which is how the head-counselor view assembles its list. Deleting
a bunk clears its campers and removes the `bunkNumber` field, freeing that number
to be claimed again.

Camper `id`s come from `crypto.randomUUID()` where available, with a
timestamp-plus-random fallback for older browsers.

## Running it

Open `index.html` in a browser, or serve the directory:

```sh
npx serve .
```

Because it loads the Firebase SDK as ES modules, open it over `http://` rather
than as a `file://` path.

## Deploying

Publish `index.html` on any static host — GitHub Pages, Netlify, Cloudflare
Pages. There is nothing to build.

Whichever domain you land on has to be listed under **Authentication →
Settings → Authorized domains** in the Firebase console, or sign-in will be
rejected.

## Firebase setup

The app talks to the `gydollars-3ee66` Firebase project; its config sits at the
top of the `<script type="module">` block in `index.html`. Pointing this at a
different project means replacing that object and enabling **Email/Password**
under Authentication.

A Firebase web API key is not a secret — it identifies the project, it doesn't
grant access, and it's meant to ship in client code. **Your Firestore security
rules are what actually protect camper data.** Worth knowing before you write
them:

- A counselor needs read and write access to `userData/{their own uid}`.
- Signup and the bunk-number prompt both query the **whole `userData`
  collection** filtered by `bunkNumber`, to check whether a number is taken. Any
  rule that restricts reads to "your own document only" will break signup.
- The head counselor and admin accounts read the entire collection, and the admin
  also writes to other people's documents.

That last point is the awkward one: the roles are recognized purely by email
address in client-side JavaScript, which is a UI affordance, not a security
boundary. Anyone can edit the JavaScript. If the rules don't independently check
`request.auth.token.email` for the admin address, admin powers aren't actually
restricted to the admin. Custom claims are the sturdier version of this.

Test rule changes against the signup path specifically — it's the one that breaks
in a way you won't notice until a new counselor tries to join.

## Repository layout

```
index.html    the entire app — markup, styles, and logic
test/         a stale copy of an older build; not used by anything
README.md
```

`test/` predates the current app and isn't wired into anything. It can be deleted
whenever you're confident you don't want it as a reference.
