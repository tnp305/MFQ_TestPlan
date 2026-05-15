# Strategy Note

## Priorities
1. Core functional stability (order lifecycle + inventory sync) to ensure business continuity.
2. Inventory synchronization with WMS/SC2P to ensure data consistency(critical to avoid overselling/underselling).
3. High-concurrency performance (flash sale scenario) to support business peak.
4. Reliability and recoverability (minimize downtime and data loss).

## Trade-offs
1. Focus on P0/P1 core scenarios; simplify low-frequency edge cases.
2. Use automation for stable core flows; manual testing for complex exceptions.
3. Balance performance scope and test duration; focus on order & inventory.
4. Accept eventual consistency between storefront and WMS/SC2P, with reconciliation to ensure final consistency.