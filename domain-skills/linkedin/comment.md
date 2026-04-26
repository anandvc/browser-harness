# LinkedIn: post a comment on a feed post

Durable shape for posting a top-level comment on a LinkedIn feed post URL of the form
`https://www.linkedin.com/posts/<author>_<slug>-activity-<id>-<hash>`.

Assumes the attached Chrome is already authenticated to LinkedIn. If not, the page will
surface a login wall; stop and ask the user.

## URL pattern

- `.../posts/<author>_<slug>-activity-<urn>-<suffix>` — canonical feed post.
- No query params needed. Hash in URL does not affect behavior.

## Stable selectors

- Editor (current, since ~April 2026): `div.ql-editor[contenteditable="true"]:not(.ql-clipboard)` —
  LinkedIn migrated from tiptap to **Quill**. Re-confirmed live 2026-04-25 (run
  20260425-170202-16907 via bh). On a cold load the page contains exactly one
  `div.ql-editor`, classed `ql-editor ql-blank`, no `.ql-clipboard` peer in DOM —
  but keep the `:not(.ql-clipboard)` filter anyway because reply composers can render
  one later when expanded.
- Editor (legacy, tiptap): `div.tiptap.ProseMirror[contenteditable="true"]` — may still
  appear in some A/B variants or older flows. Probe both selectors and use whichever is
  present.
- Post button: **no stable class for the button itself**, but the wrapping element does
  carry a stable substring `comments-comment-box__submit-button` (with a BEM modifier
  suffix like `--cr`). Two equally-durable approaches:
    1. Substring class match: `button[class*="comments-comment-box__submit-button"]:not([disabled])`.
       Confirmed live 2026-04-25 — full class set was
       `comments-comment-box__submit-button--cr artdeco-button artdeco-button--1 artdeco-button--primary ember-view`.
    2. Text-walk up from the editor and match a descendant `<button>` whose `innerText`
       is exactly `Post` / `Comment` / `Reply` and `!disabled`. The button sat 6 parents
       above the editor in the 2026-04-25 run, with `innerText === "Comment"`.
   Prefer (1) for deterministic poster scripts; (2) is the safe fallback when class
   substrings drift.
- Comment affordance (when editor isn't rendered yet on cold load): `<button>` whose
  **visible `innerText`** matches `/^add a comment$|^comment$/i`. Quill is normally
  eager-rendered, so the affordance probe is rarely needed — but if you do need it,
  **don't use `aria-label="Comment"`**: that label appears on at least two unrelated
  buttons on a feed-post page (the social-actions row AND the share-preview rail), and
  picking the wrong one collapses the editor or scroll-jumps to the comment list.
  Walk all `<button>` elements and match by visible text, not aria-label.
- Existing comments list: `.comments-comment-item, article.comments-comment-entity,
  [data-urn*="comment"]`. Use for post-submit verification if you need to confirm by
  text match.

## Posting flow that works

1. **`new_tab(target_url)`** — always new tab, never `goto()` first (per harness setup
   notes).
2. `wait_for_load(timeout=25)` + `time.sleep(3)` — LinkedIn SPA hydration continues
   after `readyState == 'complete'`. Without the extra sleep the editor isn't mounted
   yet. **Three full seconds is the floor**, not "feel free to drop to 1.5s if you're
   in a hurry" — at 1.5s the Quill editor is reliably absent and any fallback path that
   tries to mount it via a Comment-button click will misfire (see Traps).
3. Scroll to `y ≈ 900` so the comment editor (which lives below the post body) is in
   viewport. If the editor still isn't present, try `y ∈ {1200, 1800, 2400}` — post
   length varies.
4. Probe both editor selectors. If neither is present after the 3s post-load sleep,
   wait an additional `waitForSelector(editor_sel, timeout≥6000)` — Quill is
   eager-rendered but mount can still trail SPA hydration on slow networks. **Only**
   if that wait also fails should you fall back to the affordance click; and that
   click MUST match by visible `innerText` of the button, not `aria-label`.
5. Get the editor's bounding-rect center via JS and **coordinate-click** it twice (the
   editor sometimes needs a second click to actually focus, both for tiptap and Quill).
   Typing with CDP `type_text()` (`Input.insertText`) fires the `beforeinput`/`input`
   events that both tiptap and Quill listen for — don't use `press_key` char-by-char.
6. After typing, **sanity-check** the editor's `innerText` length. If the editor has
   less than 50% of the expected chars, the editor didn't capture focus — abort instead
   of clicking Post on an empty/partial editor.
7. Click the Post button by walking up from the editor element and matching text. The
   button sits 4-8 parents above the editor in the DOM.
8. Wait ~7 seconds for the submit to settle. On success the editor's `innerText`
   becomes empty (tiptap clears on successful submit). **Editor empty = success** is a
   more reliable signal than scraping the comment list, which has inconsistent
   CSS classes.

## Verification

- **Primary signal**: `editor.innerText === ''` after submit. LinkedIn only clears the
  tiptap node on a successful server ack.
- **Secondary signal** (optional): scan `.comments-comment-item` / `article.comments-comment-entity`
  for your text. Flaky because LinkedIn A/B-tests the comment-item class set and the
  new comment sometimes lands below the fold.
- **Error signal**: a toast appears (`[role="alert"]`) OR the editor retains its text
  and the Post button greys out and re-enables. If text is still there, the submit
  failed.

## Rate limiting

Pace successful submits at **≥60 seconds apart**. LinkedIn shadow-rate-limits
rapid-fire commenting — the submit succeeds but the comment is hidden from other
viewers. 65s is a safe default.

## Traps

- **`goto()` clobbers the user's active tab.** Always `new_tab()`.
- **`type_text()` on an unfocused editor silently no-ops.** Always click-to-focus
  first, then sanity-check `innerText` length before clicking Post.
- **The Post button has no stable class** across LinkedIn's rollouts. Text-match
  walking up from the editor is the durable approach.
- **Reply editors use the same selector as the top-level editor.** If a reply composer
  is already open when you arrive, you may grab the wrong editor. Filter to the first
  one with `document.querySelector` (returns the first in DOM order = top-level) or
  scope to the post container.
- **`button[aria-label="Comment" i]` is ambiguous on a feed-post page.** It matches at
  least two distinct buttons (social-actions row AND share-preview rail). Clicking the
  wrong one collapses the not-yet-mounted Quill editor or scroll-jumps the page, then
  any subsequent `waitForSelector` for the editor times out. This is the failure mode
  that triggered the 2026-04-25 self-heal run (20260425-170202-16907) — see the
  affordance bullet under "Stable selectors" for the safer text-walk.
- **Multiple screenshots of the same URL after submit can show "5h" and "now"
  duplicates** — the approvals pipeline should dedupe against the `posted` set before
  approving, not rely on LinkedIn to reject.
- Waiting on the comment list selectors to confirm is unreliable; the editor-empty
  check is the one that survives LinkedIn's A/B churn.

## Minimal working snippet

```python
# Quill is current; tiptap kept as fallback. Helpers here are the actual bh names —
# `click_at_xy` (NOT `click`) and `capture_screenshot` (NOT `screenshot`).
EDITOR_SEL = 'div.ql-editor[contenteditable="true"]:not(.ql-clipboard)'
LEGACY_SEL = 'div.tiptap.ProseMirror[contenteditable="true"]'

tid = new_tab(target_url)
wait_for_load(timeout=25); time.sleep(3)   # 3s floor — see Posting flow #2
js("window.scrollTo(0, 900)"); time.sleep(1)

# Pick whichever editor is mounted. Prefer Quill.
sel = EDITOR_SEL if js(f"!!document.querySelector('{EDITOR_SEL}')") else LEGACY_SEL

rect = js(f"""(()=>{{
  const el = document.querySelector('{sel}');
  if (!el) return null;
  el.scrollIntoView({{block:'center'}});
  const r = el.getBoundingClientRect();
  return {{x: r.left + r.width/2, y: r.top + r.height/2}};
}})()""")
click_at_xy(rect["x"], rect["y"]); time.sleep(0.3); click_at_xy(rect["x"], rect["y"])
type_text(comment_text); time.sleep(2)

# sanity-check text landed
got_len = js(f"((document.querySelector('{sel}')||{{}}).innerText||'').trim().length")
assert got_len >= len(comment_text) * 0.5, "text did not land in editor"

# click Post by walking up from the editor (durable across tiptap → Quill)
js(f"""(()=>{{
  let el = document.querySelector('{sel}');
  for (let i=0; i<10 && el; i++, el = el.parentElement) {{
    for (const b of el.querySelectorAll('button')) {{
      const t = (b.innerText||'').trim().toLowerCase();
      if ((t === 'post' || t === 'comment' || t === 'reply') && !b.disabled) {{ b.click(); return; }}
    }}
  }}
}})()""")
time.sleep(7)

# verify: editor cleared == success
empty = js(f"((document.querySelector('{sel}')||{{}}).innerText||'').trim() === ''")
capture_screenshot(f"/tmp/lc-{{item_id}}.png")
cdp("Target.closeTarget", targetId=tid)
```
