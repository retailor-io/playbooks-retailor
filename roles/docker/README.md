# docker

Installs Docker CE from the official Docker apt repository on Ubuntu/Debian.

## What changed from the original role

The original role had several bugs that would have made it fail or behave
unpredictably:

- **`tasks/main.yml` was dead code that duplicated `install.yml` incorrectly.**
  Nothing ever included it (only `main-distro-check.yml` was wired up), and
  it referenced undefined variables (`install_packages`,
  `clean_install_remove_packages` had no `become`, etc). It's been replaced
  by a single real entrypoint.
- **`install.yml` added the apt repo but never fetched the GPG key**, so
  `apt-get update` would fail signature verification. Key download and repo
  registration are now in one idempotent flow using `get_url` +
  `apt_repository` instead of raw `shell` commands.
- **`check_distro.yml` used `meta: end_play`**, which ends the *entire
  Ansible run* for *all* hosts in the play, not just this role/host. This is
  almost never what you want in a multi-host play. It's replaced with a
  `docker_distro_supported` fact that gates subsequent tasks, with an
  optional hard `fail` (`docker_fail_on_unsupported_distro`, default `true`).
- **`service.yml` had the exact same task listed twice** (one with a `when`
  guard, one without). Deduplicated into a single task.
- **Package state `latest`** was used for prerequisites and Docker itself,
  which causes unnecessary upgrades (and potential breakage) on every run.
  Changed to `present`; use `docker_install_packages` + your own
  update strategy if you want upgrades.
- **No handler was ever notified.** Installing/upgrading Docker packages now
  notifies the `Restart Docker` handler.
- **`docker` group creation used the `user` module** to create a "user" named
  docker, which is not the same as a group and wasn't added to any actual
  user. Replaced with a proper `group` task plus a `docker_users` list you
  populate to add real users to the group.
- All tasks now use fully-qualified module names (`ansible.builtin.*`) and
  `name:`/`state:` keyword syntax instead of the deprecated
  `key=value` shorthand.

## Role variables

See `defaults/main.yml` for the full list. Commonly overridden:

```yaml
docker_users:
  - deploy
  - ci

docker_install_packages:
  - docker-ce
  - docker-ce-cli
  - containerd.io
  - docker-buildx-plugin
  - docker-compose-plugin

docker_fail_on_unsupported_distro: true
```

## Example playbook

```yaml
- hosts: docker_hosts
  become: true
  roles:
    - role: docker
      vars:
        docker_users:
          - "{{ ansible_user }}"
```

## Tags

- `docker_check_distro`
- `docker_install`
- `docker_users`
- `docker_service`
