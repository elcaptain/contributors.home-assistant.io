---
title: Use strict typing
description: "Platinum rule strict-typing: the integration is fully typed and checked with strict mypy settings."
sidebarLabel: strict-typing
sidebarGroup: Platinum
order: 3
---

This is a Platinum rule. In `quality_scale.yaml`, it is called `strict-typing`.

The integration must be fully typed, and listed in the `.strict-typing` file of core, so mypy checks it with strict settings. Its library must be typed as well, and ship a `py.typed` file as described in [PEP 561](https://peps.python.org/pep-0561/).

## Why it matters

Python does not check types at runtime, so a wrong type shows up as a bug in front of a user. Type hints let mypy find those bugs before the code is merged. Strict mode makes mypy fail on missing hints, so nothing slips through untyped.

The library matters too. Without a `py.typed` file, mypy treats everything from the library as `Any`, and all the hints in the integration that touch the library check nothing.

## Example

Add the integration to `.strict-typing` in the root of the [core repository](https://github.com/home-assistant/core), in alphabetical order:

```text
homeassistant.components.my_integration.*
```

Then let hassfest update `mypy.ini`, and run mypy on the integration:

```shell
python3 -m script.hassfest -p mypy_config
mypy homeassistant/components/my_integration
```

With strict typing, use a typed config entry everywhere you get one: in `async_setup_entry`, `async_unload_entry`, the platforms, diagnostics, and the options flow. See [runtime-data](/core/quality-scale/runtime-data/) for how to define it:

```python
type MyConfigEntry = ConfigEntry[MyClient]


async def async_setup_entry(
    hass: HomeAssistant, entry: MyConfigEntry
) -> None:
    """Set up My integration from a config entry."""
```

See [Typing](/core/development/typing/) for how core uses type hints and mypy.

## How it is checked

- Hassfest, when the rule is `done`: the integration must be in `.strict-typing`, and every requirement in its manifest must ship a `py.typed` file, or have a separate `types-` stubs package. Hassfest can only check the requirements that are installed in your environment, in the version the manifest asks for.
- Hassfest, when the rule is `done` together with [runtime-data](/core/quality-scale/runtime-data/): `async_setup_entry`, `async_unload_entry`, and the other entry points in `__init__.py`, the platforms, `diagnostics.py`, and the options flow must use a typed config entry, named like `MyConfigEntry`.
- Pylint warns when a coordinator does not use the typed config entry of the integration (`home-assistant-coordinator-untyped-config-entry`).
- Mypy runs in CI and in the Git hooks, and fails on type errors.

## Exceptions

There are no exceptions to this rule.

## Related rules

- [runtime-data](/core/quality-scale/runtime-data/): store runtime data in runtime_data.
