# Dashboard Project — Claude Instructions

## Supabase Credentials

When adding Supabase sync to any page in this project, always use these exact values:

```js
const SUPABASE_URL = 'https://fntashiwcbooeqmukcak.supabase.co';
const SUPABASE_KEY = 'sb_publishable_aFr9XKECSnIQES6cpBed9g_IfjqZ7Ym';
```

Do NOT use a different Supabase project — all pages must share the same database.

---

## Supabase Sync Block — Required on Every New HTML Page

When a new HTML page is added to this project, add the following sync block **before the closing `})();`** (or at the end of the main `<script>` if there is no IIFE).

### Step 1 — Add to `<head>` if not already present

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
```

### Step 2 — Pick a unique `APP_KEY` slug

Use the filename minus `.html` (e.g. `notes`, `meals`, `budget`). Must NOT collide with:
- `dashboard`
- `po-coach`
- `health`

### Step 3 — Paste the sync block

```js
// ── Supabase Sync ─────────────────────────────────────────────────────────────
(function initSupabaseSync() {
  const SUPABASE_URL = 'https://fntashiwcbooeqmukcak.supabase.co';
  const SUPABASE_KEY = 'sb_publishable_aFr9XKECSnIQES6cpBed9g_IfjqZ7Ym';
  const APP_KEY = '<<SLUG>>'; // unique per page, e.g. 'notes'

  // ── Identify which localStorage keys this page owns ──
  // List every key this page writes. Use exact strings or prefix patterns.
  // Example: const SYNC_KEYS = ['entries', 'settings'];
  const SYNC_KEYS = [/* TODO: fill in for each page */];

  const client = supabase.createClient(SUPABASE_URL, SUPABASE_KEY);

  let lastSyncedJson = null;
  let pendingRemote = null;
  let pushTimer = null;
  let isFocused = false;

  // Track whether user is typing
  document.addEventListener('focusin', e => {
    const t = e.target;
    if (t && (t.tagName === 'INPUT' || t.tagName === 'TEXTAREA' ||
        t.tagName === 'SELECT' || t.isContentEditable)) {
      isFocused = true;
    }
  });
  document.addEventListener('focusout', () => {
    isFocused = false;
    if (pendingRemote) {
      applyRemote(pendingRemote);
      pendingRemote = null;
    }
  });

  // ── Collect local state ───────────────────────────────────────────────────
  function collectLocal() {
    const state = {};
    for (const k of SYNC_KEYS) {
      const v = localStorage.getItem(k);
      state[k] = v !== null ? JSON.parse(v) : null;
    }
    return state;
  }

  // ── Apply remote state ────────────────────────────────────────────────────
  // After adding this block to a page, fill in:
  //   1. Re-assign every in-memory variable that mirrors a localStorage key.
  //   2. Call every render/refresh function so the UI redraws.
  function applyRemote(state) {
    for (const [k, v] of Object.entries(state)) {
      if (v === null) localStorage.removeItem(k);
      else localStorage.setItem(k, JSON.stringify(v));
    }
    // TODO: reload in-memory variables and call render functions, e.g.:
    //   entries = loadEntries();
    //   renderAll();
  }

  // ── Push to Supabase ──────────────────────────────────────────────────────
  async function pushNow() {
    try {
      const state = collectLocal();
      const json = JSON.stringify(state);
      if (json === lastSyncedJson) return;
      lastSyncedJson = json;
      await client.from('app_state').upsert(
        { key: APP_KEY, state },
        { onConflict: 'key' }
      );
    } catch (err) {
      console.warn('[sync] push failed', err);
    }
  }

  function schedulePush() {
    if (pushTimer) clearTimeout(pushTimer);
    pushTimer = setTimeout(pushNow, 250);
  }

  // ── Flush on tab close ────────────────────────────────────────────────────
  async function flushOnClose() {
    if (!pushTimer) return;
    clearTimeout(pushTimer);
    pushTimer = null;
    try {
      const state = collectLocal();
      const json = JSON.stringify(state);
      if (json === lastSyncedJson) return;
      lastSyncedJson = json;
      await fetch(`${SUPABASE_URL}/rest/v1/app_state`, {
        method: 'POST',
        keepalive: true,
        headers: {
          'Content-Type': 'application/json',
          'apikey': SUPABASE_KEY,
          'Authorization': `Bearer ${SUPABASE_KEY}`,
          'Prefer': 'resolution=merge-duplicates',
        },
        body: JSON.stringify({ key: APP_KEY, state }),
      });
    } catch {}
  }

  window.addEventListener('pagehide', flushOnClose);
  window.addEventListener('beforeunload', flushOnClose);
  document.addEventListener('visibilitychange', () => {
    if (document.visibilityState === 'hidden') flushOnClose();
  });

  // ── Monkey-patch localStorage ─────────────────────────────────────────────
  const _setItem = localStorage.setItem.bind(localStorage);
  const _removeItem = localStorage.removeItem.bind(localStorage);

  localStorage.setItem = function(key, value) {
    _setItem(key, value);
    if (SYNC_KEYS.includes(key)) schedulePush();
  };
  localStorage.removeItem = function(key) {
    _removeItem(key);
    if (SYNC_KEYS.includes(key)) schedulePush();
  };

  // ── Initial pull + realtime subscription ─────────────────────────────────
  async function init() {
    try {
      const { data } = await client
        .from('app_state')
        .select('state')
        .eq('key', APP_KEY)
        .single();

      if (data?.state) {
        const remoteJson = JSON.stringify(data.state);
        const localState = collectLocal();
        const localHasData = Object.values(localState).some(v => v !== null);

        // First-launch seed: local has data but remote is empty → push up
        const remoteIsEmpty = Object.values(data.state).every(v => v === null);
        if (localHasData && remoteIsEmpty) {
          await pushNow();
        } else {
          lastSyncedJson = remoteJson;
          applyRemote(data.state);
        }
      } else {
        // No remote row yet — seed from local
        await pushNow();
      }
    } catch (err) {
      console.warn('[sync] init pull failed', err);
    }

    // Realtime subscription
    client
      .channel(`app_state:${APP_KEY}`)
      .on('postgres_changes', {
        event: '*',
        schema: 'public',
        table: 'app_state',
        filter: `key=eq.${APP_KEY}`,
      }, payload => {
        const incoming = payload.new?.state;
        if (!incoming) return;
        const incomingJson = JSON.stringify(incoming);
        if (incomingJson === lastSyncedJson) return; // echo of our own push
        lastSyncedJson = incomingJson;
        if (isFocused) {
          pendingRemote = incoming;
        } else {
          applyRemote(incoming);
        }
      })
      .subscribe();
  }

  init();
})();
```

---

## Required Supabase SQL

Run this once in the Supabase SQL editor before using any page's sync:

```sql
create table if not exists public.app_state (
  key  text primary key,
  state jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now()
);

alter table public.app_state enable row level security;

create policy "anon read" on public.app_state
  for select using (true);

create policy "anon write" on public.app_state
  for insert with check (true);

create policy "anon update" on public.app_state
  for update using (true);

-- Auto-update timestamp
create or replace function public.touch_updated_at()
returns trigger language plpgsql as $$
begin new.updated_at = now(); return new; end $$;

create trigger app_state_touch
  before update on public.app_state
  for each row execute procedure public.touch_updated_at();

-- Enable realtime
alter publication supabase_realtime add table public.app_state;
```

---

## Checklist After Adding Sync to a New Page

- [ ] Hard-refresh every device once
- [ ] Confirm the SQL above has been run in Supabase (table + RLS + realtime)
- [ ] Test: change something on one device, watch it appear on another within ~1 second
