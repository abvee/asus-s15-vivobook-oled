# ASUS S-15 Vivobook (and related laptops), linux configuration

Just a list of all configuration I have done on this particular laptop while I
have had it.

## Common Problems:
### Keyboard
If you are installing Linux on this laptop, or any related laptop, just know
that the keyboard will **NOT** work because it needs an irq override patch.

Any kernel before 6.1 needs to have this patch applied manually, and at this
point in time, that is most kernels for most distros except for Arch.

The patch in question can be found
[here](https://bugzilla.kernel.org/attachment.cgi?id=301559&action=diff).

Rebuild the kernel after applying this.

### Symless media keys
The media keys for `F8` and `F7` don't show any syms when checked with `wev` as
though they don't exist, on my machine.

## Common Configuration
Function keys and not media keys are set by default. So pressing `F5` doesn't
increase the volume, but does `F5`. This is highly annoying, and to change it,
put this in your `/etc/modprobe.d/*name of file*.conf`:
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
This is on wireplumber. If you are using `pipewire-media-session`, refer to the
Arch Wiki article on pipewire.
