# EnvyControl Fix: PCI Rescan for Integrated → Nvidia Switch

## The Problem

On some laptops (tested on Nobara Linux), switching directly from
**integrated mode** to **nvidia mode** using EnvyControl doesn't work
properly the first time.

What happens:
- You run `envycontrol -s nvidia` and reboot.
- `envycontrol -q` says you're in nvidia mode.
- But `nvidia-smi` fails with "No devices were found."
- The dedicated GPU is not actually being used.
- You have to run the switch command *again* and reboot a *second*
  time for it to finally work.

This makes switching GPU modes frustrating, especially if you're just
trying to launch a game.

## Why This Happens

When you're in integrated mode, EnvyControl adds a rule that tells
Linux to physically remove the Nvidia GPU from the system's PCI bus
(this saves battery, since the GPU isn't fully "there" anymore).

When you switch to nvidia mode, EnvyControl needs to find the Nvidia
GPU's exact location on the PCI bus so it can write that into a config
file. But if you switch straight from integrated to nvidia (without
going through hybrid mode first), the GPU is still marked as removed
at that point — so the tool can't find it, or finds the wrong
information.

The first switch+reboot ends up broken because of this. The second
attempt works because, after a full reboot, Linux naturally
re-detects all hardware from scratch.

## The Fix

Before EnvyControl tries to find the Nvidia GPU, this patch adds one
small step: it tells the Linux kernel to **rescan the PCI bus** right
then and there. This forces the system to re-check for hardware
immediately, without needing a reboot.

Because of this, the Nvidia GPU is found correctly on the first try,
and switching to nvidia mode works properly the first time — no
second switch, no second reboot.

## What Changed

One block of code was added in the "switch to nvidia mode" section,
right before the Nvidia GPU is detected:

```python
try:
    with open('/sys/bus/pci/rescan', 'w') as f:
        f.write('1\n')
    logging.info("Triggered PCI bus rescan successfully.")
except Exception as e:
    logging.warning(f"Could not rescan PCI bus: {e}")
```

This is wrapped in a safety check (`try/except`), so if rescanning
ever fails on some system, it just logs a warning and continues
normally — it won't break anything.

## Notes

- This fix does **not** change how EnvyControl behaves if you already
  have a GPU cache set up (via `--cache-create`) — that still works
  exactly as before.
- This mainly helps people who switch straight from integrated to
  nvidia mode without ever using hybrid mode first.
- Tested on: Nobara Linux, Intel iGPU + Nvidia dGPU, X11 session.
- This may not fix every case — on some laptops, the Nvidia GPU is
  powered off at a deeper hardware level, and a rescan alone won't
  bring it back. On those systems, using the cache feature is still
  the more reliable option.
