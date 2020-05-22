# ansible-role-rsync_server

This Ansible role will install rsync and configure it to run as a deamon (i.e.
"server mode"/"ryncd") with automatic start at boot.



# Installation

This repository does not have any dependencies, so just move into your `roles/`
folder and run the following:

```bash
git clone git@github.com:JonasAlfredsson/ansible-role-rsync_server.git rsync_server
```

If you would like to download any updates for this role in the future, you may
use the following command from within the previously cloned folder:

```bash
git pull
```

When the configuration is complete you may then just include this role in your
main playbook like this:

```yaml
- hosts: all
  name: Install rsync and configure it to run as a deamon
  roles:
    - rsync_server
```



# Usage

Since the rsync shares are often unique to each individual host, I usual prefer
to define these individual configurations in their respective
`host_vars/{{ ansible_hostname }}` file. However, if you have multiple
identical machines there should not be any problem to define all of this in one
of the `group_vars/` files.

> Check out the [Ansible Hashes](#ansible-hashes) section for some useful
  information about how Ansible (doesn't) handle "merging" of lists.


## Example Configuration
The following examples have all the available variables included, and all the
default values written out. So any field which is not marked with `# Required`
may be left out of your configuration if you are fine with the defaults.

If you want more information about any of the rsync deamon/sever settings I
recommend [this page][6], and then [this page][7] for anything related to the
client.

### Global Settings
```yaml
rsync_config_file: "/etc/rsyncd.conf"

# Daemon and Logging
rsync_pid_file: "/var/run/rsyncd.pid"
rsync_lock_file: "/var/run/rsync.lock"
rsync_log_directory: "/var/log/rsyncd"
rsync_log_format: '%o %h [%a] %m (%u) %f %l'
rsync_max_verbosity: 1

# Security and Users
rsync_uid: "nobody"
rsync_gid: "nogroup"
rsync_use_chroot: true
rsync_secrets_file: "/etc/rsyncd.secrets"
rsync_auth_users: []
rsync_hosts_allow: []

# Networking
rsync_port: 873
rsync_max_connections: 0
rsync_timeout: 3600

# Shares and Files
rsync_motd_file:
rsync_list: true
rsync_read_only: true
rsync_exclude: []
rsync_dont_compress: []
```

These are pretty sane defaults, so the only things you may want to look into
are perhaps locking down the server by defining
[`rsync_auth_users`](#authorizing-users), `rsync_hosts_allow` and
[`rsync_exclude`](#exclude-paths).

### Shares
```yaml
rsync_shares:
   - name:  # Required
     path:  # Required
     comment:
     list:
     uid:
     gid:
     read_only:
     auth_users: []
     exclude: []
     timeout:
   - name:
     path:
     ...
```

The `rsync_shares` variable is a list, which means that it is possible to expand
it to as many shares as you want. You should only make sure that every share has
a unique name in order to not cause conflicts.

Most of the available options here are already defined in the
"[Global](#global-settings)" section, but anything that is added here will
take precedence over those.

### Rsync Users
```yaml
rsync_users:
   - user:  # Required
     password:  # Required
     comment:
```

The username and password defined here has nothing to do with the username or
password of the user on the system (they don't even need to exist as a user on
the system). The user and group values the files/transfers will get is defined
by the `uid` and `gid` options in the
[share](#shares)/[global](#global-settings) configuration. This means that it
is possible to "impersonate" other system users if configured wrongly. Take a
look at the [Authorizing Users](#authorizing-users) section for a suggested
solution to this.

### Logrotate
```yaml
rsync_logrotate_interval: "weekly"
rsync_logrotate_count: 10
```

The logrotate settings are pretty self explanatory, and should not need tuning
unless you really want to.



# Good to Know

## Authorizing Users
As was mentioned in the [Rsync Users](#rsync-users) section, the username and
password for authorizing with the rsync server has nothing to do with the users
on the system. The access rights a connecting user will get is defined by the
`uid` and `gid` options in the [share](#shares)/[global](#global-settings)
configuration, which defaults to "nobody" and "nogroup" if nothing is changed.

The "nodbody" user is extremely limited in what it can do/access, so it might
be desirable to actually use a system user/`uid` with more access rights.
However, if you hard-code a single `uid` on the share then everyone that logs
into that share will be using the same system user, which is not ideal.

The solution to this is to use the [RSYNC_USER_NAME][1] environment variable,
in order to request that rsync run as the authorizing user. For example, if you
want a rsync to run as the same user that was received for the rsync
authentication, this setup is useful:

```conf
uid = %RSYNC_USER_NAME%
gid = *
```

This will make so that the rsync user, with username "jonas", gets assigned the
system `uid`/username of "jonas" which *could* be a real system user (but does
not need to be). However, the password the rsync user authenticates with is
still not the system password, but rather the one that was defined in the
[Rsync Users](#rsync-users) section.


## Exclude Paths
All exclude paths that are defined are always relative to the mount point of
the share. What this means is if the following file exists `/path/to/file.txt`,
and the share's is mounted to `/path/to`, then you will only need to add
"file.txt" to the [global](#global-settings)/[shares](#shares) `exclude` list
for it to be ignore. More detailed examples are available [here][8].


## Ansible Hashes
An important thing to remember is that Ansible will overwrite, and not merge,
hashes/dictionaries (like `rsync_shares` or `rsync_users`) if there are two
with the same name. You can therefore not have a part of them be defined in the
`group_vars/` and then other parts in the `host_vars/`. If you do not like this
behavior you may look into setting [`hash_behaviour = merge`][2], but be aware
that this is not a very [good solution][3]. Instead you should probably look
into the [`combine`][5] filter or the [`merge_vars`][4] action plugin.






[1]: https://download.samba.org/pub/rsync/rsyncd.conf.html#uid
[2]: https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-hash-behaviour
[3]: https://medium.com/uptime-99/3-things-ive-learned-about-ansible-the-hard-way-bae341524a86
[4]: https://pypi.org/project/ansible-merge-vars/
[5]: https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#combining-hashes-dictionaries
[6]: https://download.samba.org/pub/rsync/rsyncd.conf.html
[7]: https://linux.die.net/man/1/rsync
[8]: https://www.thegeekstuff.com/2011/01/rsync-exclude-files-and-folders/
