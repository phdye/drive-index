# Groups: one command prefix over several environments

Status: accepted 2026-09-26, implemented; record in
doc/decisions/2026-09-26.groups.md

## The problem

Three environments of one application are indexed as `x`, `y` and `z`, aliased
`app-dev`, `app-test` and `app-prod`. That works, and it does
not scale: these will not be the only dev/test/prod systems indexed,
and each new topical area multiplies into three more sections whose only
difference is one word. That duplication is what this program was written to
remove from a family of shell scripts; reintroducing it in the config file is
no better.

The wanted shape is one prefix with an environment selector:

    app-locate PATTERN          every environment
    app-locate --prod PATTERN   just Production
    app-updatedb                rebuild every environment
    app-updatedb --dev          rebuild Development

## Shape

    [group.app]
    default = all
    member.dev  = x
    member.test = y
    member.prod = z

`member.<selector>` rather than a bare `dev = x`, so that adding a group key
later -- `alias`, `description` -- cannot collide with somebody's selector.

A group is an indirection, not an index. It resolves to a list of labels and
everything downstream is unchanged, so `status`, `bulk`, the metadata
sidecars and the atomicity guard all keep working with no second code path.

## Defaulting to all members

Both verbs default to every member. `--prod` narrows; multiple selectors
union, so `--dev --prod` is two of the three.

An earlier draft of this had `updatedb` default to one environment, on the
grounds that an accidental sweep might "rebuild Production when you meant
Development". That reasoning was wrong and is recorded here so it is not
reintroduced. Indexing never writes to the mount: it reads the tree and
writes the index to local storage. The collapse guard and the atomic rename
already protect the index itself. The real cost of an accidental all-member
build is time and read load, not damage.

What does mislead is the opposite case. Searching one environment while
believing you searched all of them produces a finding about the wrong system,
and nothing in the output says so. So widening is the safe default and
narrowing is the case that needs announcing: when a selector resolves a group
to fewer than all its members, `locate` names the members it searched at
normal verbosity, on stderr, where it stays out of piped output but survives
into a pasted transcript.

## Where the selectors come from

Unknown options already fall through to a passthrough list on their way to
`locate`. Config is loaded after parsing, so by the time a group is known to
exist, its selectors are sitting in that list waiting to be claimed. Pulling
them out before the rest is forwarded is the whole mechanism.

## Validation, which is the half that earns its keep

Every selector has to be checked against two option sets when the file is
read, alongside the existing alias rules:

- A selector matching an option this program owns -- `--json`, `--limit` --
  would be consumed by the parser and never reach the group.
- A selector matching one of `locate`'s -- `--basename`, `--regex` -- would be
  stolen from `locate`.

Both are configuration errors and are refused at load, the way a one-letter
alias and a duplicate alias already are. This is the part most likely to be
skipped and most likely to bite.

## Decisions taken

**`--all-indexes` means the group's members** when the label is a group,
rather than every index on the machine. "is this file anywhere in the group"
is the search that gets made.

**`bulk` over a group** is the existing bulk with a member list: members
attempted independently, a failure not stopping the others, `bulk = no` still
honoured, non-zero exit if any member failed. That last matters more with all
as the default, since a sweep that lost one environment's index must not
report success.

**Build order is config order**, so it is deterministic and the cheap
environment can be put first.

**A bare `app-updatedb` announces what it is about to build** before
starting rather than only after. Three full scans deserve a sentence, and it
makes `--dry-run` genuinely useful here rather than a formality.

**A selector may not name a group.** Nesting is cheap to allow and cheap to
regret; one level is almost certainly enough, and a cycle check is code
nobody wants to own.

## Interaction worth knowing

The group's members are separate mounts, so a hit self-identifies: `/x/...`
versus `/z/...` says which environment without any extra machinery. That is
luck, not design. A group whose members are subtrees of one drive would give
no such signal, and the first one built that way will want the searched label
printed alongside each hit rather than only in the summary line.

## Cost

Roughly 120 to 150 lines with the validation, plus bar checks and a design
note. The parsing is the easy half.
