# Gitea SSH Proxy
**This is a fork from the awesome [Gitlab SSH Proxy](https://github.com/rendyanthony/gitlab-ssh-proxy) by [rendyanthony](https://github.com/rendyanthony).**

I modified the files to work with Gitea and adjusted the documentation.
Most of the credit goes to the original author and his amazing work.

**NOTE:** I have only tested this on my own setup on an Ubuntu 24.04 Machine without SE_LINUX. So I don't know if that module works.

---

One if the issue with running a Gitea instance in a container is to expose the Gitea SSH in the host machine without conflicting with the existing SSH port (22) on the host.

What usually happens:

```
git://git@hostname:222/username/project.git  
```

We want them to look like this instead:

```
git://git@hostname/username/project.git  
```

## Step by Step

Build and install.

```
sudo ./setup.sh install
```

It will do the following things:
1. Copy the follwoing scripts to `/usr/local/bin`
    - [`gitea-keys-check`](gitea-keys-check)
    - [`gitea-cli-proxy`](gitea-cli-proxy)
1. Build and install an SE Linux policy module: [`gitea-ssh.te`](gitea-ssh.te) to allow scripts executed from the SSH daemon to run the `ssh` binary.
1. Copy [`99-gitea-proxy.conf`](99-gitea-proxy.conf) to `/etc/ssh/sshd_config.d/`

Create the `git` user on the host

```bash
sudo useradd -m git
```

Create a new SSH key-pair

```bash
sudo su - git -c "ssh-keygen -t ed25519"
```

Assuming the Gitea container is names "gitea", write the public key to /data/git/.ssh/authorized_keys.

```bash
sudo sh -c "cat /home/git/.ssh/id_ed25519.pub | docker exec -i gitea sh -c 'cat >> /data/git/.ssh/authorized_keys'"
```

Fix the permission/ownership of the `authorized_keys` file to ensure that is only readable by the `git` user within the container. Otherwise SSH won't use the file.

```bash
# If you are using podman, substitute docker with podman
sudo docker exec -u root gitea \
    chmod 600 /data/git/.ssh/authorized_keys
sudo docker exec -u root gitea \
    chown git:git /data/git/.ssh/authorized_keys
```

Reload the SSH Service

```bash
sudo systemctl reload sshd
```

You're good to go!

### Configuration

By default the scripts assumes that the Gitea SSH service is accessible at `git@localhost` port `222`. If your setup is different, you can override this by creating a file named `gitea-ssh.conf` in `/home/git/.config` or `/etc`.

In this file you can define the following environment variables:

```bash
GITEA_HOST=git@localhost
GITEA_PORT=222
```

## Uninstall

This will remove all the script files and the SE Linux proxy module.

```
sudo ./setup.sh remove
```
