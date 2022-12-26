# Power saving
This is to document all the power saving steps I took on the ASUS S15 OLED,
version K3502ZA.306 (that last 306 might be BIOS version, so don't think too
much of it).

## Kernel Parameters

Power relevant kernel parameters I put in `/etc/default/grub`:

`GRUB_CMDLINE_LINUX_DEFAULT="intel_pstate=disable mem_sleep_default=deep
nmi_watchdog=0"` 
### intel\_pstate=disable
Intel Pstate on this laptop is a large pile of horse shit. For one, `cpupower`
settings don't work (see later section on cpupower). The CPU is constantly stuck
at a minimum frequency of 3.1 GHz, and

`echo powersave > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor`

does nothing at all.

However, the `acpi_cpufreq` driver responds correctly, so we will use it.

### mem\_sleep\_default=deep and sleep in general
Output of `cat /sys/power/state`:
```
freeze mem disk
```

Output of `cat /sys/power/mem_sleep`:
```
s2idle [deep]
```
A detailed meaning of all of these can be found in the [kernel docs](https://www.kernel.org/doc/html/v4.15/admin-guide/pm/sleep-states.html).

In breif however,
* `freeze` stands for `s2idle` sleep state, which is the suspend-to-idle state
* `mem` stands for whatever is in the `mem_sleep` file
* `disk` stands for hibernation
* `s2idle` in the `mem_sleep` file gives the value of `s2idle` to mem in the
`state` file.
* `deep` in the `mem_sleep` file gives the value of `s2idle` to `mem` in the
`state` file. Stands for `suspend to RAM` or `S3` sleep

This is used in conjucntion with `systemd-logind` to set the sleep state when
the laptop lid is closed. Note that acpid is not used because I don't want to
use another program.

Output of `cat /etc/systemd/logind.conf`:
```
# See logind.conf(5) for details.

[Login]
#NAutoVTs=6
#ReserveVT=6
#KillUserProcesses=no
#KillOnlyUsers=
#KillExcludeUsers=root
#InhibitDelayMaxSec=5
#UserStopDelaySec=10
#HandlePowerKey=poweroff
#HandlePowerKeyLongPress=ignore
#HandleRebootKey=reboot
#HandleRebootKeyLongPress=poweroff
#HandleSuspendKey=suspend
#HandleSuspendKeyLongPress=hibernate
#HandleHibernateKey=hibernate
#HandleHibernateKeyLongPress=ignore

HandleLidSwitch=suspend

#HandleLidSwitchExternalPower=lock
HandleLidSwitchDocked=ignore
#PowerKeyIgnoreInhibited=no
#SuspendKeyIgnoreInhibited=no
#HibernateKeyIgnoreInhibited=no
#LidSwitchIgnoreInhibited=yes
#RebootKeyIgnoreInhibited=no
#HoldoffTimeoutSec=30s
#IdleAction=ignore
#IdleActionSec=30min
#RuntimeDirectorySize=10%
#RuntimeDirectoryInodesMax=
#RemoveIPC=yes
#InhibitorsMax=8192
#SessionsMax=8192
#StopIdleSessionSec=infinity
```
Check out this [arch wiki](https://wiki.archlinux.org/title/Power_management#Power_management_with_systemd) page as well.

It describes all the options that `HandleLidSwitch` can have, as well as what
other options mean.

Over here though, `suspend` means put the laptop in the state that is the
contents of `mem` in the `/sys/power/state` file.

Closing the laptop lid reliably puts the suspend. Systemd also sends the right
messages to the session through `dbus`, because `swaylock` works as expected,
and locks the screen and everything before the laptop is put to sleep.
#### Problems with sleep
However, not everything is sunshine and roses.

Currently, I'm using `[deep]` by default, so let's start with that. Sleeping
only works once, ie, when I first close the lid, the laptop goes to sleep
correctly. However, when I resume from sleep, a number of things go wrong:
* Function hotkeys stop working
* The power button stops working
* The mouse stops working on sway, and needs to be clicked many times to work on
gnome
* Closing the laptop lid again, does nothing, ie, screen doesn't lock, and the
laptop stays powered on

Switching it to `[s2idle]` is not a lot better, and suffers from these problems:
* If on a tty, dmesg warnings are written to the screen
* The same warnings appear in `dmesg` for some reason
* Fans get super loud for whatever reason, and laptop is hot
* battery of course, drains like Romans draining a swamp

Sleep seems to be a glorious mess. S0 sleep / Modern standby or even regular
standby (S1) don't seem to exist. S3 is the only sleep state to exist apart from
hibernation through the `[deep]` option, and `[s2idle]` might as well not exist.

### nmi\_watchdog=0
This kernel option seems to do nothing at all, because `pgrep watchdog` still
returns something, which is annoying. Further more, the watchdog module
iTCO\_wdt don't seem to work at all.

Changing `#RuntimeWatchdogSec=off` in `/etc/systemd/system.conf` should turn it
off, but I'll keep you updated.

If that also doesn't work, I'll try blacklisting the module

## Module configuration
Some module options save power, on wifi and pcie and stuff
### wifi
My Wifi card on this particular model of laptop on this particular laptop is an
Intel Alder Lake PCH CNVi WiFi card. Running `lscpi -kvv|grep -iA10 net` gives
us all the information about the network controller.

The above command tells us that the driver and module being used are `iwlwifi`.
Thus, following this [Arch
wiki](https://wiki.archlinux.org/title/Power_management#Intel_wireless_cards_(iwlwifi)) page about wifi, I have made the following changes:

In `/etc/modprobe.d/iwlwifi.conf`:
```
options iwlwifi power_save=1
options iwlmvm power_scheme=3
```	
The `iwlmvm` option is there because for some reason, that module uses the
`iwlwifi` module, and the wiki asks me to put that line in as well.
### ASPM
Active state Power management should not exist on this model of laptop, because
the command `lspci -vv|grep -i aspm` returns absolutely nothing.
