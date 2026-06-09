# ASAM Wars — Per-Tab Participant Identity & Simplified Join (FINAL, drop-in)

Fixes the bug where two tabs in the same browser shared one `localStorage`
participant token and were treated as the same player.

Scope: participant identity + join/lobby flow **only**. No changes to
scoring, the Supabase schema, or the chart viewer.

The fix: team identity lives in **`sessionStorage`** (per-tab, so two tabs =
two players automatically), with a **`?pt=`** URL override to pin an exact
identity for testing. Solo play keeps its stable `localStorage` identity.

---

## FILE 1 — `lib/asam.ts`

Replace the old `localParticipantToken()` with this.

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

---

## FILE 2 — `TeamMode.tsx` (6 edits)

### ① Import `useMemo` (top of file)

```ts
import { useState, useEffect, useRef, useMemo } from "react";
```

### ② Replace the token line (top of the component)

```ts
// was: const token = localParticipantToken();
const token = useMemo(() => localParticipantToken({ perTab: true }), []);
```

### ③ Add `autoJoin` state (next to the other join state)

```ts
const [autoJoin, setAutoJoin] = useState(false);
```

### ④ Replace the `?join` useEffect

```ts
// Read URL params once on mount.
// ?pt  -> already consumed by localParticipantToken({ perTab: true }),
//         so per-tab identity is locked in before any Supabase insert.
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

### ⑤ Replace the `joinUrl` line

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
const testerJoinUrl = (label: string) =>
  session ? `${joinUrl}&pt=tester-${label}` : "";
```

### ⑥ Add the identity banner

Paste near the top of BOTH the `sub === "lobby"` and `sub === "round"`
returns:

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

## OPTIONAL — tester buttons for the Host Test Tools panel

Replaces the single randomized link with explicit, distinct tester links.

```tsx
<div className="flex gap-2 flex-wrap">
  {["a", "b", "c"].map((label) => (
    <Button
      key={label}
      variant="outline"
      size="sm"
      onClick={() => { navigator.clipboard.writeText(testerJoinUrl(label)); toast.success(`Copied tester-${label} link`); }}
    >
      <Copy className="w-3 h-3 mr-1" /> Tester {label.toUpperCase()} link
    </Button>
  ))}
</div>
```

---

## No edits needed

`joinSession()` and `refetchParticipants()` already key "me" off `token` and
already auto-assign joiners to the team with the fewest non-bot members. Once
`token` is per-tab (edit ②), they are correct as-is — this also satisfies the
"each tab matches its own row" requirement.

---

## How to test two teams from one machine

**Easiest — separate windows (auto identity):**
1. Host: Quick-Start a session, copy the **Join link** (now `?join=CODE`, no `pt`).
2. Open it in a **normal window**, an **incognito/private window**, and a
   **second browser** (Chrome + Firefox). Each window is its own
   `sessionStorage`, so each becomes a different participant, auto-assigned to
   the emptiest team.
   - Avoid "Duplicate tab" — it copies `sessionStorage`. Open a fresh window
     and paste the link instead, or use the `?pt=` links below.

**Most deterministic — forced identities (`?pt=`):**
1. Use the tester links (`testerJoinUrl("a"/"b"/"c")`), or append by hand:
   - `…?join=ASAM-1234&pt=tester-a`
   - `…?join=ASAM-1234&pt=tester-b`
2. Open each in any tab/window — even all in the same browser. `?pt=` pins the
   exact token, guaranteed distinct regardless of storage.
3. Name them (Alex on Team 1, Sam on Team 2). The first real member of each
   team becomes its Lead and submits for that team.

The "You are: [name] on [team]" banner confirms each tab/window is a different
person before you start a round.
