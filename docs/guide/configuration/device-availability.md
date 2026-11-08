---
sidebarDepth: 1
redirectFrom: /information/availability.md
---

# Device Availability

The availability feature checks whether your devices are online. The availability state of a device is published
to `zigbee2mqtt/[FRIENDLY_NAME]/availability` with payload `{"state":"online"}` or `{"state":"offline"}` (this message is a retained MQTT message).

```yaml
# Optional: Availability feature
availability:
    # Enable the feature (default: false)
    enabled: true
```

The availability feature works differently for active and passive devices, since passive devices cannot be pinged.

- Active devices (_non battery-powered_): by default they have to check-in\*\* every 10 minutes.
  If they don't, they will be pinged, if that fails the device will be marked as `offline`.
- Passive devices (_battery-powered_): by default they have to check-in\*\* every 25 hours.
  If they don't they will be marked as `offline`.

Note that this timeout is persisted between Zigbee2MQTT restarts. So if you for example stop Zigbee2MQTT for longer than 10
minutes, all your active devices will be marked as `offline` initially until they check-in\*\* again.

\*\* A check-in is any kind of Zigbee message from the device that reaches Zigbee2MQTT (even internal updates that are not displayed/reported).

## Advanced configuration

```yaml
# Note: all options are optional
availability:
    enabled: true
    active:
        # Time after which an active device will be marked as offline in minutes (default: 10 minutes)
        timeout: 10
        # Maximum jitter (in msec) allowed on timeout to avoid availability pings trying to trigger around the same time (default: 30000, min: 1000)
        max_jitter: 30000
        # Response timeout of each ping in milliseconds (default: 10000)
        ping_timeout: 10000
        # Delay before retrying a failed ping in milliseconds (default: 3000)
        retry_delay: 3000
        # Enable timeout backoff on failed availability pings (default: true)
        # Pattern used: x1.5, x3, x6, x12... (with default timeout of 10min: 10, 15, 30, 60, 120...)
        backoff: true
        # Pause availability pings when backoff reaches over this limit until a new Zigbee message is received from the device. (default: 0, min: 0)
        # A value of zero disables pausing, else see `backoff` pattern above.
        pause_on_backoff_gt: 0
    passive:
        # Time after which a passive device will be marked as offline in minutes (default: 1500 minutes aka 25 hours)
        timeout: 1500

devices:
    '0x12345678':
        friendly_name: 'my_bulb'
        # Set availability: false to disable the availability feature for a specific device
        availability: false
    '0x87654321':
        friendly_name: 'my_switch'
        # Change availability timeout to 3 minutes for this device only
        availability:
            timeout: 3
            # Override ping timing for this active device only
            ping_timeout: 400
            retry_delay: 100
            # active devices also can specify `max_jitter`, `backoff`, `pause_on_backoff_gt` (see above)
```

If you want to enable the availability feature for only certain devices, don't add `availability: enabled: true` in
your `configuration.yaml` but specify it for that device only, e.g.

```yaml
devices:
    '0x87654321':
        friendly_name: 'my_switch'
        # Enable availability for just 'my_switch'
        availability: true
```

### Ping timing

`timeout` is the inactivity period in **minutes** before checking a device. `ping_timeout` is the response timeout of
an individual availability ping in **milliseconds**, and `retry_delay` is the delay before the second attempt in
**milliseconds**. These ping settings apply only to active devices; passive devices are not pinged.

Device options override `availability.active` one setting at a time. Omitted options inherit the global values,
which default to `ping_timeout: 10000` and `retry_delay: 3000`. An availability object can specify only these overrides;
there is no need to repeat `timeout`. Setting `availability: false` still disables checks for that device.

`ping_timeout` must be an integer from 1 to 2147483647. `retry_delay` must be an integer from 0 to 2147483647;
zero removes the wait between attempts. To remove a device override through `bridge/request/device/options`, set
that option to `null`; the global value then applies to subsequent checks. Invalid live device ping timing overrides
fall back to the global values; invalid values in the configuration file fail startup validation.

A previously available device gets up to two attempts, with recovery enabled on the second attempt. An already
unavailable device gets one attempt. Changing ping timing does not change the attempt count, jitter, timer padding,
queue spacing, or backoff. Adapter delivery and recovery may add time, so `ping_timeout` is not an end-to-end deadline
for declaring a device offline.

Shorter response timeouts can mark powered devices offline when the network is busy or while they are starting.
Tune selected devices only after observing their response times. The example values above are not recommended defaults.

## State retrieval

When the availability feature is enabled and a device reconnects or announces itself on the network, Zigbee2MQTT will retrieve the
state of the device. This is e.g. handy when a bulb turns itself on after being reconnected to mains power. The
following attributes will be read: `state`, `brightness`, `color_temp` and `color`.

## Performance considerations

- The pinging can be heavy on the coordinator, especially if you are using a CC2530 or CC2531 adapter.
- Higher `timeout` for active devices results in less pinging so less stress on the coordinator.

## Groups

When enabling device availability, availability will also be enabled for groups. A group is marked as available when at least one device in it is available.
