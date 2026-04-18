# Wave 3.IVW: Surge 3 Integration Verification Wave

### Scope
Verify all Surge 3 Flows work with Surges 1+2 as complete system.

### User Flows to Verify

1. **Full Order Lifecycle:** Place order → confirmed → preparing → walking → delivered → rating prompt → submit rating
2. **Push Notifications:** Verify notifications on each status change on real device
3. **Merchant Analytics:** Place 5 orders to 2 orgs → verify dashboard accuracy after aggregation
4. **Walking ETA:** Verify ETA matches restaurant-to-org distance
5. **State Machine:** Attempt invalid transitions (e.g., confirmed → delivered); verify 422
6. **Regression: Individual + Group Orders:** Both order types work end-to-end
7. **Regression: Discounts:** Group discount applies through full lifecycle

### Acceptance Criteria

All previous IVW criteria plus:
- Order status state machine enforces all transitions (zero invalid transitions)
- Push notifications delivered on Android Chrome + iOS Safari
- Analytics data matches actual order records (spot-check 5 orders)
- Rating prompt appears exactly 5 minutes after delivery confirmation
- Complete regression of Surge 1 + Surge 2 flows passes

**Owner:** AO (execution) + QO (validation)
**Failure Protocol:** Failed items become fix Waves before production release.

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

