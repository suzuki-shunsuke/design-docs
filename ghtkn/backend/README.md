# Support for backends other than keyring

## Status

Draft

## Motivation

ghtkn uses the OS keyring (macOS Keychain / Linux libsecret / Windows DPAPI, etc.) as the storage destination for access tokens.
While this is a reasonable default for desktop use, there is a problem that ghtkn cannot be used in the following environments where keyring is unavailable:

- Inside containers (Docker / devcontainer, etc.)
- Development VMs (Lima / Colima / EC2, etc.)
- Minimally configured Linux without D-Bus / libsecret services running

## Goal

Add alternative backends so that ghtkn can be used even in environments where keyring does not work.
However, it is not a goal to guarantee security equivalent to keyring.
The security characteristics and intended use of each backend are documented so that users can choose according to their environment.

## Provided backends

The following 4 backends are provided. The default is keyring.

1. keyring (default): Managed by the OS keyring. Depends on keyring
2. gpg: Manages access tokens encrypted with gpg. Depends on gpg
3. agent: Performs passphrase-based encryption using ghtkn's own agent. No external dependencies
4. text: Manages in plaintext. No dependencies

## Which backend should you choose?

As a premise, regardless of which backend you choose, the fact that a short-lived (8H) User Access Token is generated from a GitHub App does not change.
Since this has a lower security risk compared to Personal Access Tokens, which tend to have long lifespans, even the lowest-security backend such as the text backend provides a certain level of security, making ghtkn worth using.

Also, installation of ghtkn is required regardless of the backend.
If you want to use it in containers or VMs, you need to install ghtkn in them.

| backend | Ease of introduction | Mechanism complexity / reliability | Security |
|---|---|---|---|
| keyring | Low: OS keyring required (difficult to work in containers/VMs) | High: Most proven, highly reliable | High: at-rest encryption, equivalent to other encryption systems when unlocked |
| gpg | Medium: External dependencies on gpg + pinentry | Medium: gpg itself is mature, but there is complexity in integration with ghtkn | High: at-rest encryption, agent pattern |
| agent | High: ghtkn alone (lightest) | Low: ghtkn's own implementation, most complex with no track record | High: at-rest encryption, agent pattern |
| text | High: ghtkn alone (lightest) | High: Simplest, least troublesome | Low: Clearly low due to plaintext storage |

In desktop environments where the OS keyring is available, we recommend using the default keyring backend.

The text backend is the backend that operates with the simplest mechanism with no external dependencies such as keyring or gpg.
Because it is simple, it is also a backend that is less prone to trouble.
One possible usage is to use the text backend when other backends cannot be used due to some problem.
However, since access tokens are managed in plaintext, the risk of leakage is higher than with other backends.

The gpg backend stores access tokens encrypted using gpg, so it has a higher security level than the text backend.
However, since it depends on gpg, the setup is somewhat cumbersome and requires an understanding of gpg itself, and because the mechanism is more complex than the text backend, problems are also more likely to occur.

The agent backend is a backend that manages access tokens encrypted by ghtkn's own agent, solving the problems of the gpg backend and the text backend.
While we aim for it to be the most recommended backend in environments where keyring is unavailable, its internal mechanism is more complex than the text backend, and it does not yet have the track record of gpg.

### Ease of introduction

Comparing the system requirements of each backend, the text backend and agent backend are easiest to introduce because they work with ghtkn alone.
The gpg backend requires external dependencies such as gpg and pinentry.
The keyring backend requires the OS keyring and has the most requirements, often making it difficult to operate in containers or VMs.

### Mechanism complexity / reliability

Compared from the perspectives of whether the mechanism works correctly and whether troubleshooting is easy when problems occur.

The keyring backend is the most proven backend and is reliable.
The text backend is the simplest backend and is less prone to problems.
The gpg backend has high reliability of gpg itself, but requires knowledge of gpg and troubleshooting can be relatively difficult.
There is a certain degree of complexity in the integration between ghtkn and gpg.
The agent backend implements the agent mechanism in-house and is the most complex.

### Security

The 3 backends excluding the text backend (keyring / gpg / agent) have almost equivalent security characteristics in practical operation:

- All are encrypted at-rest, so they are resistant to accidental leakage such as backups, mistaken commits, and cloud sync
- All are vulnerable to same-user malware in the unlocked / cached state
- Live root cannot be prevented by any of them

The differences only appear in details such as "Keychain locked state protection (before login / after explicit lock)", "cache TTL tuning", and "iCloud Keychain sync availability", and when compared with default settings, they are on par.

In the text backend, since access tokens remain on disk in plaintext, the leakage paths are wider than encrypted backends, but it is inaccurate to overestimate it as "dangerous because it's plaintext". The magnitude of the difference varies depending on the situation.

Situations where the difference is large (high value in choosing encrypted backends):

- Environments without disk encryption (FDE) / untrusted environments
- Possibility of secrets being mixed into backup tools (especially external / cloud destinations)
- Risks of human errors such as mistaken git commits / mistaken Slack attachments / cloud sync
- Possibility of file contents being picked up via secondary paths such as filesystem snapshots, Spotlight indexes, antivirus, etc.
- Handling multiple machines or persistent environments, where operational errors are prone to occur

Situations where the difference is small (practical risk is limited even with the text backend):

- FDE is enabled
- Backups exclude secrets or are equally encrypted
- Mistaken commits can be prevented with `.gitignore`, etc.
- Local development machines where operations can be done with some care
- Same-user malware can be treated as outside the threat model

In other words, whether to choose the text backend is determined less by "actual security level" and more by how much you value "insurance against human errors and secondary paths".

For details, refer to "## Security comparison".

## Specifying the backend

Specify the backend with the environment variable GHTKN_BACKEND.

```sh
export GHTKN_BACKEND=text
```

## gpg

The gpg backend stores access tokens encrypted using gpg.
ghtkn itself does not handle private keys, and delegates to gpg / gpg-agent.

There are 2 patterns for key placement:

- Pattern A: Self-contained in the same environment (recommended) — Have gpg keys in the same place as the environment where ghtkn runs (container / VM / local). The configuration is simple and works reliably especially when using Lima/Colima/Docker from macOS
- Pattern B: Forward host agent — Run the agent on the host side and request decryption from the container / VM via socket. Keys can be aggregated on the host, but the configuration is complex and the practical hurdle is high on macOS. For details, refer to the "gpg agent forwarding" section

```sh
export GHTKN_BACKEND=gpg
ghtkn get
```

### Setup procedure (Pattern A)

Procedure for generating and using gpg keys within the environment where ghtkn runs.

#### gpg installation

macOS:

```sh
brew install gnupg
```

Debian / Ubuntu:

```sh
sudo apt install gnupg
```

#### Key generation

Interactive mode (passphrase also set interactively):

```sh
gpg --full-generate-key
```

- Algorithm: default (RSA / Ed25519)
- Expiration date: zero (no expiration) is OK if it's a key dedicated to ghtkn
- Real Name / Email: If it's a key dedicated to ghtkn, `ghtkn` / `ghtkn@localhost`, etc. are fine

Non-interactive mode (for automatic provisioning, etc., no passphrase):

```sh
gpg --batch --quick-gen-key "ghtkn <ghtkn@localhost>" default default 0 \
  --passphrase ""
```

#### Obtaining the fingerprint

Obtain the fingerprint to tell ghtkn "which key to use":

```sh
gpg --list-secret-keys --with-colons --fingerprint ghtkn@localhost | \
  awk -F: '/^fpr/ {print $10; exit}'
```

The output is a 40-digit hex (e.g., `A1B2C3D4E5F6789012345678901234567890ABCD`). It is assumed to be passed to ghtkn via an environment variable, etc. (the specific interface will be decided at implementation time).

#### pinentry configuration (when using a passphrase)

Add to `~/.gnupg/gpg-agent.conf`:

```
# macOS
pinentry-program /opt/homebrew/bin/pinentry-mac

# Linux TTY
# pinentry-program /usr/bin/pinentry-tty

# Linux curses
# pinentry-program /usr/bin/pinentry-curses

default-cache-ttl 43200    # 12 hours
max-cache-ttl 43200
```

After changing the configuration, restart the agent:

```sh
gpgconf --kill gpg-agent
```

#### Operation check

```sh
echo "test" | gpg --encrypt --recipient ghtkn@localhost | gpg --decrypt
```

If the passphrase request appears via pinentry, the setup is OK.

#### Environment-specific notes

| Environment | Recommended setup |
|---|---|
| Local development (macOS / Linux desktop) | pinentry-mac / pinentry-gtk, passphrase operation |
| Development VMs (inside Lima/Colima, EC2, etc.) | TTY-based input with pinentry-tty / pinentry-curses |
| Disposable containers | Generate keys without a passphrase with `--batch --passphrase ""` |
| Persistent containers | Persist `~/.gnupg` to a volume, or import keys at startup |

## agent

We consider the agent backend as a backend that does not depend on gpg or keyring and is safer than text.

ghtkn starts an agent that runs in the background.

```sh
ghtkn agent start
```

Enter the passphrase at startup. On the first run, an encryption key is generated.

When the agent starts, a socket is created, and `ghtkn get` communicates with the agent via the socket. The agent performs encryption and decryption of access tokens.

For implementation details (encryption algorithm, socket, lifecycle, etc.), refer to "## Implementation details / agent".

## text

The access token is saved in plaintext to `${XDG_CACHE_HOME}/ghtkn/tokens/<client-id>`.
The file permissions are `0600`. No encryption is performed.

```sh
export GHTKN_BACKEND=text
ghtkn get
```

## Security comparison

### text

Since the file permissions are created with `0600`, other users cannot read it.
In addition, the following measures are necessary.

- Backups exclude secrets or are equally encrypted
   - When used locally: Under `${XDG_CACHE_HOME}` is excluded by default in many backup tools (Time Machine, etc.)
   - When used in containers/VMs: Host file-level backups do not directly recognize internal paths, so they are indirectly protected. However, the fact that VM disk images themselves of Lima/Colima, etc. become backup targets needs to be considered separately
  - Place in paths that are not subject to cloud sync (`~/.local/state/`, `$XDG_RUNTIME_DIR`)
- Disk encryption (FileVault / LUKS / EBS encryption, etc.) addresses offline theft

Note that same-user malware is outside the threat model.

#### text encryption option (not yet implemented)

Refer to [text encryption option](#text-encryption-option).

#### Comparison between text encryption option and gpg Pattern A

In gpg Pattern A, the key file `~/.gnupg/private-keys-v1.d/<keygrip>.key` is AES-encrypted with a passphrase, so behavior varies significantly depending on whether a passphrase is set.

| Aspect | gpg Pattern A (with passphrase) | gpg Pattern A (without passphrase) | text + encryption (plaintext key) |
|---|---|---|---|
| Offline theft of key file alone | Protected | Not protected | Not protected |
| Both key file and ciphertext leaked | Depends on passphrase | Decryptable | Decryptable |
| Live system same-user attack | Decryptable while cached | Decryptable | Decryptable |
| infostealer regex extraction | Avoidable | Avoidable | Avoidable |
| Ciphertext format | Standard OpenPGP | Standard OpenPGP | Custom-defined |
| Implementation | Depends on gpg / gpg-agent | Same as above | Go standard library |
| Setup | `gen-key` + pinentry configuration | `gen-key --batch --passphrase ""` | Automatic generation on first run |
| Memory cache | Managed by gpg-agent | None | None |

Important observations:

- gpg Pattern A (without passphrase) and text + encryption are essentially equivalent. The difference is only in implementation complexity, and if the security is equivalent, text + encryption is simpler
- Only gpg Pattern A (with passphrase) wins in at-rest defense. Since the key file is encrypted with a passphrase, there is an additional wall against file theft. However, while the passphrase cache is effective, it is vulnerable to live attacks, and in environments where interactive input is not possible, it tends to be without a passphrase in practice

Significance of each:

- gpg Pattern A (with passphrase): Emphasis on at-rest defense, assuming interactive use
- gpg Pattern A (without passphrase): Limited significance (can be replaced by text + encryption)
- text + encryption (plaintext key): Cases where you want infostealer countermeasures and accidental leakage countermeasures with zero external dependencies

### How different are keyring backend and agent pattern (gpg/agent backend) in practice?

"Agent pattern" here refers to the architecture of "accessing keys and passphrases cached in memory via socket". The gpg backend / agent backend of ghtkn applies. Note that this discussion assumes the state of with-passphrase + cached. Keys without a passphrase or in an uncached state have different characteristics.

In the usage scenarios of ghtkn, a CLI tool, the difference between the two is not as large as you might think:

- The per-app ACL of macOS Keychain is a theoretical advantage, but since ghtkn accesses it via `/usr/bin/security`, any process can obtain the same secret by calling the same CLI. The ACL of the Keychain item is effectively set to trust `/usr/bin/security`, and ghtkn-specific per-app protection does not function
- Linux libsecret / Windows DPAPI originally do not provide per-app ACL, only per-user isolation
- The cached state after pressing "Always Allow" is essentially the same compromise as the agent's TTL cache
- Same-user malware cannot be fundamentally prevented by either when assumed

The only clear difference is in the "protection in the Keychain locked state (before login / after explicit lock)". In normal operation after login, it becomes equivalent to the agent's cached state.

### Comparison with "plaintext 0600"

The most important observation: in many threat models, the difference between gpg and plaintext 0600 is surprisingly small.

| Threat scenario | Plaintext + `0600` | gpg + agent (cached) | gpg + agent (not cached) |
|---|---|---|---|
| Other users on the same machine | ○ | ○ | ○ |
| Other processes with same-user privileges | ✗ | ✗ (decryptable via socket) | △ (key passphrase required) |
| Live root | ✗ | ✗ (agent memory dump) | ✗ (passphrase required) |
| Offline disk theft (no FDE) | ✗ | △ (depends on passphrase strength) | ○ |
| Leakage from backup | ✗ | ○ | ○ |
| Mistakenly committed to git | ✗ | ○ | ○ |
| Mistakenly attached to Slack/email | ✗ | ○ | ○ |
| Mistakenly uploaded via cloud sync | ✗ | ○ | ○ |
| Disk with FDE + lost | ○ | ○ | ○ |

Important observations:

1. The two are nearly equivalent for live system same-user attacks — while the agent cache is loaded, decryption requests can be made via the socket, so the result is the same
2. The difference appears in at-rest scenarios — gpg makes sense as insurance against human errors and secondary leakage paths (git, backups, cloud sync, mistaken attachments)
3. gpg is one step stronger without cache — however, shortening the TTL has a trade-off of worsening UX

### agent

The passphrase is managed only in memory and is not persisted to disk. On the other hand, the encryption key itself is persisted to disk, but since it is protected by a passphrase, it cannot be used directly even if it is leaked. Even after the agent terminates, you can access the same access token by re-entering the passphrase.

The design is such that users do not need to manage long random encryption keys, and only need to manage a memorable passphrase.

### Comparison between gpg backend and agent backend

Since both adopt the same architecture of "socket + memory cache + key file encrypted with passphrase", their security characteristics largely overlap.

| Aspect | gpg backend | agent backend |
|---|---|---|
| Encryption algorithm | OpenPGP (RSA/Ed25519 + AES, etc.) | AES-256-GCM + Argon2id KDF |
| Encryption implementation | gpg binary (decades of track record) | Go standard library (new implementation) |
| Agent | gpg-agent (mature) | ghtkn's own (new) |
| Passphrase input | pinentry (mac / tty / curses / gtk) | Implemented within ghtkn |
| Key file | `~/.gnupg/private-keys-v1.d/<keygrip>.key` | `~/.config/ghtkn/key` |
| Degree of attacker targeting | `~/.gnupg/` is a known target of infostealers | ghtkn-specific paths are less likely to be patterned |
| External dependencies | gpg binary + pinentry | None |
| Setup complexity | High (pinentry / TTL / key generation / fingerprint acquisition) | Low (only passphrase input) |
| Attack surface (code amount) | Large (gpg suite + OpenPGP parser) | Small (only AES-GCM + socket) |

Nearly equivalent in major scenarios:

- Offline theft of key file alone: Protected by passphrase (both)
- Cached + same-user malware: Decryption requests can be made via socket (both)
- Backup / mistaken commit / cloud sync: No direct damage if only ciphertext (both)
- Live root: Broken through by agent memory dump (both)

Differences:

- Advantages of gpg backend: Maturity of implementation (decades of operational track record, known bugs fixed), stability of UX through pinentry integration
- Advantages of agent backend: Small attack surface (complex parsers of OpenPGP and peripheral components such as scdaemon are unnecessary), avoidance of known infostealer targets such as `~/.gnupg/` (the effect diminishes as it spreads), zero external dependencies, simplicity of design

Risk assessment of ghtkn's own implementation:

- Cryptographic primitives (AES-GCM / Argon2id) are delegated to the Go standard library and `golang.org/x/crypto`, so the risk of vulnerabilities in the crypto core itself is limited
- The remaining risks are in key wrap / unwrap logic, socket protocol, lifecycle management, etc. While they can be implemented with standard patterns, the possibility of bugs in the initial implementation cannot be denied

There is no clear security superiority or inferiority. The choice will be: gpg backend if you go conservatively, agent backend if you choose simplicity and low dependencies.

## gpg agent forwarding

A design where the agent runs on the host side and socket communication is performed from the container/VM is possible.

Key points:

- The private key does not leave the host, only the decryption result is passed to the guest
- pinentry is displayed on the host side
- Using `extra-socket` (restricted version without key management functions) is recommended

### Common preparation

> [!WARNING]
> Specific commands for gpg agent forwarding are listed below, but there may be cases where they do not work depending on the environment.
> Executing these commands does not guarantee that they will work; they are written merely for reference of the usage feel of the backend using gpg agent in determining the implementation policy.

#### Enable extra-socket on the host side

Add to `~/.gnupg/gpg-agent.conf`:

```
extra-socket /run/user/<uid>/gnupg/S.gpg-agent.extra
```

The actual path can be confirmed with `gpgconf --list-dirs agent-extra-socket`. After changing the configuration, restart the agent:

```sh
gpgconf --kill gpg-agent
```

#### Pass the public key to the guest side

A public key ring is required on the guest side to determine "which key it is encrypted with":

```sh
# Export on the host side
gpg --export <fingerprint> > /tmp/pubkey.gpg
# Import on the guest side
gpg --import /tmp/pubkey.gpg
```

In the case of a container, there is also a method of mounting `pubring.kbx` as read-only.

### Linux host → Linux container (Docker)

Mount the extra-socket and public key ring and start the container:

```sh
SOCKET=$(gpgconf --list-dirs agent-extra-socket)

docker run -it --rm \
  -v "$SOCKET:/root/.gnupg/S.gpg-agent" \
  -v "$HOME/.gnupg/pubring.kbx:/root/.gnupg/pubring.kbx:ro" \
  ghtkn-image
```

Inside the container, just execute `ghtkn get` as usual, and the host-side agent will decrypt.

### Forwarding via SSH (general VMs)

Enable `StreamLocalBindUnlink yes` in the guest-side sshd:

```
# /etc/ssh/sshd_config
StreamLocalBindUnlink yes
```

Forward the socket when connecting from the host via SSH:

```sh
HOST_SOCKET=$(gpgconf --list-dirs agent-extra-socket)
GUEST_SOCKET=/home/user/.gnupg/S.gpg-agent

ssh -R "$GUEST_SOCKET:$HOST_SOCKET" guest-vm
```

Make it permanent in `~/.ssh/config`:

```
Host guest-vm
  RemoteForward /home/user/.gnupg/S.gpg-agent /run/user/1000/gnupg/S.gpg-agent.extra
```

### Forwarding from macOS host

On macOS, Docker Desktop / Colima / Lima all have a 3-tier structure with an internal Linux VM:

```
macOS host
  └─ Linux VM of Lima/Colima/Docker Desktop
      └─ Container
```

The macOS gpg-agent socket is on the macOS filesystem and is not directly visible from the Linux VM. Since the container is further inside the VM, 2 stages of forwarding are required.

Tool-specific situations:

| Tool | VM visibility | Unix socket forwarding | Notes |
|---|---|---|---|
| Docker Desktop for Mac | Hidden | Not possible (only TCP via `host.docker.internal`) | Convert to TCP with `socat`, dependent on dedicated tools |
| Lima | Public | Possible with SSH `RemoteForward` | `ssh.forwardAgent` is officially supported |
| Colima | Public (Lima-based) | Same as above | Default `$HOME` and `/tmp/colima` mounted |

Notes:

- Unix sockets inside directories mounted with reverse-sshfs / virtiofs / 9p often do not function as sockets
- The 2-stage forwarding of macOS → VM → container has complex configuration
- Since pinentry appears on the macOS side, it has poor compatibility with automated processing

#### Lima

Since Lima is SSH-based, you can use `RemoteForward` in the same way as the VM pattern:

```sh
# Output Lima's SSH configuration
limactl show-ssh --format config default > /tmp/lima.config

# Add RemoteForward and connect
ssh -F /tmp/lima.config \
  -R "/home/$USER.linux/.gnupg/S.gpg-agent:$(gpgconf --list-dirs agent-extra-socket)" \
  lima-default
```

When running a container further inside the VM, it becomes a 2-stage mount where the forwarded socket inside the VM is volume-mounted to the container:

```sh
# Inside the VM
docker run -it --rm \
  -v "$HOME/.gnupg/S.gpg-agent:/root/.gnupg/S.gpg-agent" \
  -v "$HOME/.gnupg/pubring.kbx:/root/.gnupg/pubring.kbx:ro" \
  ghtkn-image
```

#### Colima

Colima is also internally Lima, so it works with the same pattern:

```sh
colima ssh-config > /tmp/colima.config
ssh -F /tmp/colima.config \
  -R "/home/$USER/.gnupg/S.gpg-agent:$(gpgconf --list-dirs agent-extra-socket)" \
  colima
```

#### Docker Desktop for Mac

Since Docker Desktop's internal VM is hidden, socket forwarding over SSH cannot be used. Relying on dedicated tools is realistic:

- [transifex/docker-gpg-agent-forward](https://github.com/transifex/docker-gpg-agent-forward) — A method to separately start a sidecar container that relays by converting to TCP with socat
- A custom script that converts to TCP with `socat` and connects via `host.docker.internal`

Since either has a large configuration burden, it is more realistic to migrate to Lima/Colima or have a self-contained gpg environment within the VM/container.

### Realistic options

1. Self-contain gpg-agent within the VM (recommended) — Decouple from macOS keys and have an independent gpg environment inside the Lima/Colima VM. Socket forwarding is only 1 stage from VM → container
2. 2-stage forwarding of macOS host agent — Key management can be aggregated on macOS, but configuration is complex
3. Operate keys without a passphrase inside the VM — `gpg --quick-gen-key --batch --passphrase ""`, the simplest

For a general-purpose CLI tool like ghtkn, it is less confusing for users to avoid the design of "forwarding macOS keys to containers inside the VM" and assume independent keys and token caches in each VM/container environment. By the same logic as "macOS Keychain cannot be used from a VM", "using macOS gpg-agent from a VM" also has high practical hurdles.

## Other ideas

We take up some options that were not adopted this time.

1. Manage access tokens on the host side and expose them as environment variables when creating VMs/containers
    1. While it has the advantage that no special implementation is required for ghtkn, if the VM/container lasts longer than the access token's lifespan (max 8H), the access token will expire, so recreation of the VM/container or explicit resetting of the environment variable is required
    2. Since all processes within the container can reference the access token from environment variables, the risk of leakage increases
2. Implement gpg wrapper commands such as ghtkn gpg setup / ghtkn gpg status
    1. A UX improvement measure that can compress manual fingerprint copy-pasting and a 4-step setup into 1 command. Postponed in the initial implementation, and will be added if a wrapper is judged necessary in actual use

### text encryption option

Instead of the text backend, a method of placing "encryption + plaintext key" on the same machine (encrypted with a symmetric key, with the private key placed in plaintext in `XDG_CONFIG_HOME`) can also be considered as an option.

The point that "if the key coexists in plaintext, it can be easily decrypted, so it has no meaning" is correct, but effectiveness varies depending on the type of attack:

- Indiscriminate credential harvesting (regex scanning by infostealers, etc.): Since encrypted tokens fall outside pattern matching, extraction can be avoided
- Targeted attacks: Since the key layout and encryption method can be restored by reading the code, it is equivalent to plaintext
- Accidental leakage (mistaken git commit, backup leakage, etc.): If the key and ciphertext do not ride on the same path, partial protection is possible

As an observation in the opposite direction, since automated detection like GitHub's secret scanning is regex-based, there is also a view that plaintext is revoked immediately and damage is smaller.

Trade-offs:

- Pros: Resistance to indiscriminate attacks and accidental leakage, low cost (can be implemented with the Go standard library, no external dependencies)
- Cons: Powerless against targeted attacks, risk of users overestimating that it is "encrypted", the obscurity effect diminishes as it spreads

Implementation policy proposal: Use AES-256-GCM with the Go standard library `crypto/aes` + `crypto/cipher`. AES-GCM is an authenticated encryption (AEAD) that provides confidentiality and integrity. External libraries such as `filippo.io/age` provide features such as multiple recipients, standard format, and passphrase, but are not necessary for ghtkn's use case (single user, symmetric key, small JSON).

File placement proposal:

```
~/.config/ghtkn/key       # 32 bytes symmetric key (0600)
~/.cache/ghtkn/tokens/X   # nonce || ciphertext (0600)
```

If adopted, the README needs to clearly state that "the defense effect against targeted attacks is limited", "the key layout is published in OSS", and "it is different from keyring or HSM".

## Implementation details

### gpg

ghtkn is expected to call the gpg binary via `os/exec` to perform encryption and decryption. Go OpenPGP libraries (such as `github.com/ProtonMail/go-crypto/openpgp`. `golang.org/x/crypto/openpgp` has been deprecated since 2020) are not adopted.

Reasons:

- Go OpenPGP libraries cannot directly read GnuPG's proprietary format in `~/.gnupg/private-keys-v1.d/`
- When adopting a library, keys need to be exported with `gpg --export-secret-keys --armor`, and ghtkn needs to manage them independently
- In that case, the cache mechanism of gpg-agent cannot be utilized, and the significance of the gpg backend's existence over the agent backend diminishes

Shell-out implementation image:

```go
// Encryption
cmd := exec.Command("gpg", "--encrypt", "--recipient", fingerprint, "--armor")
cmd.Stdin = bytes.NewReader(plaintext)

// Decryption
cmd := exec.Command("gpg", "--decrypt")
cmd.Stdin = bytes.NewReader(ciphertext)
```

The gpg binary internally communicates with gpg-agent and obtains the passphrase via pinentry as needed. ghtkn does not directly handle passphrases or keys.

The need to "avoid external dependencies" is to be covered by the agent backend.

#### Concept of cache TTL

gpg-agent retains the once-entered passphrase in memory. By this:

- The passphrase is not written to disk as a plaintext file (the essence of at-rest defense)
- The cache is lost upon process termination or OS reboot, and cannot be restored from disk
- The passphrase cannot in principle be leaked through at-rest leakage paths such as backups, cloud sync, and mistaken git commits

Timing when re-entry is required:

| Trigger | Description |
|---|---|
| `gpgconf --kill gpg-agent` | When the agent is explicitly killed |
| OS reboot | Disappears along with the process |
| `default-cache-ttl` elapsed | N seconds since last used (default 600 seconds) |
| `max-cache-ttl` elapsed | N seconds since first unlock (absolute upper limit, default 7200 seconds) |
| Agent crash | Abnormal termination |
| User logs out | Session-related agents terminate |

TTL trade-offs:

- Shorten (e.g., 15 minutes) — Strong at-rest defense, poor UX (pinentry may appear every time you use git/gh)
- Lengthen (e.g., 12 hours = 43200 seconds) — Input is only needed about once a day, vulnerable to live attacks during that time

Considering that ghtkn is called every time git/gh is used, the default 600 seconds is too short to be practical. The `43200` (12 hours) in the example above is a setting aimed at "no prompts during a day's work". Adjust according to the user's threat model.

Note that gpg-agent is also powerless against live memory attacks (`ptrace` or `/proc/<pid>/mem` reads by processes with same-user privileges, memory dumps by root). gpg-agent uses `mlock()` and secmem to suppress passphrases from being written to swap, but cannot completely prevent reading of running process memory.

### agent

A design proposal not yet implemented. The following are assumptions at implementation time.

#### Encryption

- Encryption algorithm: AES-256-GCM (Go standard library `crypto/aes` + `crypto/cipher`)
- Encryption key: 32 bytes, randomly generated on first startup
- KEK (key-encryption key): Derived from the passphrase with KDF, used to wrap the encryption key and save it
- KDF: Argon2id (`golang.org/x/crypto/argon2`). The salt is included in the key file

#### Passphrase management

- Input from the user when the agent starts
- The KEK derived by KDF is retained only in the agent process memory
- Passphrases and KEK are not written to disk
- Whether to set a TTL is a design judgment (similar to gpg-agent, there is a proposal to lock after idle)

#### Storage

- Encryption key: `$XDG_CONFIG_HOME/ghtkn/key` (wrapped with KEK, includes salt, `0600`)
- Encrypted tokens: `$XDG_CACHE_HOME/ghtkn/agent/<client id>` (`0600`)

#### Socket and communication

- Socket placement: `$XDG_RUNTIME_DIR/ghtkn/socket` (Linux). In environments without `XDG_RUNTIME_DIR`, use a fallback path
- Permissions: `0600`, accessible only by the same user
- Protocol: JSON-based commands (UNLOCK / LOCK / GET / SET / STATUS, etc.)
- The point that any process with same-user privileges can connect to the socket is the same structural constraint as gpg-agent

#### Lifecycle

- Explicitly managed with `ghtkn agent start` / `stop` / `status`
- If an existing socket exists at startup, judge it as stale and clean up
- Automatic restart on agent crash is not supported; the user re-runs `start`

#### At-rest defense and limits of live attacks

- Passphrase and KEK are only in memory → effective against disk at-rest attacks
- Powerless against `ptrace` / `/proc/<pid>/mem` reads with same-user privileges
- The concept of cache TTL is the same as the gpg backend (refer to "Concept of cache TTL" in "## gpg")

### Parallel writing

#### Assumed conflicts

By splitting files per client-id as `tokens/<client-id>`, write conflicts between different client-ids are structurally eliminated. What remains is only parallel writes to the same client-id:

```
Terminal A: ghtkn get --client-id=X
Terminal B: ghtkn get --client-id=X   (same time)
```

If both write to `tokens/X` simultaneously, in the worst case, the file may be corrupted due to partial writes or interleaving.

#### Design policy

Based on the premise of "if it breaks, just recreate it", we handle it with only atomic rename. Reasons:

- Even if the access token breaks in the worst case, it can be restored by re-minting from GitHub when ghtkn is executed next time. No user operation is required
- It would be a problem if the breakage frequency was too high, but it is at a level that is sufficiently tolerable in normal use
- True exclusive control such as flock is avoided because it has cross-platform differences, deadlocks, and stale lock risks

#### Atomic rename implementation

Write to a temporary file and then replace it with the main file with `os.Rename`. `os.Rename` is atomic on POSIX (within the same filesystem), so even in the worst case, it becomes a choice of either "remaining old content" or "completely replaced with new content", and an intermediate state where the file is broken does not remain.

Implementation sketch:

```go
func saveToken(path string, content []byte) error {
    dir := filepath.Dir(path)
    if err := os.MkdirAll(dir, 0700); err != nil {
        return err
    }
    tmp, err := os.CreateTemp(dir, ".ghtkn-tmp-*")
    if err != nil {
        return err
    }
    tmpPath := tmp.Name()
    defer os.Remove(tmpPath) // no-op on successful rename

    if err := os.Chmod(tmpPath, 0600); err != nil {
        tmp.Close()
        return err
    }
    if _, err := tmp.Write(content); err != nil {
        tmp.Close()
        return err
    }
    if err := tmp.Close(); err != nil {
        return err
    }
    return os.Rename(tmpPath, path)
}
```

Points:

- Create the temporary file in the same directory (to avoid cross-device rename and ensure atomicity)
- Use a hidden file prefix (`.ghtkn-tmp-*`) for the temporary file name to reduce `ls` pollution
- Explicitly set permissions to `0600`
- The `os.Remove` in defer is a no-op after successful rename, and functions as cleanup on failure

#### About lost updates

In this approach, the semantics is "the last one to rename wins", and one side's update content is lost (lost update). However, this is not a problem for the use case of access tokens:

- Both have obtained fresh tokens from GitHub
- Whichever token is ultimately saved, both are valid
- The next time ghtkn is executed, just use the saved token

#### Common to all backends

text / agent / gpg all apply this pattern at the time of per-file writes.

## References

- [security through obscurity](https://en.wikipedia.org/wiki/Security_through_obscurity)
