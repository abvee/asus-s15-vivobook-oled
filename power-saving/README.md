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

There one caveat however. Running `cpupower frequency-info` gives us this line:
```
hardware limits: 400 MHz - 3.10 GHz
```
Basically, we are limited to 3.10GHz instead of the advertised 4.5GHz on this
laptop if we use this driver. That is a sacrifice I'm willing to make for more
battery life, as I know if I need the performance, I can always switch back to
the `intel_pstate` driver, but your preference may vary. It is also possible
that with future kernel updates, the pstate driver becomes the better option, so
I'm gonna keep testing both drivers every time.

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
the laptop lid is closed. Note that acpid is not used to set the sleep state
because I don't want to use another program.

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
`HandleLidSwitch` tells systemd what to do when the laptop lid is closed. I have
set it to suspend, which is also the default option.
Check out this [arch wiki](https://wiki.archlinux.org/title/Power_management#Power_management_with_systemd) page as well.

It describes all the options that `HandleLidSwitch` can have, as well as what
other options mean.

Over here though, `suspend` means put the laptop in the state that is the
contents of `mem` in the `/sys/power/state` file.

Closing the laptop lid reliably puts the system into deep sleep, which is what
we hav set our `mem` value to. Systemd also sends the right
messages to the session through `dbus`, because `swaylock` and `swayidle` work as expected,
and locks the screen before the laptop is put to sleep.
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

UPDATE: didn't work, I stopped caring, and just let the watchdog run for now

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

However, other things that relate to ASPM like this file:
`/sys/module/pcie_aspm/parameters/policy` that dictate ASPM policy exist. I am
unsure as to if ASPM exists or not on this computer, especially, since I can
write "powersave" to that file, which should not work unless ASPM exits.

I can always try and force it with kernel line `pcie_aspm=force`, but I feel
like that might half work, and half not work, so I'm not gonna try it.

Either ways, I'll probably try a service which echos "powersave" or
"powersupersave" to the required file later on, even though no module should
exit for it.

## cpupower
This is a very simple section about how you can use cpupower to set the governor
and limit maximum frequency.

### Scaling governors
As recalled in the "Kernel Parameters" section, we are using the `acpi_cpufreq`
driver and not the `intel_pstate` driver. Thus, by default we are using the
`schedutil` scaling governor, which can be seen in this file:

`/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor`

In that folder you will also find a bunch of handy files:
* `scaling_available_governors` which contains what governors are available
* `scaling_available_frequencies` which contains what frequencies we can properly set
* `scaling_driver` and `scaling_governor` are exactly what you think

We can set the governor manually by either writing to the `scaling_governor`
file (eg `echo powersave > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor`).
However, I like to use `cpupower` manually with:

`cpupower -g *governor*` or through the `/etc/default/cpupower` file which is
pretty straight forward, and is run when the `cpupower.service` file is enabled
by systemd.

So, the file looks like this:
```
governor='powersave'

# Limit frequency range
# Valid suffixes: Hz, kHz (default), MHz, GHz, THz
#min_freq="2.25GHz"
max_freq="1.4GHz"
```
Your file might contain more detailed documentation ;)
### Scaling frequencies
Also in `/sys/devices/system/cpu/cpu0/cpufreq/` are the 2 files:
* `scaling_max_freq` which sets max frequency
* `scaling_min_freq` which sets min frequency

As usual, these can be set manually by echoing values to them, but using
`cpupower` is just more convinient.
