# Smaller gaps, and one that is not code

Status: accepted 2026-09-26. default_drive and trim_prefix implemented;
record in doc/decisions/2026-09-26.per-os-and-display.md.
The entry kinds are exercised by config/example.ini. Fixed mount points
remain open: an operator change no code here can make.

Four items. The first two are small features from the design this program
grew out of; the third is a kind of entry nothing here exercises; the fourth
is an operator project that would remove a whole class of risk.

## `default_drive`

The prior design named a default label per OS, so `locate foo` with no label
meant something. Nothing here supports a bare invocation.

Small to add. It needs one decision first: what a naked
`drive-index locate foo` should mean. Candidates are a configured default,
the drive the current directory sits on, or every index held — which is what
`--all-indexes` already does and might be the better default now that it
exists.

## `trim_prefix`

Commented out in the original and never finished. The need is real: indexing
an installation-rooted tree records paths like `/opt/root/home/user/x`,
while the same file from inside that root is `/home/user/x`. Results are
correct but
unusable without mental translation.

Two ways: store trimmed and lose the ability to reach the file from outside,
or store full and trim on output. The second is better and is a display
concern rather than an index one, which means it can be added later without
rebuilding anything.

## Kinds of entry nothing exercises

The configuration supports a label pointing at any root, but only three kinds
are configured here: whole drives, an imported index, and the registry trees.
The design it came from also had subtrees of a drive, named source trees, a
cloud-sync folder, and WSL instances.

Nothing forbids them — `[drive.usr-src]` with a root would work today — but
none is configured, so none is tested. The first one added will be the first
real test of whether the shape fits, and the likely friction is
`db_file_model` producing collisions when two labels normalize to similar
names.

## Fixed mount points

Not a code change, and the highest-value item on this page.

Windows can attach a volume to an empty directory with `mountvol` or Disk
Management, so a disk appears at `C:\mnt\usb-red` regardless of which drive
letter it is handed. Cygwin then sees `/c/mnt/usb-red` with no `fstab`
involvement, and the path is stable on both sides.

That removes the root cause the `volume` mechanism exists to detect: a label
that names a disk while its root names a slot. With fixed mounts, `volume`
should never fire, and its job becomes catching a misconfiguration rather
than catching an everyday hazard.

Worth doing before adding more removable disks, because every label added
under the current scheme is another one whose identity depends on plug order.
