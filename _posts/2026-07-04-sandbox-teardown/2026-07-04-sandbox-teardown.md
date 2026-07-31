---
title: "How Two Frontier Coding Agents Sandbox Untrusted Code"
date: 2026-07-04
categories: [AI Security, Blue Team, Sandbox]
tags: [sandbox, LLM agents, isolation, seccomp, bubblewrap, teardown]
---

# How Two Frontier Coding Agents Sandbox Untrusted Code

A comparative teardown of the OpenAI Codex CLI sandbox and the Anthropic sandbox-runtime (srt) that backs Claude Code.

> Part 1 of 2. This part explains what these sandboxes do, why it matters, and how each vendor builds them. It stays at the level of design and documented behavior. Part 2 runs a controlled attack and defense experiment against both.

## The short version

Both companies solve the same problem and reach for the same operating system parts, yet they assemble them differently. Four things are worth carrying through the whole read.

- The filter that matters most lives in the kernel, not in the program. That is what makes it something an untrusted command cannot switch off.
- The two sandboxes use the same tool, seccomp, for opposite jobs. Codex uses it to block the network. srt uses it to block local sockets and lets a namespace handle the network.
- Both leave read access wide open on purpose. They bet that blocking writes and blocking the network is enough to make a readable secret harmless.
- The interesting question is not what each sandbox blocks. It is what each one leaves open for the sake of getting real work done. Those openings are the map for part 2.

---

## Section 1. What a sandbox is and why an agent needs one

A coding agent reads your files, edits them, and runs shell commands to check its own work. That is most of its usefulness. It is also the whole problem. The moment an agent runs a command, that command inherits the same access you have. It can read anything you can read and reach anything you can reach.

Now add one fact. The instructions an agent follows do not all come from you. Some arrive inside the files, web pages, and tool output the agent reads while it works. A crafted document can carry text that the model treats as a new instruction. This is prompt injection, and it turns the agent's normal access into an attack surface.

Without isolation, a single injected instruction can lead to a command that reads SSH keys or cloud credentials, sends them to a server the attacker controls, overwrites a shell startup file so the attacker keeps a foothold, or connects to a local service socket and takes over the host.

![Threat model](/assets/img/posts/sandbox-teardown/fig1_threat_model.png)

A sandbox is the thing that stands between the command and those outcomes. It defines, ahead of time, where the command can write, what it can reach on the network, and which system calls it may use. Anything outside those limits fails.

Both vendors made the same two starting choices. First, they do not wrap the agent in a full container. They use operating system isolation features directly, which is lighter and starts faster. Second, they do not sandbox the agent process itself. They sandbox the shell commands the agent runs. The agent stays outside the box and the untrusted work happens inside it.

---

## Section 2. OpenAI Codex and Anthropic srt, side by side

Both projects are open source. Codex is written in Rust. srt is written in TypeScript with small C helpers. Both target macOS, Linux, and Windows. This post focuses on Linux, since that is the environment closest to where these agents run in production and where the isolation primitives are most transparent to inspect.

This section is the map. It says what each sandbox does and why the two designs diverge. Section 3 then explains how each piece actually works.

### Feature comparison

| Capability | OpenAI Codex | Anthropic srt |
|---|---|---|
| Linux filesystem isolation | bubblewrap. Read only root, writable roots layered on top | bubblewrap. Read only root, writable roots layered on top |
| Legacy filesystem path | Landlock, kept as a compatibility fallback | none, bubblewrap only |
| Read access default | generous, whole disk readable unless denied | generous, whole disk readable unless denied |
| Sensitive path protection | `.git`, `.agents`, `.codex` stay read only inside writable roots | `.git/hooks`, `.git/config`, dangerous dotfiles denied via a scan |
| Network isolation method | seccomp denies network syscalls | bubblewrap network namespace removes all interfaces |
| Network access when allowed | proxy routed mode plus a MITM network proxy | socat bridge over a unix socket plus a filtering proxy |
| seccomp main job | block the network | block unix domain sockets |
| io_uring blocking | yes | yes |
| ptrace and memory peeking | denied via seccomp | prevented via a nested PID namespace, plus non-dumpable init |
| Credential handling | environment inherited, sensitive env can be scrubbed | environment can be unset or masked, plus whole file credential masks |
| fail closed | refuses to run if the policy cannot be enforced | never falls back to running without isolation |
| Sandbox modes | read-only, workspace-write, danger-full-access | secure by default, poke holes explicitly |
| Weakened mode for nested hosts | compatibility paths when user namespaces are restricted | `enableWeakerNestedSandbox` binds host /proc |

### Known limitation comparison

These are limitations that each vendor documents, or that follow directly from the design and the code comments. None of these are claims that the sandbox was broken. They are the edges the vendors themselves point at, and they are where part 2 will push.

| Limitation | OpenAI Codex | Anthropic srt |
|---|---|---|
| Read of sensitive files by default | credentials remain readable unless a deny rule is added | credentials remain readable unless a deny rule is added |
| seccomp is a denylist | must chase alternate syscall paths, io_uring is the known example | must chase alternate syscall paths, io_uring is the known example |
| seccomp cannot inspect memory content | blocks by socket family, not by socket path | blocks by socket family, not by socket path, and must also cover socketcall variants |
| Inherited socket descriptors | not the focus of the network filter | documented as not blocked, including SCM_RIGHTS passing |
| Weakened isolation under containers | compatibility paths reduce strength | host /proc bind leaks process info from outside the box |
| Broad network allowlist | a wide allowed domain can become an exfiltration path | a wide allowed domain can become an exfiltration path |
| Proxy does not inspect TLS by default | request content is not examined | request content is not examined |
| Explicit escape hatch | `danger-full-access`, `--yolo` bypass | `dangerouslyDisableSandbox`, weaker nested mode |

### How to read these two choices

The tables show what each project does. What follows is a reading of why, based only on what the code and docs make visible. It is interpretation, not a claim about internal discussions, since those are not public. The evidence is the design itself.

The clearest way to see the difference is the role each project gives to seccomp. Codex makes seccomp the primary boundary and uses it to close the network. srt closes the network with a namespace and gives seccomp a narrower job. Everything else follows from that split.

**Codex, one strong filter.**

- Priority, predictable and simple enforcement. One in-process filter carries the network boundary, and the tool refuses to run if it cannot be installed.
- Upside, easy to reason about. Fewer moving parts means it is clear where a command gets stopped, and there is no silent downgrade to no protection.
- Downside, a lot rides on one denylist. A missed syscall becomes surface, and the deliberate `recvfrom` exception is one such seam.
- Downside, less granular. Network and unix socket control share the same filter, so filtering stays at the syscall level rather than the destination level.

**srt, layered jobs.**

- Priority, layered and fine-grained control. Each layer owns one job, and the network is closed at the namespace rather than at the filter.
- Upside, defense in depth. Domain-level filtering through the proxy and file-level credential masking have a natural place to live, which is hard to bolt onto a single filter.
- Downside, more moving parts. The socat bridge, the nested namespaces, and the proxy add complexity that has to be assembled correctly every time.
- Downside, more room for weak configurations. That same complexity is where nested isolation can degrade and where a misconfigured policy can quietly widen access.

**Where they agree is as telling as where they differ.** Both landed on bubblewrap for the filesystem even though both have access to Landlock. Both keep read access generous and bet on write limits plus network limits to contain damage. Both refuse to run rather than run unsandboxed. Both block io_uring specifically, because it is the known way to create sockets without the socket syscall. These shared choices suggest the constraints of the problem, not brand style, are driving most of the design.

![Architecture side by side](/assets/img/posts/sandbox-teardown/fig3_architecture.png)

---

## Section 3. How the pieces work

Section 2 covered what and why. This section is how. The three parts that carry the most weight come first and go deepest. The rest are shorter, since the design already told most of their story. Each part opens with the plain idea so you can follow it without a systems background. Precise implementation notes are collected at the end of each part for readers who want them.

### 3.1 System call filtering with seccomp

**The idea.** A program cannot touch hardware or the network on its own. When it wants to open a file or a socket, it asks the operating system kernel to do it. That request is a system call, or syscall. Think of the kernel as the only staff member allowed behind the counter, and a syscall as the order slip a program hands across that counter. Every real action goes through a slip.

Here is the point about trust. If you tried to enforce rules with checks written inside the program, the untrusted command runs in the same space as those checks and can ignore or overwrite them. The guard and the suspect would be the same person. seccomp avoids that. It registers a filter with the kernel, so the check happens at the counter, on the kernel side. The command cannot reach the counter without passing the filter, and it cannot take the filter back off once it is set.

![Where seccomp sits](/assets/img/posts/sandbox-teardown/fig2_syscall_boundary.png)

**Why it matters.** syscalls are the only path an untrusted command has to the network and to local services. If this layer is missing or has a gap, a command can open a socket to send data out, or connect to a local service socket and pivot from there. Blocking the right syscalls closes that path at the one place it cannot be avoided.

**The two designs.** srt uses seccomp for one job, stopping the command from creating unix domain sockets, which are the sockets used to talk to local services such as a docker daemon. Normal TCP and IP sockets pass, because the network is handled elsewhere by the namespace. Codex uses seccomp for the opposite job, closing the network by denying the syscalls that make and use outbound connections, while allowing local sockets so ordinary tooling still works.

![Same tool, opposite jobs](/assets/img/posts/sandbox-teardown/fig4_seccomp_roles.png)

Both filters are denylists. They allow by default and block specific calls, which means each one has to anticipate every alternate route to the same capability. That is why both block io_uring, a newer interface that can create a socket without calling the socket syscall and would otherwise slip past the rule.

*Notes from the code.* The filter is a small kernel program in a format called BPF, built by a library rather than written by hand. srt uses the C library libseccomp; Codex uses the Rust library seccompiler. The language is just the tool that builds the filter, so the honest comparison is which syscalls each one blocks. Two details are worth flagging for part 2. Codex leaves `recvfrom` allowed on purpose, with a comment that tools like `cargo clippy` need it for their helper processes. And seccomp cannot read the memory a syscall points at, so it filters a socket by its family rather than its path, and it has to cover the older `socketcall` entry point as well as the direct `socket` call. The srt filter comments on exactly those `socketcall` variants.

### 3.2 Filesystem isolation

**The idea.** On Linux, what a program sees as the filesystem is not fixed. It is a view that can be assembled per process using mounts. bubblewrap builds such a view. It presents the real disk as read only, lays a small number of writable spots on top, and hides or replaces specific files. The command runs inside that assembled view and cannot see past it.

**Why it matters.** If a command can write outside its work area, it can change a shell startup file to run attacker code later, replace a program on the system path, or edit a git hook that fires on the next commit. Write access is where persistence and privilege escalation begin, so the write boundary is the one that matters most.

**The two designs.** Both start the same way. The whole disk is mounted read only, then the working directory is made writable on top. Both protect sensitive spots inside the writable area. Codex keeps `.git`, `.agents`, and `.codex` read only even when their parent is writable. srt denies `.git/hooks` and `.git/config` and a set of dangerous dotfiles, and it scans for these paths so nested copies are caught too.

![Filesystem model](/assets/img/posts/sandbox-teardown/fig5_filesystem.png)

The design bet is in what is not protected. Read access is generous by default, so SSH keys and credential files stay readable unless you add a deny rule. Both vendors document this. The reasoning is that reading a secret is less dangerous if the command cannot write anything important and cannot reach the network to send the secret out. Whether that bet holds under pressure is a question for part 2, not a claim to make here.

*Notes from the code.* srt also handles a subtle case where a component of a path is a symlink that an attacker could delete and replace with a real directory holding malicious content. It detects this and mounts over the spot to block the swap. The code comments name this as a symlink replacement attack.

### 3.3 Network isolation

**The idea.** Blocking the network can happen at two different places. One is the network namespace, a kernel feature that gives a process its own empty set of network interfaces, so there is nothing to connect through. The other is the syscall filter from 3.1, which refuses the calls that open a connection. Either way, any allowed traffic ends up routed through a proxy, a checkpoint that can allow or deny based on the destination.

**Why it matters.** Network access is the exit door for stolen data. A command can read a secret, but it only becomes a breach when the secret leaves the machine. Closing or filtering the network is what stops exfiltration, and it also stops a command from pulling in more attacker code.

**The two designs.** srt removes all network interfaces with a network namespace, so the box starts with no network at all, then runs a small relay called socat that bridges a socket inside the box to a filtering proxy on the host. The hard boundary is the namespace. Codex closes the network with seccomp instead, and when access is allowed it switches to a proxy-routed mode that permits the sockets needed to reach a local bridge while still denying the rest. The hard boundary is the filter.

![Two ways to close the network](/assets/img/posts/sandbox-teardown/fig6_network.png)

The shared weakness is the proxy. When it does not inspect TLS, it judges traffic by the destination name the client provides, so a broad allowed domain such as a whole code host can become a path for data to leave. Both vendors document this, and it is a natural target for part 2.

### 3.4 Process isolation

Even inside a filesystem box, a command can look sideways at other processes if they share a process view. The `/proc` directory exposes information about running processes, including their environment variables, which often hold secrets. So both sandboxes give the command its own PID namespace and mount a fresh `/proc`, which limits its view to itself. Both also deny `ptrace` and the memory-reading syscalls and set `no_new_privs`, which seccomp requires and which also blocks a class of privilege escalation.

The caveat is that the fresh `/proc` depends on the host allowing it. Inside a restricted container, bubblewrap may not be able to mount one, and the weakened mode binds the host `/proc` instead. srt's own code comments that this leaks information about processes outside the box. This is why the reproduction for this series runs on a real virtual machine rather than in a container, so the sandbox runs at full strength.

### 3.5 Credential protection

Secrets reach a command as files on disk and as environment variables handed down from the parent. A sandbox that only guards files still lets the command inherit every secret in the environment, and agent workflows are full of tokens. Both sandboxes can strip sensitive environment variables before the command runs. srt goes further with whole-file credential masks, binding a harmless placeholder over a real credential file so a command that reads the path gets the placeholder instead of the secret. This is the area that connects most directly to the broader problem of keeping secrets away from agents, and it is where the two designs show the most room to grow.

### 3.6 Fail closed

A safety mechanism can fail open, running the command without protection when it cannot enforce its policy, or fail closed, refusing to run. Fail closed is the safer default for untrusted code, because a sandbox that silently downgrades is worse than none, since the user still believes they are protected. Both sandboxes fail closed. Both also provide clearly named escape hatches for users who accept the risk, such as `danger-full-access` and `--yolo` in Codex and `dangerouslyDisableSandbox` in srt. The naming is deliberate, so turning off the sandbox is always a conscious act. Whether an agent can reach for those hatches on its own is a thread for part 2.

---

## Where part 2 goes

This teardown mapped what each sandbox claims and how it is built. It stayed away from any claim that a given boundary can be crossed. Part 2 takes the limitation table and turns it into a controlled experiment, holding the attack constant and varying the sandbox and the environment, to measure which boundary stops which attempt and where the documented edges show up in practice.

The reproduction runs on a single cloud Linux virtual machine so that both sandboxes run at full strength, rather than in a nested container where their own code says protection is reduced. Environment details and exact versions will accompany part 2 for reproducibility.

---

## References

[1] OpenAI. Codex CLI source code. https://github.com/openai/codex

[2] Anthropic. sandbox-runtime source code. https://github.com/anthropic-experimental/sandbox-runtime

[3] Anthropic. Claude Code Sandboxing. https://code.claude.com/docs/en/sandboxing

[4] OpenAI. Sandboxing, Codex documentation. https://developers.openai.com/codex/concepts/sandboxing

[5] OpenAI. Agent approvals and security, Codex documentation. https://developers.openai.com/codex/agent-approvals-security

[6] OpenAI. CLI, Codex documentation. https://developers.openai.com/codex/cli

[7] OpenAI. Command line options, Codex CLI reference. https://developers.openai.com/codex/cli/reference

[8] OpenAI. Configuration reference, Codex documentation. https://developers.openai.com/codex/config-reference

[9] OpenAI. Codex full documentation, single-file Markdown export. https://developers.openai.com/codex/llms-full.txt
