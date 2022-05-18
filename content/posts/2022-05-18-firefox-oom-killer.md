---
title: "Guiding the OOM Killer toward Firefox subprocesses"
date: 2022-05-18T06:15:00-07:00
---

I recently figured out a combination of cgroups usage and scripting that
improves how my desktop behaves under web-related memory pressure.

### Systemd Units

Use `memcg` to restrict `firefox` processes in aggregate to N% of system RAM:

```
$ cat /usr/lib/systemd/system/firefox.slice
[Unit]
Description=Firefox
Before=slices.target

[Slice]
MemoryAccounting=true
MemoryMax=75%
```

Customize `OOMPolicy` such that `systemd` allows parent `firefox` processes to
continue living, even if their children are OOM-killed.

```
$ cat /usr/lib/systemd/system/firefox.service
[Unit]
Description=Firefox
After=network.target

[Service]
Environment=DISPLAY=:0
ExecStart=/usr/bin/firefox-nightly
OOMPolicy=continue
Slice=firefox.slice
User=my-username

[Install]
WantedBy=graphical.target
```

Run `firefox` under this `memcg` configuration:

```
$ sudo systemctl start firefox.service
```

### Forkstat

Build `forkstat` [from source](https://github.com/ColinIanKing/forkstat) or
install it using distro package management. The author describes it thusly:

> Forkstat is a program that logs process fork(), exec() and exit() activity.
> It is useful for monitoring system behaviour and to track down rogue
> processes that are spawning off processes and potentially abusing the system.

### Adjust OOM Killer Scores Dynamically

Use `forkstat` to monitor the main `firefox` process for new
[subprocesses](https://wiki.mozilla.org/Electrolysis), then adjust the OOM
killer score of each subprocess accordingly:

```
$ cat firefox-choom.sh
#!/bin/sh

#
# Parse PIDs of firefox content processes from `forkstat` output like:
#
# 10:00:00 fork   100000 parent          firefox-nightly
# 10:00:00 fork   100001 child           firefox-nightly
#
# ... and pass the results to `choom`, in order to guide the OOM killer toward
# content processes (children) over the main chrome process (parent).
#
# Content processes should be reaped before the main chrome process, with the
# latter being OOM-killed only as a last resort. This is because the former can
# be killed while preserving the latter, while killing the latter causes all of
# the former to be killed transitively.
#
# Put more simply, we want to avoid OOM-killing the entire browser if possible.
#

PROGNAME='firefox-nightly'
sudo forkstat -lde fork                                                     \
  | sed -uEn "s@[0-9:]+[ ]+fork[ ]+([0-9]+)[ ]+child[ ]+$PROGNAME.*@\1@p"   \
  | while read line ; do choom -n +1000 -p "$line" ; done
```

Example usage and output:

```
$ ./firefox-choom.sh
pid 700000's OOM score adjust value changed from 0 to 1000
pid 700001's OOM score adjust value changed from 0 to 1000
pid 700002's OOM score adjust value changed from 0 to 1000
[...]
```

### Summary

Use cgroups to control the memory usage of `firefox` processes in aggregate,
OOM-killing the entire process tree if necessary.

Use `forkstat` and `choom` to guide the OOM killer towards content subprocesses
and away from the main parent process, such that whenever possible, only a
single browser tab (or set of related tabs) is OOM-killed, rather than the
entire browser UI.
