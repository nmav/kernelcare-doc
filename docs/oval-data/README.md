# OVAL data

KernelCare provides automated live patching for Linux kernels with centralized management, common automation and various vulnerability management tools integration.

You can find more information here:

* [https://tuxcare.com/live-patching-services/kernelcare-enterprise/](https://tuxcare.com/live-patching-services/kernelcare-enterprise/)
* [https://docs.tuxcare.com/live-patching-services/#kernelcare-enterprise](https://docs.tuxcare.com/live-patching-services/#kernelcare-enterprise)

KernelCare OVAL feeds for supported distributions can be found here: [https://patches.kernelcare.com/oval/](https://patches.kernelcare.com/oval/).

A system managed by KernelCare can be detected by the `kernelcare` package presence:

**For RPM-based distributions**

```
$ rpm -q kernelcare
```

**For Debian-based distributions**

```
$ dpkg -s kernelcare
```

The `kernelcare` packages are signed with the `6dc3d600cdef74bb` PGP key, the public key can be found here: [https://repo.cloudlinux.com/kernelcare/kernelcare.gpg](https://repo.cloudlinux.com/kernelcare/kernelcare.gpg).




