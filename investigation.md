# Investigation

[← Back to README](../README.md)

This covers what I saw, what I checked, and how I got to each cause. The actual fixes and validation are in [Recovery and validation](recovery-and-validation.md).

---

## Part 1: the failed node

### What happened

Node A needed a full rebuild. **[VERIFY: why. Failed upgrade? Corrupt backend? Disk? A change gone wrong?]**

**[VERIFY: what was the first symptom, and who or what noticed it (an alert, a user report, a failed job)?]**

**[VERIFY: did the other three nodes keep replicating with each other and serving traffic while Node A was out?]**

**[VERIFY: one or two sentences on why you rebuilt instead of repairing in place.]**

### The encryption-settings mismatch

PingDirectory can encrypt data at rest using encryption-settings definitions. In a replicated topology that uses this, the servers need to share the same definitions. A server that doesn't have the right definition can't work with data that was encrypted using it. **[VERIFY this explanation against the Ping 10.3 docs before publishing.]**

The rebuilt Node A didn't match its peers, so this had to be fixed before replication could work.

**[VERIFY: how you noticed the mismatch. An error message? `encryption-settings list` output compared across nodes? Only add commands you actually ran.]**

### Lockdown mode

After initialization, Node A came up in lockdown mode. In lockdown, PingDirectory only accepts requests from administrators. So the server can be running and still clearly not be ready for normal traffic.

**[VERIFY: what caused it and what you checked to find out.]**

---

## Part 2: the config drift

This is the part that wasn't obvious. Replication on Node A was already confirmed healthy (see [Recovery and validation](recovery-and-validation.md#5-confirming-replication-health)).

### The symptom

I ran the same query against the same test user on Node A and Node B. I asked for `ds-pwp-state-json`, a virtual attribute that shows a user's password policy state as JSON.

Node B returned it. Node A returned nothing.

```bash
# Sanitized. Real hostnames, DNs and paths replaced.
bin/ldapsearch \
  --hostname node-a.example.com --port 1636 --useSSL \
  --bindDN "cn=admin-user" \
  --bindPasswordFile /path/to/bindPasswordFile \
  --baseDN "cn=test-user,ou=people,dc=example,dc=com" \
  --searchScope base \
  "(objectClass=*)" ds-pwp-state-json
```

Same data, different answer. That told me the problem wasn't in the data. It was in how the server was configured to process it.

A small thing that slowed me down: one of the test entries had an RDN ending in an escaped trailing space. Copying the DN by hand kept breaking my queries. I switched to capturing DNs straight from `ldapsearch --dontWrap` output with command substitution, so the exact DN went into the next command untouched.

### Finding 1: a virtual attribute and a plugin were disabled

I compared the relevant server configuration on both nodes:

| Config object | Node A | Node B |
|---|---|---|
| Password Policy State JSON (virtual attribute) | disabled | enabled |
| Modifiable Password Policy State (plugin) | disabled | enabled |

That explained why `ds-pwp-state-json` was missing on Node A.

### Finding 2: users were getting the wrong password policy

This one was more serious. On Node B, the test user resolved to a custom password policy. On Node A, the same user fell back to the default policy.

In this environment, password policies weren't stored on each user. They were assigned by three custom virtual attributes, each one setting `ds-pwp-password-policy-dn` for a different group of entries. Node B had all three. Node A had none of them.

If Node A had taken real bind traffic in that state, users who landed on it would have been checked against the default policy instead of their real one. That means different lockout and password rules depending on which server they happened to hit.

### Root cause

Replication copies directory data between servers. It doesn't copy each server's own configuration. That lives separately on every server.

When Node A was rebuilt, it started with default configuration. Anything that had been customized on the old Node A over time, like enabled plugins and custom virtual attributes, was gone unless someone added it back. `dsreplication initialize` brought the data back. It couldn't bring back the config, because config was never part of replication.

So every replication health check passed, and the node still behaved differently from the others.

**Confirmed vs. not confirmed:**

- **Confirmed:** the specific config differences above, and that fixing them made Node A return the same results as Node B.
- **Not confirmed:** whether these differences existed on the old Node A before the rebuild, or only appeared because of it. Without a config export from before the rebuild, there's no way to prove which. "Lost in the rebuild" is the most likely explanation, not a proven one.

---

Next: [Recovery and validation →](recovery-and-validation.md)
