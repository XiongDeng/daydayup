`sudo loginctl enable-linger $USER` makes systemd keep your user's services running even when you aren't logged in.

**The pieces**

- `loginctl` is the command-line tool for systemd's login manager, `systemd-logind`. It manages user sessions.
- `enable-linger <user>` marks that user as "lingering".
- `$USER` is your current username, so it applies to your account only. You need `sudo` because it changes a system-wide setting.

**What it changes**

- **Without linger:** when your last login session ends (logout, closing the SSH session), systemd stops your user service manager and kills your user services. When you log in again, it starts them again.
- **With linger:** systemd starts your user manager at boot and keeps it running. Services you enabled with `systemctl --user enable` run all the time, including after logouts and with nobody logged in.

**How it's stored**  
It creates an empty file at `/var/lib/systemd/linger/<username>`. That is why it persists across reboots and you only run it once.

**Check and undo**

```sh
loginctl show-user $USER | grep Linger   # Linger=yes
ls /var/lib/systemd/linger/              # your username is listed
sudo loginctl disable-linger $USER       # back to the default
```

**Notes**

- It doesn't give the service extra privileges. Your services still run as you.
- It applies to all of your enabled user services, not only `when-we-sleep`.