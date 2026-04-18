# Wave 1.IVW: Surge 1 Integration Verification Wave

### Scope
Verify all Surge 1 Flows work together as an integrated system.

### User Flows to Verify

1. **Full Onboarding → QR → Marketplace:** Create org (all 3 channels) → approve → QR generated → scan QR → marketplace loads with nearby restaurants
2. **Marketplace Browse → Individual Order → POS:** Scan QR → browse → menu → add items → checkout → pay (cash) → order in POS
3. **Card Payment Flow:** Same as #2 with card payment → redirect → callback → confirmation
4. **Restaurant Opt-In Effect:** Merchant enables walking delivery → appears in marketplace; disables → disappears
5. **User Auth + Org Linking:** Guest checkout → individual order; OTP auth → org linked → marketplace shows org context
6. **Edge: Org Deactivation:** Deactivate org → QR shows inactive → no new orders

### Acceptance Criteria (Fixed)

1. Every end-to-end user flow runs successfully on a real mobile device browser
2. Every API call hits a real backend endpoint (zero mocks)
3. Every button, form, and navigation action wired to real handlers
4. Every route resolves correctly (zero 404s)
5. Cross-Wave data flows work end-to-end (org → QR → scan → marketplace → order → POS)
6. All SDK/library features used are native capabilities
7. Runtime execution succeeds on iOS Safari + Android Chrome

**Owner:** AO (execution) + QO (validation)
**Failure Protocol:** Failed items become fix Waves before Surge 2 begins.

---

<a name="surge-2"></a>

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

