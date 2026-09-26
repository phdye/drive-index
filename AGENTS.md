# drive-index

One program behind every `<label>-locate` and `<label>-updatedb` symlink. It
reads the label and the action out of the name it was invoked under, so
`c-locate` and `4tb-updatedb` are the same executable reached by two paths.
It wraps GNU findutils' `locate` and `updatedb`; it does not reimplement
them.

## The risk to understand before changing anything

**Most of the indexes this builds describe disks that are not attached.**
They are built once, from a disk that then goes in a drawer, and consulted
for months afterwards. For those, the index is not a cache of something you
could regenerate by asking the filesystem. It is the only surviving record of
what was on that disk.

Everything awkward about this program follows from that. Builds go to
`<database>.new` and are renamed into place only after the result is checked
against what it would replace. A result that collapsed is discarded rather
than installed. A build refuses to run when the volume mounted at the root is
not the one the index describes, because an index confidently labelled with
the wrong disk is worse than no index: you will believe it.

If you find yourself simplifying any of that, you have probably assumed the
disk is reachable. It usually is not.

## Wrapping, not reimplementing

`updatedb` and `locate` are called as programs. This is a deliberate
constraint, not an accident of history, and it has one cost worth knowing
before you hit it.

`updatedb` takes `--localpaths` and `--prunepaths` as single
whitespace-separated lists, with no escaping form. **A path containing a
space cannot be expressed to it at all.** Such a path is refused with an
explanation rather than silently read as two paths that match nothing.
Driving `find` and `frcode` directly would fix this and is out of scope.

`locate` keeps its own interface. Options this program has no opinion about
are passed through untouched, and `locate` rejects what it does not know in
its own words. Four short options are claimed here and mean something else:
`-d` is `--debug` (use `--db`), `-q` is the volume control, `-a` searches
every index, and `-l` is a limit in both. The long `--all` reaches `locate`
unchanged.

## Labels

A single-letter label is a drive and needs no configuration: its root is
`/<letter>/` and its database comes from `db_file_model`. That is the common
case and it must stay free.

Anything else — an imported index, a subtree, a registry tree — names
something that cannot be guessed, and needs a `[drive.<label>]` section
before it will run. **Single letters are reserved to drives**, so a
one-letter alias is a configuration error rather than a silent hijacking.
Duplicate aliases are refused too: otherwise which index you searched would
depend on the order sections happened to be read in.

A label with no root is an index built elsewhere and copied here. It can be
searched but never built, and gets no `updatedb` symlink at all.

## Layout

    drive-index           the program
    drive-index-certify   the certification bar
    config/example.ini    a template; deployed config is not tracked
    doc/design/           how what exists works, and why it is that way
    doc/proposal/         work not yet done, each with a Status line
    a/                    working notes, never committed

## Conventions

**Python 3.6.** This has to run on RHEL 8.10, whose Python is 3.6.8. Nothing
from 3.7 or later: no `dataclasses`, no `subprocess.run(capture_output=)`, no
walrus, no `from __future__ import annotations`. `configparser` rather than
`tomllib`, which arrived in 3.11.

**The bar is the gate.** `./drive-index-certify` must pass before a commit.
It takes an interpreter as its first argument, so it can be run against the
oldest one available. Every defect fixed gets a check that fails without the
fix; the bar is currently the most valuable artifact here, because three
separate defects in it reported green while the code was wrong.

It runs entirely against a fixture it builds, with `HOME` and
`DRIVE_INDEX_CONFIG` redirected into a scratch directory. Nothing it does
reads or writes the machine's own configuration, so it passes on a clean
checkout and cannot quietly depend on whoever is running it. Redirecting
`HOME` as well as the config is deliberate: a check that forgets to isolate
itself then fails against an empty scratch home rather than passing against a
real one, which is how a broken alias once hid behind a live config entry.

    env -i PATH=/usr/bin:/bin ./drive-index-certify

is the check that it is still true.

**`--help` prints the hand-written Usage block verbatim** rather than
argparse's rendering. The two are separate artifacts and nothing keeps them
in sync, so the text a caller reads is at least the text that was reviewed.
When you add an option, edit both.

**Diagnostics carry provenance.** A failure names the value, where it came
from (option, environment, which config file, or the default), what was
tried, and what exists nearby. When no config was found at all, the failure
lists every name that was looked for -- reporting `none` answers the wrong
question, and a config sitting under an unsearched name reads as no config
at all. This is not decoration: a config key that was
never read passed its tests for a whole session because the configured value
and the built-in default happened to be the same string.

## State

Working and covered by the bar: dispatch, config layers with aliases and
templates, atomic builds with the collapse guard, volume identity, removable
and imported labels, `status`, `bulk`, `locate --all-indexes`, metadata
sidecars.

Deployment is one command. The repository is the source of record and
`install --into DIR` places the program and every symlink the config names
where it runs from. Editing in one place and running from another without
that step drifted twice in a single sitting, both times silently.

A config can describe more than one machine. `[os.<name>]` sits between
`[defaults]` and `[drive.<label>]` and carries what differs between systems
rather than between drives: the `locate` and `updatedb` binaries, the
database model, which label a bare action means.

Nothing in `doc/proposal/` is now unimplemented code. What remains there is
fixed mount points, which is an operator change rather than a program one.
