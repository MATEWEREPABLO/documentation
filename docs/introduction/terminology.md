# Terminology

Use these terms consistently across your implementation, support workflows, and internal documentation.

## Merchant

A merchant is the business account that integrates with the Yonne External API.

The merchant owns the API key, wallet balance, pickup settings, and order lifecycle from the platform side.

## Courier

A courier is the delivery provider or rider network that fulfills a shipment.

Yonne routes orders to eligible couriers based on availability, affiliation, and vehicle capacity rules.

## Order

An order is a delivery request created through `POST /api/v1/external/create-order`.

An order includes pickup and delivery details, receiver details, pricing context, and metadata. Once created, the order can be tracked, cancelled when still eligible, and reconciled through your own merchant references.

## Tracking ID

A tracking ID is the customer-facing delivery identifier returned after successful order creation.

Use it for tracking links and support conversations.

## Merchant reference ID

A merchant reference ID is your own external identifier for the order.

Use it to reconcile Yonne deliveries with your checkout, OMS, or ERP records.
