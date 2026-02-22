---
name: grid-load-shifter
description: >
  Energy-aware load shifting using Home Assistant data. Read electricity
  prices, consumption, solar production, and battery levels from HA. Schedule
  deferrable loads (EV charging, dishwasher, pool pump, laundry) during
  cheapest rate periods. Use when asked about electricity costs, energy
  optimization, peak/off-peak scheduling, solar self-consumption, or
  smart appliance timing. Works with any utility rate plan worldwide.
metadata: { "openclaw": { "emoji": "⚡" } }
---

# Grid-Aware Load Shifter

Shift heavy residential loads to the cheapest electricity hours using Home Assistant energy data.

## Quick Start

```bash
# Find all energy-related entities in HA
python3 {baseDir}/scripts/ha_bridge.py discover

# Get a full energy dashboard snapshot (prices, solar, consumption, batteries)
python3 {baseDir}/scripts/ha_bridge.py energy-summary

# Turn on the EV charger
python3 {baseDir}/scripts/ha_bridge.py call-service switch/turn_on --entity-id switch.ev_charger
```

## Connection

Two paths to reach Home Assistant:

1. **MCP (preferred):** If the HA MCP server is configured, use `mcporter call homeassistant.<tool>` directly.
2. **REST API:** Use `python3 {baseDir}/scripts/ha_bridge.py`. Requires `HA_URL` and `HA_TOKEN` environment variables (see `.env.example` in `{baseDir}`).

## Commands

| Command | What it does | Example |
|---|---|---|
| `discover` | List all energy entities | `ha_bridge.py discover` |
| `energy-summary` | One-shot dashboard (prices + consumption + solar + storage) | `ha_bridge.py energy-summary` |
| `status <entity>` | Read a single entity's state and attributes | `ha_bridge.py status sensor.electricity_price` |
| `call-service <d/s>` | Call any HA service | `ha_bridge.py call-service switch/turn_on --entity-id switch.ev_charger` |
| `history <entity>` | Get state changes over last N hours | `ha_bridge.py history sensor.grid_import --hours 24` |

All commands output JSON to stdout.

## Load-Shifting Workflow

Follow these steps when asked about energy optimization:

1. **Discover** available energy entities: run `discover` or `energy-summary`
2. **Read prices**: Check pricing entities' state and attributes — look for:
   - Hourly price arrays in `today` / `tomorrow` / `prices_today` / `rates` attributes
   - `price_level` attribute (CHEAP / NORMAL / EXPENSIVE)
   - Current vs. average price comparison
3. **Identify deferrable loads**: Find `switch.*` entities for schedulable devices (EV charger, pool pump, dishwasher, washer/dryer)
4. **Find the cheapest window**: Scan hourly prices for the contiguous N-hour block with the lowest sum (N = estimated run time of device)
5. **Execute**: Call `switch/turn_on` at the optimal time, or `automation/trigger` if the user has an existing automation

## Interpreting Price Data

Different integrations expose prices differently:

- **Hourly arrays** (Nordpool, ENTSO-e, Octopus): Read `today`/`tomorrow` attributes → find cheapest hours
- **Price level** (Tibber): Read `price_level` → act when CHEAP or VERY_CHEAP
- **Utility meter tariffs**: Read `sensor.*_peak` vs `sensor.*_offpeak` → user's HA automations switch tariffs at configured times
- **Static price**: Read `current_price` attribute → compare against historical average

## Cost Savings Estimate

When recommending a shift, show estimated savings:

```
savings = (current_rate - cheapest_rate) × device_power_kw × run_duration_hours
```

## Solar Self-Consumption

If solar sensors exist, align loads with peak production:

- Read `sensor.forecast_solar_*` or `sensor.solcast_*` for today's forecast
- Shift loads to hours with highest expected production
- This avoids grid import entirely — savings = full retail rate × kWh shifted

## Entity Reference

For detailed entity patterns across providers, read: [energy_entities.md]({baseDir}/references/energy_entities.md)
