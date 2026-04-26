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
  LinkedIn migrated from tiptap to **Quill**. Confirmed live 2026-04-25 (LC-95 post via bh).
  The `:not(.ql-clipboard)` filter excludes Quill's hidden paste buffer that shares the
  parent class. There can be multiple on the page when reply editors are open; the first
  is the top-level composer.
- Editor (legacy, tiptap): `div.tiptap.ProseMirror[contenteditable="true"]` — may still
  appear in some A/B variants or older flows. Probe both selectors and use whichever is
  present.
- Post button: **no stable class**. Find by walking up from the editor and matching a
  descendant `<button>` whose `innerText` is exactly `Post` (or `Comment` / `Reply` on
  some variants), and is `!disabled`. Works unchanged across the tiptap → Quill migration;
  text-walk found the button at depth 6 in the LC-95 test.
- Comment affordance (when editor isn't rendered yet on cold load): `<button>` whose
  `innerText` matches `/add a comment|^comment$/i`. Clicking it mounts the editor. With
  Quill the editor is now usually rendered eagerly on cold load, so the affordance probe
  is often unnecessary — but keep it as a fallback.
- Existing comments list: `.comments-comment-item, article.comments-comment-entity,
  [data-urn*="comment"]`. Use for post-submit verification if you need to confirm by
  text match.

## Posting flow that works

1. **`new_tab(target_url)`** — always new tab, never `goto()` first (per harness setup
   notes).
2. `wait_for_load(timeout=25)` + `time.sleep(3)` — LinkedIn SPA hydration continues
   after `readyState == 'complete'`. Without the extra sleep the editor isn't mounted
   yet.
3. Scroll to `y ≈ 900` so the comment editor (which lives below the post body) is in
   viewport. If the editor still isn't present, try `y ∈ {1200, 1800, 2400}` — post
   length varies.
4. Probe both editor selectors. If neither is present, click the "Add a
   comment" button first to mount the editor. Quill is usually eager-rendered;
   tiptap was historically lazier.
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
- **Multiple screenshots of the same URL after submit can show "5h" and "now"
  duplicates** — the approvals pipeline should dedupe against the `posted` set before
  approving, not rely on LinkedIn to reject.
- Waiting on the comment list selectors to confirm is unreliable; the editor-empty
  check is the one that survives LinkedIn's A/B churn.

## Minimal working snippet

```python
tid = new_tab(target_url)
wait_for_load(timeout=25); time.sleep(3)
js("window.scrollTo(0, 900)"); time.sleep(1)

rect = js("""(()=>{
  const el = document.querySelector('div.tiptap.ProseMirror[contenteditable="true"]');
  if (!el) return null;
  el.scrollIntoView({block:'center'});
  const r = el.getBoundingClientRect();
  return {x: r.left + r.width/2, y: r.top + r.height/2};
})()""")
click(rect["x"], rect["y"]); time.sleep(0.3); click(rect["x"], rect["y"])
type_text(comment_text); time.sleep(2)

# sanity-check text landed
got_len = js("(document.querySelector('div.tiptap.ProseMirror[contenteditable=\"true\"]').innerText||'').trim().length")
assert got_len >= len(comment_text) * 0.5, "text did not land in editor"

# click Post by walking up from the editor
js("""(()=>{
  let el = document.querySelector('div.tiptap.ProseMirror[contenteditable="true"]');
  for (let i=0; i<10 && el; i++, el = el.parentElement) {
    for (const b of el.querySelectorAll('button')) {
      const t = (b.innerText||'').trim().toLowerCase();
      if ((t === 'post' || t === 'comment' || t === 'reply') && !b.disabled) { b.click(); return; }
    }
  }
})()""")
time.sleep(7)

# verify: editor cleared == success
empty = js("((document.querySelector('div.tiptap.ProseMirror[contenteditable=\"true\"]')||{}).innerText||'').trim() === ''")
screenshot(f"/tmp/lc-{item_id}.png")
cdp("Target.closeTarget", targetId=tid)
```
