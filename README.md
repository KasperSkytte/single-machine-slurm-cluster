# Slurm all-in-one (personal single-machine cluster)

Turns one Ubuntu/Debian box into a full Slurm cluster — login node,
controller, accounting database, and compute node, all in one place.
Everything's built from source, memory (and a GPU, if you have one) gets
carved out so jobs can't starve the host, and job e-mails go out over
whatever SMTP account you give it.

## What it does

- Builds Slurm and slurm-mail from source, at whatever version you pin in
  `group_vars/all.yml`, plus the contrib tools (`seff`, `pam_slurm_adopt`,
  the Perl API, ...) unless you set `slurm_build_contrib: false`.
- Reserves `mem_reserved_gb` of RAM for the host via cgroups, so jobs can't
  eat all of it.
- Detects whether an NVIDIA GPU is present, then installs drivers, container
  toolkit, and makes it schedulable (`--gres=gpu:<model>:<n>`).
- E-mails you when jobs start/finish via slurm-mail, using whatever SMTP
  account you give it — or skip mail entirely with `slurm_mail_enable: false`.

## Prerequisites

1. Install Ansible and friends:
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
2. If you want job e-mails, have an SMTP account ready — Gmail, Proton
   Mail, Fastmail, your own relay, anything works. You'll need the server,
   port, username, and password (some providers want an app password
   instead of your real one).
3. Edit `group_vars/all.yml`: `cluster_name`, `mem_reserved_gb`, and either
   `smtp_username`/`smtp_server`/`smtp_port` or `slurm_mail_enable: false`
   if you'd rather skip mail.
4. Encrypt the `vault_mysql_slurm_password` and `vault_smtp_password` secrets in place — there's no separate vault file for secrets:
   ```
   ansible-vault encrypt_string --name vault_mysql_slurm_password
   ansible-vault encrypt_string --name vault_smtp_password
   ```

## Running
Check whether `ansible.cfg` makes sense for you first. Then run with
```
ansible-playbook playbook.yml --ask-become-pass
```

Add `--ask-vault-pass` too if you haven't set up `.vault_pass` yet. The
inventory targets `localhost` by default — edit `inventory.ini` if you'd
rather run this against another box over SSH.

The first run compiles Slurm from source, so it takes a while. Later runs
skip the rebuild unless `slurm_version` changed or you set
`slurm_force_rebuild: true`.

## A few things worth knowing
- **New users can't submit jobs until you add them.** Accounting
  enforcement is on, so run `sacctmgr add account <name>` and
  `sacctmgr add user <user> account=<name>` first, or else jobs get
  rejected.
- Only RAM is reserved for the host, not CPU. Add `CoreSpecCount=N` to the
  `NodeName` line in `roles/slurm/templates/slurm.conf.j2` if you want
  that too.
- No SSH/PAM restrictions — anyone with an account on the box can log in
  and submit jobs, with running jobs or not - so don't run things outside of slurm jobs!
- Turned on GPU support or contrib tools after Slurm was already built?
  It won't rebuild on its own since the version hasn't changed — set
  `slurm_force_rebuild: true` once to pick it up.
- Versions are pinned, not auto-updated. Bump `slurm_version` or
  `slurm_mail_version` yourself when you want something newer.
