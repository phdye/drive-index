# Protecting an index that cannot be rebuilt

The design constraint everything here follows from, and the three mechanisms
that implement it.

## Why an index is not a cache

A cache is something you can throw away because the authority is still
reachable. These indexes mostly are not caches. A disk is indexed once, then
unplugged and shelved, and the index is consulted for months while the disk
itself is in a drawer. During that time the index is the only record of what
was on it.

That changes what a wrong answer costs. A search that misses is a nuisance.
An index that was quietly replaced with a scan of something else, or
truncated by a run that died, destroys information that cannot be recovered
without finding the physical disk.

Three things follow.

## Builds are atomic

`updatedb` is given `--output=<database>.new`. The result is checked, and
only then renamed over the real path with `os.replace`, which is atomic
within a filesystem.

A run that fails, or is interrupted, leaves the temporary file and the
previous index untouched. The temporary is removed on the next build rather
than accumulating.

This is what wrapping rather than reimplementing costs and permits: the
program cannot change how `updatedb` writes, but it can decide where it
writes and whether the result is accepted.

Evidence that this was needed and not theoretical: `/var/mlocate/x.db` is 10
bytes and `c.db.n` is 0 bytes, both left by interrupted runs of the scripts
this replaced.

## A collapsed result is refused

After a build, the candidate is compared with what it would replace:

- empty, or
- under half the bytes, or
- under half the path count

Any of those and the candidate is discarded, the existing index is left in
place, and the run exits 6. `--force` accepts it anyway and says that it is
doing so.

Half is a chosen number, not a derived one, and lives in `SHRINK_RATIO` at
the top of the program. It is meant to catch the difference between a disk
that lost a directory and a disk that was not mounted, not to police ordinary
churn.

The path count comes from `locate -c` against the candidate, and from the
metadata sidecar for the incumbent, falling back to counting it too.

## An index knows which disk it describes

A label may declare `volume = <name>`. Before any build, `<root>/.volume-id`
is read and compared. A mismatch refuses the build and exits 5, naming both
sides.

This exists because a drive letter is a slot, not a disk. Windows hands out
`/d` to whatever is plugged in, so an index built as one disk's while another
is mounted would be labelled with the wrong hardware — and would be believed,
because by the time anyone checks, neither disk is attached.

With fixed mount points this never fires. That is the point: it costs nothing
when the setup is right and catches the case where it is not.

## What an index carries with it

Beside each built index, `<database>.meta` holds the source host, the root,
the build time, the path count, and the tool version that wrote it.

Searching offline, the question is never only whether a path exists. It is
how old this picture is and which machine took it. The index itself cannot be
asked, so the answer has to be stored beside it.

`locate` reports an index's age at `-v` and warns past 90 days. `locate`'s
own age warning is suppressed, because its default threshold is eight days
and an index of a shelved disk is past that forever — a warning that fires on
every search is one nobody reads.

An index copied from another machine may have no sidecar. Age then falls back
to the file's mtime, which is an approximation and is reported as one.

## Removable and imported labels

`removable = yes` says an absent root is normal. A build against one is a
quiet no-op rather than a failure, and a bulk sweep passes over it without
noise. A fixed root that has vanished is still an error.

A label with no root at all is an **imported** index: built elsewhere,
copied here, searchable but never buildable. It gets no `updatedb` symlink,
and asking for a build is refused with the reason. Without this, one
`4tb-updatedb` would replace 129 MB of another machine's filesystem with a
scan of a path that does not exist on this one.

## `bulk`

`bulk = no` keeps a label out of a sweep. The registry trees use it: they
change faster than any index of them can stay true, so they are worth having
on demand and not worth rebuilding on a schedule.

A sweep that lost one index exits non-zero. A partial success that reports
success to whatever scheduled it is how a broken index goes unnoticed until
the disk is gone.
