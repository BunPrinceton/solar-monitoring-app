# Solar Rebate Reporting - Solution Options

## Current Requirement

**What needs to happen:**
- Submit lifetime kilowatt hours (kWh) to state rebate website
- Frequency: Monthly (between 1st-15th)
- Data source: Tesla app (lifetime total)
- State site: Manual login and entry

**Current process:**
- Check Tesla app for lifetime kWh total
- Log into state rebate website
- Enter date and total kWh number
- Submit

---

## Solution Options

### Option 1: Full Automation Script
**What it does:** Completely hands-off monthly reporting

**How it works:**
- Script runs automatically on 1st of each month
- Fetches lifetime kWh from Tesla API
- Logs into state website (Playwright/Selenium)
- Submits the data automatically
- Sends confirmation email/notification

**Pros:**
- Zero manual effort after setup
- Never forget to submit
- Reliable and consistent

**Cons:**
- Most complex to build (~8-12 hours development)
- Requires server/computer running monthly ($5-15/month or home computer)
- Tesla API access needs auth token refresh
- State website changes could break automation
- Maintenance if either site changes

**Estimated effort:** 8-12 hours development + 1-2 hours setup
**Ongoing cost:** $5-15/month (cloud hosting) or $0 (home computer)

---

### Option 2: Semi-Automated Helper
**What it does:** Fetch Tesla data automatically, manual submission to state site

**How it works:**
- Simple web page or mobile shortcut
- Click button → fetches Tesla lifetime kWh via API
- Displays number prominently
- "Submit to State" button opens state website
- Copy/paste the number manually

**Pros:**
- Much simpler to build (~3-5 hours)
- No server needed (can be static webpage)
- Less fragile - only Tesla API integration
- Still saves time vs opening Tesla app
- Free to host (GitHub Pages, Vercel, etc.)

**Cons:**
- Still requires manual login and submission
- Need to remember to do it monthly
- ~2 minutes of work each month

**Estimated effort:** 3-5 hours development + 1 hour setup
**Ongoing cost:** $0

---

### Option 3: Monthly Reminder (Minimal Tech)
**What it does:** Calendar reminder with streamlined instructions

**How it works:**
- Monthly calendar reminder (1st of month)
- Reminder includes direct links:
  - Link to Tesla app/website
  - Link to state rebate site
- Quick checklist in reminder:
  1. Open Tesla app → lifetime kWh
  2. Open state site → login
  3. Enter date + kWh number
  4. Submit

**Pros:**
- Zero development time
- No maintenance
- No dependencies on APIs or automation
- Always works regardless of website changes
- Completely free

**Cons:**
- Manual process (~2-3 minutes monthly)
- Requires remembering to check reminder
- Most effort each month

**Estimated effort:** 5 minutes to set up
**Ongoing cost:** $0

---

### Option 4: Voice Assistant Shortcut
**What it does:** "Hey Siri/Alexa, check my solar rebate"

**How it works:**
- iOS Shortcut or Alexa Skill
- Fetches Tesla lifetime kWh
- Speaks the number out loud
- Optionally texts it to your phone
- Still manual state site submission

**Pros:**
- Very quick (30 seconds)
- Works from anywhere
- Simple to build on iOS (~2 hours)
- No server needed
- Free

**Cons:**
- Still manual state site login
- Limited to iOS/specific platform
- Tesla API setup still required

**Estimated effort:** 2-3 hours setup
**Ongoing cost:** $0

---

## Recommendation

**For minimal effort now:** Option 3 (Calendar Reminder)
- 5 minutes to set up, works forever
- 2-3 minutes per month to complete
- Zero risk of technical issues

**For best long-term convenience:** Option 2 (Semi-Automated Helper)
- 3-5 hours one-time build
- Reduces monthly time to 1 minute
- No ongoing costs
- Flexible and maintainable

**Only if truly hands-off is critical:** Option 1 (Full Automation)
- Significant upfront investment
- Ongoing maintenance risk
- Probably overkill for 12 submissions/year

---

## Next Steps

1. Discuss which option fits best
2. Determine comfort level with:
   - Initial time investment
   - Ongoing maintenance
   - Monthly manual work acceptable?
3. If automation desired, verify:
   - Tesla account API access
   - State website URL and login method
   - Preferred notification method

---

## Technical Notes

**Tesla API access:**
- Requires Tesla account credentials
- Can use unofficial Tesla API libraries
- Auth tokens expire, need refresh mechanism
- Alternative: Tesla Fleet API (official but requires app registration)

**State website automation:**
- Depends on site structure
- May have CAPTCHA (breaks automation)
- Changes to site can break scripts
- Check their API/data submission options first

**Hosting options (if needed):**
- Free: GitHub Pages, Vercel, Cloudflare Pages
- Paid: Railway ($5/mo), Heroku ($7/mo), DigitalOcean ($4/mo)
- Home: Raspberry Pi, old computer, NAS
