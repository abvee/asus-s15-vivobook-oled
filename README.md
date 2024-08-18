# ASUS S-15 Vivobook (and related laptops) linux configuration

A list of all configuration I have done on this particular laptop while I
have had it.

Details of my particular model:
* Year: 2022
* Model Number: K3502ZA
* CPU: Intel i5-12500H
* Current Kernel version tested: 6.6.36-lts

Note that all the contents have been tested only for the kernel version given
above. Any problems with older kernel *might* be present in previous versions of
the article, but any kernels after are untested.

## Common Problems:
### Keyboard
Any kernel before 6.1 needs an irq override patch, and will not work without it.

The patch in question can be found
[here](https://bugzilla.kernel.org/attachment.cgi?id=301559&action=diff).

Rebuild the kernel after applying this.

### Symless media keys
The media keys for `F8` and `F7` don't show any syms when checked with `wev` as
though they don't exist.

## Common Configuration
Function keys and not media keys are set by default. So pressing `F5` doesn't
increase the volume. This is highly annoying, and to change it,
put this in your `/etc/modprobe.d/*name of your choice*.conf`:
```
options asus_wmi fnlock_default=N
```
You can find the current value in `/sys/module/asus_wmi/parameters/fnlock_default`

## Bluetooth
My headphones don't like when pipewire does automatic switching of HSP/HFP and
A2DP profiles, and it crackles the audio.

Adding this to `~/.config/wireplumber/policy.lua.d/11-bluetooth-policy.lua`
fixes the issue:
```
bluetooth_policy.policy["media-role.use-headset-profile"] = false
```
This is on wireplumber. If you are using `pipewire-media-session`, refer to [this](https://wiki.archlinux.org/title/PipeWire#Bluetooth_devices)
Arch Wiki article on pipewire.

***NOTE:*** as of pipewire version 0.3.71, this no longer occurs, and the file
can be removed or commented out with `--`.
