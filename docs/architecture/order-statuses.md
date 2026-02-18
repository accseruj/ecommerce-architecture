# Order Statuses in OMS

This document defines all order statuses in the Order Management System (OMS) and describes their triggers and ownership.

## Status Lifecycle Overview

Orders in OMS progress through various statuses based on events from sales channels, payment systems, warehouse, and delivery tracking. The OMS is the central orchestrator, but status updates come from different systems.

## Complete Status List

| Status | Trigger | Updated by |
|--------|---------|------------|
| `pending_payment` | Order created, waiting for payment | Magento 2 |
| `paid` | Payment callback received | Magento 2 → OMS |
| `processing` | Order accepted in OMS | OMS |
| `picking` | WMS started picking | WMS → OMS |
| `partially_shipped` | Some items shipped | WMS → OMS |
| `shipped` | All items shipped | WMS → OMS |
| `delivered` | Courier delivered | Tracking API → OMS |
| `return_requested` | Customer requested return | Magento / BL → OMS |
| `return_in_transit` | Item in transit to warehouse | Tracking → OMS |
| `return_received` | WMS accepted return | WMS → OMS |
| `refunded` | Money returned | Verto → Pay → OMS |
| `cancelled` | Order cancelled | OMS |
| `closed` | Order finalized | OMS |

## Detailed Status Descriptions

### `pending_payment`
- **Trigger**: Order is created in Magento but payment is not yet confirmed
- **Used for**: Bank transfer, pending online payments
- **Next status**: `paid` (success) or `cancelled` (timeout/failure)
- **Owner**: Magento 2 creates order with this status

### `paid`
- **Trigger**: Payment Gateway confirms successful payment
- **Flow**: Payment Gateway → Magento 2 → OMS
- **Next status**: `processing`
- **Note**: For COD orders, this status is set when courier confirms cash collection

### `processing`
- **Trigger**: OMS accepts the order and prepares it for fulfillment
- **Actions**: OMS validates order, allocates inventory, creates fulfillment task
- **Next status**: `picking` or `cancelled` (if issues found)
- **Owner**: OMS

### `picking`
- **Trigger**: WMS starts picking items from warehouse
- **Flow**: WMS → OMS
- **Next status**: `partially_shipped`, `shipped`, or `cancelled` (if items unavailable)
- **Note**: Indicates active warehouse work

### `partially_shipped`
- **Trigger**: WMS ships some items but not all
- **Use cases**:
  - Split shipment due to partial stock availability
  - Large orders shipped in multiple packages
  - Backordered items shipped separately
- **Flow**: WMS → OMS (with shipment details)
- **Next status**: `shipped` (when remaining items ship) or `cancelled` (remaining items)
- **Owner**: WMS reports partial fulfillment

### `shipped`
- **Trigger**: All order items have been shipped
- **Flow**: WMS → OMS (with tracking numbers)
- **Next status**: `delivered` or `return_requested`
- **Customer notification**: Tracking information sent

### `delivered`
- **Trigger**: Courier confirms delivery
- **Flow**: Tracking API → OMS
- **Next status**: `return_requested` or `closed`
- **Note**: Some implementations auto-close after delivery + X days

### `return_requested`
- **Trigger**: Customer initiates return through sales channel
- **Flow**: Magento 2 / Baselinker → OMS
- **Next status**: `return_in_transit` or `cancelled` (return denied)
- **Actions**: OMS creates return task for WMS

### `return_in_transit`
- **Trigger**: Customer sends item back, tracking shows in transit
- **Flow**: Tracking API → OMS
- **Next status**: `return_received`
- **Note**: Reverse logistics tracking

### `return_received`
- **Trigger**: WMS receives and inspects returned item
- **Flow**: WMS → OMS (with item condition)
- **Next status**: `refunded` or `closed` (if refund denied)
- **Actions**: OMS initiates refund process

### `refunded`
- **Trigger**: Money returned to customer
- **Flow**: Verto ERP → Payment Gateway → OMS
- **Next status**: `closed`
- **Note**: Refund may be full or partial

### `cancelled`
- **Trigger**: Order cancelled by customer, admin, or system
- **Reasons**: Payment failed, items unavailable, customer request, fraud detection
- **Owner**: OMS (can be triggered by any system)
- **Next status**: `closed` (final state)
- **Actions**: Release inventory, cancel fulfillment, process refund if paid

### `closed`
- **Trigger**: Order lifecycle complete
- **Conditions**: 
  - Delivered + no return window
  - Refunded + no further actions
  - Cancelled + cleanup complete
- **Owner**: OMS
- **Note**: Final state, no further status changes

## Status Transition Rules

### Normal Flow
```
pending_payment → paid → processing → picking → shipped → delivered → closed
```

### Partial Shipment Flow
```
picking → partially_shipped → shipped → delivered → closed
```

### Return Flow
```
delivered → return_requested → return_in_transit → return_received → refunded → closed
```

### Cancellation Flow
```
[any status] → cancelled → closed
```

## Status Categories

For reporting and filtering, statuses can be grouped:

- **Pre-fulfillment**: `pending_payment`, `paid`, `processing`
- **In-fulfillment**: `picking`, `partially_shipped`, `shipped`
- **Post-fulfillment**: `delivered`
- **Returns**: `return_requested`, `return_in_transit`, `return_received`, `refunded`
- **Terminal**: `cancelled`, `closed`

## Integration Notes

### For Sales Channels (Magento 2, Baselinker, AMS)
- Push: `pending_payment`, `paid`, `return_requested`
- Pull: Subscribe to all status updates from OMS
- Display: Show OMS status to customers

### For WMS
- Push: `picking`, `partially_shipped`, `shipped`, `return_received`
- Pull: Receive fulfillment tasks from OMS
- Track: Shipment details with each status update

### For Verto ERP
- Push: `refunded` (after processing refund)
- Pull: `paid`, `shipped`, `cancelled` for financial records
- Generate: Invoices, credit memos

### For Payment Gateway
- Push: `paid` (via Magento/Baselinker), `refunded`
- Process: Payments and refunds

## See Also

- [OMS as Source of Truth](ADR-001-oms-source-of-truth.md) - Why OMS orchestrates status
- [Return Flow](return-flow.md) - Detailed return and partial shipment scenarios
- [Order Lifecycle](order-lifecycle.mmd) - Visual order flow diagram
