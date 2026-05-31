# OpsOlive Landing Page — Implementation Plan

## Decisions

- **Hosting:** Plain HTML/CSS + GitHub Pages (custom domain supported via CNAME + DNS)
- **Email collection:** Google Sheets via Apps Script (free, unlimited submissions)
- **Form UX:** Inline JS — success message without page redirect
- **Design:** Port OpsOlive rebrand tokens (sage olive + warm coral + warm cream)
- **Repo:** Separate GitHub repo `opsolive-site`, not part of the hoprs monorepo

---

## [DONE] Step 1 — Create repo and file structure

Create a new GitHub repo `opsolive-site` with this structure:

```
opsolive-site/
├── index.html
├── css/
│   └── style.css
├── assets/
│   ├── opsolive-mark.svg      (from rebrand bundle)
│   ├── favicon.svg             (copy of mark, or sized variant)
│   └── og-image.png            (create for social sharing — 1200x630)
├── CNAME                       (custom domain, e.g. opsolive.com)
└── README.md
```

**Acceptance:** Repo exists on GitHub, `git clone` works, folder structure in place.

---

## [DONE] Step 2 — Set up Google Sheet + Apps Script endpoint

### 2a. Create the Google Sheet

1. Create a new Google Sheet named "OpsOlive — Waitlist Signups"
2. Add headers in Row 1: `Timestamp | Email`

### 2b. Create the Apps Script web app

1. In the Sheet → Extensions → Apps Script
2. Replace `Code.gs` with:

```javascript
function doPost(e) {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  var data = JSON.parse(e.postData.contents);
  sheet.appendRow([new Date(), data.email]);
  return ContentService
    .createTextOutput(JSON.stringify({ result: "success" }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

3. Deploy → New deployment → Web app
   - Execute as: **Me**
   - Who has access: **Anyone**
4. Copy the deployment URL (looks like `https://script.google.com/macros/s/.../exec`)

### 2c. Test the endpoint

```bash
curl -L -X POST \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com"}' \
  "YOUR_DEPLOYMENT_URL"
```

Verify the row appears in the Sheet, then delete the test row.

**Acceptance:** POST to the endpoint appends a row with timestamp + email. CORS works from browser origin.

---

## [DONE] Step 3 — Build `style.css` with OpsOlive design tokens

Port tokens directly from the rebrand handoff (`colors_and_type.css` values). The CSS file should contain:

### 3a. Custom properties (`:root`)

```
Brand scale:     --brand-50 through --brand-900 (sage olive, hue 142)
Accent scale:    --accent-50 through --accent-900 (warm coral, hue 35)
Surfaces:        --bg-1 through --bg-4, --bg-inv
Foregrounds:     --fg-1, --fg-2, --fg-3, --fg-inv
Lines:           --line-1, --line-2, --line-3
Shadows:         --shadow-xs through --shadow-xl (olive-ink hue 135)
Focus ring:      --focus-ring (olive)
Typography:      --font-sans (Plus Jakarta Sans), --font-mono (JetBrains Mono)
Type scale:      --text-xs through --text-2xl
Spacing:         --sp-1 through --sp-12
Radii:           --radius-sm, --radius-md, --radius-lg, --radius-xl
```

### 3b. Base styles

- `body`: bg-1 background, fg-1 text, font-sans, antialiased
- Responsive container: `max-width: 720px`, centered, padding
- Headings: font-display weight 800, fg-1
- Links: brand-500, hover brand-600

### 3c. Component styles

- `.brand-link` — wordmark: "Ops" in fg-1, "Olive" in brand-700, weight 800, letter-spacing -0.04em
- `.hero` — centered text, large heading, subheading in fg-2
- `.form` — inline email input + submit button
- `.form input` — bg-1, line-2 border, radius-md, focus ring
- `.form button` — brand-500 bg, white text, radius-md, hover brand-600, press brand-700
- `.features` — 2-3 column grid (responsive), each feature card as surface with shadow-sm
- `.footer` — fg-3 text, small, centered
- `.success-msg` — hidden by default, shown after submit, brand-soft bg, brand-700 text

### 3d. Responsive

- Single breakpoint at ~640px: stack features to 1 column, form input + button stack vertically
- No horizontal scroll on mobile

**Acceptance:** Page renders with correct olive/coral/cream palette, matches the hoprs app's visual feel. Plus Jakarta Sans loads from Google Fonts.

---

## [DONE] Step 4 — Build `index.html`

### 4a. `<head>`

- Meta charset, viewport
- Page title: "OpsOlive — The oil that keeps your operations running."
- Meta description: "OpsOlive gives ops teams the tools to manage data, run workflows, and automate the rest — without touching code."
- OG tags (title, description, image, url)
- Favicon: `<link rel="icon" href="assets/favicon.svg" type="image/svg+xml">`
- Google Fonts: Plus Jakarta Sans (400, 600, 800)
- Link to `css/style.css`

### 4b. Page structure

```html
<body>
  <header>
    <!-- Logo mark (inline SVG) + OpsOlive wordmark -->
  </header>

  <main>
    <section class="hero">
      <!-- Headline: value proposition -->
      <!-- Subheadline: one-sentence explanation -->

      <form id="waitlist-form" class="form">
        <input type="email" name="email" placeholder="you@company.com" required>
        <button type="submit">Join the waiting list</button>
      </form>
      <p id="success-msg" class="success-msg" hidden>
        You're on the list. We'll be in touch.
      </p>
    </section>

    <section class="features">
      <!-- 2-3 feature cards with icon + title + short description -->
      <!-- Content TBD — needs your input on messaging -->
    </section>
  </main>

  <footer>
    <!-- Copyright + optional links -->
  </footer>

  <script>
    // Inline form handler (see Step 5)
  </script>
</body>
```

**Acceptance:** Semantic HTML, accessible form with labels, no external JS dependencies.

---

## [DONE] Step 5 — Inline form submission script

At the bottom of `index.html`, a small inline `<script>`:

```javascript
document.getElementById('waitlist-form').addEventListener('submit', async (e) => {
  e.preventDefault();
  const form = e.target;
  const btn = form.querySelector('button');
  const email = form.querySelector('input[name="email"]').value;

  btn.disabled = true;
  btn.textContent = 'Sending...';

  try {
    await fetch('APPS_SCRIPT_URL', {
      method: 'POST',
      body: JSON.stringify({ email }),
    });
    form.hidden = true;
    document.getElementById('success-msg').hidden = false;
  } catch {
    btn.disabled = false;
    btn.textContent = 'Get early access';
    alert('Something went wrong. Please try again.');
  }
});
```

Key behaviors:
- Prevents default form submission (no redirect)
- Disables button + shows "Sending..." during request
- On success: hides form, shows success message
- On failure: re-enables button, shows alert
- No external dependencies — vanilla JS, ~20 lines

**Acceptance:** Submit → success message appears inline. Email + timestamp appear in Google Sheet. No page navigation.

---

## [DONE] Step 6 — Deploy to GitHub Pages

1. Push the repo to GitHub
2. Settings → Pages → Source: "Deploy from a branch" → `main` / `/ (root)`
3. Site goes live at `https://<username>.github.io/opsolive-site/`

**Acceptance:** Site loads at the GitHub Pages URL, form submits successfully.

---

## [DONE] Step 7 — Configure custom domain

1. Add a `CNAME` file to repo root containing the bare domain (e.g. `opsolive.com`)
2. At your DNS provider, add:
   - **A records** pointing to GitHub Pages IPs:
     ```
     185.199.108.153
     185.199.109.153
     185.199.110.153
     185.199.111.153
     ```
   - **CNAME record** for `www` → `<username>.github.io`
3. In GitHub repo Settings → Pages → Custom domain: enter your domain
4. Check "Enforce HTTPS" (wait for certificate provisioning, usually ~15 min)

**Acceptance:** `https://opsolive.com` serves the landing page with valid HTTPS.

---

## Step 8 — Create OG image

Create a 1200x630 PNG for social sharing:
- Warm cream background (bg-2)
- OpsOlive logo mark centered
- Wordmark below
- Tagline text in fg-2

Can be created with any image tool or generated as an HTML page screenshotted at 1200x630.

**Acceptance:** Sharing the URL on Slack/Twitter/LinkedIn shows the branded card.

---

## Content (resolved)

### Hero headline
"The oil that keeps your operations running."

### Hero subheadline
"Stop wrangling spreadsheets. OpsOlive gives ops teams the tools they need to manage data, run workflows, and automate the rest — without touching code."

### Feature cards

| # | Title | Body |
|---|---|---|
| 1 | Single source of truth | All your operational data in one place, always up to date — accessible to everyone who needs it, from any tool you build. |
| 2 | Run your processes, not your inbox | Turn the workflows your team describes out loud into real tools — without asking a developer to translate. |
| 3 | Automate what shouldn't need you | The repetitive tasks that eat your week? Let OpsOlive handle them. You handle the decisions. |

### Domain
`opsolive.com` (already registered)

### CTA button text
"Join the waiting list"

---

## What this does NOT include (future scope)

- Dark mode (can be added later with the dark tokens from the rebrand handoff)
- Blog / multiple pages (would warrant switching to Astro or Jekyll)
- Analytics (add Plausible or Fathom later if needed — both have free tiers)
- Cookie banner (not needed if no tracking)
