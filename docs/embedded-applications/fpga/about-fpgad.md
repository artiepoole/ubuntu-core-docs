(about-fpgad)=

# About FPGAd and provider snaps

A provider snap is a snap which contains bitstreams, device tree overlays, metadata (e.g. shell.json for dfx-mgr bitstreams) and user space applications to interface with the running bitstream. It should also contain at least one snap application which can be configured to load at startup or run on command. This could be simple like a bash/python script to make a single call to load a bitstream/apply an overlay using busctl, or a full binary application which prepares the device, ensures everything is working and runs a GUI application with continuous monitoring.

The next subsection explains how FPGAd
`<keeps>` the original functionality of vendor provided solutions for controlling fpga devices (such as dfx-mgr for Xilinx devices). The following sections describe the process of using FPGAd directly (using the command line interface ([CLI](about-fpgad.md#command-line-interface)) and offer guidance for writing and using provider snaps.

## Platforms and Softeners

In order to maintain vendor provided functionality and user space helper applications, softeners were included as part of FPGAd. Softeners are simple a gatekeeper for calling those vendor specific applications, (such as dfx-mgr for Xilinx machines), and are used automatically if the device's platform compatibility string matches one of the softeners' compatibility strings. It can be overridden
`NO IT CAN'T - WIP` by
`STEPS TBC`.

[//]: # (TODO: update once an override is available.)

# Command line interface

FPGAd provides a command line interface (CLI) to make manual control of the underlying FPGA subsystem possible without the need for a provider snap. This is useful for rapid prototyping and verification reasons, as well as being enough for situations requiring less complexity. The following subsections describe how to use the CLI to check the status, load a bitstream/apply and overlay and set properties (e.g. flags)

## Usage
```
Usage: [snap run] fpgad [OPTIONS] <COMMAND>

OPTIONs:
  -h, --help            Print help
      --handle <DEVICE_HANDLE>  fpga device `HANDLE` to be used for the operations.
                       Default value for this option is calculated in runtime
                       and the application picks the first available fpga device
                       in the system (under `/sys/class/fpga_manager/`)

COMMANDs:
├── load                Load a bitstream or overlay
│   ├── overlay <FILE> [--handle <OVERLAY_HANDLE>]
│   │       Load overlay (.dtbo) into the system using the default OVERLAY_HANDLE
│   │           (either the provided DEVICE_HANDLE or "overlay0") or provide
│   │       --handle: to name the overlay directory
│   └── bitstream <FILE>
│           Load bitstream (e.g. `.bit.bin` file) into the FPGA
│
├── set <ATTRIBUTE> <VALUE>
│       Set an attribute/flag under `/sys/class/fpga_manager/<DEVICE_HANDLE>/<ATTRIBUTE>`
│
├── status [--handle <DEVICE_HANDLE>]
│       Show FPGA status (all devices and overlays) or provide
│       --handle: for a specific device status
│
└── remove              Remove an overlay or bitstream
    ├── overlay [--handle <HANDLE>]
    │       Removes the first overlay found (call repeatedly to remove all) or provide
    │       --handle: to remove overlay previously loaded with given handle
    └── bitstream
            Remove active bitstream from FPGA (bitstream removal is vendor specific)
```
## Explicit versions

### Apply an overlay

```shell
fpgad [--handle=<device_handle>] load ( (overlay <file> [--handle=<handle>]) | (bitstream <file>) )
```

### Remove an overlay

```shell
fpgad [--handle=<device_handle>] remove ( ( overlay <HANDLE> ) | ( bitstream ) )
```

### Set

```shell
fpgad [--handle=<device_handle>] set ATTRIBUTE VALUE
```

### Status

```shell
fpgad [--handle=<device_handle>] status
```

## Examples

### Load

```shell
fpgad load bitstream /lib/firmware/k26-starter-kits.bit.bin
fpgad --handle=fpga0 load bitstream /lib/firmware/k26-starter-kits.bit.bin

fpgad load overlay /lib/firmware/k26-starter-kits.dtbo
fpgad load overlay /lib/firmware/k26-starter-kits.dtbo --handle=overlay_handle
fpgad --handle=fpga0 load overlay /lib/firmware/k26-starter-kits.dtbo --handle=overlay_handle
```

### Remove

```shell
fpgad --handle=fpga0 remove overlay
fpgad --handle=fpga0 remove overlay --handle=overlay_handle
```

### Set

```shell
fpgad set flags 0
fpgad --handle=fpga0 set flags 0

        fpgad set key ABADC0DE
fpgad --handle=fpga0 set key ABADC0DE
```

### Status

```shell
fpgad status
fpgad --handle=fpga0 status
```

# Provider snaps

[//]: # (TODO: what is a provider snap)

## Writing a provider snap

[//]: # (TODO: explain why you'd want to, summarise the following sections, point to DBus docs)

### Before starting

Before endeavouring to write a provider snap, please come up to speed with the [Craft a snap](https://documentation.ubuntu.com/snapcraft/stable/tutorials/craft-a-snap/) tutorial. It will describe the overarching process of packing a snap.

If you intend to make the snap package publicly available (or available unlisted but still hosted on the snap store), please also see the guides in [Publishing](https://documentation.ubuntu.com/snapcraft/stable/how-to/publishing/) especially [How to register a snap](https://documentation.ubuntu.com/snapcraft/stable/how-to/publishing/register-a-snap/#how-to-register-a-snap) and [Publish a snap](https://documentation.ubuntu.com/snapcraft/stable/how-to/publishing/publish-a-snap/).

The snap can be built from source automatically, alleviating some of the maintenance burden. See [Manage revisions and releases](https://documentation.ubuntu.com/snapcraft/stable/how-to/publishing/manage-revisions-and-releases/) for more information on that subject.

It is also definitely work looking at the [k26-default-bitstreams](https://github.com/canonical/k26-default-bitstreams/) snap as an an example of a provider snap. It is written in rust and uses FPGAd's [DBus](#dbus) interface to initiate the load on startup. The provided [README](https://github.com/canonical/k26-default-bitstreams/blob/main/README.md) on the [k26-default-bitstreams](https://github.com/canonical/k26-default-bitstreams/) repository provides an explanation of the contained
`snap/snapcraft.yaml`. If you're planning to use Rust for your application, the associated source code can be used as a basis for using the [zbus](https://docs.rs/zbus/latest/zbus/) crate to interface with FPGAd via DBus (examples in other languages may come in the future).

### Process overview

In order to write a provider snap, you need to undertake the following steps:

- create a
  `snap/snapcraft.yaml` file relative to your content root
- provide the contents to the snap as a
  `part:` in the
  `snapcraft.yaml`, probably by using the [dump plugin](https://documentation.ubuntu.com/snapcraft/stable/common/craft-parts/reference/plugins/dump_plugin/) in one of the two ways:
    1) including the content files inside a subdirectory such as
       `data/<my-snap>/<content>` using
       `parts: plugin: dump:` with
       `source-type: local` with
       `organize: ...` specified ([old example](https://github.com/canonical/k26-default-bitstreams/blob/ffe6513770c9ae5bf25e59dcf747ea3108b82161/snap/snapcraft.yaml#L36)).
    2) including the content files from a remote git repository using
       `parts: plugin: dump:` with
       `source-type: git` and
       `source: <url to git repository>` with
       `override-build: ...` specified ([newer example](https://github.com/canonical/k26-default-bitstreams/blob/43ff1dbe4fc56c4b6e4e943bc60ff27d0025988f/snap/snapcraft.yaml#L39)).
- write at least one application to communicate with FPGAd via DBus
- add at least one
  `app:` to the
  `snapcraft.yaml`file which builds your application or runs a script (e.g. python) to communicate with FPGAd via DBus to load (some of) the provided files. See []() for more on building from source in a
  `snapcraft.yaml`.
- add a
  `plug:` for the DBus communication to the
  `snapcraft.yaml` (see [here](#snapcraftyaml-explained)) for an example)
- add the above
  `plugs: <your-plug-name>` to each application requiring access to FPGAd's DBus interfaces. See [snapcraft.yaml explained](#snapcraftyaml-explained) for more about the DBus plug.

Each of these steps is outlined by following the below subsections.

### creating the snapcraft.yaml

The first steps is to create a
`snapcraft.yaml`. For this you can run

```
cd path/to/project/root
mkdir snap
touch snap/snapcraft.yaml
```

or create the file in any way you normally would.

```{note}
The snapcraft.yaml must be inside the snap directory at the project root
```

Below is a template which can be used to get started, please note that anything inside of
`<>` is to be replaced:

```yaml
name: <name of snap> # note: must match any registration if you registered a snap
base: core24 # core2* - should match Core image base version
summary: <your summary here>
description: |
  <Your longer description here>
grade: devel # or stable etc
confinement: strict # must be strict for Ubuntu Core - can be devmode if for classical image
plugs:
  fpgad-dbus: # It is recommended to use `fpgad-dbus` as the name here, but it can be customized
    interface: dbus
    bus: system
    name: com.canonical.fpgad
  #<any other plugs you may need>:
apps:
  <your startup app name>: # this application runs on startup due to `daemon: oneshot`
    command: bin/<startup binary name>
    daemon: oneshot # to run once on startup
    plugs:
      - fpgad-dbus
      <...>
    restart-condition: <always> # optional
    start-timeout: <30s> # not optional if restart-condition specified
    install-mode: disable # see the "run on startup" section for explanation and required hooks
  <your manual app name>: # this application can be run manually. If it matches the snap name it can be called using the snap name wihtout <snap-name>.<app-name> syntax
    command: bin/<manual-run binary name>
  plugs:
    - fpgad-dbus
    <...>
parts:
  <startup binary name>:
    plugin: <plugin> # see LINK for information on building inside a snap
    source: <relative/path/to/source>
    <...>
  <manual-run binary name>:
    plugin: <plugin> # see LINK for information on building inside a snap
    source: <relative/path/to/source>
    <...>
<remote-bitstream-data>:
  plugin: dump
  source: <git repository url>
  source-type: git
  override-build: |
    mkdir -p $SNAPCRAFT_PART_INSTALL/data/<name of snap>
    cp <repository/path/to/file(s)> $SNAPCRAFT_PART_INSTALL/data/<name of snap>/
<local-bitstream-data>:
  plugin: dump
  source: <relative/path/to/source>
  source-type: local
  organize:
    <path/to/source/dir>: data/<name of snap>
```

See [the snapcraft docs](https://documentation.ubuntu.com/snapcraft/stable/) for more details on using and crafting snaps. See [this specific page](https://documentation.ubuntu.com/snapcraft/stable/reference/project-file/snapcraft-yaml/) for more on the keys and values available for `snapcraft.yaml` files.

#### Run on startup

As seen in the two applications in the [snapcraft.yaml template](#creating-the-snapcraftyaml) above, applications can either be normal (absence of `daemon:` keyword) or defined as a `daemon: <type>` (run on startup, for example). If specified, there are multiple types of daemons, and they are similar to those available in systemd service files. See [services and daemons](https://snapcraft.io/docs/services-and-daemons) for more information on snap daemon types.

If your intention is to load a bitstream on startup, there are a few details to be made aware of in order to be able to get the snap containing the daemon installed, connected and enabled. For applications communicating with `fpgad:daemon-dbus`, that connection must be made before running your daemon. If this wasn't the case, then everything could be left as default and during install the snap would be installed, any hooks would be run, and then the daemons with "install-mode: enable" (the default value) would be started. If this application fails to start, then the snap install fails and the install does not stick.

To allow the snap to install without running the daemon, but to minimise the number of manual steps required, the following process can be used:
- use `install-mode: disable` in the daemon tags, as shown in [the template](#creating-the-snapcraftyaml)
- make two hooks in the `snap/hooks/` directory called `connect-plug-fpgad-dbus` and `disconnect-plug-fpgad-dbus` (disconnect is optional) (`fpgad-dbus` is the recommended name of the plug but the format is `disconnect-plug-<name-of-interface>`)
- write hooks to enable the daemon on connect, and disable it on disconnect (as follows, or see [k26-default-bitstreams/snap/hooks](https://github.com/canonical/k26-default-bitstreams/tree/main/snap/hooks) for an example)
- make these files executable using `chmod +x /path/to/file`

```shell
# File: snap/hooks/connect-plug-fpgad-dbus

#!/bin/sh

set -e

echo "enabling <name of snap service> on startup"
snapctl start --enable <name of snap service>
echo "<name of snap service> enabled on startup"
```


```shell
# File: snap/hooks/disconnect-plug-fpgad-dbus

#!/bin/sh

set -e

echo "enabling <name of snap service> on startup"
snapctl stop --disable <name of snap service>
echo "<name of snap service> enabled on startup"
```


```{note}
These commands run from inside your snap's shell so snapctl always controls the snap to which the hook belongs
```
For more information on hooks, see the snapcraft docs on [Connect Hooks](https://snapcraft.io/docs/interface-hooks#p-36664-connect-hooks).

If manual control of the service is desired, the following commands can be used.
 To start and enable it:
```
[sudo] snap start <name of snap>.<name of application> --enable
```
To stop and disable it:
```
[sudo] snap stop <name of snap>.<name of application> --disable
```

## writing the dbus application

[//]: # ( TODO: edit the following to reference "my-snap" or something)

### Using softeners

### Using the content Interface

## Using a provider snap

If running on target device:

```shell
snapcraft
sudo snap install <name of snap> # or use /path/to/snap if built locally
sudo snap connect <name of snap>:fpgad-dbus fpgad:dbus-daemon
```

```{note}
the  name "fpgad:dbus-daemon" is defined in the fpgad snapcraft.yaml, so may be subject to change. Check [fpgad's snapcraft.yaml](https://github.com/canonical/fpgad/blob/main/snap/snapcraft.yaml) for changes if this command fails.
```

### publishing your provider snap

# DBus

[//]: # (TODO: use the link below as reference of API doc)
https://networkmanager.dev/docs/api/latest/spec.html

```
busctl introspect com.canonical.fpgad /com/canonical/fpgad/status
busctl introspect com.canonical.fpgad /com/canonical/fpgad/control
```

In order to access the FPGA subsystem from a snap, an application within a snap must communicate with FPGAd backend using two DBus interfaces:

1. [<code>com.canonical.fpgad.status</code>](#status-interface) - for read only access to properties and attributes (i.e. getters)
2. [<code>com.canonical.fpgad.control</code>](#control-interface) - for read/write access to properties and attributes (i.e. setters)

These interfaces can be accessed using any DBus framework (e.g. zbus for Rust) or DBus tool (e.g. busctl) so long as these applications/scripts are run from inside the context of a snap application which has made the necessary plug-slot connection to the
`fpgad:daemon-dbus` interface. Without this connection, FPGAd's daemon will not be detectable since it is hosted through the snap interface subsystem.

The following subsections describe inputs which are common to many methods are described before followed by a full description of the DBus API, starting with a summary followed by the full description of each method.

### Common arguments

The following input arguments are used in multiple interface methods and therefore are described in detail here for brevity.

#### platform_string

The platform_string input is the compatibility string for the device, or
`universal`, and is necessary in order for FPGAd to know what platform to use in order to control the device in the "best" way. See [Platforms and Softeners](#platforms-and-softeners) for more information.

There are 3 valid types of input:

1. <code>''</code> - providing an empty platform_string will prompt FPGAd to read the platform string and to use the appropriate softener if available, falling back to universal if no valid softener is known.
2. <code>'universal'</code> - tell FPGAd to not use any softeners, instead using the basic system files approach to control the device.
3. manually provide compatibility string, such as by using the output of [GetPlatformType](#getplatformtype) e.g.
   `'xlnx,zynqmp-pcap-fpga'` - using an empty string will usually result in the same behaviour.

Use [GetPlatformType](#getplatformtype) to get the compatibility string of your device

#### device_handle

The
`device_handle` is the name of the device as it appears in
`/sys/class/fpga_manager/` e.g. the device at
`/sys/class/fpga_manager/fpga0` has a device handle of
`fpga0`. If the device_handle argument is part of a method's signature, is it always required to be a valid string.

The [GetPlatformTypes](#getplatformtypes) method can be used to fetch all valid device handles.

#### overlay_handle

This is the name used to create or access an overlay. These are the names of the directories created in
`/sys/kernel/config/device-tree/overlays/` (be default). They can be fetched using the [GetOverlays](#getoverlays) method. If the
`overlay_handle` argument is part of a method's signature, is it always required to be a valid string.

#### firmware_lookup_path

[//]: # (TODO: populate this properly)
- empty string or full path to the directory containing the dtbo file and associated bitstreams and helper files (typically points to `$SNAP_DATA/...` or a snap's content interface directory)

### Error strings

If the method call makes it to the interface and fails inside FPGAd, the returned string will be the string representation of the encountered error. We have made every effort to make it possible to switch on these errors and so all of our errors are presented in the following format
`FpgadError::<error_type>: ....`. For example, the following are two errors. Firstly
`"FpgadError::Argument: Device fpga1 not found."` is a catch of a bad input, and secondly
`"FpgadError::IODelete: An IO error occurred when deleting "/sys/kernel/config/device-tree/overlays/fpga0": No such file or directory (os error 2)"` is reporting an error caught by the system (because the "fpga0" [overlay_handle](#overlay-handle) does not exist).

#### FpgadError Summary

Below is a table summarizing the various FpgadError types (printed as
`"FpgadError::<Variant>: ....`").

| Variant       | Description/Reason                                                                                                                             |
|---------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Flag          | The flags could not be read, or setting the flags failed to stick                                                                              |
| OverlayStatus | The overlay did not show "applied" in status or when applying the overlay, the system cleared the path file (so did not apply the overlay)     |
| FPGAState     | The fpga device's state file did not contain "operating" after writing a bitstream                                                             |
| Argument      | The user provided a bad input argument                                                                                                         |
| IORead        | The system IO failed to read from a file - a mapping of Rust's std:io::Error to FpgadError                                                     |
| IOWrite       | The system IO failed to write to a file - a mapping of Rust's std:io::Error to FpgadError                                                      |
| IOCreate      | The system IO failed to create a dir or file - a mapping of Rust's std:io::Error to FpgadError                                                 |
| IODelete      | The system IO failed to delete a file or directory - a mapping of Rust's std:io::Error to FpgadError                                           |
| IOReadDir     | The system IO failed to read the contents of a directory - a mapping of Rust's std:io::Error to FpgadError                                     |
| Softener      | Error using a softener (only if `softeners` feature enabled)                                                                                   |
| Internal      | Internal error occurred  which cannot be explained by the above (e.g. the device-tree overlay subsystem is not active or not mounted correctly |

## Status interface

`com.canonical.fpgad.status` -- FPGAd interface for read only access to properties and attributes (i.e. getters)

---

### Summary

| Method                                | Signature | Description                                                                                     |
|---------------------------------------|-----------|-------------------------------------------------------------------------------------------------|
| [GetFpgaFlags](#getfpgaflags)         | ss → s    | Read from `/sys/class/fpga_manager/<device_handle>/flags`                                       |
| [GetFpgaState](#getfpgastate)         | ss → s    | Read from `/sys/class/fpga_manager/<device_handle>/status`                                      |
| [GetOverlayStatus](#getoverlaystatus) | ss → s    | Read the `status` and source `path` of an overlay in `/sys/kernel/config/device-tree/overlays/` |
| [GetOverlays](#getoverlays)           | - → s     | Get a list of all present overlay handles                                                       |
| [GetPlatformType](#getplatformtype)   | s → s     | Get platform type for a given device                                                            |
| [GetPlatformTypes](#getplatformtypes) | - → s     | Get list of platform types for all present devices in `/sys/class/fpga_manager/`                |
| [ReadProperty](#readproperty)         | s → s     | Read a property by name                                                                         |

---

### Methods

#### GetFpgaFlags

- <b>Description:</b> Read the current flags assigned to an FPGA device

- <b>Signature:</b> ss → s

- <b>Inputs:</b>
    - str: [platform_string](#platform-string) - <code>''</code> or valid platform
    - str: [device_handle](#device-handle) - valid device handle required.

- <b>Output:</b>
    - str: contents of
      `/sys/class/fpga_manager/<device_handle>/flags` in hexadecimal format or [FpgadError string](#error-strings)

---

#### GetFpgaState

- <b>Description:</b> Report the contents of state file in
  `/sys/class/fpga_manager/<device_handle>/state`

- <b>Signature:</b> ss → s

- <b>Inputs:</b>
    - str: [platform_string](#platform-string) - <code>''</code> or valid platform
    - str: [device_handle](#device-handle) - valid device handle required.

- <b>Output:</b>
    - str: The contents of
      `/sys/class/fpga_manager/<device_handle>/state` - typically
      `"operating"`,
      `"unknown"` or [FpgadError string](#error-strings)

---

#### GetOverlayStatus

- <b>Description:</b> Read the current `status` and `path` attributes of a device-tree overlay directory in `/sys/kernel/config/device-tree/overlays/<overlay_handle>`

- <b>Signature:</b> ss → s

- <b>Inputs:</b>
    - str: [platform_string](#platform-string) - empty string or valid platform
    - str: [overlay_handle](#overlay_handle) - valid device handle required.

- <b>Output:</b>
    - str: the contents `/sys/kernel/config/device-tree/overlays/<overlay_handle>/path` followed by a space and then `/sys/kernel/config/device-tree/overlays/<overlay_handle>/status` e.g. `"\"k26_starter_kits.dtbo\" applied"`
      or [FpgadError string](#error-strings)

---

#### GetOverlays

- <b>Description:</b> Provide a list of valid existing overlay_handles

- <b>Signature:</b> - → s

- <b>Inputs:</b>
    - <i>(none)</i>

- <b>Output:</b>
    - str: the names of all subdirs of `/sys/kernel/config/device-tree/overlays/` separated by newline (`\n`) characters e.g. `"overlay0\noverlay1"` or [FpgadError string](#error-strings)

---

#### GetPlatformType

- <b>Description:</b> Get the compatibility string for a device for use as the [platform_string](#platform-string) input for other method calls.

- <b>Signature:</b> s → s

- <b>Inputs:</b>
    - str: [device_handle](#device-handle) - valid device handle required.

- <b>Output:</b>
    - str: The compatibility string of the requested device_handle as required for [platform_string](#platform-string) inputs (e.g. `"dev0:vendor,specific-platform"`) or [FpgadError string](#error-strings)

---

#### GetPlatformTypes

- <b>Description:</b> Find all devices and associated platform compatibility strings

- <b>Signature:</b> - → s

- <b>Inputs:</b>
    - <i>(none)</i>

- <b>Output:</b>
    - str: A single string containing a map of [<code>device_handles</code>](#device-handle) to [<code>platform_string</code>](#platform-string) with one device per line, with a colon as the separator between the
      `device_hande` and the
      `platform_string`, with the commas of the platform string being retained i.e.
      `"dev0:vendor,specific-platform\ndev1:vendor,specific-platform\n"` or [FpgadError string](#error-strings)

---

#### ReadProperty

- <b>Description:</b> Report the contents of any attribute file at the provided path, which must be a child of the `/sys/class/fpga_manager/` directory

- <b>Signature:</b> s → s

- <b>Inputs:</b>
    - str: The path of the desired fpga_manager attribute e.g.  `"/sys/class/fpga_manager/<device_handle>/flags"`, `"/sys/class/fpga_manager/<device_handle>/name"` or `"/sys/class/fpga_manager/<device_handle>/key"`

- <b>Output:</b>
    - str: The contents of the requested file or [FpgadError string](#error-strings)

## Control interface

`com.canonical.fpgad.control` - for read/write access to properties and attributes (i.e. setters)

---

### Summary

| Method                                        | Signature | Description                            |
|-----------------------------------------------|-----------|----------------------------------------|
| [ApplyOverlay](#applyoverlay)                 | ssss → s  | Apply an overlay to a device           |
| [RemoveOverlay](#removeoverlay)               | ss → s    | Remove an overlay from a device        |
| [SetFpgaFlags](#setfpgaflags)                 | ssu → s   | Set FPGA flags for a given device      |
| [WriteBitstreamDirect](#writebitstreamdirect) | ssss → s  | Write a bitstream directly to the FPGA |
| [WriteProperty](#writeproperty)               | ss → s    | Write a property value                 |

---

### Methods

#### ApplyOverlay

- <b>Description:</b> Create a new directory in `/sys/kernel/config/device-tree/overlays/`, and write the `.dtbo` file's path (relative to firmware_lookup_path) to `/sys/kernel/config/device-tree/overlays/<overlay_handle>/path`, before checking that the path write stuck, and the status changed to "applied" or return an error.

- <b>Signature:</b> ssss → s

- <b>Inputs:</b>
    - str: [platform_string](#platform-string)
    - str: [overlay_handle](#overlay_handle)
    - str: overlay_source_path - full path to the `.dtbo` overlay file to be applied
    - str: [firmware_lookup_path](#firmware-lookup-path)

- <b>Output:</b>
    - str: Result of the operation in the form of `"<overlay_source_path> loaded via \"/sys/kernel/config/device-tree/overlays/<overlay_handle>\" using firmware lookup path: \'\"<firmware_lookup_path>\"\'"` or [FpgadError string](#error-strings)

---

#### RemoveOverlay

- <b>Description:</b> Delete the directory at `/sys/kernel/config/device-tree/overlays/<overlay_handle>`, thus removing the overlay. Note, this may not unload any loaded bitstreams - this is driver/platform specific.

- <b>Signature:</b> ss → s

- <b>Inputs:</b>
    - str: [platform_string](#platform-string)
    - str: [overlay_handle](#overlay-handle)

- <b>Output:</b>
    - str: Result of the operation in the form of `"<overlay_handle> removed by deleting \"/sys/kernel/config/device-tree/overlays/<overlay_handle>\""` or [FpgadError string](#error-strings)

---

#### SetFpgaFlags

- <b>Description:</b> Write `flags` to `/sys/class/fpga_manager/<device_handle>/flags`

- <b>Signature:</b> ssu → s

- <b>Inputs:</b>
    - str: [platform_string](#platform-string)
    - str: [device_handle](#device-handle)
    - u32: flags - decimal representation of the flags

- <b>Output:</b>
    - str: Result of the operation in the form of `"Flags set to <flags> for <device_handle>"`or [FpgadError string](#error-strings)

---

#### WriteBitstreamDirect

- <b>Description:</b> Set the kernel firmware path to `firmware_lookup_path` before writing the `bitstream_path_str` relative to `firmware_lookup_path` to `/sys/class/fpga_manager/<device_handle>/firmware`

- <b>Signature:</b> ssss → s

- <b>Inputs:</b>
    - str: [platform_string](#platform-string)
    - str: [device_handle](#device-handle)
    - str: bitstream_path_str - full path to the bitstream file to be loaded
    - str: [firmware_lookup_path](#firmware-lookup-path)

- <b>Output:</b>
    - str: Result of the operation in the form of `"<bitstream_path_str> loaded to fpga0 using firmware lookup path: \'\"<firmware_lookup_path>\"\'"` or [FpgadError string](#error-strings)

---

#### WriteProperty

- <b>Description:</b> Write to any attribute file at the provided path, which must be a child of the `/sys/class/fpga_manager/` directory

- <b>Signature:</b> ss → s

- <b>Inputs:</b>
    - str: property_path_str - full path to the desired attribute file to write into. Must be a child of `/sys/class/fpga_manager/`
    - str: data - the data to write to the provided attribute file

- <b>Output:</b>
    - str: Result of the operation in the form of `"<data> written to <property_path_str>"` or [FpgadError string](#error-strings)

## Typical control sequence

#### FPGA only:

1. control.SetFpgaFlags(fpga_handle, flags)
2. control.WriteBitstreamDirect(fpga_handle, bitstream_path)

#### Overlay only:

1. status.GetOverlayStatus(overlay_handle) <- check doesn't exist
2. control.SetFpgaFlags(device_handle, flags) <- does check for sticking internally
3. control.CreateOverlay(overlay_handle) <- just makes a dir and checks the subsystem created the internal files
4. control.ApplyOverlay(overlay_handle, dtbo_path) <- writes dtbo_path to overlay and asserts overlay status
5. status.GetFpgaState(fpga_handle) <- check it is
   `operating`

#### Combined:

1. control.SetFpgaFlags(device_handle, flags) <- >does check for sticking internally
2. control.WriteBitstreamDirect
3. control.CreateOverlay(overlay_handle) <- just makes a dir and checks the subsystem created the internal files
4. control.ApplyOverlay(overlay_handle, dtbo_path) <- writes dtbo_path to overlay and asserts overlay status
5. status.GetFpgaState(fpga_handle) <- check it is
   `operating`

#### Removing:

The FPGA subsystem does not have a way to remove an overlay. Instead, you must write a new one.

To remove an overlay simply call:

1. control.RemoveOverlay(overlay_handle)

## Busctrl Call Examples

### Status (unprivileged)

To get the state of an FPGA device:

```shell
busctl call --system com.canonical.fpgad /com/canonical/fpgad/status com.canonical.fpgad.status GetFpgaState ss "" "fpga0"
```

To get the currently set flags for an FPGA device:

```shell
busctl call --system com.canonical.fpgad /com/canonical/fpgad/status com.canonical.fpgad.status GetFpgaFlags ss "" "fpga0"
```

To get the current status of an overlay with given handle and platform:

```shell
busctl call --system com.canonical.fpgad /com/canonical/fpgad/status com.canonical.fpgad.status GetOverlayStatus ss "xlnx" "fpga0"
```

To get the compatibility string of a given FPGA device:

```shell
busctl call --system com.canonical.fpgad /com/canonical/fpgad/status com.canonical.fpgad.status GetPlatformType s "fpga0"
```

To get all platforms for all devices:

```shell
busctl call --system com.canonical.fpgad /com/canonical/fpgad/status com.canonical.fpgad.status GetPlatformTypes
```

To get all currently present overlay handles:

```shell
busctl call --system com.canonical.fpgad /com/canonical/fpgad/status com.canonical.fpgad.status GetOverlays
```

### Control (privileged)

#### set flags

To set the flags of an FPGA device:

```shell
sudo busctl call --system com.canonical.fpgad /com/canonical/fpgad/control com.canonical.fpgad.control SetFpgaFlags ssu "" "fpga0" 0

sudo busctl call --system com.canonical.fpgad /com/canonical/fpgad/control com.canonical.fpgad.control SetFpgaFlags ssu "" "fpga0" 12 (converts to hex so is stored as 0xC
sudo busctl call --system com.canonical.fpgad /com/canonical/fpgad/control com.canonical.fpgad.control SetFpgaFlags ssu "" "fpga0" 0x0C
sudo busctl call --system com.canonical.fpgad /com/canonical/fpgad/control com.canonical.fpgad.control SetFpgaFlags ssu "" "fpga0" 0b1100 (converts to hex so is stored as 0xC)
```

#### apply an overlay

Using default
`fw_search_path` generation:

```shell
sudo busctl call --system com.canonical.fpgad /com/canonical/fpgad/control com.canonical.fpgad.control ApplyOverlay ssss "xlnx" "fpga0" "/lib/firmware/k26-starter-kits.dtbo" ""
```

or manually specified
`fw_search_path`:

```shell
sudo busctl call --system com.canonical.fpgad /com/canonical/fpgad/control com.canonical.fpgad.control ApplyOverlay ssss "xlnx" "fpga0" "/lib/firmware/xilinx/k26-starter-kits/k26_starter_kits.dtbo" "/lib/firmware/xilinx/k26-starter-kits"
```

#### write a bitstream

Using automated platform detectoin and default
`fw_search_path` generation:

```shell
sudo busctl call --system com.canonical.fpgad /com/canonical/fpgad/control com.canonical.fpgad.control WriteBitstreamDirect ssss "" "fpga0" "/lib/firmware/k26-starter-kits.bit.bin" ""
```

or using specific platform and specific
`fw_search_path`:

```shell
sudo busctl call --system com.canonical.fpgad /com/canonical/fpgad/control com.canonical.fpgad.control WriteBitstreamDirect ssss "xlnx" "fpga0" "/lib/firmware/xilinx/k26-starter-kits/k26_starter_kits.bit.bin" "/lib/firmware/"
```

#### remove an overlay

To remove an overlay with provided platform and handle:

```shell
sudo busctl call --system com.canonical.fpgad /com/canonical/fpgad/control com.canonical.fpgad.control RemoveOverlay ss "xlnx" "fpga0"
```

Consider using
`GetOverlays` and/or
`GetOverlayStatus` if you don't know the handle.

#### other properties

The virtual files contained within
`/sys/class/fpga_manager/fpga*/`, which do not have specific interfaces, can be accessed by using ReadProperty or WriteProperty e.g.

```shell
sudo busctl call --system com.canonical.fpgad /com/canonical/fpgad/status com.canonical.fpgad.status ReadProperty s "/sys/class/fpga_manager/fpga0/name"
```

```shell
sudo busctl call --system com.canonical.fpgad /com/canonical/fpgad/control com.canonical.fpgad.control WritePropertyBytes s ay "/sys/class/fpga_manager/fpga0/key" 0xAB 0xAD 0xC0 0xDE
```

### Snap

```shell
sudo snap install fpgad
sudo snap connect fpgad:fpga
```


