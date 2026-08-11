# Multi-Echelon Supply Chain Optimization

## Progress Log
- Explored Kaggle "Supply Chain Logistics Problem" dataset (7 sheets: OrderList, FreightRates, WhCosts, WhCapacities, ProductsPerPlant, PlantPorts, VmiCustomers)
- Found single destination port in data; identified Customer column (46 unique) as true demand layer
- Merged OrderList (9,215 orders) with FreightRates on Carrier + Origin/Destination Port + Weight bracket - 90.7% match rate (8,361 orders)
- Deduplicating to one rate per order (cheapest matching option) - in progress
