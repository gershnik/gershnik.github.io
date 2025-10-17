---
layout: post
title:  "Installing Gitea from binary on macOS"
description: "How to install Gitea from binary instead of Homebrew on macOS"
tags: macos
---

_Update 2025-10-16_: Added [workaround for macOS Tahoe upgrade bug](#upon-upgrade-to-macos-tahoe-user-git-gets-its-primary-group-reset-to-staff).
<hr style="border:none;height:1px;background-color:#f0f0f0;">

[Gitea](https://about.gitea.com/products/gitea/) is an awesome open source Git server that almost completely mimics Github.
Unfortunately, its only officially supported method of installation on macOS is via Homebrew (see 
[here](https://docs.gitea.com/installation/install-from-package#macos)).

If you, like me, are not a fan of Homebrew, it is possible to install it directly from binary.
Gitea's website provides reasonably detailed [instructions](https://docs.gitea.com/installation/install-from-binary) on how set it up on Linux. This page attempts to translate them to macOS.

If you follow this guide you might want to keep the original instructions open too for comparison and additional information.

* TOC
{:toc}

## Preliminaries

* If you are new to Gitea start by reading the manual on [Database Preparation](https://docs.gitea.com/installation/database-prep). 
Gitea can work perfectly with built-in SQLite database that does not require any preparation. 
If you plan to support many users and "scale" you might want to consider using a different DB.

* Make sure that your macOS system has Git installed (either from development tools or Xcode).


## Download

You can find the file matching your platform from the [downloads page](https://dl.gitea.com/gitea/) after navigating to the version you want to download.

You should choose `darwin-arm64` suffix if your hardware uses Apple Silicon, or `darwin-amd64` for Intel.

The following assumes that you have downloaded the file into `~/Downloads/gitea-xxx-darwin-yyy-arch`

## Install the executable and remove the mark of the web

```bash
sudo cp ~/Downloads/gitea-xxx-darwin-yyy-arch /usr/local/bin/gitea
sudo chmod a+x /usr/local/bin/gitea
sudo xattr -r -d com.apple.quarantine /usr/local/bin/gitea
```

Check that it works

```bash
/usr/local/bin/gitea --version
```

## Create a user to run Gitea

This is the part that differs most from how things are done on Linux.

In order to create a user and user's group for Gitea first of all you need to figure out what user ID (UID)
and group ID (GID) to use. You might already know the available numbers for your system but, if not,
the following two scripts will find them for you. Run them in terminal.

```bash
# Print a free UID and store it in the `GITEA_UID` variable.
GITEA_UID=200; while true ; do
  if ! (id $GITEA_UID>/dev/null 2>&1); then break; else ((GITEA_UID++)); fi
done; echo $GITEA_UID
```

```bash
# Print a free GID and store it in the `GITEA_GID` variable.
GITEA_GID=200; while true ; do
  if [[ $(dscl -plist . -search /Groups PrimaryGroupID $GITEA_GID) == "" ]]
    then break; else ((GITEA_GID++))
  fi
done; echo $GITEA_GID
```

The following assumes that the UID/GID are available in `$GITEA_UID` and `$GITEA_GID` variables.

### Create the group

```bash
sudo dscl . -create /Groups/git
sudo dscl . -create /Groups/git PrimaryGroupID $GITEA_GID
sudo dscl . -create /Groups/git RealName "Gitea Server"
sudo dscl . -create /Groups/git dsAttrTypeNative:IsHidden 1
```

### Create the user

```bash
sudo dscl . -create /Users/git
sudo dscl . -create /Users/git UniqueID $GITEA_UID
sudo dscl . -create /Users/git PrimaryGroupID $GITEA_GID
sudo dscl . -create /Users/git Password '*'
sudo dscl . -create /Users/git NFSHomeDirectory "/var/lib/gitea/home"
sudo dscl . -create /Users/git UserShell /bin/bash
sudo dscl . -create /Users/git RealName "Gitea Server"
sudo dscl . -create /Users/git dsAttrTypeNative:IsHidden 1
```

**Note 1**: having shell be `/bin/bash` is important. Do not try to set it to `/usr/bin/false` like other 
macOS daemon accounts do.

**Note 2**: I have chosen to put the users home directory in `/var/lib/gitea/home` rather than more "normal"
`Users/git`. This is deliberate because daemon users normally do not get a directory under `Users` on macOS.


### Allow SSH access for the user

Note: technically this step is only necessary if you plan to use SSH protocol with your Git server. If you 
do not (probably a bad idea) you don't need it.

```bash
sudo dseditgroup -o edit -a git -t user com.apple.access_ssh
```

## Create required directory structure

```bash
sudo mkdir -p /var/lib/gitea/{custom,data,log,home}
sudo chown -R git:git /var/lib/gitea/
sudo chmod -R 750 /var/lib/gitea/
sudo mkdir /etc/gitea
sudo chown root:git /etc/gitea
sudo chmod 770 /etc/gitea
```

## Create the daemon plist file

The following uses `nano` editor from command line. Feel free to use your favorite GUI editor 
but keep in mind that you will need to be root to edit the config file.

```bash
sudo nano /Library/LaunchDaemons/io.gitea.web.plist
```

Copy the following into it, save (Ctrl-O) and exit (Ctrl-X)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>io.gitea.web</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/gitea</string>
        <string>web</string>
        <string>-c</string>
        <string>/etc/gitea/app.ini</string>
    </array>
    <key>UserName</key>
    <string>git</string>
    <key>GroupName</key>
    <string>git</string>
    <key>WorkingDirectory</key>
    <string>/var/lib/gitea</string>
    <key>EnvironmentVariables</key>
	<dict>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/sbin</string>
        <key>GITEA_WORK_DIR</key>
        <string>/var/lib/gitea</string>
        <key>HOME</key>
        <string>/var/lib/gitea/home</string>
    </dict>
    <key>KeepAlive</key>
    <dict>
        <key>Crashed</key>
        <true/>
    </dict>
    <key>RunAtLoad</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/var/lib/gitea/log/stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/var/lib/gitea/log/stderr.log</string>
</dict>
</plist>
```

## Load and start the daemon

```bash
sudo launchctl load -w /Library/LaunchDaemons/io.gitea.web.plist
```

If everything works fine `/var/lib/gitea/log/stdout.log` should contain sane output and you should be able to navigate to `http://localhost:3000` to perform the first time setup. 

You can start/stop the daemon via:

```bash
sudo launchctl stop io.gitea.web
sudo launchctl start io.gitea.web
```

## Post-installation

Once the initial set up is complete you should do the following:

### Remove write access to Gitea config from its user.

```bash
sudo chmod 750 /etc/gitea
sudo chmod 640 /etc/gitea/app.ini
```

**Note**: do not be tempted provide read access to other users. Gitea stores security sensitive information in 
`app.ini` that nobody other than `root` should see.

### Configure proper logging

By default Gitea logs to `stdout`/`stderr` which will go to `/var/lib/gitea/log/std{out|err}.log`. You could,
if you wanted, use `newsyslogd` to manage this file but it is far simpler to just let Gitea log into a file.
It automatically handles proper log rotation and compression (see [here](https://docs.gitea.com/administration/logging-config) for details).

To do this, edit `/etc/gitea/app.ini`. Locate the `[log]` section and change `MODE` to `file`

```ini
[log]
MODE = file
```

Now the log will be in `/var/lib/gitea/log/gitea.log`.


That's pretty much it. Further tweaks and configuration can be done by editing `/etc/gitea/app.ini`, configuring
`sshd` and Gitea web interface.

## Known Issues

### Upon upgrade to macOS Tahoe user `git` gets its primary group reset to `staff`

macOS had issues with resetting users group membership upon upgrade for a long time. These get reported, fixed and
then surface again. 😠

If upon upgrade to Tahoe SSH connections to Gitea server suddenly stop working and you get the dreaded "Could not read from remote repository" error check the `git` user primary group:

```bash
id git
```

If it reports something like "uid=xxx **gid=20(staff)** then you have hit this issue. It should be
"uid=xxx gid=yyy(**git**)`. To rectify:

- Find the group ID of the `git` group:
  ```bash
  dscl . -read /Groups/git PrimaryGroupID
  ```
- Then re-apply this group ID to the `git` user:
  ```bash
  sudo dscl . -create /Users/git PrimaryGroupID <the id from previous step>
  ```

