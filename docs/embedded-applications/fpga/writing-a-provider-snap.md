(writing-a-provider-snaps)=
# Writing a provider snap

- [k26-default-bitstreams](https://github.com/canonical/k26-default-bitstreams/)

[//]: # ( TODO: edit the following to reference "my-snap" or something)

## Example snapcraft.yaml


## snapcraft.yaml explained

The `plugs` entry here allows the connection to be made between this snap and the fpgad daemon
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
here `daemon: oneshot` means "run once on startup and then it is finished".

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
Here `version` just runs a simple script to generate a unique version string, `k26-default-bitstreams` part defines how to build the rust package which creates the `bin/k26-default-bitstreams` used in the app section and `bitstream-data` makes a copy of the project's `./data` folder available from the snap root at `$SNAP/data`.

## Content Interface

[//]: # ( TODO: edit the following to reference "my-snap" or something)