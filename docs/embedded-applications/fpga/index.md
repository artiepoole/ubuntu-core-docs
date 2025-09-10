(index)=

# Ubuntu Core for FPGA devices

- [About FPGAd](about-fpgad)
- [Writing a provider snap](writing-a-provider-snap)
- [Using a provider snap](using-provider-snap)

Due to the strict containerisation of applications, which is fundamental for Ubuntu Core, access to the various system files required to interface with hardware and their drivers are not available to applications on Ubuntu Core.
In a classic (as in, non-Core) version of Ubuntu/Linux, these files can be written to and read from by any application, with access being controlled by the Discretionary Access Control (DAC) permission system.
This means that if a user doesn't have access, `sudo` can be used to elevate the permissions and grant access.

When using Ubuntu core, access to these files is controlled by AppArmor, and no amount of `sudo` will let the application see files it is not permitted to access.
As far as the application can tell, these files don't even exist.
To alleviate these rules, applications can be installed as snaps and these snaps can be installed using either the `--devmode` flag to bypass AppArmor entirely, temporarily, or the snap can be created with `classic` confinement (which cannot be installed on Ubuntu Core).
Both of these approaches, however, are insecure and go against the principles of snaps.
Therefore, in order to have a snap be accepted by the store team to be installable from the app store, and for it to be usable on Ubuntu Core, the appropriate permissions must be added via the snap's `snapcraft.yaml` definition file in the form of `interfaces`.
Highly privileged permissions cannot automatically be enabled (interfaces will not be auto-connected), unless explicitly enabled by the store team.
Therefore, FPGAd, a canonical owned snap, creates a gateway to access the underlying FPGA subsystem without the need for your snap to have these super-privileged interfaces.

There are two ways to interface with FPGAd to control FPGA devices:

1) using the provided DBus interface, see \<here\> for details.
2) using the provided FPGAd command line interface (CLI) detailed \<here\>

## Background

## Getting Started

## Writing a provider snap

### DBus
### CLI




```{toctree}
:hidden:
:titlesonly:
:maxdepth: 2
:glob:

About FPGAd and provider snaps <about-fpgad>
Writing a Provider Snap <writing-a-provider-snap>
Using a provider snap <using-provider-snap>
FPGAd interface reference <fpgad-reference/index>
Command line interface reference <cli>
DBus interface reference <dbus>