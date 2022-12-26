# ASUS S-15 Vivobook (and related laptops), linux configuration

Just a list of all configuration I have done on this particular laptop while I
have had it.

## Common Problems:
If you are installing Linux on this laptop, or any related laptop, just know
that the keyboard will not work because it needs an irq override patch.

Any kernel before 6.1 needs to have this patch applied manually, and at this
point in time, that is most kernels for most distros except for Arch.

The patch in question can be found
[here](https://bugzilla.kernel.org/attachment.cgi?id=301559&action=diff).

Rebuild the kernel after applying this.
