# ASAM Wars — Per-Tab Participant Identity & Simplified Join

This fixes the bug where two tabs in the same browser shared one
`localStorage` participant token and were treated as the same player.

Scope: participant identity + join/lobby flow **only**. No changes to
scoring, the Supabase schema, or the chart viewer.

---

## 1. Token resolution — `lib/asam.ts`

Replace the old `localParticipantToken()` with the version below. It adds a
`perTab` option:

- **Team play** (`perTab: true`): `?pt=` URL override → else a per-tab
  `sessionStorage` token. Two tabs in one browser get two identities.
- **Solo play** (`perTab: false`, the default): one stable `localStorage`
  identity per device — existing single-player behaviour is preserved.

```ts
const PT_KEY = "asam_participant_token";

function newToken() {
  const rnd =
    typeof crypto !== "undefined" && crypto.randomUUID
      ? crypto.randomUUID()
      : Math.random().toString(36).slice(2);
  return `pt-${rnd}`;
}

/**
 * Resolve the participant token for THIS browsing context.
 *
 * Team play (perTab: true):
 *   1) ?pt=<value>     -> use that exact value (force a distinct identity per tab)
 *   2) sessionStorage  -> per-tab token (two tabs in one browser = two identities)
 *
 * Solo play (perTab: false, the default):
 *   3) localStorage    -> one stable identity for the single player on this device
 */
export function localParticipantToken(opts: { perTab?: boolean } = {}): string {
  const perTab = opts.perTab ?? false;

  if (perTab) {
    // (a) explicit override from the URL — highest priority
    try {
      const pt = new URL(window.location.href).searchParams.get("pt");
      if (pt && pt.trim()) {
        const v = pt.trim();
        try { sessionStorage.setItem(PT_KEY, v); } catch {}
        return v;
      }
    } catch {}

    // (b) per-tab token kept in sessionStorage (NOT shared between tabs)
    try {
      const existing = sessionStorage.getItem(PT_KEY);
      if (existing) return existing;
      const fresh = newToken();
      sessionStorage.setItem(PT_KEY, fresh);
      return fresh;
    } catch {}
  }

  // (c) solo single-player fallback — stable per device in localStorage
  try {
    const solo = localStorage.getItem(PT_KEY);
    if (solo) return solo;
    const fresh = newToken();
    localStorage.setItem(PT_KEY, fresh);
    return fresh;
  } catch {}

  return newToken();
}
```

Why `sessionStorage` solves the bug: `localStorage` is shared by every tab
of an origin; `sessionStorage` is scoped to a single tab. So a second tab
automatically gets its own token with zero manual work — and `?pt=` lets you
pin an exact identity when you want one.

---

## 2. Resolve the token per-tab in `TeamMode.tsx`

Change the top-of-component resolution so team mode uses the per-tab path.
`useMemo` keeps it stable across re-renders (and reads `?pt=` exactly once,
before any Supabase insert):

```ts
// was: const token = localParticipantToken();
const token = useMemo(() => localParticipantToken({ perTab: true }), []);
```

Make sure `useMemo` is imported from React.

Because every Supabase insert (`createSession`, `joinSession`, `pickTeam`,
flags, attempts) and `refetchParticipants()`'s "find me" already use this
`token`, fixing the token here fixes identity everywhere — including
requirement 5 (each tab matches its own row by the resolved per-tab token).

---

## 3. URL params — read both `?join` and `?pt`

`?pt=` is consumed inside `localParticipantToken()` at mount, so this effect
only needs to handle `?join`. Setting `autoJoin` lets the join screen skip
manual code entry (requirement 3) while keeping the typed-code path:

```ts
const [autoJoin, setAutoJoin] = useState(false);

// Read URL params once on mount.
// ?pt  -> already consumed by localParticipantToken({ perTab: true }) above,
//         so the per-tab identity is locked in before any insert.
// ?join -> auto-resolve the session and drop straight to the name prompt.
useEffect(() => {
  try {
    const url = new URL(window.location.href);
    const j = url.searchParams.get("join");
    if (j) {
      setJoinCode(j.toUpperCase());
      setAutoJoin(true);   // code came from the link → no manual entry needed
      setSub("join");      // straight to the streamlined name prompt
    }
  } catch {}
}, []);
```

---

## 4. `joinSession()` — unchanged logic, now correct identity

Your existing `joinSession()` already auto-assigns the joiner to the team
with the fewest **real** (non-bot) members, and already keys "me" off
`token`. With the per-tab `token` from step 2 it now works correctly with no
further change. The one thing to confirm is the guard at the top so the
auto-join path can call it as soon as a name exists:

```ts
const joinSession = async () => {
  if (!orgCode || !joinCode.trim() || !joinName.trim())
    return toast.error("Code & name required");
  // ... rest unchanged ...
  // existing = currentList.find((p) => p.client_token === token)  // per-tab now
  // targetTeam = team with fewest non-bot members
  // insert { ..., client_token: token }
};
```

---

## 5. Join URL generation — don't bake a shared `pt` into the link

The old line generated a fresh random `pt` on every render and embedded it
in the shared link:

```ts
// OLD — buggy: re-randomises each render, and anyone opening the SAME copied
// link would collide on the SAME pt.
const joinUrl = session
  ? `${window.location.origin}${window.location.pathname}?join=${session.code}&pt=tester-${Math.random().toString(36).slice(2,7)}`
  : "";
```

Replace it with a **stable** share link that carries no `pt` (each tab then
self-assigns a per-tab `sessionStorage` identity), plus a helper that mints
**distinct** forced-identity links for solo testing:

```ts
// Stable share link — no pt. Every tab/person who opens it self-assigns a
// distinct per-tab identity via sessionStorage. Safe to give to many people.
const joinUrl = useMemo(
  () =>
    session
      ? `${window.location.origin}${window.location.pathname}?join=${session.code}`
      : "",
  [session?.code]
);

// Forced-identity links for solo testing two teams from one machine.
// Each label pins an exact, distinct token via ?pt=.
const testerJoinUrl = (label: string) =>
  session ? `${joinUrl}&pt=tester-${label}` : "";
```

Then in the Host Test Tools panel, offer two explicit tester links instead of
one randomised one, e.g. `testerJoinUrl("a")` and `testerJoinUrl("b")`.

---

## 6. "You are: [name] on [team]" banner (lobby + round)

Drop this banner near the top of both the `lobby` and `round` renders so each
tab visibly proves it is a different person:

```tsx
{me && (
  <div className="rounded-lg border border-teal-500/40 bg-teal-500/10 px-4 py-2 text-sm">
    You are:{" "}
    <span className="font-semibold text-teal-200">{me.display_name}</span>
    {" on "}
    <span className="font-semibold">
      {teams.find((t) => t.id === me.team_id)?.name ?? "no team yet"}
    </span>
    {me.is_team_lead && <span className="ml-2 opacity-80">· Team Lead</span>}
  </div>
)}
```

---

## How to test two teams from one machine

The session token now lives in `sessionStorage`, which is per-tab. You have
two ways to get distinct players:

**Easiest — separate windows (auto identity):**
1. Host: Quick-Start a session, copy the **Join link** (now `?join=CODE`,
   no `pt`).
2. Open that link in a **normal window**, an **incognito/private window**,
   and **another browser** (Chrome + Firefox). Each window is a separate
   `sessionStorage`, so each becomes a different participant. They auto-assign
   to the emptiest team and land on the name prompt.
   - Note: two **tabs of the same normal window** that were opened with
     "Duplicate tab" can copy `sessionStorage`; opening a fresh tab and
     pasting the link does not. When in doubt, use separate windows or the
     `?pt=` links below.

**Most deterministic — forced identities (`?pt=`):**
1. In Host Test Tools, use the two tester links (`testerJoinUrl("a")`,
   `testerJoinUrl("b")`), or just append your own:
   - `…?join=ASAM-1234&pt=tester-a`
   - `…?join=ASAM-1234&pt=tester-b`
   - `…?join=ASAM-1234&pt=tester-c`
2. Open each in any tab/window — even all in the same browser. `?pt=` pins
   the exact token, so they are guaranteed distinct regardless of storage.
3. Give each a name (Alex on Team 1, Sam on Team 2). The first real member of
   each team becomes its Lead and submits for that team.

Either way, the "You are: [name] on [team]" banner confirms each tab/window
is a different person before you start a round.
