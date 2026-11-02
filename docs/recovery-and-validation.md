# Recovery and validation

[← Back to README](../README.md)

The steps below are in the order I did them. For each one: what I did, why, and how I checked it before moving on.

---

## Part 1: getting Node A back into replication

### 1. Fix the encryption settings

**What:** brought Node A's encryption-settings definitions in line with its peers.

**Why first:** without matching definitions, the rebuilt node couldn't work with encrypted data from the rest of the topology. There was no point initializing until this was fixed.

**[VERIFY: exactly how you fixed it. For example, exported the definition from a healthy node and imported it on Node A? Only add commands you actually ran.]**

### 2. Re-enable the `userRoot` backend

**What:** the `userRoot` backend on Node A was disabled and had to be re-enabled before replication could be set up for it. **[VERIFY: why it was disabled, if you know.]**

```bash
# Illustrative. Confirm the exact command you used.
bin/dsconfig set-backend-prop \
  --backend-name userRoot \
  --set enabled:true
```

### 3. `dsreplication enable`, then `dsreplication initialize`

**What:** added Node A back into the topology with `dsreplication enable`, then ran `dsreplication initialize` to load its data from a healthy peer.

**Risk I watched for:** `initialize` replaces the target server's data with the source server's data. Running it in the wrong direction would have pushed Node A's empty or incomplete data onto a good node. Before running it, I made sure the source was a healthy node and the target was Node A. **[VERIFY: which node you used as the source.]**

### 4. Clear lockdown mode

**What:** after initialization, Node A was in lockdown mode, so it wasn't ready for normal traffic.

**[VERIFY: how you cleared it. `bin/leave-lockdown-mode`? A restart? Fixing something first?]**

### 5. Confirming replication health

Working with Ping support, we confirmed Node A's replication state:

| Check | Result |
|---|---|
| Configured replication peers | 3 |
| Replication connections | 3 inbound, 3 outbound |
| Replication backlog | 0 |
| `userRoot` entry count | 110,885 |
| Failed replication operations | 0 (Node A and Node B) |

**Scope of these numbers:** "0 failed" is the replication counter. It means no replicated changes failed to apply. It does **not** prove that no client LDAP writes failed during the incident, and I'm not claiming that.

**[VERIFY: entry counts and failed-op counters on Node C and Node D. Keep the `dsreplication status` / `status` output as evidence.]**

By the usual checks, Node A was recovered. It wasn't, quite. See [Investigation, Part 2](investigation.md#part-2-the-config-drift).

---

## Part 2: fixing the config drift

### 6. Enable the password policy state virtual attribute and plugin

**What:** enabled the Password Policy State JSON virtual attribute and the Modifiable Password Policy State plugin on Node A, with the same base DN scope and filter settings as Node B.

```bash
# Illustrative. Check exact object names with list-virtual-attributes / list-plugins.
bin/dsconfig set-virtual-attribute-prop \
  --name "Password Policy State JSON" \
  --set enabled:true

bin/dsconfig set-plugin-prop \
  --plugin-name "Modifiable Password Policy State" \
  --set enabled:true
```

**Validation:** re-ran the same `ldapsearch` against the same test user. `ds-pwp-state-json` was now returned on Node A.

### 7. Recreate the password-policy virtual attributes

**What:** recreated the three custom virtual attributes on Node A to match Node B exactly, including their `evaluation-order-index` values (10, 2 and 2).

**Why the order index mattered:** it controls which rule applies when more than one could match an entry. Copying the attributes without the same order could have assigned the wrong policy to some users, which is the exact problem I was fixing.

**Validation:** the test user resolved to the correct custom policy on Node A, same as on Node B.

---

## Part 3: why I didn't apply the full config diff

I generated a configuration comparison between Node A and Node B. The obvious move was to take every difference and apply it to Node A.

Ping support advised against that, and once I read through the diff, I understood why. It mixed shared settings with settings that are *supposed* to be different on each server: certificates and keystores, trust settings, debug logging. Applying it wholesale could have broken Node A's TLS setup or copied Node B's server-specific settings onto it.

So the plan became:

1. Split the shared differences into functional groups (password validators, virtual attributes, password policy settings, and so on).
2. Take a config snapshot before each group.
3. Apply one group, then validate it with real queries before moving on.
4. Leave node-specific values alone.
5. Handle index differences last, as their own separate change.

**[VERIFY: current status of the config-parity work. Is it finished? If yes, how did you confirm parity at the end?]**

---

Next: [Lessons learned →](lessons-learned.md)
