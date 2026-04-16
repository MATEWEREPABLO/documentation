# Webhooks

Use webhooks to receive order status updates in your backend instead of relying only on polling.

This gives you faster state changes for merchant systems, customer notifications, and operational dashboards.

## Why webhooks matter

Webhooks help you:

- keep order status in sync with less polling
- trigger downstream notifications
- update merchant dashboards and support tools
- reconcile Yonne events with your own order records

## Include metadata on order creation

When you create an order, send `metadata` for any values you want echoed back in the create-order response and webhook payloads.

Examples include:

- sales channel
- internal order number
- tenant ID
- checkout session ID

## Test webhook handling

The API includes a testing endpoint to simulate webhook status transitions during development and QA.

Use the testing tools in the API reference to verify that your webhook consumer:

- accepts the payload
- validates the event source according to your internal controls
- updates the correct merchant order
- handles duplicate deliveries safely

## Delivery model

Your webhook consumer should be idempotent.

A single event may be retried by network infrastructure or replayed in internal testing, so you should process it safely more than once.

## Fallback strategy

Keep the order and tracking endpoints as a fallback for reconciliation and support workflows.

Webhooks should drive your primary state updates, but polling remains useful when you need to recover from missed events.
