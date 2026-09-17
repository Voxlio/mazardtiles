# Turning on online play

Mazard Tiles plays fine with no setup at all — **Pass & Play** on one device works
out of the box. This file is only about the **Play Online** button, which needs a
free Firebase Realtime Database to pass moves between players.

Budget about five minutes. Everything below is on Firebase's free Spark plan.

---

## 1. Create the project

1. Go to <https://console.firebase.google.com> and sign in with a Google account.
2. **Add project** → name it anything (`mazard-tiles` is fine) → Continue.
3. It'll offer Google Analytics. **Turn it off** — you don't need it and it adds
   a step. Continue → Create project.

## 2. Create the database

1. In the left sidebar: **Build → Realtime Database** → **Create Database**.
2. Pick the location closest to your players.
3. Choose **Start in test mode** → Enable.

> Watch out: **Realtime Database**, not **Firestore**. They're different products
> and they sit next to each other in the sidebar. If the page you land on talks
> about "collections" and "documents", you're in Firestore — go back.

Test mode leaves the database open to anyone for 30 days. Step 5 fixes that.

## 3. Register a web app

1. Click the **gear icon** (top left, next to Project Overview) → **Project settings**.
2. Scroll to **Your apps** → click the **`</>`** (web) icon.
3. Give it a nickname → **Register app**. Skip Firebase Hosting.
4. It shows you a `firebaseConfig` snippet. Leave this page open.

## 4. Paste the config into the game

Open `index.html`, scroll to the top of the first `<script>` block (around line
610), and fill in the empty strings from that snippet:

```js
var FIREBASE_CONFIG = {
  apiKey:       "AIzaSy...",
  authDomain:   "your-project.firebaseapp.com",
  databaseURL:  "https://your-project-default-rtdb.firebaseio.com",
  projectId:    "your-project",
  appId:        "1:123456789:web:abc123"
};
```

**`databaseURL` is the one that actually matters.** Firebase sometimes omits it
from the snippet — if it's missing, copy it off the top of the Realtime Database
page. It looks like `https://your-project-default-rtdb.firebaseio.com` (or
`...-rtdb.europe-west1.firebasedatabase.app` outside the US).

Save, hard-refresh the page, and **Play Online** will light up. Until you do this,
the online screen shows these instructions instead of a broken button.

## 5. Lock the rules down before you share it

Test mode expires after 30 days and, until then, lets anyone on the internet read
and write your entire database. Replace it before you send the link to anyone.

**Realtime Database → Rules** tab, paste this, **Publish**:

```json
{
  "rules": {
    "rooms": {
      "$code": {
        ".read": "$code.length === 4",
        ".write": "$code.length === 4",
        ".validate": "newData.hasChildren(['status', 'seats']) || !newData.exists()"
      }
    }
  }
}
```

This confines everything to four-character room codes under `rooms/`, so nobody
can write junk elsewhere in your database, and it doesn't expire.

**Be honest with yourself about what this does and doesn't do.** There are no user
accounts, so anyone who knows or guesses a room code can read that room — which
includes the other players' racks. With 4 characters there are about 1.6 million
combinations and only your live rooms exist at any moment, so a random guess
essentially never lands. But it's guessable in principle. For a game with friends
that's fine. Don't put anything sensitive in a player name.

If you ever want this properly sealed, the fix is Firebase Anonymous Auth plus a
rule requiring `auth != null` and checking the writer owns the seat they're
editing. That's a bigger change and it isn't wired up here.

---

## If something doesn't work

**"Couldn't reach the server"** — `databaseURL` is missing, has a typo, or points
at a Firestore URL. Check it character by character against the Realtime Database
page.

**"The database rejected that (permission denied)"** — your rules are blocking the
write. If you pasted the rules from step 5, confirm you're using 4-character room
codes (the game generates these itself, so this usually means the rules got edited).
If test mode expired, that's this error too.

**"Your Firebase API key looks wrong"** — `apiKey` was copied incompletely. It
starts with `AIza` and is about 39 characters.

**Online button is greyed out** — `FIREBASE_CONFIG` still has empty strings, or
you're looking at a cached copy. Hard-refresh: Ctrl+Shift+R.

**Works on your laptop, not on your phone** — you opened `index.html` as a local
file. Firebase needs a real origin. Serve it over HTTP (`python -m http.server`)
or push it to GitHub Pages.

---

## Cost

Free tier is 1 GB stored and 10 GB/month transferred. A room is a few hundred
bytes and rooms delete themselves when the last player leaves. You would need to
be running tens of thousands of games a month to notice. No card required on the
Spark plan — it refuses service rather than billing you.
