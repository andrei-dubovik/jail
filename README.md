# A wrapper around bwrap and pasta for per-project Claude sandboxes

## Usage

Run `jail project-dir` to set up a sandboxed environment inside `project-dir`.

The local directory `project-dir/.jail/claude` gets mounted on `~/.claude` inside the sandbox. Select Claude configuration files (auth, user-level config, plugins) from the host `~/.claude` directory get overlaid on top. That is, base configuration is shared between projects but sessions, memory, caches—those stay project-specific.

## Design Choices
- The network namespace is isolated with [pasta](https://passt.top). By default, abstract unix sockets and local services are not available from inside the sandbox.
- The user, mount, pid, ipc, and cgroup namespaces are isolated with [bwrap](https://github.com/containers/bubblewrap).
- `/usr/` is shared read-only with the sandbox. (Sharing `/usr` in full is convenient, however it also increases the attack surface for malicious scripts.)
- There is additional modular configuration for select programs. Currently, the following programs get configured: Python, R, LaTeX.
- As noted, Claude configuration is split between shared and project-specific config files, where notably sessions and memory are mounted as project-specific. This is a more subjective design choice, and it comes from my own preference.
- Sandboxes get their own git configuration (user.name=Claude, user.email=noreply@anthropic.com). Given that project directories are bind-mounted, it is possible to simultaneously work and commit changes as Claude in one terminal, and to work and commit changes as a human developer in another terminal.

Claude credentials are currently accessible from within the sandbox, specifically they get mounted to `~/.claude/.credentials.json`. It should be possible to store the credentials outside the sandbox and reroute the sandboxed Claude Code via a custom gateway that then takes care of authentication with the Anthropic servers. This security enhancement is currently not on the todo list.

## Configuration

The script uses a global git-style configuration file located at `~/.config/jail.conf`. The following example configuration covers all available options.

```
[dir "/realpath/to/project-dir"]
  tcpns = 8080  # gets passed to --tcp-ns; defaults to none
  udpns = 53    # gets passed to --udp-ns; defaults to none

  # The following options allow to specify additional paths that will be
  # bind-mounted as is. These options are intended for when the project folder
  # contains symbolic links with outside targets. Once sandboxed, those symbolic
  # links will be dead. Bind-mounting their targets restores the links. For
  # security reasons, automatic mounting of symbolic links is not done (symbolic
  # links can be changed from within the sandbox).
  rw = /var/tmp/cache1
  rw = /var/tmp/cache2
  ro = /var/tmp/data
```

The config file should be viewed as sensitive, because it allows configuring arbitrary sandbox escaping. For that reason, it is implemented as a global config file rather than as a set of local config files located inside project directories. Doing the latter would risk pulling a compromised local config file from a project repository.

## Compatibility

The script is currently written for the standard installation paths on Arch Linux. There is no auto configuration in that regard. The paths will likely need to be adjusted when running on other distributions.

## Security

This repository should be viewed as an example script only and should not be used in production where security guarantees are required. I do not have a background in security. This code has not received a security audit.

Having disclaimed that, let me nevertheless briefly discuss security choices when running Claude Code.

### Unsandboxed Claude

Claude Code can be run without a sandbox. In theory, if every tool call is inspected, that could be a secure option. In practice, doing so would of course defeat the whole purpose of agentic automation. And once blanket permissions are given to, e.g., `python`, anything goes.

Any program can have unintended behaviour, but arguably there is a difference between running, say, unsandboxed neovim and running unsandboxed Claude Code. The latter is stochastic by design and can cause data loss or a data leak by accident if not specifically by malign intent.

### Claude with a Native Sandbox

Claude Code has a built-in sandbox, available on Linux and MacOS, that can be enabled with the `/sandbox` command. Claude's sandbox has limited configuration. For instance, it offers only write-protection (reading outside the sandbox is allowed) and there are limited options for more granular network sandboxing, say local services and abstract unix sockets are still available. Also, IMHO, Claude Code is fast-evolving software and it would not be surprising if the default security settings change without much notice.

### Claude in a Container

A container is a lightweight solution, but containers share the kernel code and therefore remain potentially vulnerable to local privilege escalation exploits. The same applies to `bwrap`, of course. A different type of drawback, however, is usability. Standard containers such as Docker or Podman not only provide sandboxing but also provide reproducible system environments. The latter necessitates image management, which out-of-the-box is more work than simply `jail project-dir`.

### Claude in a Virtual Machine

Short of running Claude Code on a dedicated machine, full virtualization provides the best security, but it also complicates usability still further compared to containers. Initially, when coding agents were becoming popular, I was running mine in Qemu. That brought along yet another Linux system to manage. Eventually, I decided to trade some security to gain usability, hence this script.
