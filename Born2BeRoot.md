*This project has been created as part of the 42 curriculum by gvan-box.*

# Born2BeRoot

## Description

Born2BeRoot is a system administration project: set up a minimal, hardened **Debian** server inside a virtual machine, following strict security rules. There is no code to hand in; the deliverable is a `signature.txt` with the SHA1 hash of the virtual disk, and the setup is checked live during the defence.

No graphical interface is allowed, so everything is configured from the command line.

## What was set up

### Disk

- At least two **encrypted partitions**, managed with **LVM** (logical volumes for root, swap, home and so on).

### Access

- An **SSH** server on port **4242**, with root login over SSH disabled.
- **UFW** as firewall, active at boot and leaving only port 4242 open.
- **AppArmor** running at boot.
- The hostname is the login followed by `42`; a user with that login belongs to the `user42` and `sudo` groups.

### Password policy

- Passwords expire every 30 days; they can be changed at most once every 2 days; a warning is shown 7 days before expiry.
- At least 10 characters, with an uppercase letter, a lowercase letter and a digit, and no more than 3 identical characters in a row.
- A password may not contain the user's name, and a new password must differ from the old one in at least 7 characters (this last rule does not apply to root).

### sudo

- At most 3 attempts for a wrong password, with a custom error message.
- Every `sudo` command, with its input and output, is logged in `/var/log/sudo/`.
- TTY mode is required, and the paths `sudo` may use are restricted (`secure_path`).

### monitoring.sh

A Bash script that broadcasts system information to all terminals with `wall` at startup and every 10 minutes, scheduled with `cron`:

```
#Architecture: Linux ... x86_64 GNU/Linux
#CPU physical : 1
#vCPU : 1
#Memory Usage: 74/987MB (7.50%)
#Disk Usage: 1009/2Gb (49%)
#CPU load: 6.7%
#Last boot: 2021-04-25 14:45
#LVM use: yes
#Connections TCP : 1 ESTABLISHED
#User log: 1
#Network: IP 10.0.2.15 (08:00:27:51:9b:a5)
#Sudo : 42 cmd
```

## What I learned

- How a Linux system boots and how disks, partitions, LVM and encryption fit together.
- Package management on Debian (`apt` versus `aptitude`) and what AppArmor does.
- The difference between users, groups and privileges, and why `sudo` is preferred over logging in as root.
- How PAM enforces a password policy, and how a firewall and SSH configuration limit what can reach a machine.
- Writing a Bash script that gathers system information from `/proc`, `free`, `df`, `lsblk`, `ss`, `who` and the sudo log.

## Resources

- The 42 Born2BeRoot subject.
- `man` pages of `sshd_config`, `ufw`, `sudoers`, `login.defs`, `pam_pwquality`, `lvm`, `cron` and `wall`.

### AI usage

This README was written with Claude (Anthropic), based on the project subject.
