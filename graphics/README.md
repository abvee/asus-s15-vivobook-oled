# Intel Graphics
A good place to start is always the Arch Wiki. See
[this](https://wiki.archlinux.org/title/Intel_graphics) page about intel
graphics for full details.
## GUC
You can safely enable GUC / HUC since this is an Alder lake 12th gen platform.
Refer to the Arch Wiki linked above on how to enable them. TL;DR, put this in
your kernel command line:
```
i915.enable_guc=3
```

## SR-IOV
In progress...

## Console Fonts not working
When messing around with intel graphics, it is possible that the `i915` module
gets loaded after the console font is set from `/etc/vconsole.conf` for systemd.
To avoid this, load the graphics driver early in the initramfs. This will differ
for different initramfs generators, but for `mkinitcpio`, add it to the modules
array in `/etc/mkinitcpio.conf`:
```
MODULES=(i915)
```
Then rebuilt initramfs:
```
mkinitcpio -P
```
