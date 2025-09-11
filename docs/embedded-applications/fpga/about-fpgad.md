(about-fpgad)=

# About FPGAd and provider snaps

A provider snap is a snap which contains bitstreams, device tree overlays, metadata (e.g. shell.json for dfx-mgr bitstreams) and user space applications to interface with the running bitstream.
It should also contain at least one snap application which can be configured to load at startup or run on command.
This could be simple like a python script to make a single call to load a bitstream/apply an overlay, or a full binary application which prepares the device, ensures everything is working and runs a GUI application with continuous monitoring.

The following sections describe the process of using FPGAd directly (using the command line interface (CLI) as well as providing guidance for writing and using provider snaps.

## Provider snaps

### Writing a provider snap

#### Before starting

Before endeavouring to write a provider snap, please come up to speed with the [Craft a snap](https://documentation.ubuntu.com/snapcraft/stable/tutorials/craft-a-snap/) tutorial. It will describe the overarching process of packing a snap.

If you intend to make the snap package publicly available (or available unlisted but still hosted on the snap store), please also see the guides in [Publishing](https://documentation.ubuntu.com/snapcraft/stable/how-to/publishing/) especially [How to register a snap](https://documentation.ubuntu.com/snapcraft/stable/how-to/publishing/register-a-snap/#how-to-register-a-snap) and [Publish a snap](https://documentation.ubuntu.com/snapcraft/stable/how-to/publishing/publish-a-snap/).

The snap can be built from source automatically, alleviating some of the maintenance burden. See [Manage revisions and releases](https://documentation.ubuntu.com/snapcraft/stable/how-to/publishing/manage-revisions-and-releases/) for more information on that subject.

It is also definitely work looking at the [k26-default-bitstreams](https://github.com/canonical/k26-default-bitstreams/) snap as an an example of a provider snap.
It is written in rust and uses FPGAd's [DBus](#dbus) interface to initiate the load on startup.
The provided [README](https://github.com/canonical/k26-default-bitstreams/blob/main/README.md) on the [k26-default-bitstreams](https://github.com/canonical/k26-default-bitstreams/) repository provides an explanation of the contained
`snap/snapcraft.yaml`.
If you're planning to use Rust for your application, the associated source code can be used as a basis for using the [zbus](https://docs.rs/zbus/latest/zbus/) crate to interface with FPGAd via DBus (examples in other languages may come in the future).

#### Process overview

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

#### creating the snapcraft.yaml

The first steps is to create a
`snapcraft.yaml`.
For this you can run

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
base: core24 # or core26 - should match Core image base version
summary: <your summary here>
description: |
  <Your longer description here>
grade: devel # or stable etc
confinement: strict # must be strict for Ubuntu Core - can be devmode if for classical image
plugs:
  <dbus-iface-name>: # It is recommended to use `fpgad-dbus` as the name here
    interface: dbus
    bus: system
    name: com.canonical.fpgad
  #<any other plugs you may need>:
apps:
  <your startup app name>: # this applicatoin runs on startup due to `daemon: oneshot`
    command: bin/<app1 binary name>
    daemon: oneshot # to run once on startup
    plugs:
      - fpgad-dbus
      <...>
    restart-condition: <always> # optional
    start-timeout: <30s> # not optional if restart-condition specified
  <your manual app name>: # this application can be run manually. If it matches the snap name it can be called using the snap name wihtout <snap-name>.<app-name> syntax
    command: bin/<app2 binary name>
  plugs:
    - fpgad-dbus
    <...>
parts:
  <app1 binary name>:
    plugin: <plugin> # see LINK for information on building inside a snap
    source: <relative/path/to/source>
    <...>
  <app2 binary name>:
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

#### writing the dbus application

####

[//]: # ( TODO: edit the following to reference "my-snap" or something)

### snapcraft.yaml template

#### Using the content Interface

#### Using softeners

#### snapcraft.yaml explained

The
`plugs` entry here allows the connection to be made between this snap and the fpgad daemon

```yaml
plugs:
  fgpad-dbus:
    interface: dbus
    bus: system
    name: com.canonical.fpgad
```

but it must also be added to the application:

```yaml
apps:
  k26-default-bitstreams:
    command: bin/k26-default-bitstreams
    daemon: oneshot
    plugs:
      - fpgad-dbus
```

here
`daemon: oneshot` means "run once on startup and then it is finished".

The parts section describes how to form the snap package

```yaml

parts:
  version:
    plugin: nil
    source: .
    build-snaps:
      - jq
    override-pull: |
      craftctl default
      cargo_version=$(cargo metadata --no-deps --format-version 1 | jq -r .packages[0].version)
      craftctl set version="$cargo_version+git$(date +'%Y%m%d').$(git describe --always --exclude '*')"
  k26-default-bitstreams:
    plugin: rust
    source: .
    rust-path:
      - k26-default-bitstreams
  bitstream-data:
    plugin: dump
    source: ./data/
    source-type: local
    organize:
      default-bitstreams: data/k26-starter-kits
```

Here
`version` just runs a simple script to generate a unique version string,
`k26-default-bitstreams` part defines how to build the rust package which creates the
`bin/k26-default-bitstreams` used in the app section and
`bitstream-data` makes a copy of the project's
`./data` folder available from the snap root at
`$SNAP/data`.

### Using a provider snap

If running on target device:

```shell
snapcraft
sudo snap install k26-default-bitstreams..._arm64.snap
sudo snap connect k26-default-bitstreams:fpgad-dbus fpgad:dbus-daemon
```

```{note}
the `fpgad:dbus-daemon` is external to this repo so may be subject to change. Check [fpgad's snapcraft.yaml](https://github.com/canonical/fpgad/blob/main/snap/snapcraft.yaml) for changes if this command fails.
```

### publishing your provider snap

## Platforms and Softeners

## Interfaces

# DBus

(dbus)=

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
`GetOverlayStatus` if you don't
know the handle.

#### other properties

The virtual files contained within
`/sys/class/fpga_manager/fpga*/`, which do not have specific interfaces, can be
accessed by using ReadProperty or WriteProperty e.g.

```shell
sudo busctl call --system com.canonical.fpgad /com/canonical/fpgad/status com.canonical.fpgad.status ReadProperty s "/sys/class/fpga_manager/fpga0/name"
```

```shell
sudo busctl call --system com.canonical.fpgad /com/canonical/fpgad/control com.canonical.fpgad.control WriteProperty ss "/sys/class/fpga_manager/fpga0/key" ""
```

### Snap

```shell
sudo snap install fpgad
sudo snap connect fpgad:fpga
```

### CLI

# FPGAd's Command Line Interface (CLI)

## Usage

```
Usage: [snap run] fpgad [OPTIONS] <COMMAND>

Commands:
  load    Load a bitstream or an overlay for the given device handle
  remove  Remove bitstream or an overlay
  set     Write a value to an attribute within the sysfs folder e.g. to edit /sys/class/fpga_manager/fpga0/flags
  status  Get the status information for the given device handle
  help    Print this message or the help of the given subcommand(s)

Options:
      --handle <HANDLE>  fpga device `HANDLE` to be used for the operations. Default value for this option is calculated in runtime and the application picks the first available fpga in the system (under /sys/class/fpga_manager)
  -h, --help             Print help

```

### Loading

```shell
fpgad [--handle=<device_handle>] load ( (overlay <file> [--handle=<handle>]) | (bitstream <file>) )
```

### Removing

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

## examples (for testing)

### Load

```shell
sudo ./target/debug/cli load bitstream /lib/firmware/k26-starter-kits.bit.bin
sudo ./target/debug/cli --handle=fpga0 load bitstream /lib/firmware/k26-starter-kits.bit.bin

sudo ./target/debug/cli load overlay /lib/firmware/k26-starter-kits.dtbo
sudo ./target/debug/cli load overlay /lib/firmware/k26-starter-kits.dtbo --handle=overlay_handle
sudo ./target/debug/cli --handle=fpga0 load overlay /lib/firmware/k26-starter-kits.dtbo --handle=overlay_handle
```

### Remove

```shell
sudo ./target/debug/cli --handle=fpga0 remove overlay
sudo ./target/debug/cli --handle=fpga0 remove overlay --handle=overlay_handle
```

### Set

```shell
sudo ./target/debug/cli set flags 0
sudo ./target/debug/cli --handle=fpga0 set flags 0
```

### Status

```shell
./target/debug/cli status
./target/debug/cli --handle=fpga0 status
```


