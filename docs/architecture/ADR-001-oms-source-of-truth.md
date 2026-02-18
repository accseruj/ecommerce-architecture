# ADR-001: OMS as Source of Truth for Orders

**Status**: Accepted

**Date**: 2026-02-18

## Context

We operate a multichannel e-commerce platform with orders coming from multiple sources:
- Magento 2 (main e-commerce platform)
- Baselinker (marketplace aggregator for Allegro, Empik)
- AMS (Amazon Seller)
- POS (Point of Sale systems)

Each channel has its own order management capabilities, but we need a single system to orchestrate order status across all channels and ensure consistency.

## Decision

**OMS (Order Management System) is designated as the single source of truth for order status orchestration.**

However, "source of truth" is not absolute - different aspects of an order have different authoritative systems:

### Source of Truth Breakdown by Aspect

| Aspect | Source of Truth | Why |
|--------|----------------|-----|
| Order composition (items, qty, price) | OMS | OMS stores canonical order after intake from channels |
| Payment status | Payment Gateway | Only the payment gateway knows real transaction status |
| Fulfillment status (picked, shipped, tracking) | WMS | Warehouse is the only one knowing physical state |
| Financial data (invoices, accounting, refunds) | Verto ERP | ERP is the master for accounting |
| Customer-facing display | Magento 2 | But this is a mirror, not the source |
| Aggregated order status | ⭐ OMS | OMS collects statuses from all systems and orchestrates |

## Rationale

1. **Centralized Orchestration**: Without a central orchestrator, each channel would need to integrate with all other systems (WMS, ERP, Payment Gateway), creating N×M integration complexity.

2. **Consistency**: OMS ensures all channels display the same order status by being the single point of status aggregation and distribution.

3. **Business Logic**: Complex order processing rules (partial shipments, split orders, returns) are handled once in OMS rather than duplicated across channels.

4. **Audit Trail**: Single system maintains complete order lifecycle history across all channels.

5. **Scalability**: Adding new sales channels requires integration with OMS only, not with every other system.

## Implementation

- All sales channels (Magento 2, Baselinker, AMS, POS) send orders to OMS
- OMS receives status updates from:
  - Payment Gateway (via sales channels)
  - WMS (fulfillment status)
  - Verto ERP (financial status)
  - Tracking APIs (delivery status)
- OMS pushes aggregated status back to sales channels
- Sales channels display status to customers but don't modify it independently

## Consequences

### Positive
- Single integration point for new channels
- Consistent order status across all customer touchpoints
- Simplified business logic maintenance
- Complete order lifecycle visibility
- Easier compliance and auditing

### Negative
- OMS becomes a critical dependency - if OMS is down, order processing stops
- Initial setup requires more complex OMS implementation
- Status updates have slight delay due to synchronization

### Risks & Mitigation
- **Risk**: OMS downtime impacts all channels
  - **Mitigation**: High availability setup, order queuing in channels
- **Risk**: Status synchronization delays
  - **Mitigation**: Webhook-based real-time updates, status caching in channels

## Related Decisions
- [ADR-002: Payment Flow Design](ADR-002-payment-flow.md) - Explains why payment status comes through channels

## References
- Order Status Documentation: [order-statuses.md](order-statuses.md)
- Order Lifecycle: [order-lifecycle.mmd](order-lifecycle.mmd)
