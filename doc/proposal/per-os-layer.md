# A per-OS layer

Status: proposal 2026-09-26, not started

## The gap

Three things vary by operating system, and none can currently be expressed:

**Where the tools are.** `locate` and `updatedb` are found on `PATH`. That
works on one machine and is wrong on two. The design this program grew out of
carried an explicit `applications` block naming the binaries per OS,
precisely because Cygwin and Linux do not put them in the same place.

**Where the databases go.** `db_file_model` is a single value. The prior
design used `/usr/local/var/mlocate/{}.db` on Cygwin and `/var/mlocate/{}.db`
on Linux. One file cannot currently say both, so a config shared between
machines has to be edited per machine — which defeats having one.

**Which label is meant by default.** Covered separately below.

The project declares a Python 3.6 floor specifically so it can run on RHEL
8.10. That floor is pointless if the configuration cannot describe two
machines at once.

## Shape

An `[os.<name>]` section, matched against a normalized system name
(`Cygwin`, `Linux`), sitting between `[defaults]` and `[drive.<label>]` in
precedence:

    [os.Cygwin]
    db_file_model = /usr/local/var/mlocate/{name}.db
    locate = /usr/bin/locate
    updatedb = /usr/bin/updatedb

    [os.Linux]
    db_file_model = /var/mlocate/{name}.db

The `{os}` field already exists in `db_file_model`, which covers some cases
without any new section — `/var/mlocate/{os}/{name}.db` works today. It does
not cover the tool paths, and it forces the OS into the path shape whether
that suits or not.

## Open questions

Whether `locate` and `updatedb` should also be overridable per label. An
imported index built by mlocate rather than findutils would need a different
`locate` to read it, and the two are not compatible. This may be worth
having, or may be a case that never arises.

Whether to verify the configured binary exists at startup or let the failure
come from `subprocess`. The second is simpler; the first gives a diagnostic
naming the config key and the file it came from, which is the convention
everywhere else here.

## Cost

Small. The lookup already walks a section list; this adds one entry to it.
The bar would need a case that fakes the OS name, which means the normalized
name has to be injectable rather than read directly from `platform.system()`
at the point of use.

## Why it has not been done

Nothing has run this anywhere but the one machine, so every value that would
go in the OS section currently has exactly one correct answer. The first
attempt to run it on actual RHEL is what will make this urgent, and that has
never been attempted.
