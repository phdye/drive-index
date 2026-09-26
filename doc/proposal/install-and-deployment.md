# Where the runnable copy lives

Status: accepted 2026-09-26, implemented; record in
doc/decisions/2026-09-26.deployment.md

## The problem

There are two copies of this program. One sits in a directory on PATH and is
what the symlinks point at and what actually runs; the copy in this
repository is the one being edited. Nothing reconciles them.

This is not hypothetical. Stale copies caused two separate wrong results in a
single session: a fix made in one copy was overwritten by an older version of
the other, and a check that existed in one bar never reached the other, so
the most important regression test silently was not running.

Until this is settled, the source of truth is ambiguous, and that ambiguity
is the defect.

## What makes it more than a `cp`

`install` computes where to write the symlinks from `os.path.realpath` of its
own path. That is correct today, when the program and its links sit in one
directory.

Make the deployed copy a symlink into this repository and it becomes
wrong in a way that is easy to miss: `realpath` resolves through the symlink,
so all eleven links would be created **inside the repository** rather than
beside the symlink that was invoked. The program would appear to work and
would quietly scatter links into the wrong tree.

Any option below that involves a symlink has to change that logic first.

## The options

### Symlink in, and teach `install` to use the invoked directory

The deployed copy becomes a symlink to `<repo>/drive-index`. `install`
uses `os.path.dirname(os.path.abspath(argv[0]))` without resolving, and links
to the basename as now.

One copy. Edits land immediately with nothing to deploy. Against: the working
setup then depends on the repository being present at a fixed path, and a
half-finished edit is live the moment it is saved — there is no moment at
which the deployed program is known-good and the edit is not yet.

### `install --into DIR`

The repository is the source. The directory on PATH is a deployment target,
refreshed by an explicit act.

Two copies, deliberately, with a named moment where the bar has passed and
the deployment happens. Against: two copies can drift, which is the problem
this document exists about — though drift by omission is more visible than
drift by accident.

### Deploy step, matching whatever the machine already does

If a general deployment tool is already in use for other repositories, this
should use it rather than invent a third thing.

## Recommendation

`install --into DIR`, unless the existing deploy pattern already covers this,
in which case use that. The deciding argument is that both stale-copy
defects happened because deployment was implicit. A deploy that is an act
someone performs, after the bar passes, is the one that fails loudly.

This is the operator's call, not a technical one — it turns on a machine
convention that is not visible from inside this repository.

## What is blocked on it

The originals have been copied here and verified
identical, but **not deleted**. The delete phase waits on this decision,
because where the files end up depends on which option is taken.
