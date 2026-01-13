# Stash Extension - Security Quick Reference

## At a Glance

| Metric | Value |
|--------|-------|
| Total Lines of Code | 1,641 (excluding Readability.js) |
| Third-party Code | Readability.js (Mozilla, 2,314 lines) |
| Network Destinations | 1 (Supabase only) |
| Data Collected | Article text when user clicks Save |
| Analytics/Tracking | None |

---

## Code to Review (by priority)

| File | Lines | What to Check |
|------|-------|---------------|
| `background.js` | 511 | Network requests, message handlers |
| `supabase.js` | 158 | Auth handling, API calls |
| `content.js` | 120 | DOM access, data extraction |
| `kindle-scraper.js` | 178 | Amazon page scraping |
| `popup.js` | 259 | User input handling |
| `manifest.json` | 47 | Permissions |
| `config.js` | 12 | Configuration values |

---

## Permissions Summary

```
activeTab        - Read current tab when user saves
contextMenus     - Add right-click "Save highlight"
storage          - Store auth token locally
tabs             - Open Amazon page for Kindle sync
scripting        - Inject scraper on Amazon only
<all_urls>       - Save from any website
```

**Why `<all_urls>`?** Required to extract article content from any website the user wants to save. Only activated by explicit user action.

---

## Data Flow (Simple)

```
[User clicks Save] → [Extract text from page] → [Send to Supabase] → [Done]
```

**No data leaves the extension except to user's Supabase database.**

---

## What This Extension Does NOT Do

- Track browsing history
- Collect analytics
- Send data to third parties
- Access passwords or forms
- Run code in background without user action
- Store any data except auth token
- Modify web page content

---

## Known Security Considerations

1. **Broad host permission (`<all_urls>`)** - Necessary for saving from any site, but only used on user action

2. **Supabase anon key in source** - This is intentional and safe. Anon keys are designed to be public; security is enforced by Row Level Security in the database

3. **Amazon scraping** - Only reads DOM of notebook page, does not access credentials

---

## Quick Audit Commands

```bash
# Search for network calls
grep -n "fetch\|XMLHttpRequest" *.js

# Search for eval (should find none)
grep -n "eval\|Function(" *.js

# Search for external URLs
grep -n "http" *.js | grep -v supabase

# Search for storage access
grep -n "chrome.storage\|localStorage" *.js
```

---

## Approval Checklist

- [ ] Reviewed manifest.json permissions
- [ ] Reviewed network requests in background.js
- [ ] Confirmed no analytics/tracking code
- [ ] Confirmed no external dependencies loaded at runtime
- [ ] Confirmed data only sent to Supabase
- [ ] Reviewed content script DOM access
