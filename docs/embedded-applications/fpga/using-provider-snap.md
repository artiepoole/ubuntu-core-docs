(using-provider-snap)=
# Using a provider snap

If running on target device:
```shell
snapcraft
sudo snap install k26-default-bitstreams..._arm64.snap
sudo snap connect k26-default-bitstreams:fpgad-dbus fpgad:dbus-daemon
```

```{note}
the `fpgad:dbus-daemon` is external to this repo so may be subject to change. Check [fpgad's snapcraft.yaml](https://github.com/canonical/fpgad/blob/main/snap/snapcraft.yaml) for changes if this command fails.
```
