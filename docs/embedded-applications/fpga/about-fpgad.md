(about-fpgad)=

# About FPGAd and provider snaps

(writing-a-provider-snaps)=



A provider snap is a snap which contains bitstreams, device tree overlays, meta data (e.g. shell.json for dfx-mgr bitstreams) and user space applications to interface with the running bitstream.
It should also contain at least one snap application which can be configured to load at startup or run on command.
This can be anything from a python script to make a single call to load a bitstream/apply an overlay, to a full binary application which prepares the device, ensures everything is working and runs a GUI application.


##Before starting

Before endeavouring to write a provider snap, please come up to speed with the [Craft a snap](https://documentation.ubuntu.com/snapcraft/stable/tutorials/craft-a-snap/) tutorial. It will describe the overarching process of packing a snap.

If you intend to make the snap package publicly available (or available unlisted but still hosted on the snap store), please also see the guides in [Publishing](https://documentation.ubuntu.com/snapcraft/stable/how-to/publishing/) especially [How to register a snap](https://documentation.ubuntu.com/snapcraft/stable/how-to/publishing/register-a-snap/#how-to-register-a-snap) and [Publish a snap](https://documentation.ubuntu.com/snapcraft/stable/how-to/publishing/publish-a-snap/).

The snap can be built from source automatically, alleviating some of the maintenance burden. See [Manage revisions and releases](https://documentation.ubuntu.com/snapcraft/stable/how-to/publishing/manage-revisions-and-releases/) for more information on that subject.

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

## Writing a provider snap

The [k26-default-bitstreams](https://github.com/canonical/k26-default-bitstreams/) snap is an example of a provider snap.
It is written in rust and uses FPGAd's [DBus](#dbus) interface to initiate the load on startup.
The provided [README](https://github.com/canonical/k26-default-bitstreams/blob/main/README.md) on the [k26-default-bitstreams](https://github.com/canonical/k26-default-bitstreams/) repository provides an explanation of the contained
`snap/snapcraft.yaml`.
If you're planning to use Rust for your application, the associated source code can be used as a basis for using the [zbus](https://docs.rs/zbus/latest/zbus/) crate to interface with FPGAd via DBus (examples in other languages may come in the future).

[//]: # ( TODO: edit the following to reference "my-snap" or something)

### snapcraft.yaml template

```yaml
<some template>
```


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


