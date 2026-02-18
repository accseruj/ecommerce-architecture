# ADR-002: Payment Flow Design

**Status**: Accepted

**Date**: 2026-02-18

## Context

We need to decide how the Payment Gateway integrates with our multichannel e-commerce architecture. The key question is: Should the Payment Gateway communicate directly with OMS, or should it communicate through the sales channels (Magento 2, Baselinker)?

## Decision

**Payment Gateway communicates with sales channels (Magento 2 and Baselinker) directly, NOT with OMS. OMS receives payment status from sales channels as part of order data.**

## Rationale

### Technical Considerations

1. **Payment Module Integration**: Payment Gateways (Stripe, PayU, Przelewy24, etc.) integrate with Magento via payment modules. These modules are designed to:
   - Register payment callbacks/webhooks with Magento URLs
   - Handle payment redirects back to Magento
   - Validate payment signatures using Magento's security context

2. **Callback URLs**: Payment Gateway callbacks are configured to point to the sales channel's domain (e.g., `https://shop.example.com/payment/callback`), not to OMS.

### User Experience

- Customer must immediately see payment result on the Magento storefront
- Payment confirmation page, order confirmation, and thank-you page are all part of the Magento UX flow
- Any delay in payment status would break the customer experience

### Security (PCI DSS Compliance)

- Payment data should pass through the minimal number of systems
- Magento (as the customer-facing application) is already PCI DSS compliant
- Adding OMS as a payment intermediary would:
  - Extend the PCI compliance scope
  - Introduce an additional attack surface
  - Require OMS to handle sensitive payment data

### Business Logic

- In Magento, an order is created **only AFTER** successful payment
- If payment fails, no order is created, and OMS receives nothing
- This prevents OMS from dealing with "pending payment" orders that may never complete

## Payment Flows for Different Scenarios

### 1. Online Payment (Card, BLIK, PayPal)

```
1. Customer initiates payment on Magento
2. Magento → Payment Gateway (payment request)
3. Payment Gateway → Customer (payment page/redirect)
4. Customer completes payment
5. Payment Gateway → Magento (callback with payment status)
6. Magento creates order (if payment successful)
7. Magento → OMS (order with payment_status: "paid")
```

### 2. Cash on Delivery (COD)

```
1. Customer selects COD on Magento
2. Magento creates order with payment_status: "cod_pending"
3. Magento → OMS (order with COD flag)
4. OMS → WMS (fulfillment task)
5. Courier collects money from customer
6. Courier → WMS (cash collected confirmation)
7. WMS → OMS (payment confirmed)
8. OMS → Verto ERP (financial record)
```

### 3. Installment/Deferred Payment (Buy Now Pay Later)

```
1. Customer initiates installment payment
2. Magento → Payment Gateway (installment request)
3. Payment Gateway approves (first installment)
4. Magento creates order with payment_status: "installment_pending"
5. Magento → OMS (order with installment details)
6. Payment Gateway → Magento (installment confirmations over time)
7. Magento → OMS (payment status updates)
8. When fully paid: OMS updates status to "fully_paid"
```

### 4. Marketplace Orders (via Baselinker)

```
1. Customer pays on Allegro/Empik marketplace
2. Marketplace holds payment
3. Marketplace → Baselinker (order with payment_held)
4. Baselinker → OMS (order with marketplace_payment flag)
5. After shipment confirmation:
6. OMS → Baselinker (shipped status)
7. Baselinker → Marketplace (shipped)
8. Marketplace → Baselinker (payment released)
9. Marketplace → Payment Gateway (payout)
10. Payment Gateway → Verto ERP (reconciliation)
```

## Consequences

### Positive

- **No direct OMS-Payment Gateway integration needed**: Reduces integration complexity
- **PCI compliance scope limited**: Only sales channels handle payment data
- **Immediate payment feedback**: Customer sees payment result instantly
- **Standard payment flow**: Uses proven e-commerce patterns

### Negative

- **Payment status comes indirectly**: OMS learns about payment status from sales channels, not directly from source
- **Channel dependency**: If Magento is down, payment processing stops (but this is unavoidable)
- **Reconciliation complexity**: Payment Gateway reconciliation data must be matched with orders via Verto ERP

### Neutral

- **Payment Gateway → Verto ERP for reconciliation**: Financial data still flows to ERP for accounting
- **OMS trusts sales channels**: OMS assumes payment status from channels is accurate

## Alternative Considered: Direct Payment Gateway → OMS Integration

**Why rejected:**

1. Would require duplicating payment module logic in OMS
2. Customer wouldn't see immediate payment confirmation
3. Would expand PCI compliance scope
4. Would complicate order creation flow (order before or after payment?)
5. Payment Gateway webhooks are designed for e-commerce platforms, not OMS systems

## Implementation Notes

- Magento payment modules must be configured to send order data to OMS after successful payment
- OMS should accept payment status from trusted channels only (via API keys/signatures)
- For COD, OMS must handle the "pending payment" state until courier confirms
- For installments, OMS must track partial payments and update status accordingly

## Related Decisions

- [ADR-001: OMS as Source of Truth](ADR-001-oms-source-of-truth.md) - Explains OMS role in order orchestration

## References

- System Architecture: [overview.mmd](overview.mmd)
- Order Statuses: [order-statuses.md](order-statuses.md)
