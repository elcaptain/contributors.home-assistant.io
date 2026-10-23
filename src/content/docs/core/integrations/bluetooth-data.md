---
title: Fetching Bluetooth data
description: Pick the right coordinator for a Bluetooth device, and turn its advertisements into entities.
sidebarGroup: Discovery
order: 4
---

Many Bluetooth devices send their data in advertisements: small packets they broadcast over and over, without anyone connecting to them. Home Assistant has coordinators that are driven by those advertisements instead of by polling. They work like the [data update coordinator](/core/integrations/fetching-data/), and take care of availability for you.

## Pick a coordinator

| Your device | Use |
| --- | --- |
| Mainly a sensor, binary sensor, or source of events, with all data in advertisements | `PassiveBluetoothProcessorCoordinator` |
| Mainly a sensor, binary sensor, or source of events, but some data needs a connection | `ActiveBluetoothProcessorCoordinator` |
| Something else, with all data in advertisements | `PassiveBluetoothDataUpdateCoordinator` |
| Something else, and some data needs a connection | `ActiveBluetoothDataUpdateCoordinator` |
| Only works with a connection, does not use advertisements | The normal [`DataUpdateCoordinator`](/core/integrations/fetching-data/) |

## Processor coordinators

The processor coordinators cut the code of sensor-like integrations down to almost nothing. Your library parses an advertisement, you turn the result into a `PassiveBluetoothDataUpdate`, and the framework creates and updates the entities when they show up.

A `PassiveBluetoothDataUpdate` has four mappings. The keys are `PassiveBluetoothEntityKey(key, device_id)`, like `PassiveBluetoothEntityKey("temperature", None)`:

| Field | What it holds |
| --- | --- |
| `devices` | The `DeviceInfo` per device ID. `None` is the main device. |
| `entity_descriptions` | The [entity description](/glossary/entity-description/) per entity key |
| `entity_names` | The name per entity key |
| `entity_data` | The value per entity key |

### Set up the coordinator

In `__init__.py`, create the coordinator with the address of the device and the parse method of your library. Start it after the platforms are set up, so they can subscribe first:

```python
from homeassistant.components.bluetooth import BluetoothScanningMode
from homeassistant.components.bluetooth.passive_update_processor import (
    PassiveBluetoothProcessorCoordinator,
)

type ExampleConfigEntry = ConfigEntry[PassiveBluetoothProcessorCoordinator]


async def async_setup_entry(
    hass: HomeAssistant, entry: ExampleConfigEntry
) -> None:
    """Set up an Example Bluetooth device."""
    assert entry.unique_id is not None
    parser = ExampleParser()

    coordinator = PassiveBluetoothProcessorCoordinator(
        hass,
        _LOGGER,
        address=entry.unique_id,
        mode=BluetoothScanningMode.ACTIVE,
        update_method=parser.update,
    )
    entry.runtime_data = coordinator

    await hass.config_entries.async_forward_entry_setups(entry, PLATFORMS)

    # Start after the platforms subscribed
    entry.async_on_unload(coordinator.async_start())
```

`connectable` defaults to `False` here, so you get advertisements from every adapter.

### The sensor platform

Each platform registers a processor that turns the parsed data into a `PassiveBluetoothDataUpdate`, and an entity class:

```python
from homeassistant.components.bluetooth.passive_update_processor import (
    PassiveBluetoothDataProcessor,
    PassiveBluetoothDataUpdate,
    PassiveBluetoothProcessorEntity,
)
from homeassistant.components.sensor import SensorEntity


def sensor_update_to_bluetooth_data_update(
    parsed: ExampleUpdate,
) -> PassiveBluetoothDataUpdate:
    """Turn parsed data into a Bluetooth data update."""
    return PassiveBluetoothDataUpdate(
        devices={None: parsed.device_info},
        entity_descriptions={...},
        entity_data={...},
        entity_names={...},
    )


async def async_setup_entry(
    hass: HomeAssistant,
    entry: ExampleConfigEntry,
    async_add_entities: AddConfigEntryEntitiesCallback,
) -> None:
    """Set up the Example Bluetooth sensors."""
    coordinator = entry.runtime_data
    processor = PassiveBluetoothDataProcessor(
        sensor_update_to_bluetooth_data_update
    )

    entry.async_on_unload(
        processor.async_add_entities_listener(
            ExampleBluetoothSensor, async_add_entities
        )
    )
    entry.async_on_unload(coordinator.async_register_processor(processor))


class ExampleBluetoothSensor(PassiveBluetoothProcessorEntity, SensorEntity):
    """A sensor of an Example Bluetooth device."""

    @property
    def native_value(self) -> float | int | str | None:
        """Return the value of the sensor."""
        return self.processor.entity_data.get(self.entity_key)
```

### Connect when needed

`ActiveBluetoothProcessorCoordinator` works the same way, and the platforms do not change. It adds two functions that run when an advertisement changes:

- `needs_poll_method(service_info, last_poll)`: returns whether a connection is needed now. `last_poll` is the number of seconds since the last poll, or `None`.
- `poll_method(service_info)`: connects to the device and returns the data.

```python
def _needs_poll(
    service_info: BluetoothServiceInfoBleak, last_poll: float | None
) -> bool:
    # Only poll when Home Assistant is running, the device needs it,
    # and a connectable adapter can reach it
    return (
        hass.state is CoreState.running
        and parser.poll_needed(service_info, last_poll)
        and bool(
            async_ble_device_from_address(
                hass, service_info.device.address, connectable=True
            )
        )
    )


async def _async_poll(service_info: BluetoothServiceInfoBleak):
    # The advertisement may come from an adapter that cannot connect,
    # so get a connectable BLEDevice for the same address
    if service_info.connectable:
        device = service_info.device
    elif not (
        device := async_ble_device_from_address(
            hass, service_info.device.address, True
        )
    ):
        raise RuntimeError(
            f"No connectable device for {service_info.device.address}"
        )

    return await parser.async_poll(device)


coordinator = ActiveBluetoothProcessorCoordinator(
    hass,
    _LOGGER,
    address=address,
    mode=BluetoothScanningMode.PASSIVE,
    update_method=parser.update,
    needs_poll_method=_needs_poll,
    poll_method=_async_poll,
    # Take advertisements from every adapter, and swap the BLEDevice
    # for a connectable one when polling
    connectable=False,
)
```

Import it from `homeassistant.components.bluetooth.active_update_processor`.

### Ask for active scans

Both processor coordinators accept `scan_interval` and `scan_duration`. They ask scanners in `AUTO` mode to scan this address actively for `scan_duration` seconds, every `scan_interval` seconds. Some devices only send their full data when they are scanned actively, and fall back to needing a connection when they have not been for a while. Pick a `scan_interval` shorter than that, and a `scan_duration` long enough to catch the next advertisement:

```python
coordinator = ActiveBluetoothProcessorCoordinator(
    ...,
    scan_interval=165.0,
    scan_duration=10.0,
)
```

Both default to `None`, which keeps the normal pace. Scanners the user set to `PASSIVE` or `ACTIVE` are not affected.

## Data update coordinators

For devices that are not mainly sensors, like a lock or a light, use `PassiveBluetoothDataUpdateCoordinator` (from `passive_update_coordinator`) or `ActiveBluetoothDataUpdateCoordinator` (from `active_update_coordinator`). Subclass them, and handle the advertisements in `_async_handle_bluetooth_event`:

```python
from homeassistant.components import bluetooth
from homeassistant.components.bluetooth.active_update_coordinator import (
    ActiveBluetoothDataUpdateCoordinator,
)
from homeassistant.core import CoreState, callback


class ExampleCoordinator(ActiveBluetoothDataUpdateCoordinator[None]):
    """Handle the data of an Example lock."""

    def __init__(
        self,
        hass: HomeAssistant,
        ble_device: BLEDevice,
        device: ExampleLock,
    ) -> None:
        """Initialize the coordinator."""
        super().__init__(
            hass=hass,
            logger=_LOGGER,
            address=ble_device.address,
            needs_poll_method=self._needs_poll,
            poll_method=self._async_update,
            mode=bluetooth.BluetoothScanningMode.ACTIVE,
            connectable=True,
        )
        self.device = device

    @callback
    def _needs_poll(
        self,
        service_info: bluetooth.BluetoothServiceInfoBleak,
        seconds_since_last_poll: float | None,
    ) -> bool:
        """Return whether to connect and poll the device."""
        return (
            self.hass.state is CoreState.running
            and self.device.poll_needed(seconds_since_last_poll)
            and bool(
                bluetooth.async_ble_device_from_address(
                    self.hass, service_info.device.address, connectable=True
                )
            )
        )

    async def _async_update(
        self, service_info: bluetooth.BluetoothServiceInfoBleak
    ) -> None:
        """Connect to the device and poll it."""

    @callback
    def _async_handle_bluetooth_event(
        self,
        service_info: bluetooth.BluetoothServiceInfoBleak,
        change: bluetooth.BluetoothChange,
    ) -> None:
        """Handle a new advertisement."""
        self.device.update_from_advertisement(service_info.advertisement)
        super()._async_handle_bluetooth_event(service_info, change)
```

Call the parent `_async_handle_bluetooth_event` from yours: it marks the device available, updates the listeners, and in the active version, schedules a poll when one is needed.

The passive version has no `needs_poll_method` and `poll_method`, and `connectable` defaults to `False`. Override `_async_handle_unavailable` to act when the device stops advertising.

> [!NOTE]
> Unavailability depends on the Bluetooth stack noticing that a device stopped advertising. On macOS, CoreBluetooth caches advertisements and may never report that, so `_async_handle_unavailable` might not be called.
