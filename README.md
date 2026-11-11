# Sonatype Nexus Repository Manager — DigitalOcean Lab

This repository documents my practical learning while installing and configuring **Sonatype Nexus Repository Manager** on an Ubuntu DigitalOcean Droplet.

The objective is to understand artifact management and how Nexus fits into a CI/CD workflow, while practising Linux and cloud administration.

> **Lab environment:** DigitalOcean Droplet, Ubuntu, 4 vCPUs, 8 GB RAM and 80 GB disk. This is a learning environment, not a production sizing recommendation.

## 1. What is artifact management?

A software build can produce artifacts such as Java JAR/WAR files, npm packages and Docker images. These need to be stored, versioned and distributed reliably.

- **GitHub** is used to version-control source code and collaborate.
- **Nexus** stores, organizes and distributes build artifacts and dependencies.

### Repository types

| Repository | Purpose |
|---|---|
| Hosted | Stores packages published by an organization |
| Proxy | Connects to external repositories and caches requested packages |
| Group | Combines hosted and proxy repositories behind one endpoint |

A group repository does not replace hosted or proxy repositories. It provides a combined access point.

### Important Nexus concepts

- **Repository:** A logical location and endpoint for components.
- **Blob store:** Stores the actual binary content.
- **Metadata:** Helps Nexus identify and organize components and assets.
- **Component:** A package at a particular version.
- **Asset:** An individual file belonging to a component, such as a JAR, POM or source archive.

Nexus supports formats such as Maven, npm, Docker, PyPI and NuGet.

## 2. Lab environment

| Item | Value |
|---|---|
| Cloud provider | DigitalOcean |
| Operating system | Ubuntu |
| Droplet resources | 4 vCPUs, 8 GB RAM, 80 GB disk |
| Installation method | Native Linux installation from Sonatype distribution |
| Linux account running Nexus | `nexusnsai` |
| Nexus web port | `8081` |
| Installation directory | `/opt/nexus-3.98.1-01` |
| Data directory | `/opt/sonatype-work/nexus3` |

For this exercise I used a native installation rather than Docker, so I could practise Linux account ownership, permissions, process management and troubleshooting.

## 3. Installation and configuration

### Connect to the Droplet

```bash
ssh root@YOUR_DROPLET_PUBLIC_IP
```

Replace the placeholder with the Droplet's public IPv4 address.

### Inspect the existing Linux account

I chose to use the existing `nexusnsai` account instead of creating a separate account named `nexus`.

```bash
whoami
id nexusnsai
groups nexusnsai
```

### Download and extract Nexus

Download the appropriate Nexus Repository Manager archive from Sonatype. The version used in this lab was `3.98.1-01`.

Extract the downloaded archive (the filename may differ by release):

```bash
tar -xvzf nexus-3.98.1-01-unix.tar.gz
```

### Configure the account that runs Nexus

The startup configuration file is:

```text
/opt/nexus-3.98.1-01/bin/nexus.rc
```

I configured it to use the existing account:

```bash
run_as_user="nexusnsai"
```

This tells the Nexus startup script which Linux account should run the application.

### Check ownership and permissions

```bash
ls -ld /opt/nexus-3.98.1-01
ls -ld /opt/sonatype-work
ls -la /opt/sonatype-work/nexus3
```

If ownership needs to be corrected, an administrator can run:

```bash
chown -R nexusnsai:nexusnsai /opt/nexus-3.98.1-01
chown -R nexusnsai:nexusnsai /opt/sonatype-work
```

**Important:** Run administrative commands only from an authorized root shell. In this lab, `nexusnsai` does not have `sudo` privileges. I used `su -` to switch to root when system-level changes were needed, and `su - nexusnsai` to return to the normal account. Do not recursively change ownership of unrelated system directories.

### Start Nexus

```bash
/opt/nexus-3.98.1-01/bin/nexus start
```

The executable is named `nexus`; `nexusnsai` is the Linux account name.

## 4. Verify the service

Starting an application is not enough. I checked its process, listening port and logs.

### Inspect the process

```bash
ps -ef | grep '[n]exus'
ps -u nexusnsai -f
```

### Check port 8081

```bash
ss -lnt | grep ':8081'
```

A listening socket on port `8081` indicates that Nexus is listening. An address such as `*:8081` means it is bound to all network interfaces; firewall rules must still restrict external access.

### Inspect the application log

```bash
tail -n 40 /opt/sonatype-work/nexus3/log/nexus.log
```

In this lab, the log confirmed that the Sonatype Nexus Repository application had started successfully.

## 5. Access the web interface

After confirming the service and configuring network access, open:

```text
http://YOUR_DROPLET_PUBLIC_IP:8081
```

The initial administrator username is:

```text
admin
```

The initial password is generated in:

```text
/opt/sonatype-work/nexus3/admin.password
```

Read it locally on the server:

```bash
cat /opt/sonatype-work/nexus3/admin.password
```

Keep this password private. Never commit it, publish it in screenshots or share it in a repository. Complete the initial setup and change the administrator password when prompted.

## 6. Network security

- Restrict SSH access to trusted source IP addresses where practical.
- Do not expose the Nexus administration interface to the entire internet.
- If browser access is required for this lab, allow inbound TCP port `8081` only from your own public IP in the DigitalOcean Cloud Firewall; check the Ubuntu firewall too.
- For production, configure HTTPS using a suitable reverse proxy or supported TLS configuration.
- Never commit passwords, access tokens, private keys or database credentials.

## 7. Troubleshooting lessons

| Problem | Lesson |
|---|---|
| `su: user nexus does not exist` | The configured account must exist. I selected `nexusnsai` and updated `nexus.rc`. |
| Permission denied under `/opt` | System installation paths may require root privileges to modify. |
| `nexusnsai is not in the sudoers file` | A normal account cannot automatically use `sudo`; use an authorized administrator account for system changes. |
| `netstat: command not found` | `netstat` is part of the optional `net-tools` package. The built-in `ss` command can check listening sockets without installing it. |
| Need to confirm startup | Check the process, listening port and application log rather than relying only on the startup message. |

Useful account commands:

- `su -` switches to root when the root password is available.
- `su - nexusnsai` switches to the Nexus Linux account.
- `whoami` shows the active account.
- `exit` returns to the previous shell.

## 8. Where Nexus fits in CI/CD

A typical workflow I am working towards:

```text
Developer
   |
   v
GitHub (source code)
   |
   v
Jenkins (CI pipeline)
   |
   v
Build and test (for example, Maven)
   |
   v
Nexus (publish and store versioned artifacts)
   |
   v
Deployment (retrieve the required artifact version)
```

Jenkins and the build tool produce and publish the artifact. Nexus stores and distributes it. Nexus does not compile the application's source code.

## 9. Next practical steps

- Complete the Nexus web-based initial setup.
- Create a Maven hosted repository for internally published artifacts.
- Create a Maven proxy repository for external dependencies.
- Create a Maven group repository as a combined dependency endpoint.
- Publish and retrieve a sample Maven artifact.
- Integrate Maven and Jenkins with Nexus.
- Explore npm and Docker repository formats.
- Review backup, access control, HTTPS and production database considerations.

---

**Learning outcome:** This exercise connected artifact repository concepts with practical cloud administration: Linux users and groups, file ownership, application configuration, process monitoring, port verification, logs and firewall restrictions.

This document records the learning lab, not a production deployment guide. Adapt and verify commands for the Nexus release and operating system in use.
