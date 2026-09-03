# PATCH — SCP raise flow: no more silent failure

**Status:** proposed, NOT applied. Apply to `bushido_coreproject.html` (via canvas first per pipeline).
**Raised:** 2026-09-03, after Chark tapped "Raise this project" and nothing happened.
**Scope:** 3 small edits, display-only. Writes no user data, adds no state key, changes no data shape.

## The defects

1. **`sprRaiseCommit` fails silently on an empty name** — `if(!name){ nameEl.focus(); return; }`.
   Focus, no message. This is the exact failure the craft guard three lines below was fixed for
   (*"A3: require a craft — SAY so, don't scroll in silence"*). Never applied to the name.
2. **The craft message never clears** — nothing sets `#spr-raise-craft-msg` back to `display:none`,
   so it stays on screen after the user picks a craft.

## Constraint that governs the implementation

`sprRenderRaise` rebuilds `ov.innerHTML` wholesale, and the typed name/hours live ONLY in the DOM.
**Any re-render wipes what the user typed** — see the existing chip handler comment: *"toggle in place —
re-rendering would wipe the typed name/hours"*. Every edit below is therefore an in-place style toggle.
Do not "fix" this by calling `sprRenderRaise()`.

## Edit 1 — helper (new, next to the other spr helpers)

```js
function sprRaiseMsg(id, show){ var m=document.getElementById(id); if(m) m.style.display = show ? 'block' : 'none'; }
```

## Edit 2 — `sprRenderRaise`: add the name message node, after the `#spr-raise-name` input

```js
+ '<div id="spr-raise-name-msg" style="display:none;font-size:9px;letter-spacing:1.5px;text-transform:uppercase;color:var(--spr-red);margin-top:6px">COPY — JAMES TO RULE</div>'
```

Mirrors `#spr-raise-craft-msg` exactly (same size, tracking, colour, margin).

## Edit 3 — `sprRaiseCommit`: say it, and clear stale messages

```js
function sprRaiseCommit(){
  var nameEl = document.getElementById('spr-raise-name');
  var name = nameEl ? (nameEl.value||'').trim() : '';
  sprRaiseMsg('spr-raise-name-msg', false);            // clear both first — neither may outlive its cause
  sprRaiseMsg('spr-raise-craft-msg', false);
  if(!name){
    sprRaiseMsg('spr-raise-name-msg', true);           // was a SILENT focus() — the Chark case
    if(nameEl){ nameEl.focus(); nameEl.scrollIntoView({block:'center'}); }
    return;
  }
  if(SPR_RAISE_CRAFT == null){
    sprRaiseMsg('spr-raise-craft-msg', true);
    var cap = document.querySelector('#sp-works-overlay .spr-crow'); if(cap) cap.scrollIntoView({block:'center'});
    return;
  }
  /* …unchanged from here… */
}
```

## Edit 4 — clear the craft message when a chip is tapped (`sprEnsureWorksDelegation`)

```js
else if(act==='raise-craft'){ SPR_RAISE_CRAFT = t.getAttribute('data-v');
  sprRaiseMsg('spr-raise-craft-msg', false);           // fixes the never-clears bug
  /* …existing in-place chip toggle, unchanged… */ }
```

## Deliberately NOT in this patch

- **The no-crafts empty state.** It ALREADY exists (`sprRenderRaise`, the `chips` else-branch:
  *"You have no crafts yet — a project hangs off one. Add a craft in Shokunin first."*).
  The real gap there is PLACEMENT — that line sits under the craft heading while the absent
  button is ~40px lower, past the hours field, with nothing connecting them. A separate,
  design-led change; do not add a second line saying the same thing.
- **Exact copy.** A23/A24 govern this surface ("project" is the prose noun, never "work";
  no em-dash in the intro; measured at 360px). James rules the wording.

## Before it ships

- `verify_raise_flow` (6/6) and `verify_project_entrypoints` (11/11) assert this DOM — expect RED,
  update deliberately, RED-first. Do not relax an assertion to green it.
- Canvas may be behind — run canvas catch-up before building.
- Re-test the typed-name-survives case explicitly: type a name, tap with NO craft picked, confirm
  the name is still in the box when the message appears. That is the regression this patch risks.

## Open

Chark's actual cause is still UNCONFIRMED. If he was signed out, this patch does not help him —
his project would be in localStorage only, with a ⚠ chip offering a retry. Confirm before assuming
this fixed it.
