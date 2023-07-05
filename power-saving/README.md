# Power saving
This is to document all the power saving steps I took on the ASUS S15 OLED,
version K3502ZA.306 (that last 306 might be BIOS version, so don't think too
much of it).

The goal is to lower power consumption as much as possible for a long battery
life. Most of these modifications can be done with only installing `cpupower` or
installing nothing at all.

## Monitoring Power
On this particular laptop, there are 2 folders in the `/sys/class/power_supply`
directory. They are `AC0` and `BAT0` standing for the AC input and battery
respectively.

We want the power drawn from the battery, and `AC0` is does not contain any
particularly useful information.

The `/sys/class/power_supply/BAT0` directory on this laptop with Arch's 6.1.2
kernel does **NOT** have a commonly found file called `power_now`, which would
tell us how much power was drawn.
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

## Kernel Parameters

Power relevant kernel parameters I put in `/etc/default/grub`:

`GRUB_CMDLINE_LINUX_DEFAULT="mem_sleep_default=deep nmi_watchdog=0"` 
### intel\_pstate
This has to do with drivers controlling cpu frequency. By default, most kernels
will use the `intel_pstate` driver. Usually, a fallback driver is
`acpi_cpufreq`.

The initial reason for doing this is that `cpupower` did not seem to work when I
used the `intel_pstate` governor. I monitered frequencies with
`watch grep -i mhz /proc/cpuinfo`, and it showed that most cores were running at
a constant 3.1GHz. However, switching to `acpi_cpufreq` allowed `cpupower` to
work flawlessly, and I could set the cpu frequency to whatever I desired. This
was the initial reason for disabling the `intel_pstate` driver.

However, upon further
testing in a newer kernel (6.3.9), there are some interesting observations to be
made.

#### Power usage and CPU Frequency discrepancies
For one, the power usage as monitored above, returns lower power draw than if I
were to use just the `acpi_cpufreq` driver. It consistently draws 5-6 Watts
rather than the 7-8 Watts when the battery is high, and draws <++> Watts
compared to the 2-5 Watts when the battery is low.

Furthermore, the actual realtime cpufrequency is a bit questionable. While
`watch grep -i mhz /proc/cpuinfo` returns earch cpu core running at mostly
3.1GHz the entire time, running `cpupower frequency-info|grep -i kernel` reveals
that the cpu frequency is actually sitting around the 1GHz mark.

There have been [other issues](https://github.com/htop-dev/htop/issues/953)
saying that cpu frequency is not being reported correctly by `/proc/cpuinfo`.
All of them advice to use `cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq`
instead, but on my laptop, that gives the same values as `/proc/cpuinfo`.

#### Energy savings with intel\_pstate
[This](https://wiki.archlinux.org/title/CPU_frequency_scaling#Autonomous_frequency_scaling)
section of the Archwiki (at the bottom) also says intel and amd cpus using their
respective pstate drivers will regulate themselves, and provide two governors:
powersave and performance.

It goes on to say that these are not like other governors, and that these levels
are translated to energy performance preference hints for the cpu's internal
governor. As a result, they both provide dynamic scaling, similar to schedutil
and ondemand.

It also claims that the performance governor should give better power saving
than the old ondemand governor.

#### Conclusion
I have decided to test out if `intel\_pstate` really saves power, as well as
which file/program to rely on for accurate cpu  frequency reporting.

### mem\_sleep\_default=deep and sleep in general
Output of `cat /sys/power/state`:
```
freeze mem disk
```

Output of `cat /sys/power/mem_sleep`:
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

On the laptop, I want this type of sleep to only occur when the lid is closed,
or perhaps through `swayidle`, though I have not done anything about that yet.

`systemd-logind` is the default way to handle `acpi` events. `acpid` can also be
used, but I don't want to use it becuase it's just another program, and it makes
things more complicated.

However, it might be possible that the problems that occur with sleep
(described later on) might not occur if `acpid` is used. I personally have not
tested it.

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

As noted by [@naveenjohnsonv](https://github.com/naveenjohnsonv), this seems to be an issue on Windows as 
well.
[This](https://zentalk.asus.com/t5/zenbook/zenbook-13-ux325-fn-keys-not-working-after-sleep-anyone/m-p/78104)
forum post is about the ASUS ZenBook 13 facing the same problem described on
Windows as well. This might be a firmware bug that spans multiple ASUS laptops
at the moment.

[@naveenjohnsonv](https://github.com/naveenjohnsonv), also noted that switching to S0 sleep on Windows fixed the
issue, but S0 doesn't exist for my variant of K305* , atleast on the Arch Linux
kernel 6.1.8, or the latest fedora 36 kernel. If anyone has a kernel or
patch that enables S0 sleep, I request you to open an issue to tell me about it.

Switching it to `[s2idle]` is not a lot better, and suffers from these problems:
* The worst offender is that after waking from sleep, shutdown does not occur as
normal. Systemd starts a stop job that terminates after an infinite time to
close the user session (usually 1000 on my single user system), and it never
closes, forcing me to hard shutdown.
* If on a tty, dmesg warnings are written to the screen
* The same warnings appear in `dmesg`
* Fans get super loud for whatever reason, and laptop is hot
* battery of course, drains like Romans draining a swamp

Sleep seems to be a glorious mess. S0 sleep / Modern standby or even regular
standby (S1) don't seem to exist. S3 is the only sleep state to exist apart from
hibernation through the `[deep]` option, and `[s2idle]` might as well not exist
becuase it is practically useless.

### nmi\_watchdog=0
This kernel option seems to do nothing at all, because `pgrep watchdog` still
returns something, which is annoying. Further more, the watchdog module
iTCO\_wdt won't disable itself after the kernel line option is given.

Changing `#RuntimeWatchdogSec=off` in `/etc/systemd/system.conf` should turn it
off, but it doesn't, and `pgrep watchdog` still returns a value, and iTCO\_wdt
is still loaded.

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
exist for it.

## cpupower
This is a very simple section about how you can use cpupower to set the governor
and limit maximum frequency.

### Scaling with acpi\_cpufreq

#### Scaling governors
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

#### Scaling frequencies
Also in `/sys/devices/system/cpu/cpu0/cpufreq/` are the 2 files:
* `scaling_max_freq` which sets max frequency
* `scaling_min_freq` which sets min frequency

As usual, these can be set manually by echoing values to them, but using
`cpupower` is just more convinient.

### Scaling with intel\_pstate
Refer to the `intel_pstate` section on why I am using the `intel_pstate`
governor.

#### Setting scaling governor
Similar to how you set the governor for `acpi_cpufreq`, just that there are only
2 options. One is `powersave` and the other is `performance`. As mentioned
above, this controls their energy performance preference hint. I'm currently
using `powersave`.

#### Setting performance and energy bias hint
Just set it in the `/etc/default/cpupower` file, and set the `perf_bias` option
to 15. Refer to
[this](https://wiki.archlinux.org/title/CPU_frequency_scaling#Intel_performance_and_energy_bias_hint)
section in the Arch Wiki for what values mean what.

`/etc/default/cpupower - for intel_pstate`
```
governor='powersave'

perf_bias=15
```
**NOTE:** I have chosen to not limit minimum and maximum cpu frequencies because
I don't know if it's even having an effect thanks to the missreporting from
`/proc/cpuinfo`.

#### Setting energy preference
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

## Problems
Idle power usage changes with battery level. For whatever reason, idle power
draw when the battery level is higher is also higher.

Idles at around `5.8W` when the battery is around 80%, but drops to `3.4W` when
the battery is around 25%. The goal is to try and get the laptop to idle at
around `3W` even when the battery is at 80%.

If you find a solution to this issue, please let me know by creating a pull
request, or even just creating an issue.

## Results
Previously, without any configuration, Fedora with GNOME would idle at around
7-8 W. Switching to Arch and using something lighter (sway) reduced that number
to around 5-6 W on idle, when battery is high (refer previous problem section).

As it is now however, it is perfectly usable, and you get decent battery life.

### intel\_pstate results
The system idles at a very low 5.3 Watts when on 80% charge, though cpu
frequency is reported to be very high.
