# Wave 2.IVW: Surge 2 Integration Verification Wave

### Scope
Verify all Surge 2 Flows work with Surge 1 as integrated system.

### User Flows to Verify

1. **Full Group Order Lifecycle:** Create → share → 3 join → add items → close → discount applied → organizer pays → POS
2. **Individual Pay Flow:** Same as #1 with individual pay → each pays → partial timeout
3. **Discount Tiers Live:** 3 participants (no discount) → add 2 (10% activates) → add 5 (15% activates) → remove 1 (back to 10%)
4. **Savings Messaging:** Complete discounted order → confirmation shows savings
5. **Regression: Individual Orders Still Work:** Full Surge 1 flow unaffected

### Acceptance Criteria

All Surge 1 IVW criteria plus:
- WebSocket real-time updates verified across 3+ concurrent browser sessions
- Discount calculation verified for 5 different participant counts
- Both payment modes produce correct POS orders

---

<a name="surge-3"></a>

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

