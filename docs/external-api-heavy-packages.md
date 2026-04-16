# Guide: Heavy Packages and Capacity-Aware Dispatch

This guide explains how Yonne handles heavier shipments and why rider assignment may fail even when riders are online.

## The dispatch model

Yonne uses **capacity-qualified dispatch**.

That means the system does **not** simply pick the nearest rider first.

Instead, dispatch works conceptually like this:

1. find online riders
2. filter by courier affiliation
3. filter by vehicle capacity
4. enforce vehicle-class rules
5. prefer the smallest valid vehicle class
6. then sort by distance

## Required weight

Dispatch uses a computed `required_weight_kg`.

This comes from:

- `weight_estimate`
- optionally `total_weight_kg`

If both are present, the system uses the greater value.

## Weight bucket interpretation

The routing layer treats weight buckets conservatively:

- `0-5kg` -> `5`
- `5-10kg` -> `10`
- `10-20kg` -> `20`
- `20-50kg` -> `50`
- `50kg+` -> above `50` for routing purposes

## The bike rule

`BIKE` riders are excluded when the shipment is above **50 kg required weight**.

This is a hard routing rule.

Even if a bike rider has a custom capacity override, the system does not assign bikes to shipments above that threshold.

## Suggested vehicle class in quotes

Quote responses may include:

- `required_weight_kg`
- `suggested_vehicle_class`
- `service_id`

Use these values in checkout UX to explain expected routing.

Example:

```json
{
  "suggested_vehicle_class": "VAN",
  "required_weight_kg": 95,
  "service_id": "van_std"
}
```

## Heavy-order example

```json
{
  "delivery_address": "Area 18, Lilongwe",
  "receiver_name": "John Phiri",
  "delivery_fee": 18500,
  "delivery_lat": -13.954,
  "delivery_lng": 33.792,
  "item_name": "Commercial freezer",
  "product_type": "Appliance",
  "weight_estimate": "50kg+",
  "total_weight_kg": 95
}
```

## No-capacity outcome

If no online rider can carry the shipment, the API may return:

```json
{
  "success": false,
  "error": "ERR_NO_CAPACITY_AVAILABLE",
  "message": "No online rider has sufficient vehicle capacity for this package weight.",
  "required_weight_kg": 95
}
```

HTTP status:

```text
422
```

## Recommended client behavior

When you receive `ERR_NO_CAPACITY_AVAILABLE`:

- do not retry immediately in a tight loop
- show a clear user-facing message
- suggest retrying later or contacting support
- keep the merchant order in a pre-submit or failed state until capacity is available

## Best practice

Always send:

- `weight_estimate`
- `total_weight_kg` when the actual weight is known

That gives pricing and routing the best chance to agree on the correct vehicle class.
