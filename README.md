# Multichannel E-commerce Architecture

Comprehensive architecture documentation for a multichannel e-commerce platform integrating multiple sales channels, marketplaces, and supporting systems.

## System Overview

This architecture connects the following components:

- **ERP**: Verto ERP (master for SKU, stock, price, and financial data)
- **PIM**: Akeneo PIM (product information management)
- **Sales Channels**:
  - Magento 2 (main e-commerce platform)
  - Baselinker (marketplace aggregator)
  - AMS (Amazon Seller)
  - POS (Point of Sale systems)
- **Marketplaces**: Allegro, Empik (via Baselinker)
- **Order Management**: OMS (Source of Truth for order status)
- **Warehouse**: WMS (fulfillment and inventory management)
- **Support Systems**:
  - CRM (customer relationship management)
  - Payment Gateway (payment processing)
  - BI/Analytics (business intelligence and reporting)

## Architecture Documentation

### Architecture Diagrams

- [System Architecture Overview](docs/architecture/overview.mmd) - Complete system architecture with all components and data flows
- [Order Lifecycle](docs/architecture/order-lifecycle.mmd) - Detailed order processing flow including partial shipments and returns

### Architecture Decision Records (ADRs)

- [ADR-001: OMS as Source of Truth](docs/architecture/ADR-001-oms-source-of-truth.md) - Why OMS is the single source of truth for order status
- [ADR-002: Payment Flow Design](docs/architecture/ADR-002-payment-flow.md) - Payment Gateway integration strategy

### Process Documentation

- [Order Statuses](docs/architecture/order-statuses.md) - Complete list of order statuses and their triggers
- [Return Flow & Partial Shipments](docs/architecture/return-flow.md) - Return processing and partial shipment scenarios

## Key Architectural Principles

1. **Single Source of Truth**: OMS orchestrates order status across all channels
2. **Separation of Concerns**: Each system handles its core responsibility (ERP for finance, WMS for fulfillment, etc.)
3. **Payment Security**: Payment Gateway integrates directly with sales channels (Magento 2, Baselinker) for PCI DSS compliance
4. **Data Synchronization**: PIM distributes product data from ERP to all channels
5. **Real-time Updates**: Status changes propagate from OMS back to sales channels

## Integration Patterns

### Product Data Flow
```
Verto ERP → Akeneo PIM → Sales Channels (Magento 2, Baselinker, AMS)
```

### Order Flow
```
Sales Channels → OMS → WMS → OMS → Verto ERP
```

### Payment Flow
```
Payment Gateway ↔ Sales Channels → OMS
```

### Returns Flow
```
Sales Channel → OMS → WMS → OMS → Verto ERP → Payment Gateway
```

## Getting Started

To view the Mermaid diagrams:
1. Use a Mermaid-compatible viewer (GitHub, VS Code with Mermaid extension, or https://mermaid.live)
2. Navigate to the [docs/architecture](docs/architecture/) directory
3. Open any `.mmd` file to view the diagrams

## Contributing

When documenting new integrations or architectural changes:
1. Update relevant architecture diagrams
2. Create an ADR if introducing a significant architectural decision
3. Update process documentation as needed
4. Keep diagrams and documentation in sync

## License

Internal documentation for e-commerce architecture.
