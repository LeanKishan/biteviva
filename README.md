# BiteViva — AI nutrition from a photo

**Live app: [biteviva.vercel.app](https://biteviva.vercel.app)**

BiteViva is a full-stack web app that turns a photo of a meal into calories, protein, carbs and fat in a few seconds, then helps people plan, log and improve what they eat. I designed and built it end to end on my own — product, interface, backend, database, AI integration, deployment and ongoing maintenance.

> **The source code for this project is private.** This repository documents what I built and how. I'm happy to walk through the code in an interview — reach out on [LinkedIn](https://www.linkedin.com/in/kishanpatel-isu/).

---

## At a glance

| | |
|---|---|
| **Role** | Sole designer and developer |
| **Type** | Full-stack web app + installable mobile app (PWA) |
| **Stack** | Next.js 16 (App Router) · React 19 · TypeScript · Tailwind CSS · Postgres · Drizzle ORM · Auth.js · Google Gemini |
| **Hosting** | Vercel (app) · Neon (serverless Postgres) |
| **Size** | ~11,000 lines of TypeScript · 15 pages · 18 API routes · 50+ components · 28 library modules |
| **Running cost** | $0/month — the whole product runs on free tiers |

---

## The problem

Tracking food is tedious. Most apps make you search a database item by item and guess portion sizes, so people give up within a week. I wanted something where the fastest path — take a photo — also gives a trustworthy answer, and where the app helps you decide *what to eat next*, not just record what you already ate.

---

## What the app does

### 1. Log a meal in seconds — five ways
- **Snap a photo.** The AI identifies every food on the plate, estimates the portion of each, and returns calories, protein, carbs and fat per item and in total, plus a 0–100 health score and a short coaching note. An animated scanning overlay shows progress while the photo is analyzed.
- **Describe it in words.** "2 rotis and a bowl of dal" gets the same breakdown as a photo — useful when you've already eaten.
- **Scan a barcode.** The phone camera reads the barcode in the browser; the product is looked up in the Open Food Facts database for exact label data. I convert per-100 g values to the serving size, convert kilojoules to calories when needed, and compute a health score from the macros when no AI is involved.
- **Scan a restaurant menu.** Photograph a menu and BiteViva ranks the dishes against your goal (e.g. high protein, lower calorie) and explains why.
- **Repeat a past meal** with one tap from your history.

### 2. Review and correct before anything is saved
AI estimates can be wrong, so nothing counts toward your day until you confirm it. On the result screen you can change the portion of any item (½×, 1.5×, 2× …), remove items the AI got wrong, and watch the totals update instantly. The same nutrition math runs in the browser (for live updates) and on the server (to re-check the numbers), and the server validates every food item sent from the browser before saving it.

### 3. Today dashboard and history
- Daily progress rings for calories and each macro against your personal targets.
- A 7-day chart and weekly stats.
- Edit, delete or re-log any meal.
- Day boundaries follow **your** time zone: the browser reports its zone and the server works out "today" from it, so a late dinner never lands on the wrong day.

### 4. Personal goals
- Six goal presets (Cutting, Maintenance, Bulking, High protein, Keto, Athlete) with a live macro pie chart preview.
- A built-in calorie calculator using the Mifflin–St Jeor formula with five activity levels.
- Validation that keeps targets realistic and the macro split consistent with the calorie goal.

### 5. Ideas for what to eat next
- **Discover:** meal recommendations filtered by goal, cuisine (with photo tiles) and diet (vegetarian, vegan, etc.), each with macros and a "why it fits" note.
- **Recipes:** any idea opens a full recipe with ingredients and step-by-step instructions, and can be logged directly.
- **Fridge scan:** photograph your fridge and get meals you can make from what's inside.
- **Meal builder:** type the ingredients you have and get meal suggestions.

### 6. AI nutrition coach
A chat coach that, when you're signed in, knows your goals and what you've eaten today, so its advice is specific ("you're 40 g short on protein — here are three dinners that close the gap"). Replies render as formatted text. Any meal result has an "Ask the coach about this" shortcut that opens the chat with the question already written.

### 7. Weekly meal plan and grocery list
- Generates a full week of meals that hits your targets, viewable day by day.
- One tap turns ingredients into a grocery list with check-offs. The list is stored on the device, so it works without an account.

### 8. Accounts and data control
- Sign in with **Google** or **GitHub** (OAuth). Using the scanners doesn't require an account.
- If you scan a meal while signed out and then sign in, the result carries over so you can still log it.
- **Download my data** exports everything stored about you as JSON.
- **Delete account** permanently removes the user and all of their data. Every table cascades from the user record, so nothing is left behind.
- A plain-language **"Your data"** page explains exactly what is stored, what is sent to which service, and why.

### 9. Sharing
Any meal can be turned into a branded image card and shared through the phone's native share sheet.

### 10. Private owner dashboard
An owner-only `/admin` page shows how the product is being used:
- AI uses per day for 30 days, split by signed-in vs. anonymous visitors
- Usage and failure counts per feature
- New and returning accounts, and the latest activity

Usage is recorded *after* each response is sent, so tracking can never slow down or break a request. It stores only the feature name, the time, success or failure, and whether the visitor was signed in — no photos, text or personal identifiers. Access is limited to an allow-list kept in environment variables, and everyone else gets a standard 404, so the page doesn't reveal that it exists. Page-level traffic comes from cookieless Vercel Web Analytics.

---

## Engineering highlights

**Reliable AI output.** Every AI feature asks Gemini for **structured JSON that must match a schema** I defined (foods, portions, macros, scores), rather than free text that has to be parsed. Analysis runs at a low temperature (0.2) for consistent numbers. For long outputs such as a full week's meal plan, and for chat, I turned off the model's "thinking" mode, which otherwise spent the free-tier token budget on reasoning and returned truncated answers.

**Debugging AI behavior.** I traced a bug where unrelated foods (e.g. chai or curry) occasionally appeared in results that had nothing to do with them, and tightened the prompts and validation so results only contain what is actually in the photo or description.

**Built for $0.** The entire product runs on free tiers: Gemini's free API tier, Vercel hosting, Neon serverless Postgres and Open Food Facts. Photos are resized **on the device** to 1280 px JPEGs before upload, which makes scans faster on mobile data and keeps requests small. Because the free AI tier may use submitted content, the app never sends a user's name to the AI, and the privacy page tells people to keep photos to food.

**Data integrity.** Nothing the browser sends is trusted. Food lists are validated and re-totaled on the server, and every database read and write is scoped to the signed-in user. Deleting an account removes everything through `ON DELETE CASCADE`.

**Friendly failure.** Raw network and provider errors are translated into plain, actionable messages (for example, asking people to try again in a minute when the AI is busy) instead of stack traces.

**Time zones done right.** Daily totals, streaks and charts are computed in the user's own time zone, not the server's, which fixed meals showing up on the wrong day for evening users.

**Database design.** Postgres tables for users, OAuth accounts, sessions, meals (with foods stored as JSON plus indexed totals), meal plans and usage events. The schema is managed with Drizzle ORM, and migrations are written to be additive and safe to re-run.

---

## Design

I designed the interface myself, without paid templates or UI kits, aiming for a calm, editorial feel rather than a typical fitness app.

- **Typography:** Fraunces (an expressive serif) for headlines, paired with Geist for the interface.
- **Color system:** a warm cream light theme and a deep green-black dark theme, built from CSS-variable color tokens so every component switches themes automatically. Light, dark or follow-the-system, with no flash of the wrong theme on load.
- **Photography:** real food photos, sized responsively with `srcset` so phones never download desktop-sized images.
- **Motion:** short, snappy animations (0.25–0.35 s) for page transitions, reveals, progress rings and charts, all turned off for people who enable "reduce motion."
- **Mobile first:** a floating bottom tab bar on phones that slides away while the keyboard is open, bottom sheets for menus, and no scroll areas nested inside other scroll areas.

---

## Mobile app (PWA)

BiteViva can be added to a phone's home screen and runs full-screen like a native app: web manifest, app icons, a service worker and a standalone display mode. I handled iPhone-specific layout details so content stays clear of the notch, status bar and home indicator, and controls stay tappable.

---

## Quality and accessibility

I tested changes systematically rather than by eye alone:
- Automated layout checks at 360, 390, 768 and 1280 px widths for horizontal overflow, tap targets under 44 px, and nested scrolling.
- Text contrast checked against WCAG AA in both themes.
- Before/after Lighthouse comparisons for performance-related changes; a change that didn't measurably help was reverted.
- Keyboard support (Escape closes menus and sheets), labelled controls for screen readers, and reduced-motion support.
- Strict TypeScript across the codebase, with a type-check and production build before every deploy.

---

## How it evolved

1. **MealSnap** — a single-purpose prototype: upload a photo, get protein and calories.
2. **Accounts and history** — sign-in, saved meals, daily goals and a history view.
3. **More ways to log** — barcode, menu, fridge and meal-builder tools.
4. **Planning** — recommendations, recipes, a weekly meal plan and a grocery list.
5. **BiteViva** — a full rebrand and redesign. I ran a UX audit of every page and then implemented the fixes: review-before-logging, a "Today" home, time-zone handling, text logging, an AI coach, data export and account deletion, Google sign-in, the PWA, and a consistent design system across every page.
6. **Operating it** — an owner analytics dashboard, privacy documentation and ongoing mobile polish based on real-device testing.

---

## Tech stack

| Area | Tools |
|---|---|
| Framework | Next.js 16 (App Router, Server Components, Route Handlers), React 19 |
| Language | TypeScript |
| Styling and motion | Tailwind CSS, CSS custom properties, Framer Motion |
| AI | Google Gemini 2.5 Flash (vision and text, schema-constrained JSON output) |
| Database | Neon serverless Postgres, Drizzle ORM |
| Auth | Auth.js v5 (Google and GitHub OAuth) with a database session adapter |
| External data | Open Food Facts API (barcodes), Unsplash (photography) |
| Browser APIs | Camera and barcode scanning (html5-qrcode), Canvas image resizing, Web Share API, localStorage |
| Hosting and ops | Vercel (hosting, environment secrets, Web Analytics), Vercel CLI deploys |
| Mobile | Progressive Web App (manifest, service worker, safe-area layout) |

---

## What I learned

- Designing **around AI's uncertainty**: making the user the final check (review before logging) instead of trusting a single model answer.
- Getting **dependable structured output** from a language model with schemas, low temperature and server-side validation.
- **Shipping and operating** a real product: OAuth setup, secrets management, database migrations against live data, analytics that respect privacy, and debugging on real phones.
- Building a **cohesive design system** and holding every page to it.
- Working within **hard constraints** (a $0 budget, free-tier AI limits) without the product feeling cheap.

---

## About me

**Kishan Patel** — B.S. Management Information Systems, Iowa State University · Chicago, IL

[LinkedIn](https://www.linkedin.com/in/kishanpatel-isu/) · [GitHub](https://github.com/LeanKishan) · [Live app](https://biteviva.vercel.app)

---

© 2026 Kishan Patel. All rights reserved. No license is granted to use, copy, modify or distribute any part of BiteViva.
