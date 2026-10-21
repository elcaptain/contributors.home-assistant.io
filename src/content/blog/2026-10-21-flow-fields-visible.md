---
title: "Show flow fields based on other fields"
description: As of Home Assistant Core 2026.11, config and options flows can show a field only while other fields of the same step have certain values.
date: 2026-10-21
authors:
  - Paul Bottein
areas:
  - core
---

As of Home Assistant Core 2026.11, config and options flows can show a field only while other fields of the same step have certain values. You no longer need an extra step, or to show the form again after submit, just to show or hide a few fields.

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
