# Start here: the safety model, and checking vs. changing

This is the document `README.md`'s *Your first hour in order* points you to. It is not a
tutorial and not a reference — it is the difference between looking at a machine and changing
one, and the rules that make the second safe. Everything here is enforced by the program, not
promised in prose; the test suite keeps the two honest.

## Checking never changes anything

`shall check`, `shall plan`, `shall eval` and every `--dry-run` are read-only. `--dry-run` is
not a flag some commands remember to honour — it is enforced by the single function every file
Shall owns is written through, so a preview cannot quietly record state, write a snapshot or
touch a package. **`shall --dry-run sync` is the command to try first on any machine.**

The one question worth asking before you allow anything: *what would this command change?*
`check` answers drift, unmanaged software and backend health; `plan` freezes what `sync` would
do; `--dry-run` previews it. All three are free to run as often as you like.

## Changing is transactional

When you do let Shall change the machine:

- **The file is the truth.** `sync` makes the machine match your manifests. It adds what is
  missing, removes what is no longer listed, and leaves everything else alone. Every change is
  first a change to a file you own.
- **A write-ahead log records every mutation that cannot be recomputed** — every package, every
  `exec:` script, every `@undo=` — *before* it runs, whichever command issues it. If Shall is
  killed mid-command, the next run heals: packages are replayed or reverted, and an interrupted
  script is reported by name. A crash that went unattended for hours is still healable.
- **Snapshots.** btrfs, ZFS, Timeshift and Windows Restore Points are taken automatically before
  a sync or upgrade where a provider exists. Filesystem-level rollback is Linux-first: macOS has
  no adapted provider yet, so there the git history is the undo.
- **Non-interactive refusals.** `sync`, `rollback` and `remove-orphans` refuse to apply
  unconfirmed changes in a pipe, cron job or CI run without `--yes`.

## The removal guard

**Every path that removes anything** — packages *and* the resources a declaration puts in
place (`link:`, `service:`, `setting:`, `shim:`, `schedule:`, `repo:`) — goes through one
guard. It refuses a removal that:

- exceeds a ceiling (`max_removals`, `max_extra_removals`, `max_port_closures`,
  `max_total_changes`),
- is out of proportion to what Shall manages (`purge_ratio`),
- touches a protected package (a built-in list, anything you add via `protected_packages`, and
  the OS's own essential flags where it has them),
- or trips a `[guard]` policy rule.

`shall protected` prints the effective rules. `--yes` is deliberately **not** an override — every
script and CI job passes `-y`, and an unattended run is exactly the one that cannot notice a
system being taken apart. Protection is a refusal, not a confirmation: nothing overrides it.

## Hooks and adapters are locked

A repo you clone cannot teach your machine anything you have not read. `after_install` hooks
and everything in `adapters/` (see *The eight things you can teach it* in the README) go through
the approval ledger: the first sync after you write or change one refuses it by name and tells
you to run `shall lock`.

## Your first change

1. `shall init` — scaffolds a config repo with one active profile.
2. `shall check` — read-only; what needs you.
3. `echo 'cargo:ripgrep' > ~/.config/shall/modules/tools.txt` and `echo 'use tools' >> ~/.config/shall/profiles/Main`.
4. `shall --dry-run sync` — preview it.
5. `shall sync` — make it so.
6. `shall git status` — the change is a commit's worth of your own file.