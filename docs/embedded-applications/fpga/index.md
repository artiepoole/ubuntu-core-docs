(index)=

# Ubuntu Core for FPGA devices

[//]: # (TODO: background and justification for this page existing)

Due to the strict containerization of applications, which is fundamental for Ubuntu Core, access to the various system files required to interface with hardware and their drivers are not available to applications on Ubuntu Core. In a classic (as in, non-Core) version of Ubuntu/Linux, these files can be written to and read from by any application, with access being controlled by the Discretionary Access Control (DAC) permission system. This means that if a user doesn't have access,
`sudo` can be used to elevate the permissions and grant access.

When using Ubuntu core, access to these files is controlled by AppArmor, and no amount of
`sudo` will let the application see files it is not permitted to access. As far as the application can tell, these files don't even exist. To alleviate these rules, applications can be installed as snaps and these snaps can be installed using either the
`--devmode` flag to bypass AppArmor entirely, temporarily, or the snap can be created with
`classic` confinement (which cannot be installed on Ubuntu Core). Both of these approaches, however, are insecure and go against the principles of snaps. Therefore, in order to have a snap be accepted by the store team to be installable from the app store, and for it to be usable on Ubuntu Core, the appropriate permissions must be added via the snap's
`snapcraft.yaml` definition file in the form of
`interfaces`. Highly privileged permissions cannot automatically be enabled (interfaces will not be auto-connected), unless explicitly enabled by the store team. Therefore, FPGAd, a canonical owned snap, creates a gateway to access the underlying FPGA subsystem without the need for your snap to have these super-privileged interfaces.

See [About FPGAd and provider snaps](about-fpgad) for more information on making use of FPGAd on Ubuntu Core.

## Overview

The manual loading of bitstreams can still be conducted using bash commands (i.e. manually writing copying firmware into
`/lib/firmware/` and loading it by writing to the sysfs files) but vendor provided helper applications (such as dfx-mgr on AMD-Xilinx products) cannot be installed or run on Ubuntu Core. Therefore, FPGAd contains these vendor provided applications, named [softeners](about-fpgad.md#platforms-and-softeners) in the context of FPGAd, and provide access to these via the [command line interface (CLI)](about-fpgad.md#command-line-interface). This [CLI](about-fpgad.md#command-line-interface) also provides a way to control bitstream loading and overlays from

## Getting Started with FPGAd

First you must install an Ubuntu Core image on an FPGA enabled device. If you are using a custom image, add

```yaml
<some image definition yaml snippet here>
```

to the image definition to add FPGAd to the image, or run

```shell
sudo snap install fpgad
```

to install FPGAd in an already installed image.

The necessary interfaces are enabled automatically so FPGAd is ready to use e.g.

```shell
fpgad status
```

As soon as a snap application attempts to connect to the
`fpgad:daemon-dbus` interface (a DBus interface hosted at
`com.canonical.fpgad`), the FPGA daemon will start and handle DBus calls. If you intend to use a [provider snap](about-fpgad.md#provider-snaps) then you must manually connect your snap's DBus plug to the
`fpgad:daemon-dbus` e.g.:

```shell
sudo snap connect <your-snap>:fpgad-dbus fpgad:daemon-dbus
```

where
`fpgad-dbus` is a recommended name for the DBus plug which is defined in your provider snap's
`snapcraft.yaml`.

For more information on the following topics, follow the provided links:

- [FPGAd's command line interface](about-fpgad.md#command-line-interface)
- [About provider snaps](about-fpgad.md#provider-snaps)
- [Writing provider snaps](about-fpgad.md#writing-a-provider-snap)
- [Using provider snaps](about-fpgad.md#using-a-provider-snap)
- [Publishing your provider snap](about-fpgad.md#publishing-your-provider-snap)

```{toctree}
:hidden:
:titlesonly:
:maxdepth: 2
:glob:

About FPGAd and provider snaps <about-fpgad>
