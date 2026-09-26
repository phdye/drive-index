# Dispatch and configuration

How the program decides what to search and what to build.

## One executable, many names

`drive-index` reads its own `argv[0]`, takes the basename, and splits it at
the **last** dash. The left side is the label, the right side must be an
action. So `c-locate` is label `c`, action `locate`, and
`web-01-locate` is label `web-01` rather than label `web`.

Splitting at the last dash rather than the first is what allows labels that
are not single letters. The original scripts assumed a drive letter, which
broke as soon as `4tb-locate` existed.

Invoked under its own name, the label and action are the first two
arguments instead: `drive-index c locate PATTERN`. Nothing therefore needs a
symlink in place to be tested, which is why the bar can exercise labels that
have no symlinks at all.

## Symlink targets are bare basenames

`install` writes `c-locate -> drive-index`, not an absolute path. The
directory can then be moved or copied without every link breaking.

This has a consequence worth knowing before changing `install`: it computes
where to put the links from `os.path.realpath` of its own path. If the
program itself were reached through a symlink from somewhere else, the links
would land beside the *resolved* file rather than beside the invoked one.
That is currently correct and would become wrong under one of the options in
`doc/proposal/install-and-deployment.md`.

## Resolution order

Four sources, most specific first:

1. the command-line option
2. `DRIVE_INDEX_<OPTION>` in the environment
3. the config files, a label's own section outranking `[defaults]`
4. the built-in default

Every resolved value records which of the four it came from, and failures
print it. This is load-bearing rather than cosmetic. The option is `--db`
while the config key is `database`, and an early version used one string to
look up both — so the config layer was skipped entirely and nothing noticed,
because the configured path and the built-in default were the same string.
The provenance line is what makes that class of defect visible.

## Config layers

Read least specific first, each merging over the last:

    /etc/drive-index.conf
    $XDG_CONFIG_DIRS/drive-index/{config.ini,conf.d/*.ini}
    ~/.drive-index.ini
    $XDG_CONFIG_HOME/drive-index/{config.ini,conf.d/*.ini}

`XDG_CONFIG_DIRS` runs the opposite way to every other list here — earlier
entries win — so it is walked in reverse.

`--config` or `DRIVE_INDEX_CONFIG` replaces the whole stack rather than
adding to it, and says so at normal verbosity, because a stray value in an
environment is exactly how that happens unnoticed.

**There is deliberately no per-directory layer and no ancestor walk.** Which
database belongs to drive C is a fact about the machine, not about the
directory you are standing in. A `.drive-index.ini` found two levels up would
be meaningless at best. This is also why `--config-stop` does not exist: its
purpose is to bound a walk there isn't one of.

Each file is parsed into its own `ConfigParser` rather than merged into one,
because a merged parser cannot say which file a value came from.

## Labels, aliases, and the reserved namespace

A single-letter label is a drive: root `/<letter>/`, database from
`db_file_model`. No configuration at all. Anything else must have a section,
and is refused with the name of the section it wants if it does not.

Two alias rules, both enforced when the file is read rather than when
someone relies on them:

- **A one-letter alias is an error.** It would shadow a drive, and the
  guarantee that `c-locate` searches drive C is the one thing a user should
  never have to check.
- **A duplicate alias is an error.** Otherwise which index it reached would
  depend on the order sections happened to be read in.

## db_file_model

A template over `{name}`, `{host}` and `{os}`, defaulting to
`/var/mlocate/{name}.db`.

Named fields rather than positional. The design this replaced used a single
positional `{}`, which could not express a host-scoped path — and one of the
index already in use lives under a directory named for the host that built
it, which a single positional field cannot produce.

## Prune paths

`prunepaths` replaces the inherited set; `add_prunepaths` extends it; the
`--prune` option adds to whatever those produced. `--no-prune` discards all
of it.

Stated in config rather than inherited from whatever `updatedb` was compiled
with, so `drive-index config` can show the effective set and say where each
part of it came from.

Two failure modes are caught rather than allowed:

- **A prune path that does not exist** prunes nothing and says so to nobody.
  Reported at normal verbosity. The scripts this replaced carried
  a `proc` path under an installation root, which has never existed on
  disk, since `/proc` is synthesized by Cygwin — and one written as
  `/d/-/.*/root/dev`, which `find -path` reads with a literal dot and so
  matched only a directory actually named `.*`.
- **A prune path containing whitespace** is refused outright. See `AGENTS.md`
  on why there is no escaping form.
