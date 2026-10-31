# opencas-nix

[Open CAS](https://github.com/lhns/open-cas-linux) block caching for NixOS: the kernel module, `casadm`/`casctl`, udev
rules and units, and a NixOS module that generates `/etc/opencas/opencas.conf` from options. It builds the
[lhns fork](https://github.com/lhns/open-cas-linux), which adds a per-IO-class cache mode, pinned as the
`open-cas-src` input. Decisions are in [`docs/adr/`](docs/adr).

Originally part of [lhns/ceph-nix](https://github.com/lhns/ceph-nix), which now consumes it as a flake input.

## Outputs (x86_64-linux)

- `overlays.default`: `pkgs.open-cas` (userspace) and `open-cas` in every `linuxPackages` set (kernel module).
- `packages.{open-cas, open-cas-kmod, default}`: userspace and the module for the default kernel.
- `nixosModules.{opencas, default}`: `services.opencas`; the module adds the overlay itself.
- `checks`: `opencas` (VM test), `open-cas-kmod-lts` (6.12), `open-cas-kmod-latest` (6.18), `formatting`.

## Usage

```nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";
  inputs.opencas-nix = {
    url = "github:lhns/opencas-nix";
    inputs.nixpkgs.follows = "nixpkgs";
  };

  outputs = { nixpkgs, opencas-nix, ... }: {
    nixosConfigurations.host = nixpkgs.lib.nixosSystem {
      modules = [
        opencas-nix.nixosModules.opencas
        {
          services.opencas = {
            enable = true;
            initializeBlankCaches = true;
            caches."1" = {
              device = "/dev/disk/by-id/nvme-EXAMPLE";
              mode = "wb";
              cores = [ "/dev/disk/by-id/wwn-0x123456789abcdef0" ]; # /dev/cas1-1
            };
          };
        }
      ];
    };
  };
}
```

## The module

`services.opencas.caches` maps a cache id to `device`, `mode`, `cacheLineSize`, `extraFlags`, `ioClasses` and `cores`.
LVM is taught about `/dev/cas*` devices and the raw devices are hidden from it, and a shutdown with active LVs on a
cas device does not leave the cache dirty. `initializeBlankCaches` creates a cache once on a device without any
signature (a marker in `/var/lib/opencas` prevents ever formatting it again); without it, create the cache once with
`casctl init`. Devices must be `/dev/disk/by-id` or `by-path` links.

The kernel module is built for `boot.kernelPackages` and marked broken for kernels outside what this Open CAS release
supports (5.10 to 7.0). The VM test is `nix build .#checks.x86_64-linux.opencas` (also part of `nix flake check`; it
needs KVM).

## License

The Nix expressions, scripts and docs here are Apache-2.0. Open CAS itself is BSD-3-Clause; its source is the pinned
`open-cas-src` input in `flake.lock`.
