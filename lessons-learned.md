# Lessons learned

[← Back to README](../README.md)

---

## What I'd do differently

**Capture config before rebuilding.** Before tearing down a node, I'd export its configuration so there's a known baseline to compare against afterward. Diffing the rebuilt node against a peer works, but a before/after of the same node is much cleaner. It would also have answered the one question I couldn't: whether the drift was caused by the rebuild or was already there.

**Don't stop at "replication is healthy."** Backlog 0 and matching entry counts prove the data is in sync. They say nothing about how each server processes that data. Next time I'd add a behavior check: the same queries against the same test users on every node, comparing password policy state and effective policy.

**Keep custom config as code.** Custom virtual attributes and plugin settings should live in version-controlled `dsconfig` batch files. Then a rebuilt node gets the same config by running a script, instead of depending on someone remembering what was changed by hand.

**Use a bind password file from the first command.** Passwords typed on the command line end up in shell history. `--bindPasswordFile` should be the default, not something you switch to halfway through.

**Be sure of the direction before running `initialize`.** It overwrites the target. Write down the source and target before pressing enter.

---

## A post-rebuild checklist I'd use now

- [ ] Encryption-settings definitions match the rest of the topology
- [ ] Backends enabled as expected
- [ ] `dsreplication status`: all peers connected, backlog 0
- [ ] Entry counts match on **every** node, not just the recovered one
- [ ] Server not in lockdown mode
- [ ] Same test queries return the same results on every node (including password policy state and effective policy)
- [ ] Config compared against a healthy peer, with node-specific settings (certs, keystores, trust, logging) excluded
- [ ] Shared config differences applied in small groups, each validated before the next

---

## Closing

The replication recovery was the part I expected to be hard, and it was: encryption settings, a disabled backend, lockdown mode. But the issue that would have actually affected users was the quiet one afterward. A node with correct data and the wrong configuration passes every replication check.

[← Back to README](../README.md)
