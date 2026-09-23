# GPredict_K4KDR_N6RFM

**A patched build of Gpredict** carrying forward K4KDR's Alpha-5
catalog-number work, plus a fix for a ~2-year-old upstream regression
that silently breaks rotator tracking for a subset of users — found,
root-caused, and verified through a full investigation documented
below.

Based on current upstream [`csete/gpredict`](https://github.com/csete/gpredict)
master (`9399b12`, version `2.6.1`).

---

## TL;DR

If your Gpredict rotator control's Azimuth/Elevation "Read" fields
are permanently stuck at `---` — Engage works, rotctld is healthy,
the hardware responds fine, but the display never updates, no matter
how many times you rebuild, reinstall, or toggle Engage — this repo
fixes it. The cause is a December 2023 upstream commit that
accidentally deleted the timer driving that display, and whether you
ever notice depends on the exact value of an unrelated setting
(Cycle) in your rotator config. See below for the full story.

This repo also carries K4KDR's Alpha-5 catalog-number decoding, so
satellites with NORAD IDs ≥ 100000 (CelesTrak's Alpha-5 encoding,
introduced 2026-07-11) load and track correctly.

---

## The bug: rotator display permanently stuck at `---`

### Symptom

In Gpredict's Rotator Control window, Azimuth and Elevation "Read:"
labels show `---` forever after clicking Engage — even though:

- `rotctld` accepts the connection and logs successful position
  queries
- A manual `rotctl -m 2 -r <host>:<port> p` query returns correct
  live values
- The Engage button correctly toggles to "Disengage"
- Toggling Engage/Disengage repeatedly, restarting `rotctld`,
  rebuilding from a clean checkout, and reinstalling all have **no
  effect**

### Root cause

Commit [`dfb7b02`](https://github.com/csete/gpredict/commit/dfb7b02276a9f2887be5de62a32e36c79c128637)
("Closing the rotor control window while a rotor is engaged no
longer crashes Gpredict", 2023-12-24) fixed a real crash-on-close
bug, but in doing so deleted the only line that created the rotator
widget's own periodic display-refresh timer:

```diff
--- a/src/gtk-rot-ctrl.c
+++ b/src/gtk-rot-ctrl.c
@@ -1681,9 +1681,6 @@ GtkWidget      *gtk_rot_ctrl_new(GtkSatModule * module)
     gtk_box_pack_start(GTK_BOX(rot_ctrl), table, FALSE, FALSE, 5);
     gtk_container_set_border_width(GTK_CONTAINER(rot_ctrl), 5);

-    rot_ctrl->timerid = g_timeout_add(rot_ctrl->delay,
-                                      rot_ctrl_timeout_cb, rot_ctrl);
-
     if (module->target > 0)
         gtk_rot_ctrl_select_sat(rot_ctrl, module->target);
```

Even though the satellite module's own periodic update loop still
calls `gtk_rot_ctrl_update()` on the widget, that alone is **not
sufficient** — confirmed by explicitly wiring `module->rotctrl` to
the widget (matching the real UI code path exactly) and still
observing the stuck display in a controlled test. Without
`rot_ctrl->timerid` ever being set at creation, the widget never
establishes its own refresh cycle.

This has been broken in **every Gpredict release since December
2023** — current master, 2.4, 2.5, 2.5.1, 2.5.2 — for anyone it
affects (see below for who).

### Why it's intermittent

A separate, unrelated handler happens to recreate the same timer as
a side effect:

```c
static void delay_changed_cb(GtkSpinButton * spin, gpointer data)
{
    GtkRotCtrl *ctrl = GTK_ROT_CTRL(data);
    ctrl->delay = (guint) gtk_spin_button_get_value(spin);
    if (ctrl->conf)
        ctrl->conf->cycle = ctrl->delay;
    if (ctrl->timerid > 0)
        g_source_remove(ctrl->timerid);
    ctrl->timerid = g_timeout_add(ctrl->delay, rot_ctrl_timeout_cb, ctrl);
}
```

This fires on the Cycle spinbox's `"value-changed"` GTK signal,
which GTK also emits when the value is set **programmatically** — as
happens automatically at startup when a saved rotator device is
auto-selected:

```c
gtk_spin_button_set_value(GTK_SPIN_BUTTON(ctrl->cycle_spin), ctrl->conf->cycle);
```

The Cycle spinbox is constructed with
`gtk_spin_button_new_with_range(10, 10000, 10)`, so its value is
`10` before any config is applied. GTK only fires `"value-changed"`
when the new value **differs** from the current one:

| Saved `Cycle` value | Triggers signal? | Timer created? | Read fields update? |
| --- | --- | --- | --- |
| `10` (equals spinbox default) | No | No | **Stuck at `---`, permanently** |
| Anything else (e.g. `1000`) | Yes | Yes | Works normally |

Gpredict's own actual default (`DEFAULT_CYCLE_MS = 1000`) is used
whenever the saved config has no `Cycle` key at all — which is the
normal case for a rotator set up fresh through Preferences, since
that dialog doesn't even expose a Cycle field. So most brand-new
setups are accidentally fine.

`Cycle=10` specifically tends to show up when:
- Someone drags the live Cycle spinbox down to its minimum, chasing
  the fastest possible update rate — the more aggressively you tune
  for responsiveness, the more likely you are to hit the exact value
  that silently disables the display meant to show it.
- A `.rot` config file was copied or adapted from an external
  source (a shared config, a forum post, a tutorial) with that value
  already baked in.
- A historical, since-fixed **scroll-wheel bug** on Gpredict's own
  spin buttons (see the "Fixed broken scroll wheel in radio and
  rotator controllers" changelog entry) could have nudged the value
  to its floor from a single stray scroll, with no typing involved.
  Combined with a 2019 feature that auto-saves the Cycle field on
  every window close, this value can end up locked in without
  anyone ever deliberately choosing it.

### How this was found

Manual testing on real hardware (a GS-232-protocol rotator via
`rotctld`) ruled out the antenna, the serial connection, and the
network path — `rotctld` itself was always confirmed healthy. To
isolate the actual code defect, a fully sandboxed, deterministic
reproduction was built:

1. A headless Linux environment (Xvfb virtual display) with
   Hamlib's built-in "Dummy" rotator model — no physical hardware
   needed.
2. A small instrumented test hook compiled directly into
   `mod-mgr.c` that opens the Rotator Control widget, engages it
   programmatically, waits a few seconds, then reads the
   `AzRead`/`ElRead` label text directly and prints a pass/fail
   verdict — no GUI clicking, screenshots, or human interaction
   required.
3. `git bisect run` against this script, walking automatically
   between a confirmed-good commit (`0f3beb6`, Aug 2022) and current
   upstream master, rebuilding and retesting at each step.

This pinpointed `dfb7b02` as the exact commit where behavior
changed, and follow-up tests confirmed the precise mechanism (the
Cycle-value coincidence above) rather than just the commit boundary
— including confirming that Engage/Disengage toggling, which is what
real-world troubleshooting on this bug naturally reaches for first,
genuinely cannot fix it, since that code path never touches the
timer at all.

### The fix

Restore the timer creation in `gtk_rot_ctrl_new()`, keeping the
double-free guard added in the very next upstream commit
([`b74d9bd`](https://github.com/csete/gpredict/commit/b74d9bdf9fcfc27d7ea606d779a8f6d6f33a6cb7))
so the original 2022 crash-on-close bug does not resurface:

```c
rot_ctrl->timerid = g_timeout_add(rot_ctrl->delay,
                                  rot_ctrl_timeout_cb, rot_ctrl);
```

Verified directly in the sandbox: the same binary, built from this
repo, shows `AzRead='1.80°'` with `Cycle=10` set — the exact value
that leaves stock upstream Gpredict permanently broken.

---

## Alpha-5 catalog number support (K4KDR / N6RFM)

CelesTrak's SATCAT crossed 100000 on 2026-07-11. Objects cataloged
past that point use Alpha-5 encoding in the TLE catalog number
field: the leading digit is replaced with a letter (A–Z, excluding
I/O) to represent values up to 339999 in the existing 5-character
field width.

Stock Gpredict parses this field with plain numeric conversion, so
any Alpha-5-encoded catalog number silently decoded to `0`, causing
those satellites to collide or fail to load correctly.

This repo adds an Alpha-5-aware decoder (in `sgp_in.c` and
`tle-update.c`) that correctly handles:
- Legacy zero-padded numeric catalog numbers (`00900`)
- Legacy space-padded numeric catalog numbers (` 1234`)
- Alpha-5-encoded numbers (`A0057` → `100057`)

Verified against Space-Track's published Alpha-5 test values
(`A0000`→100000 … `Z9999`→339999), and confirmed end-to-end: loading
a CelesTrak TLE file containing an Alpha-5 satellite (`SOYUZ-MS 29`,
field `A0057`) correctly shows **Catalogue number: 100057** in
Gpredict's own Satellite Info panel.

**Origin note:** this feature was originally written for
[K4KDR's gpredict fork](https://github.com/K4KDR/gpredict), whose
`master` branch is itself downstream of the `dfb7b02` regression
above (confirmed via `git merge-base --is-ancestor`) — meaning the
bug was already present in that codebase before this Alpha-5 work
was ever added to it. This repo carries the Alpha-5 feature forward
onto a base with the timer bug actually fixed, rather than relying
on the Cycle-value coincidence to mask it.

---

## Build

```
git clone https://github.com/<your-username>/GPredict_K4KDR_N6RFM.git
cd GPredict_K4KDR_N6RFM
./autogen.sh
make
sudo make install
```

## Commit history

- `Restore rotor display refresh timer removed by dfb7b02` — the fix
- `Add Alpha-5 catalog number support` — the K4KDR/N6RFM feature,
  rebased onto current upstream master

## Credit

- [csete/gpredict](https://github.com/csete/gpredict) — original
  project
- [K4KDR/gpredict](https://github.com/K4KDR/gpredict) — origin of
  the Alpha-5 catalog-number work carried forward here

See [CONTRIBUTORS.md](CONTRIBUTORS.md) for details on this fork's
specific contributions (K4KDR, N6RFM, and Claude/Anthropic).
