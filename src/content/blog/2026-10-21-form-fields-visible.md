---
title: "Show form fields based on other fields"
description: As of Home Assistant Core 2026.11, config flows, options flows, and blueprints can show a field only while other fields have certain values.
date: 2026-10-21
authors:
  - Paul Bottein
areas:
  - core
---

As of Home Assistant Core 2026.11, config flows, options flows, and blueprints can show a field only while other fields have certain values. You no longer need an extra step, or to show the form again after submit, just to show or hide a few fields. Flows and blueprints use the same `visible` condition.

## Config and options flows

Give the `Required` or `Optional` marker from `homeassistant.data_entry_flow` a `visible` condition:

```python
from homeassistant import data_entry_flow
from homeassistant.helpers.selector import BooleanSelector

DATA_SCHEMA = probatio.Schema(
    {
        probatio.Required(CONF_SSL, default=False): BooleanSelector(),
        data_entry_flow.Required(
            CONF_VERIFY_SSL,
            default=True,
            visible={"field": CONF_SSL, "value": True},
        ): BooleanSelector(),
    }
)
```

The **Verify SSL certificate** checkbox only shows once the user checks **Use SSL**. While it is hidden, it is not required and its value is not saved.

Read [fields that show based on another field](/core/integrations/flow-steps/#fields-that-show-based-on-another-field) for the available conditions and how hidden fields behave.

## Blueprints

Blueprint inputs and input sections take the same `visible` key, next to `default` and `collapsed`:

```yaml
blueprint:
  name: Motion light
  domain: automation
  input:
    dim_mode:
      name: Dimming
      default: none
      selector:
        select:
          options:
            - none
            - fixed
    brightness:
      name: Brightness
      default: 50
      visible:
        field: dim_mode
        value: fixed
      selector:
        number:
          min: 1
          max: 100
```

**Brightness** only shows when **Dimming** is set to `fixed`. A hidden input does not need a value: where the blueprint uses it with `!input`, it gets its `default`, or nothing if it has none. Setting `visible` on an input section hides the whole section and its inputs.

Read [showing inputs based on other inputs](https://www.home-assistant.io/docs/blueprint/schema/#showing-inputs-based-on-other-inputs) in the user documentation for the details.
