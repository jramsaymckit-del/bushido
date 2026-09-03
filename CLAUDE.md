# BUSHIDO — deploy repo

> **This repo is the GitHub Pages deploy target (`bushidolife.net`), not the dev repo.**
> Source of truth (`bushido_coreproject.html`, `Workshop/CANVAS.html`, the `verify_*` gates,
> `DIRECTION.md`, `SCP_CLAN_MIGRATION_DESIGN`) lives outside this checkout. Commits on `main`
> are machine-written `deploy: app <hash> <ts>` entries. Do not hand-edit `app/index.html` here —
> changes go live immediately and would be overwritten by the next deploy.

---

## ⚑ OPEN REQUEST FOR CECE — Special Clan Projects: full editability + real cross-user assignment

**Raised by James, 2026-09-03. Nothing has been built. This is a note, not a change.**

**The ask, in his words:** *"Special projects needs to be all editable and assign other users other tasks."*

This is the **B2 clanProjects build** that SCP v1 explicitly parked. The `BUILD_ID` changelog
already says it: *"NO assignment, editing or uid work — that is the separate clanProjects build."*

### Verified state of the code (read 2026-09-03, `app/index.html` @ `def3a5b`)

✓ **Own room is ALREADY fully editable** — don't rebuild it. `openProjectRoom` → add branch/sub-task,
edit text, re-home, set steward, set due, tick, seal 印, post to The Record, delete-unsealed→tombstone.
All of it works today.

✗ **A clan-mate's room is read-only BY CONSTRUCTION** — this is the "all editable" half.
- `sprOpenClanView` (`app/index.html:13891`) sets `SPR_OPEN = null` at :13897 with the comment
  *"structural read-only: no owned project open → every edit act is a no-op"*.
- `sprOnClick` (:13579) then bails at :13584 on `if(!sprGetProject(SPR_OPEN)) return;` — every
  `data-act` after `close` is swallowed.
- The view renders via `sprBranchStatic` / `sprForgeStaticHtml`, which emit **no inputs and no
  handlers at all**. There is nothing to enable; the editable DOM does not exist on this path.
- Footer copy at :13888 states the current contract outright: *"This project lives with X.
  Only X can change it."* Changing behaviour means changing that promise deliberately.

✗ **Assignment is a NAME STRING, never a real user** — this is the "assign other users" half.
The shape is right but the field is dead: `steward` / `consult` carry `{name, uid, outside}`
(designed at :13472 as *"an ADDITIVE layer later, never a rewrite"*), and **`uid` is hardcoded
`null` at every single write site**:
- `sprAddSubtaskCommit` :13627 — `steward:{name:'James',uid:null,outside:false}`
- `sprAddBranchCommit` :13636 — same
- `sprCommitSteward` :13660 — `{name:nm, uid:null, outside:!!outside}`; the form (`sprStewardForm`
  :13989) is a free-text `placeholder="a name"` box.

So you can type "Susan" onto a task and it binds to no account. Susan's app never learns of it.

### 🐞 Live bug found while verifying — `'James'` is hardcoded as an identity test

Not a feature gap; this is **already wrong in production for Susan, Chark, Elijah and Lisa**,
and it sits inside exactly the code this work has to touch:

| Line | Problem |
|---|---|
| `:13627`, `:13636` | New sub-task/branch is stamped `name:'James'` **regardless of who is signed in** |
| `:13830`, `:13978`, `:13983` | `steward.name==='James' ? 'You' : …` — the *string* "James" **is** the personhood test |

Net effect: any non-James user adding a task gets it silently attributed to "James" and rendered
back to them as **"You"**. Two different people both read as "You" on the same row. This is the
`uid` gap showing through — fix it as part of the identity switch, not separately.

### What CECE should weigh before writing code

1. **Read `SCP_CLAN_MIGRATION_DESIGN` §3 first** — the uid identity switch is deferred *there*, by
   design. This request is that deferral coming due. Don't re-derive it.
2. **Transport is the real blocker, not the UI.** Today `/clan/{uid}` is a one-way mirror the owner
   republishes on their own save — hence the *"as of"* staleness stamp at :13877. Assigning a task
   to Susan means Susan's app must **write into a doc she doesn't own**. A one-way per-uid mirror
   cannot carry that. Expect Firestore rules work, not just client work.
3. **Editability ≠ a boolean.** Decide who may change what: can a steward tick only their own row?
   Can a non-owner add a branch? Seal? Delete? The four rendered-DOM boundaries at :13475–13477
   (esp. **A13** — outside-hand rows show no seal/tick affordance) are ratified rules, not
   incidental styling. Keep them.
4. **Merge net.** `projects` / `projectRemovals` already ride recursive id-union + per-entity
   `updatedAt` CAS + tombstones (Finding #1 discipline, §7.2). Two people editing one project is
   the first genuine **concurrent-writer** test of that net. Incident A8 (2026-07-02) was caused by
   exactly this class of bug — a second writer clobbering newer data. Treat it as the headline risk.
5. **Gates.** `verify_project_room` (15/15), `verify_projects_merge`, `verify_clan_project_visibility`
   (16/16) and `verify_scp_witness_band` all currently assert the read-only contract. They will —
   correctly — go RED. Update them deliberately; a green suite after this work must be green on the
   *new* contract, not on a weakened assertion.
6. **Pipeline.** Canvas → coreproject → sync → deploy. This work starts in `Workshop/CANVAS.html`,
   never here, never in `app/index.html`.

### Naming note
`DIRECTION.md` A24 records the standing tension: the feature is called *Special **Clan** Project*
while v1 shares nothing — the "Clan" was the aspiration, parked to B2. **This request is B2.**
Building it closes that tension rather than working around it.

---

## ⚑ ALSO FOR CECE — live incident, SCP raise flow (2026-09-03, OPEN)

Chark tapped "Raise this project" and nothing saved. He **has crafts**, so the commit button rendered
and his tap did reach `sprRaiseCommit`. Cause still **unconfirmed** — narrowed to:

1. **Empty name** — `sprRaiseCommit` does `if(!name){ nameEl.focus(); return; }`. Focus, **no message**.
   Truly silent. (If he has exactly one craft it is auto-selected at `:14315`, which rules the craft
   guard out entirely and leaves this as the only dead-tap path.)
2. **No craft picked** (2+ crafts only) — there IS a message, but it is 9px uppercase micro-copy.
3. **Signed out** — project lands in localStorage only; a ⚠ "not saved to cloud" chip offers a retry.
   `saveStateToFirestore` returns early at `:10355`. **This one no copy fix helps.**

The discriminator: *did the project room open?* Yes → created and stranded locally (#3).
No → rejected by a guard (#1/#2), nothing written.

**A patch for #1 and #2 is written and ready:** see `PATCH-raise-flow-silent-failure.md` in this repo.
Not applied. It carries the one trap that matters — `sprRenderRaise` rebuilds `innerHTML` and the typed
name/hours live only in the DOM, so surfacing a message via re-render **wipes what the user typed**.
Every edit in the patch is an in-place style toggle for that reason.
