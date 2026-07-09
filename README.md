# Slurm all-in-one (personal single-machine cluster)

Sets up one machine to act simultaneously as login node, controller
(`slurmctld`), accounting database (`slurmdbd` + MariaDB) and compute node
(`slurmd`). Slurm is built from source at a pinned version.
Job memory is capped via cgroups so the host OS always keeps a fixed amount
of free RAM. Job notification e-mails are sent through
[slurm-mail](https://github.com/neilmunday/slurm-mail) over SMTP -- any
provider works (Gmail, Proton Mail, a self-hosted relay, ...). NVIDIA GPUs
are auto-detected and made schedulable.

Targets Ubuntu/Debian only (uses `apt`).

## What it does

- Builds munge auth key, MariaDB accounting DB, and Slurm (`slurmctld`,
  `slurmd`, `slurmdbd`, `sacct`/`sacctmgr`/`sinfo`/`squeue`/...) from source
  at the version pinned by `slurm_version` in `group_vars/all.yml`, installed
  under the standard `/usr/local` prefix (`slurm_prefix`) — binaries land in
  `/usr/local/bin`/`/usr/local/sbin`, already on everyone's `$PATH`, no
  profile.d shim needed.
- Configures `cgroup.conf`/`slurm.conf` so that `mem_reserved_gb` (default 4
  GiB) is always reserved for the host OS via `MemSpecLimit` +
  `ConstrainRAMSpace=yes`; everything else is schedulable by Slurm jobs.
- Installs [slurm-mail](https://github.com/neilmunday/slurm-mail) (version
  pinned by `slurm_mail_version`) and wires it up as `MailProg`, configured
  to relay through the SMTP server/account you provide.
- Detects an NVIDIA GPU via `lspci` (override with `has_gpus` in
  `group_vars/all.yml` if needed) and, when found: installs the NVIDIA
  driver (only if none is already working), NVML dev headers so Slurm is
  built with GPU autodetect support, and the NVIDIA Container Toolkit; then
  configures `gres.conf`/`slurm.conf` so the GPU is schedulable
  (`--gres=gpu:1`).

## Prerequisites

1. Install Ansible (via `pipx`, with Mitogen for speed) and the
   roles/collections this playbook needs:
   ```
   sudo apt-get install pipx
   pipx install --include-deps "ansible-core>=2.17.12"
   pipx inject ansible-core passlib
   ansible-galaxy install -r roles/requirements.yml -p roles/galaxy
   ansible-galaxy collection install -r roles/requirements.yml -p roles/collections
   wget https://files.pythonhosted.org/packages/source/m/mitogen/mitogen-0.3.22.tar.gz
   tar zxf mitogen-0.3.22.tar.gz
   rm mitogen-0.3.22.tar.gz
   ```
2. An SMTP account to send job notification mail through -- any provider
   works (Gmail, Proton Mail, Fastmail, a self-hosted Postfix relay,
   SendGrid, ...). You'll need its server hostname, port, a username, and a
   password. Some providers require explicitly enabling SMTP
   access/submission and/or generating an app-specific password rather than
   using your normal login password -- check your provider's docs.
3. Edit `group_vars/all.yml`:
   - `cluster_name`
   - `mem_reserved_gb` — GiB to always keep free for the host OS
   - `smtp_username` — your SMTP account's address (used as both the SMTP
     username and the mail's From address), `smtp_server`, `smtp_port`
4. Encrypt the two secrets **in place**, directly inside `group_vars/all.yml`
   — there's no separate vault file to manage:
   ```
   ansible-vault encrypt_string --name vault_mysql_slurm_password
   ansible-vault encrypt_string --name vault_smtp_password
   ```
   Leave off the plaintext argument so each one prompts interactively
   instead of landing in your shell history (it'll ask you to confirm by
   typing it twice). Paste each resulting `vault_xxx: !vault |` block over
   the matching `vault_xxx: CHANGE_ME` placeholder already in
   `group_vars/all.yml`.

   Either of these needs a vault password to encrypt/decrypt with:
   answer the prompt (`--ask-vault-pass`), or create `./.vault_pass` as an
   executable script that prints the password from your password
   manager — `ansible.cfg` already points `VAULT_PASSWORD_FILE` at it, so
   once it exists you won't need `--ask-vault-pass` at all (don't combine
   the two; passing `--ask-vault-pass` *and* having `VAULT_PASSWORD_FILE`
   configured makes Ansible ignore the file and demand interactive input).

## Running

By default the inventory targets `localhost` (`ansible_connection=local`).
Edit `inventory.ini` if you'd rather run this over SSH against another host.

```
ansible-playbook playbook.yml --ask-become-pass
```

Add `--ask-vault-pass` instead if you haven't set up `./.vault_pass` yet
(see Prerequisites above) — just don't pass both at once.

The build step compiles Slurm from source (`make -j<nproc>`), so the first
run takes a while. Re-runs are idempotent: the download/configure/make/install
sequence only runs if `slurmd --version` doesn't already match `slurm_version`
(or `slurm_force_rebuild: true` is set) — e.g. after bumping the pinned
version, or after changing a build dependency or configure flag, where you'd
set `slurm_force_rebuild: true` for one run to force a rebuild against the
same version. Config file changes only trigger restarts of the affected
daemons.

## Verifying it worked

```
sinfo                      # should show your node in state "idle"
sbatch --wrap="sleep 30" --mail-type=ALL --mail-user=you@example.com
squeue
sacct
```

You should get a slurm-mail HTML e-mail when the job starts and finishes.
If mail doesn't arrive, check:

```
sudo journalctl -u slurmctld -e
tail -f /var/log/slurm-mail/*.log
sudo cat /etc/slurm-mail/slurm-mail.conf   # confirm SMTP settings
```

`slurm-send-mail` runs once a minute via `/etc/cron.d/slurm-mail`, so allow
up to a minute for delivery after a job event.

## Notes / things you may want to tune

- **Memory reservation mechanism**: `slurm.conf`'s `NodeName` line sets
  `RealMemory` to the full installed RAM and `MemSpecLimit` to
  `mem_reserved_gb * 1024` MiB. Slurm automatically subtracts
  `MemSpecLimit` from what it hands out to jobs, and (with
  `ConstrainRAMSpace=yes` in `cgroup.conf`) confines `slurmd`/`slurmstepd`
  themselves to that reserved amount — so the OS keeps that RAM available
  even under a full job load.
- **CPU reservation**: not configured — all CPUs are schedulable. Add
  `CoreSpecCount=N` to the `NodeName` line in
  `roles/slurm/templates/slurm.conf.j2` if you also want cores reserved
  for the OS.
- **Accounting**: the cluster is registered with `sacctmgr add cluster`, and
  `AccountingStorageEnforce=associations,limits,qos` is set in `slurm.conf`,
  which means a user needs an association before they can submit *any*
  job — add one per user with `sacctmgr add account <name>` and
  `sacctmgr add user <user> account=<name>` before they try `sbatch`, or
  jobs will be rejected.
- **PAM job restriction / multi-user SSH**: not configured, since this is a
  personal single-node setup. Look at `pam_slurm_adopt` if you ever want to
  restrict SSH access to nodes with a running job for that user.
- **Upgrading Slurm/slurm-mail**: bump `slurm_version` / `slurm_mail_version`
  in `group_vars/all.yml` and re-run — the playbook will rebuild and the
  restart handlers will bounce the daemons in the right order (`slurmdbd`
  -> `slurmctld` -> `slurmd`). Versions are pinned rather than
  auto-resolved, so check the upstream release pages yourself when you
  want to move to a newer one.
- **GPU support was added after Slurm was already built**: if `slurmd
  --version` already matches `slurm_version` from before GPU support
  existed, the build-skip logic won't know to rebuild with NVML support.
  Set `slurm_force_rebuild: true` for one run to force it.
