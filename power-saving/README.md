This is to document all the power saving steps I took on the ASUS S15 OLED,
version K3502ZA.306 (306 is the current BIOS version).

Most of the article is focused on lowering power consumption to get as much
battery life as possible . Most of these modifications can be done with only
installing `cpupower` or installing nothing at all.
# Monitoring
## Monitoring Power
There are 2 symlinks in the `/sys/class/power_supply`
called `AC0` and `BAT0`, standing for the AC input and battery
respectively.

We want the power drawn from the battery, and `AC0` is does not contain any
particularly useful information.

The `/sys/class/power_supply/BAT0` directory does **NOT** have a commonly found
file called `power_now`, which would tell us how much power was drawn.
It does have `current_now` and `voltage_now` files though, which tell us in real
time the current and voltage from the battery when it is discharging.

A simple bash script (found on [stack exchange](https://unix.stackexchange.com/questions/10418/how-to-find-power-draw-in-watts))
looks like this:
```
echo - | awk "{printf \"%.1f\", \
$(( \
  $(cat /sys/class/power_supply/BAT0/current_now) * \
  $(cat /sys/class/power_supply/BAT0/voltage_now) \
)) / 1000000000000 }" ; echo " W "

```
Note that we need `awk` here for precision calculation. You could write the same
thing in python or any other high level language.
## Monitoring CPU frequency
CPU frequency is monitored with this command:
```
watch grep MHz /proc/cpuinfo
```
# Kernel Parameters
Power relevant kernel parameters in `/etc/default/grub` (or a bootloader of your
choice):
```
GRUB_CMDLINE_LINUX_DEFAULT="mem_sleep_default=s2idle nmi_watchdog=0"
```
## nmi\_watchdog=0
This kernel option seems to do nothing at all, because `pgrep watchdog` still
returns something, which is annoying. Further more, the watchdog module
iTCO\_wdt won't disable itself after the kernel line option is given.

Changing `#RuntimeWatchdogSec=off` in `/etc/systemd/system.conf` should turn it
off, but it doesn't, and `pgrep watchdog` still returns a value, and iTCO\_wdt
is still loaded.

It's unclear if this actually contributes anything to powersaving, however, the
Arch Wiki recommends it [here](https://wiki.archlinux.org/title/Power_management#Disabling_NMI_watchdog).
# CPU configuration
From the (Arch Wiki)[https://wiki.archlinux.org/title/CPU_frequency_scaling]:
>The Linux kernel offers CPU performance scaling via the CPUFreq subsystem, which defines two layers of abstraction:
>	* Scaling governors implement the algorithms to compute the desired CPU frequency, potentially based off of the system's needs.
>	* Scaling drivers interact with the CPU directly, enacting the desired frequencies that the current governor is requesting

Scaling governors and scaling drivers are the main way to configure the cpu for
power saving.
## Scaling Drivers
There are 2 drivers that can be used:
* `intel_pstate`, which is the main driver and
* `acpi_cpufreq`, which is a fallback driver

By default, most kernels will use the `intel_pstate` driver. This is the
recommended driver and you should switch to the fallback only if you have a good
reason to do so. Add this to your kernel parameters to disable `intel_pstate`
and use the fallback driver:
```
GRUB_CMDLINE_LINUX_DEFAULT="intel_pstate=disable"
```
### Problems with intel_pstate
`cpupower` allows you to set the frequency of the CPU at a fixed clock speed.
However, using the `intel_pstate`, you cannot set the frequeny and instead get
this error:
```
cpupower frequency-set -f 4GHz
------------------------------
Setting cpu: 0
Error setting new values. Common errors:
- Do you have proper administration rights? (super-user?)
- Is the governor you requested available and modprobed?
- Trying to set an invalid policy?
- Trying to set a specific frequency, but userspace governor is not available,
   for example because of hardware which cannot be set to a specific frequency
   or because the userspace governor isn't loaded?
```
Switching to `acpi_cpufreq` does allow `cpupower` set the cpu frequency
to whatever is desired.
### Power usage and CPU Frequency discrepancies between drivers
The `acpi_cpufreq` driver draws more power (around 7-8W at idle) when the
battery is relatively full (90-100%). As the battery level decreases, so does
the power usage that is reported. When the battery is relatively low, the
reported power draw falls to 2-5W at idle.

**NOTE:** I have no way of verifying if this is the actual power draw or if
using the `acpi_cpufreq` driver causes the kernel to misreport poweer draw.

`intel_pstate` on the other hand reports consistently drawing 5-6 Watts at idle,
given all power saving methods are applied.
## Scaling Governors
### intel_pstate governors
[This](https://wiki.archlinux.org/title/CPU_frequency_scaling#Autonomous_frequency_scaling)
section of the Archwiki says intel and amd cpus using their respective pstate
drivers will regulate themselves, and provide two governors:
* powersave, and
* performance.

These are not to be confused with the kernel's default performance and powersave
governors. According to the Arch Wiki:
>They do not work at all like their normal counterpart, however: these levels are
>translated into an Energy Performance Preference hint for the CPU's internal
>governor. As a result, they both provide dynamic scaling, similar to the
>schedutil or ondemand generic governors respectively, differing mostly in
>latency.

You can set the governor manually:
```
cpupower -g powersave
```
If you want to set to performance:
```
cpupower -g performance
```
As mentioned above, these control the CPU's Energy Performance Preference hint.
### acpi_cpufreq governors
By default we are using the `schedutil` scaling governor, which can be
seen in `/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor`.

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
## Scaling frequencies
Scaling CPU frequencies manually is **only** possible if you are using the
`acpi_cpufreq` driver. See ["Problems with intel_pstate"] for more info.

Also in `/sys/devices/system/cpu/cpu0/cpufreq/` are the 2 files:
* `scaling_max_freq` which sets max frequency
* `scaling_min_freq` which sets min frequency

As usual, these can be set manually by echoing values to them, but using
`cpupower` is just more convinient.
## Setting performance and energy bias hint for intel_pstate
Open the `/etc/default/cpupower` file, and set the `perf_bias` option
to 15 for maximum power saving. Refer to
[this](https://wiki.archlinux.org/title/CPU_frequency_scaling#Intel_performance_and_energy_bias_hint)
section in the Arch Wiki for which values mean what.

```
/etc/default/cpupower
---------------------
governor='powersave'

perf_bias=15
```
## Setting energy preference for intel_pstate
The Arch Wiki states that most modern CPUs don't rely on the ACPI P-state, and
just scale according to their internal governors. However, there is
[this](https://wiki.archlinux.org/title/Power_management#Processors_with_Intel_HWP_\(Intel_Hardware_P-state\)\_support\))
section that says that processors that support HWP (hardware P-states), which I
think this one does can choose between one of the options in the file: 
```
/sys/devices/system/cpu/cpufreq/policy?/energy_performance_available_preferences
```
The one that optimises for power seems to be the `power` option. We modify a
file in `/etc/tmpfiles.d` to set this option:
`/etc/tmpfiles.d/energy_performance_preference.conf`
```
w /sys/devices/system/cpu/cpufreq/policy?/energy_performance_preference - - - - balance_power
```
We can confirm that the option has been set:
```
cat /sys/devices/system/cpu/cpufreq/policy?/energy_performance_preference
```
## Turbo Boost
Similar to setting `energy_performance_preference` with `tmpfiles` (see later
on), we can edit a file in `/etc/tmpfiles.d` with the `.conf` ending. We can
then put this line inside it:
```
w /sys/devices/system/cpu/intel_pstate/no_turbo - - - - 1
```
This disables turbo boost. Some people may want this enabled for higher
performance.
# Sleep
There are 2 files and their contents that are of relevenace to us:
* `/sys/power/state`:
```
freeze mem disk
```
* `/sys/power/mem_sleep`:
```
s2idle [deep]
```
A detailed meaning of all of these can be found in the [kernel docs](https://www.kernel.org/doc/html/v4.15/admin-guide/pm/sleep-states.html)
as well as a detailed explanation of what sleep states are and what benefits
each has. I recommend reading the linked kernel doc first before proceeding.

In breif however,
* `freeze` stands for `s2idle` sleep state, which is the suspend-to-idle state
* `mem` stands for whatever is in the `mem_sleep` file
* `disk` stands for hibernation
* `s2idle` in the `mem_sleep` file gives the value of `s2idle` to mem in the
`state` file.
* `deep` in the `mem_sleep` file gives the value of `deep` to `mem` in the
`state` file. Stands for `suspend to RAM` or `S3` sleep

We need to overwrite the default contents of `/sys/power/mem_sleep`. By default
it is set to `[deep]`, which causes problems discussed later. You can do this by
adding this to the kernel command line:
```
GRUB_CMDLINE_LINUX_DEFAULT="mem_sleep_default=s2idle"
```
or, you can manually set it by writing to it:
```
echo s2idle > /sys/power/mem_sleep
```

`systemd-logind` is the default way to handle `acpi` events. `acpid` can also be
used, but I have opted not to use it becuase `systemd-logind` comes preinstalled
in Arch.

Output of `cat /etc/systemd/logind.conf` (systemd-logind's configuration file):
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
Uncomment `HandleLidSwitch` from `/etc/systemd/logind.conf`. This tells systemd what to do when the laptop lid
is closed. It is set to suspend, which the default option.

[This Arch Wiki page](https://wiki.archlinux.org/title/Power_management#Power_management_with_systemd)
describes all the options that `HandleLidSwitch` can have, as well as what
other options mean.

Over here though, `suspend` means put the laptop in the state that is the
contents of `mem` in the `/sys/power/state` file.

Closing the laptop lid reliably puts the system into s2idle, which is what
we hav set our `mem` value to. Systemd also sends the right
messages to the session through `dbus`.

We can verify this with `swaylock` and `swayidle`, and they work as expected,
locking the screen before the laptop is put to sleep.
## Problems with deep sleep or S3 idle
Using `[deep]` sleep only works once, ie, only on the first close the lid.

When I resume from sleep, a number of things go wrong:
* Function hotkeys stop working
* The power button stops working
* The mouse stops working on sway, and needs to be clicked many times to work on
gnome
* Closing the laptop lid again, does nothing, ie, screen doesn't lock, and the
laptop stays powered on

As noted by [@naveenjohnsonv](https://github.com/naveenjohnsonv), this seems to be an issue on Windows as 
well.
[This](https://zentalk.asus.com/t5/zenbook/zenbook-13-ux325-fn-keys-not-working-after-sleep-anyone/m-p/78104)
forum post is about the ASUS ZenBook 13 facing the same problem described on
Windows as well. This might be a firmware bug that spans multiple ASUS laptops
at the moment.

[@naveenjohnsonv](https://github.com/naveenjohnsonv), also noted that switching to S0 sleep on Windows fixed the
issue, but S0 doesn't exist for this variant of K305*.

If anyone patch that enables S0 sleep, I request you to open an issue to tell me
about it.
### Problems with s2idle
Switching it to `[s2idle]` is better, but still suffers from these problems:
* ~~The worst offender is that after waking from sleep, shutdown does not occur as
normal. Systemd starts a stop job that terminates after an infinite time to
close the user session (usually 1000 on my single user system), and it never
closes, forcing me to hard shutdown.~~
**NOTE:** This is solved as of kernel 6.3.9 and the laptop shuts down like
normal
* ~~If on a tty, dmesg warnings are written to the screen~~. Solved as of kernel
6.3.9
* Warnings appear in `dmesg`.
* ~~Fans get super loud for whatever reason, and laptop is hot~~ Not tested.
Installing nbfc\_linux seems to have solved the issue though.
* The battery drains like Romans draining a swamp.
# Module configuration
Some module options save power, on wifi and pcie and stuff
## wifi
My Wifi card on this particular model of laptop on this particular laptop is an
Intel Alder Lake PCH CNVi WiFi card. Running `lscpi -kvv|grep -iA10 net` gives
us all the information about the network controller.

The driver and module being used (in my laptop) are `iwlwifi`.
Thus, following this [Arch wiki](https://wiki.archlinux.org/title/Power_management#Intel_wireless_cards_(iwlwifi))
page about wifi, the following changes have been made:

In `/etc/modprobe.d/iwlwifi.conf`:
```
options iwlwifi power_save=1
options iwlmvm power_scheme=3
```
**NOTE:** Some networks I have connected to suffer a massive performance drop
(~90%) when these options are enabled
## NVME ssd
Follow
[this](https://wiki.archlinux.org/title/Power_management#Bus_power_management)
Arch Wiki article.
`lspci -vv|grep 'ASPM.*abled'` checks if Active State Power Mangement is
enabled. If it is, you can check `/sys/module/pcie_aspm/parameters/policy` which
dictates ASPM policy.

You can check all the ASPM policies by running:
```
cat /sys/module/pcie_aspm/parameters/policy
```
You can then set it permenantly by setting a kernel parameter:
```
GRUB_CMDLINE_LINUX_DEFAULT="pcie_aspm.policy=powersave"
```
