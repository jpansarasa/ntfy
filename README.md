# ntfy

[ntfy.sh](https://ntfy.sh) push-notification server, run as a Docker container
under systemd.

## Layout

| File | Role |
| --- | --- |
| `install` | Idempotent installer. Run it for the first deploy and after every change. |
| `compose.yml` | Container definition. Bind-mounts `/tank/ntfy/{config,cache,auth}`. |
| `server.yml.tmpl` | ntfy config, with the one secret as `${WEBPUSH_PRIVATE_KEY}`. |
| `ntfy.service` | Owns the container. No pull on start, so boot is fast and network-order-independent. |
| `ntfy-update.service` | Oneshot wrapper around `check-update`. No `[Install]` section — the timer resolves it by name. |
| `ntfy-update.timer` | Daily update check, off the boot critical path. |
| `check-update` | Pulls the configured tag, compares to the running image, **stages** (never applies) and pushes a notification. |

Paths are pinned: the systemd units hardcode `/opt/ntfy` and `/tank/ntfy`, and
`install` refuses to run from anywhere else.

## What is not in this repo

Two root-only files, by design — the repo stays safe to publish:

| File | Contents | Created by |
| --- | --- | --- |
| `/tank/ntfy/secrets.env` | `WEBPUSH_PRIVATE_KEY`, `WEBPUSH_EMAIL`, `NTFY_BASE_URL` | You, by hand (see below) |
| `/tank/ntfy/notify.env` | `NTFY_URL`, `NTFY_TOPIC`, `NTFY_TOKEN` | `install` seeds a template; you fill in the token |

A third, optional and not secret: `/tank/ntfy/image.env`, holding only
`NTFY_TAG` when a release is pinned (see [Pinning a bad release](#pinning-a-bad-release)).
Absent in the normal case.

Both live on the ZFS dataset rather than in `/etc` **on purpose**, so that
whatever backs up the dataset captures them too. That makes the recovery set
exactly two things:

> **this repo** (configuration) + **the `tank/ntfy` dataset** (secrets and state)

They sit at the dataset root, which `compose.yml` does not bind-mount — only
`config/`, `cache/` and `auth/` are — so the container never sees them. They
are `0600 root:root`, and `install` re-asserts that after its recursive
`chown`, which would otherwise hand them to the `ntfy` user.

Also on the dataset, and likewise not in git: the user database, message cache
and attachments. That is runtime state, not configuration.

### Snapshots

When `install` creates the dataset from scratch it sets
`com.sun:auto-snapshot:monthly=true` and takes a first snapshot, so that
snapshot-based backup tooling has something to work from immediately. It does
this **only** on creation, so a later decision to opt the dataset out is not
silently undone by a subsequent run.

Whatever the snapshot cadence, note what a stale snapshot actually costs here:
the message cache is ephemeral (`cache-duration: 12h`) and the credential files
effectively never change, so the exposure is a recently added user or token.
Snapshot by hand after a user change you care about:

```bash
sudo zfs-auto-snapshot --label=monthly --keep=12 tank/ntfy
```

## Recreating on a fresh machine

Prerequisites: Docker with the compose plugin, ZFS with a pool named `tank`,
`envsubst` (`gettext-base`), systemd.

```bash
# 1. Restore the dataset from backup. This brings back secrets.env,
#    notify.env, the user database, and the cache.
sudo zfs recv -F tank/ntfy < <stream>

# 2. Clone this repo.
sudo git clone https://github.com/jpansarasa/ntfy.git /opt/ntfy

# 3. Converge. Creates the ntfy user, renders server.yml, registers the
#    units, and starts everything.
sudo /opt/ntfy/install
```

Starting **without** the dataset, you must supply the secrets by hand first:

Run `install` first: it creates and mounts the dataset, then stops and tells
you what it needs. Do **not** create `/tank/ntfy` by hand and write the file
there — ZFS mounts over a non-empty directory, so the dataset would hide it.

```bash
sudo /opt/ntfy/install          # creates the dataset, then exits asking for these
sudo sh -c 'umask 077; cat > /tank/ntfy/secrets.env' <<'VARS'
WEBPUSH_PRIVATE_KEY=<key matching web-push-public-key in server.yml.tmpl>
WEBPUSH_EMAIL=<contact address reported to the push provider>
NTFY_BASE_URL=<https://ntfy.example.com>
VARS
sudo /opt/ntfy/install

# install seeds notify.env with an empty token; fill it in to get update pushes
sudo sed -i 's/^NTFY_TOKEN=.*/NTFY_TOKEN=tk_.../' /tank/ntfy/notify.env
```

If you do not have the original private key, generate a new pair — but note
this invalidates every existing browser web-push subscription, so every browser
client has to re-subscribe:

```bash
docker run --rm binwiederhier/ntfy:latest webpush keys
# put the private half in /tank/ntfy/secrets.env,
# and commit the public half into server.yml.tmpl
```

### Users

Accounts live in `/tank/ntfy/auth/user.db`, which comes with the dataset
rather than from this repo. Only if you are starting without that dataset does
the config's `auth-default-access: deny-all` leave you with no users, in which
case:

```bash
docker exec -it ntfy ntfy user add --role=admin <admin>
docker exec -it ntfy ntfy access <user> <topic> rw
docker exec -it ntfy ntfy token add <user>
```

## Day-to-day

```bash
# Change config: edit server.yml.tmpl, then re-run. Restarts ntfy.
sudo /opt/ntfy/install

# Check for an image update by hand (stages, does not apply)
sudo systemctl start ntfy-update.service
journalctl -u ntfy-update.service -n 20

# Apply a staged update
sudo systemctl restart ntfy.service

# Is an update waiting?
cat /run/ntfy-update-available
```

### Pinning a bad release

The normal state is `:latest`, and the normal response to a new release is to
take it — upstream releases are far more often fixes than regressions. So
`latest` is the default, and pinning is the exception, kept to one file that is
not in git:

```bash
# Which version am I on, and which have I run before? (the container logs it
# at startup, and the unit runs attached, so the journal has the whole history)
docker logs ntfy | head -1
journalctl -u ntfy.service | grep -oE 'ntfy [0-9]+\.[0-9]+\.[0-9]+' | uniq

# Pin, then apply
sudo sh -c 'echo NTFY_TAG=v2.26.3 > /tank/ntfy/image.env'
sudo systemctl restart ntfy.service

# Un-pin once upstream supersedes the bad release
sudo rm /tank/ntfy/image.env
sudo systemctl restart ntfy.service
```

`ntfy.service` sets `NTFY_TAG=latest` and then reads `/tank/ntfy/image.env`,
which systemd applies **after** `Environment=` and therefore wins; the `-`
prefix makes the file optional, so its absence just means `latest`. The pin
lives on the dataset rather than in git so it survives a restore and never
collides with a `git pull`.

While pinned, `check-update` tracks the **pinned** tag — it will not claim an
update is staged that a restart would not actually apply. It still watches
`:latest` separately and pushes a one-line "the pin can likely be retired"
notice once upstream moves past the release you pinned away from.

`install` is safe to run repeatedly; it converges rather than erroring on
things that already exist. The only side effect of a no-change run is the
service restart in the final step.

It finishes by polling `/v1/health` until ntfy actually answers, so a zero exit
means "serving", not merely "systemd accepted the job". That distinction is the
whole point: a bad value in `server.yml.tmpl` renders cleanly and starts
cleanly, then crash-loops inside the container. Since editing the template and
re-running is the most common operation here, that is the failure most likely to
be introduced — and without the poll it exits 0 and looks like a success.

## Notes

- The container is unprivileged: `cap_drop: ALL`, then only `CHOWN`/`SETGID`/`SETUID` back, running as uid/gid 2101.
- `ntfy.service` deliberately does **not** pull on start. Pulling is the update timer's job; a start that waits on the network makes boot order fragile.

### Waiting, not skipping

Every prerequisite that can be *late* — the ZFS dataset, the rendered config,
dockerd — is checked in a way that **fails and retries**, never one that skips.
That is a deliberate choice, and the reasoning is worth keeping:

`Condition*` directives (and `Assert*`, and a failed `Requires=`) abort the
start *job*. The unit never enters start, so `Restart=` is never armed, the unit
sits at `inactive (dead)` with `Result=success`, and **nothing ever
re-evaluates**. Repairing the prerequisite does not bring it back — only a new
start job does. For the box that carries your alerts, that is the worst failure
shape there is: silent, absent from `systemctl --failed`, and permanent.

So the dataset and config checks are `ExecStartPre=` (a real start failure, which
`Restart=always` retries every 30s, re-running the check each cycle), docker is
`Wants=` rather than `Requires=`, and `StartLimitIntervalSec=0` means it never
gives up. A late pool import, a hand `zfs mount`, or a dockerd that comes back
all recover the service unattended within 30 seconds.

The cost, which you need to know when looking for a sick service: a unit that
retries forever never reaches `failed`, so **`systemctl --failed` stays empty
while ntfy is down**. Look instead at:

```bash
systemctl is-active ntfy            # "activating" (not "active") while stuck
journalctl -p err -u ntfy -n 20     # the guards log why, at priority err, every cycle
```
