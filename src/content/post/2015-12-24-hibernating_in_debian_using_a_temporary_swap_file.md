---
date: 2015-12-24
title: Hibernating in Debian using a temporary swap file
categories:
    - how-to
tags:
    - power management
    - Linux
    - Debian
    - swap
    - system administration
---

This article explains how to configure a Debian Jessie system to
hibernate using swap space that's created just before suspending and
destroyed just after resuming. This trick allows hibernation on a
swap-free or swap-lite system—useful on a desktop system running with
enough RAM to make swap space otherwise unnecessary.

## How to make it work

### Determine the root partition of your system

```
$ rootdev=$(mount | grep ' / ' | awk '{ print $1 }')
```

_Important!_ Make sure you get this right. Otherwise your system may
fail to suspend or—much worse—fail to resume.

```
$ echo $rootdev
/dev/sda1
```

The second line (e.g., `/dev/sda1`) should be your system's root
partition.

### Install and configure `uswsusp`

```
$ sudo apt-get install uswsusp &&
  sudo sh -c "cat <<EOF >/etc/uswsusp.conf
resume device = $rootdev
compress = y
early writeout = y
shutdown method = platform
image size = 0
resume offset = 0
EOF"
```

Hibernating using a swap file—as opposed to a swap partition—requires
special support. We'll use `uswsusp`.

The `/etc/uswsusp.conf` is the configuration file for `uswsusp`. The
`image size` and `resume offset` lines are placeholders that we'll
overwrite each time before suspending.

_Note._ It's also possible to hibernate to a swap file using `systemd`,
but my only experience doing this is with Arch Linux, and, as far as I
know, the swap file must exist upon boot-up, thus preventing the swap
file from being created on demand.

### Update Grub to add the `resume` kernel parameter

```
$ { grep -q '\<GRUB_CMDLINE_LINUX_DEFAULT=.*resume=' /etc/default/grub &&
      (echo 'Warning: The `resume` parameter is already set.' >&2 || true) } ||
    { sudo sed --in-place \
        -e "s#^\\(GRUB_CMDLINE_LINUX_DEFAULT=\"\\)#\\1resume=$rootdev #" \
        /etc/default/grub &&
      sudo update-grub; }
```

The `resume` kernel is necessary to resume from a swap file.
Otherwise the kernel will not find the swap file, and the system will
boot normally—as though rebooting.

If the `resume` parameter is already set, you'll need to take
intelligent action to resolve.

### Install the `dynamic-hibernate` script

```
$ sudo wget -O /usr/local/sbin/dynamic-hibernate \
    https://raw.githubusercontent.com/cmbrandenburg/pcconf/e7dbe6128a9267b1f55fed27b22fcbfdc1734b3e/bin/dynamic-hibernate &&
  sudo chmod +x /usr/local/sbin/dynamic-hibernate
```

The `dynamic-hibernate` script does the actual hibernation. It:

1. Sets up a temporary swap file,
1. Updates the `/etc/uswsusp.conf` with the location of the new swap
   file,
1. Re-creates the system's initial RAM disk,
1. Suspends the system, and
1. Deletes the swap file after resuming.

By default, the script requests the kernel to store the smallest image
possible when suspending, presumably by discarding disk buffers and
disk-backed pages that may be safely reloaded after resuming. In my
experience, smaller images yield faster resume speeds when using a
spinning disk, and SSDs are fast enough to make any difference
negligible. Nevertheless, if you want to change this behavior then
modify the `image size` value in the script.

### Give yourself `sudo` permission to hibernate (optional)

```
$ sudo sed --in-place \
  -e "\$a$USER ALL = NOPASSWD: /usr/local/sbin/dynamic-hibernate" \
  /etc/sudoers
```

This gives permission to your current user account to run the
`dynamic-hibernate` script without entering a password.

### Create a script to lock the screen and hibernate (optional)

```
$ mkdir -p ~/bin &&
  touch ~/bin/hibernate &&
  chmod +x ~/bin/hibernate &&
  cat <<EOF >~/bin/hibernate
#!/bin/sh
gnome-screensaver-command --lock
sudo /usr/local/sbin/dynamic-hibernate
EOF
```

Locking the screen requires the user to enter a password after resuming.
Without locking the screen, anyone who resumes your computer during
hibernation will gain access to your system.

The above script works only if you're using the Gnome screensaver. If
you're using a different screensaver then you must change the
`~/bin/hibernate` script appropriately.

### Verify everything works

```
$ ~/bin/hibernate
```

Or:

```
$ sudo /usr/local/sbin/dynamic-hibernate
```

Your system will take a few moments to save state before powering off.
Power the system back on, and the system should restore its state as it
was before suspending.

## Additional considerations

Once you get hibernation working, it should continue to work reliably.
However, here are some further considerations.

* Don't hibernate after upgrading the kernel. Reboot first. Otherwise
  the kernel you resume with will be a different kernel than what you
  suspended with, and the mismatch will cause mayhem.

* If your system fails to suspend then there's a good chance the
  hibernation script will print a useful error message.

* If your system fails to suspend or resume then you may need to
  manually delete the swap file that gets left behind
  (`/swap.hibernate`).

* If your system fails to resume then your system will suffer the same
  consequences as though it crashed. This includes loss of application
  state, file corruption, etc.

## References

* https://github.com/cmbrandenburg/pcconf#hibernation. These are the
  notes I use to set up a desktop system.
