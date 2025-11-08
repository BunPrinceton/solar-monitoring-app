# Simplified Project Scope

> Based on conversation: Once-a-month utility submission app for family use

**Last Updated:** November 8, 2025

---

## Project Summary

**What:** iOS app to simplify monthly solar/electric meter reading submission
**Who:** Dad (primary user), possibly other family members
**When:** Use once a month when readings are due
**Goal:** "Set it and forget it" - reliable, professional, doesn't crash

---

## Confirmed Requirements

### User Profile
- **Device:** iPhone (2018 or newer - iPhone X and later)
- **Usage Frequency:** Once per month
- **Tech Level:** TBD (asking dad)
- **Number of Users:** 1-3 family members

### Your Constraints
- **Have:** Mac (can build iOS app ✓)
- **Time:** 10 hours per week
- **Timeline:** 4-6 weeks is acceptable
- **Quality:** Semi-professional, reliable, polished

### Hardware Situation
- Solar system: Converter/inverter (charges Tesla, feeds grid)
- Utility meter: Likely analog (60+ years old, no smart features)
- Tesla app: Already shows production in real-time
- **NO hardware modification wanted** (warranty concerns)

---

## What We're Building (MVP)

### Phase 1: Core Functionality (Weeks 1-3)

**iOS App:**
- Clean, professional UI (large buttons, clear text)
- Manual entry form for readings
- Submit button (saves to backend)
- Success confirmation
- History view (past submissions)

**Backend:**
- Node.js + Express API
- Google Sheets logging (permanent record)
- JWT authentication
- Deploy on Railway.app (~$5-15/month)

**Utility Submission:**
- TBD based on dad's answers
- Options: Web automation, API, email, etc.

### Phase 2: Polish (Week 4)
- Error handling & retry logic
- Loading states & animations
- Dark mode support
- App icon & launch screen
- Accessibility features

### Phase 3: Deployment (Week 5-6)
- TestFlight for family testing
- Bug fixes based on feedback
- Final polish
- Ready for monthly use

---

## What We're NOT Building (For Now)

### Excluded from MVP:
- ❌ Real-time monitoring (use Tesla app for this)
- ❌ Complex offline sync (monthly use = usually online)
- ❌ ESP32-CAM meter reading (manual is fine)
- ❌ Automated Tesla API integration (may add later)
- ❌ Push notifications / reminders (can add later)
- ❌ Data visualization / charts (can add later)
- ❌ App Store publication (TestFlight is enough)

### Possible Future Enhancements:
- Monthly reminder notifications
- Tesla app integration (if useful)
- Photo capture of meter (for records)
- Export to PDF/CSV
- Multiple user accounts
- Camera OCR (if dad wants it)

---

## Questions for Dad (Critical)

**Sent questionnaire to:** `/mnt/c/Users/benja/Desktop/askdad.txt`

**Need to know:**
1. What exact numbers need to be submitted?
2. How is submission currently done?
3. Who is the electric company?
4. What makes current process annoying?
5. Tesla equipment details
6. Utility meter type and location

**Timeline:** Get answers by end of weekend
**Impact:** Determines exact features and submission method

---

## Technical Decisions

### iOS App
- **Language:** Swift 5.5+
- **UI:** SwiftUI (modern, declarative)
- **Min iOS:** iOS 15 (supports iPhone X/2018+)
- **Auth:** Google OAuth 2.0
- **Networking:** URLSession + async/await
- **Storage:** UserDefaults (simple) + Keychain (auth tokens)
- **No offline queue for MVP** (can add later if needed)

### Backend
- **Runtime:** Node.js 18
- **Framework:** Express.js
- **Database:** Google Sheets API (primary storage)
- **Auth:** JWT tokens (7-day expiration)
- **Hosting:** Railway.app
- **Cost:** $5-15/month

### Utility Submission
- **TBD:** Depends on dad's answers
- **Options:**
  - Direct API (if utility provides)
  - Web automation (Puppeteer)
  - Email submission
  - Manual copy-paste helper

---

## Development Timeline (10h/week)

### Week 1: Backend Foundation
- Setup Node.js project
- Google Sheets integration working
- Basic API endpoints
- Test with cURL
- **Deliverable:** Can save data to Sheets

### Week 2: iOS Skeleton
- Xcode project setup
- Main UI layout (big buttons, clear text)
- Manual entry form with validation
- **Deliverable:** Good-looking UI with mock data

### Week 3: Integration
- Connect iOS to backend
- Google OAuth login
- Submit real data
- History view
- **Deliverable:** Full flow works end-to-end

### Week 4: Polish
- Error handling
- Loading states
- Dark mode
- App icon
- **Deliverable:** Professional quality

### Week 5: Utility Submission
- Implement submission to electric company
- Confirmation emails/receipts
- **Deliverable:** Actually submits to utility

### Week 6: Testing & Deployment
- TestFlight setup
- Family testing
- Bug fixes
- **Deliverable:** Ready for monthly use

**Total: 6 weeks (60 hours)**

---

## Success Criteria

### Functional
- ✓ Can enter meter reading and submit
- ✓ Saves to Google Sheets successfully
- ✓ Submits to electric company
- ✓ Shows confirmation
- ✓ Can view past submissions

### Quality
- ✓ No crashes during normal use
- ✓ Loads in <1 second
- ✓ Smooth animations (feels native)
- ✓ Clear error messages
- ✓ Works on iPhone X and newer
- ✓ Looks professional (dad would show friends)

### Reliability
- ✓ Network errors handled gracefully
- ✓ Invalid input prevented
- ✓ Retry logic if submission fails
- ✓ Receipt/proof of submission

---

## Risks & Mitigations

### Risk 1: Utility Submission Method Unknown
**Impact:** Can't build submission until we know how
**Mitigation:** Get dad's answers this weekend
**Fallback:** Build everything else, add submission in Week 5

### Risk 2: Tesla Integration Complexity
**Impact:** May be difficult to auto-pull production
**Mitigation:** Start with manual entry (good enough)
**Fallback:** Dad already uses Tesla app, can check there

### Risk 3: Learning Curve
**Impact:** First iOS app may take longer
**Mitigation:** Excellent documentation available
**Fallback:** Build web app instead (faster)

---

## Cost Estimate

### One-Time Costs
- Apple Developer Account: $99/year
- ESP32-CAM (if wanted later): $10
- **Total: $99-109**

### Monthly Costs
- Railway hosting: $5-15/month
- Google Sheets: $0 (free tier)
- **Total: $5-15/month**

### Development Time
- 60 hours @ your time
- **Value if hired out:** ~$4,500 ($75/hr)

---

## Next Steps

### This Weekend
1. ✓ Created askdad.txt questionnaire
2. ⏳ Get dad's answers
3. ⏳ Finalize exact scope based on answers

### Week 1 (Starting Monday)
1. Setup development environment
2. Create Google Cloud project
3. Build backend + Google Sheets integration
4. Test with cURL

### Week 2+
- Continue as per timeline above
- Adjust based on dad's feedback

---

## Design Philosophy

**Keep It Simple:**
- Manual entry is fine (reliable, no hardware)
- Once-a-month usage (don't over-engineer)
- Dad-friendly UI (large text, clear actions)

**Make It Professional:**
- Smooth animations
- Proper error handling
- Clear success states
- Feels polished

**Make It Reliable:**
- Network error recovery
- Confirmation receipts
- History/audit trail
- Never lose data

---

## Open Questions (Need Dad's Input)

1. What exact numbers to submit?
2. How is submission done currently?
3. What's the electric company?
4. Is there a monthly deadline?
5. Does Tesla app show what we need?
6. What type of meter (analog/digital)?
7. Who else might use the app?
8. Any other pain points?

**Answers expected:** By end of weekend
**Impact:** Will finalize features and submission method

---

**Status:** Waiting for dad's input to finalize scope
**Next Action:** Get questionnaire answered, then start Week 1 development
**Confidence:** High (simplified scope is very achievable)
