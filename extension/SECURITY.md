# Stash Chrome Extension - Security Review Document

**Prepared for:** NYT Security Team
**Version:** 1.0.0
**Date:** January 2025

---

## Executive Summary

Stash is a personal article/bookmark manager Chrome extension that allows users to save web pages and text highlights to a private Supabase database. The extension also includes optional Kindle highlight syncing from Amazon's read.amazon.com.

**Key Security Points:**
- No data is sent to third parties (only to user's own Supabase instance)
- No browsing history is collected or transmitted
- No tracking or analytics
- All network requests are to user-configured Supabase endpoints
- Open source - all code is auditable

---

## Files Overview

| File | Purpose | Security Relevance |
|------|---------|-------------------|
| `manifest.json` | Extension configuration | Defines permissions |
| `background.js` | Service worker | Handles saves, API calls |
| `content.js` | Page content extraction | Runs on web pages |
| `kindle-scraper.js` | Amazon page scraping | Only runs on read.amazon.com |
| `popup.js` | Extension popup UI | User interface |
| `supabase.js` | Database client | API communication |
| `config.js` | Configuration | Supabase URL/key |
| `Readability.js` | Mozilla's article parser | Content extraction (MIT licensed) |

---

## Permissions Requested

### manifest.json permissions:

```json
{
  "permissions": [
    "activeTab",
    "contextMenus",
    "storage",
    "tabs",
    "scripting"
  ],
  "host_permissions": [
    "<all_urls>",
    "https://read.amazon.com/*"
  ]
}
```

### Permission Justifications:

| Permission | Why Needed | Data Accessed |
|------------|-----------|---------------|
| `activeTab` | Extract article content from current tab when user clicks "Save" | Page title, content, metadata |
| `contextMenus` | Right-click menu to save highlighted text | Selected text only |
| `storage` | Store authentication session locally | Auth token (encrypted by Chrome) |
| `tabs` | Open Amazon notebook page for Kindle sync | Tab URL only |
| `scripting` | Inject content script for Kindle scraping | Only on read.amazon.com |
| `<all_urls>` | Save articles from any website | Only when user initiates save |
| `read.amazon.com` | Kindle highlight sync feature | Highlight text, book titles |

---

## Data Flow

### 1. Saving an Article

```
User clicks "Save"
    → content.js extracts article text via Readability.js
    → background.js receives extracted content
    → background.js sends POST to Supabase REST API
    → Data stored in user's Supabase database
```

**Data transmitted:**
- Page URL
- Page title
- Article content (text only, no scripts/styles)
- Author, publish date (if available)
- Featured image URL (if available)

**Data NOT transmitted:**
- Cookies
- Form data
- Passwords
- Local storage from other sites
- Browsing history

### 2. Saving a Highlight

```
User selects text → Right-clicks → "Save highlight to Stash"
    → background.js captures selected text
    → POST to Supabase with highlight text + source URL
```

### 3. Kindle Sync (Optional)

```
User clicks "Sync Kindle"
    → Extension opens read.amazon.com/notebook
    → kindle-scraper.js extracts visible highlights from DOM
    → Highlights sent to Supabase
```

**Important:** This only works if user is already logged into Amazon. The extension does NOT:
- Access Amazon credentials
- Store Amazon cookies
- Access any Amazon data outside the notebook page

---

## Authentication

The extension uses Supabase Auth:

1. User signs in with email/password in extension popup
2. Supabase returns a JWT access token
3. Token stored in `chrome.storage.local` (encrypted by Chrome)
4. Token included in Authorization header for API requests
5. Token refreshed automatically by Supabase client

**The extension never:**
- Stores passwords
- Transmits credentials to any server except Supabase auth endpoint
- Has access to credentials for other services

---

## Network Requests

All network requests go to a single destination:

**Endpoint:** `https://[project-id].supabase.co`

| Request Type | Endpoint | Purpose |
|-------------|----------|---------|
| POST | `/auth/v1/token` | User login |
| GET | `/auth/v1/user` | Get current user |
| POST | `/rest/v1/saves` | Save article/highlight |
| GET | `/rest/v1/saves` | Load recent saves |

**No requests are made to:**
- Analytics services
- Ad networks
- Third-party APIs
- Any domain other than configured Supabase instance

---

## Content Script Behavior

### content.js (runs on all pages)

**Triggers:** Only when user initiates a save action
**Actions:**
1. Clones DOM (does not modify original page)
2. Runs Readability.js to extract article text
3. Returns extracted text to background script

**Does NOT:**
- Run automatically on page load
- Access cookies or localStorage
- Modify page content
- Inject tracking scripts
- Monitor user behavior

### kindle-scraper.js (runs only on read.amazon.com/notebook)

**Triggers:** Only when user clicks "Sync Kindle"
**Actions:**
1. Queries DOM for highlight elements
2. Extracts text content from visible highlights
3. Returns highlight array to background script

---

## Local Storage

Data stored in `chrome.storage.local`:

| Key | Contents | Purpose |
|-----|----------|---------|
| `stash_session` | JWT token, refresh token, expiry | Authentication |

**No other data is stored locally.**

---

## Third-Party Code

### Readability.js
- **Source:** Mozilla (https://github.com/mozilla/readability)
- **License:** Apache 2.0
- **Purpose:** Article content extraction
- **Security:** Widely used, maintained by Mozilla, no network access

---

## Configuration

`config.js` contains:
```javascript
const CONFIG = {
  SUPABASE_URL: 'https://[project-id].supabase.co',
  SUPABASE_ANON_KEY: '[anon-key]',
};
```

**Note:** The `SUPABASE_ANON_KEY` is a **public client key** (similar to a Firebase API key). It is designed to be exposed in client-side code. Security is enforced by:
1. Row Level Security (RLS) policies in Supabase
2. JWT authentication for data access
3. Users can only access their own data

---

## Security Considerations

### What the extension CAN do:
- Read page content when user clicks "Save"
- Store data in user's Supabase database
- Read highlights from Amazon notebook (when user initiates)

### What the extension CANNOT do:
- Access data without user action
- Read passwords or form inputs
- Access other extensions' data
- Modify page content
- Execute remote code
- Track browsing history
- Send data to third parties

---

## Source Code Audit Checklist

- [ ] `manifest.json` - Review permissions
- [ ] `background.js` - Review message handlers, network requests
- [ ] `content.js` - Review DOM access, data extraction
- [ ] `kindle-scraper.js` - Review Amazon page scraping
- [ ] `supabase.js` - Review API client, auth handling
- [ ] `popup.js` - Review UI interactions
- [ ] `config.js` - Review configuration (anon key is intentionally public)

---

## Contact

For questions about this security review:
- Repository: [internal repo location]
- Developer: Kevin Roose

---

## Appendix: Full Source Code

All source files are available in the `extension/` directory. The extension contains approximately 1,500 lines of JavaScript (excluding Readability.js which is 2,200 lines of Mozilla-maintained code).
