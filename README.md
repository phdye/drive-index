# drive-index

One program behind every `<label>-locate` and `<label>-updatedb` symlink,
wrapping GNU findutils' `locate` and `updatedb`.

    c-locate -r 'max.*loop'      search drive C's index
    c-updatedb                   rebuild it
    drive-index status           what is indexed, how old, what is missing
    drive-index bulk             rebuild everything worth rebuilding
    drive-index install c d      make the symlinks for two drives

A single-letter label is a drive and needs no configuration: its root is
`/<letter>/` and its database is `/var/mlocate/<letter>.db`. Anything else —
an index built on another machine, a subtree, a registry tree — needs a
section in a config file. See `config/example.ini`.

Options `drive-index` has no opinion about are passed through to `locate`
untouched, so `-r`, `-b`, `-w` and the rest work as they always did.

## Why it is careful

Most of the indexes this builds describe disks that are not attached. They
are built once, from a disk that then goes in a drawer, and consulted for
months. For those the index is not a cache — it is the only surviving record
of what was on the disk.

So builds are written to a temporary path and renamed into place only after
the result is checked against what it would replace; a result that collapsed
is discarded rather than installed; and a build refuses to run when the
volume mounted at the root is not the one the index describes.

`doc/design/protecting-the-index.md` covers the reasoning.

## Requirements

Python 3.6 or later, and GNU findutils. No third-party packages.

## Verifying

    ./drive-index-certify [PYTHON]

The bar must pass before a commit. Pass an interpreter path to certify
against the oldest one you have.

## Documentation

    doc/design/      how what exists works, and why
    doc/proposal/    what is not done, each with a status line
    AGENTS.md        conventions, for a contributor or an AI agent

## License

MIT. Copyright (c) 2026 Philip Hardy Dye.
