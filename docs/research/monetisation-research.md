# Monetisation Research: AI-Powered Study Apps (macOS/iOS)

**Date**: 2026-03-15
**Status**: Research Complete
**Applies to**: Socratic Study Mentor

---

## Table of Contents

1. [Monetisation Models That Work for Study Apps](#1-monetisation-models-that-work-for-study-apps)
2. [Competitor Pricing Analysis](#2-competitor-pricing-analysis)
3. [BYOK (Bring Your Own Key) Pricing Models](#3-byok-bring-your-own-key-pricing-models)
4. [Apple App Store Considerations](#4-apple-app-store-considerations)
5. [Homebrew Cask Distribution](#5-homebrew-cask-distribution)
6. [Legal/Terms Implications of Proxying User API Keys](#6-legalterms-implications-of-proxying-user-api-keys)
7. [Free CLI / Paid API Precedent](#7-free-cli--paid-api-precedent)
8. [Recommended Strategy for Socratic Study Mentor](#8-recommended-strategy-for-socratic-study-mentor)

---

## 1. Monetisation Models That Work for Study Apps

### Model Comparison

| Model | Examples | Pros | Cons |
|-------|----------|------|------|
| **Freemium** | Quizlet, Brainscape, Knowt | Low barrier to entry, large user base, upsell path | Must deliver enough free value; conversion rates typically 2-5% |
| **Subscription** | Quizlet Plus, Knowt Ultra | Recurring revenue, predictable income | Subscription fatigue; users expect continuous new value |
| **One-time Purchase** | Anki (iOS), BoltAI | Simple to understand, no churn | No recurring revenue; must sell new versions or add-ons |
| **BYOK + Paid App** | BoltAI, TypingMind | User controls costs; app revenue is decoupled from API costs | Smaller addressable market (technical users only) |
| **BYOK + Free App** | Jan AI, ChatBox AI | Maximum adoption; community-driven | No direct revenue; must monetise elsewhere (donations, cloud services) |
| **Hybrid (Free core + Paid premium)** | Brainscape | Best of both worlds | Complex to implement; must find the right paywall boundary |

### Key Insight: Study Apps Trend Toward Freemium/Subscription

The dominant pattern in 2025-2026 is **freemium with annual subscription upsell**. Users expect to try before they buy. The notable exception is Anki, which uses a one-time iOS purchase to fund open-source development -- but Anki has 15+ years of brand equity and a captive audience of medical students.

---

## 2. Competitor Pricing Analysis

### Anki

- **Desktop (macOS/Windows/Linux)**: Free, open-source
- **iOS (AnkiMobile)**: USD $24.99 one-time purchase
- **Android (AnkiDroid)**: Free, open-source (separate dev team)
- **Web (AnkiWeb)**: Free sync service
- **Model**: One-time purchase on iOS funds all development. The App Store listing explicitly states: "Sales of this app support the development of both the computer and mobile version, which is why the app is priced as a computer application."
- **Lesson**: A dedicated, technical user base (medical students, language learners) will pay a premium one-time price if the value proposition is clear. No subscription fatigue. Rating: 4.1 stars with 2,212 reviews.

### Quizlet

- **Free tier**: Limited -- 3 practice tests/month, 20 rounds of Learn questions/month
- **Quizlet Plus**: GBP 2.99/month (billed annually at GBP 35.99/year) -- 3 practice tests, 3 textbook solutions, 20 Learn rounds, ad-free
- **Quizlet Plus Unlimited**: GBP 3.75/month (billed annually at GBP 44.99/year) -- complete access to everything, unlimited Learn, millions of textbook solutions
- **Family Plan**: GBP 6.99/month (billed at GBP 83.99/year) -- up to 5 accounts
- **Free trial**: 7-day trial on annual plans
- **Model**: Aggressive freemium with tight usage caps on the free tier to drive conversions. Monthly plans have no free trial. Tiered subscription creates a natural upsell ladder.
- **Lesson**: Usage-based limits (X questions per month) are effective gatekeeping for AI-powered features. The free tier is deliberately frustrating to drive upgrades.

### Brainscape

- **Basic (Free)**: Unlimited flashcard creation, import, AI generation (100s of cards), sharing with peers
- **Pro (Individual)**: USD 11.99/month or USD 47.88/year (USD 3.99/month) -- unlimited access to certified classes, advanced study analytics, bookmarks, offline mode, no ads
- **Enterprise**: Custom pricing with bulk discounts (up to 70% off consumer Pro), dedicated support, branded landing pages, student data analytics
- **Model**: Generous free tier for creation; paywall on premium content library and advanced analytics. The free tier is genuinely useful -- you can create and study your own cards indefinitely.
- **Lesson**: Separating "creation tools" (free) from "premium content + analytics" (paid) is a clean paywall boundary that doesn't frustrate users.

### Knowt

- **Basic (Free)**: Notes, flashcards, free study modes, custom hints, browse community content
- **Ultra Annual**: USD 12.49/month (billed at USD 149.99/year) -- unlimited AI summaries, unlimited Kai AI chat, auto-graded assessments, AP mock exams
- **Ultra Monthly**: USD 24.99/month
- **Model**: Freemium with aggressive AI feature gating. The annual price is notably higher than competitors (USD 150/year), reflecting the cost of AI features.
- **Lesson**: AI-powered features command a price premium. Knowt's higher price point suggests the market accepts that "AI costs money" -- but also that the annual discount must be substantial (50% in Knowt's case) to drive annual commitment.

### Summary Table

| App | Free Tier | Paid Tier | Annual Price | Model |
|-----|-----------|-----------|-------------|-------|
| Anki | Desktop only | iOS one-time | $24.99 (once) | One-time purchase |
| Quizlet | Usage-capped | Plus / Unlimited | ~$44-55/year | Freemium + subscription |
| Brainscape | Create + study own | Pro | ~$48/year | Freemium + subscription |
| Knowt | Basic study | Ultra | $150/year | Freemium + subscription |

---

## 3. BYOK (Bring Your Own Key) Pricing Models

### What Is BYOK?

The user provides their own API key for Claude, Gemini, OpenRouter, etc. The app acts as a frontend/interface layer. The user pays the LLM provider directly for token usage.

### Real-World BYOK App Pricing

#### BoltAI (Mac-native AI chat app)

- **Model**: One-time purchase + BYOK
- **Essential**: USD $79 (1 seat, perpetual license, 1 year updates, 512MB cloud)
- **Pro**: USD $99 (2 seats + 1 mobile, perpetual, 1 year updates, 1GB cloud)
- **Team**: USD $400 (5 seats, then $80/seat/year)
- **Update renewal**: Optional, ~40% discount after first year lapses
- **Key insight**: BoltAI proves that BYOK apps can charge significant one-time fees. Users are paying for the UI/UX, prompt management, and workflow features -- not the AI itself. The perpetual license with optional update renewals avoids subscription fatigue while creating recurring revenue opportunity.

#### TypingMind (Web + Desktop AI chat)

- **Model**: One-time license purchase + BYOK
- **Pricing**: One-time license fee (exact current pricing requires JS-rendered page, but historically around $39-79 for personal use)
- **Key insight**: Pioneered the "pay once for the interface, bring your own key" model. Proved there's a market for premium AI frontends even when the underlying API access is commodity.

#### ChatBox AI (Cross-platform AI client)

- **Model**: Free + BYOK with optional premium
- **Pricing**: Free download, open-source core. Premium features available.
- **Key insight**: The "free BYOK" model works for building community and adoption but requires alternative monetisation (premium features, cloud services, or donations).

#### Jan AI (Open-source local AI)

- **Model**: Free, open-source
- **Pricing**: Completely free. 5.3M+ downloads, 41K GitHub stars
- **Key insight**: The fully-free model works when backed by a company with other revenue streams (Homebrew Labs has enterprise/cloud offerings). Not viable for solo developers.

### BYOK Pricing Patterns

The market has settled into three clear tiers:

1. **Free + BYOK** (Jan, ChatBox): Maximum adoption, no direct app revenue. Works for VC-backed companies or projects with enterprise upsells.
2. **One-time purchase + BYOK** (BoltAI at $79-99, TypingMind at ~$39-79): User pays once for the interface. Sustainable for indie developers. The "Sketch/Figma" model.
3. **Subscription + BYOK**: Rare in practice. Users resist paying both a subscription AND their own API costs. The exception is when the subscription funds significant server-side features (sync, cloud storage, team management).

**For Socratic Study Mentor, the one-time purchase + BYOK model (a la BoltAI) is the most natural fit.** The app provides genuine value through Socratic pedagogy, SRS scheduling, course management, and TUI/web interfaces -- all of which are independent of the LLM.

---

## 4. Apple App Store Considerations

### Commission Structure

| Scenario | Commission Rate | Notes |
|----------|----------------|-------|
| Standard rate | 30% | Default for all paid apps and IAP |
| Small Business Program | 15% | Developers earning < $1M/year in proceeds |
| Subscription year 2+ | 15% | After subscriber retains for > 1 year |
| Reader apps (external link entitlement) | Reduced/0% | For "reader" apps that allow linking to external purchase |

#### Small Business Program Details (from Apple)

- Developers earning up to USD $1M in proceeds in the prior calendar year qualify
- If you surpass $1M in the current year, standard 30% rate applies to future sales
- If proceeds fall below $1M in a future year, you re-qualify the following year
- Must identify all Associated Developer Accounts
- **This is the relevant tier for Socratic Study Mentor** -- we would almost certainly qualify

### Subscription vs One-Time Purchase on App Store

#### Subscription Pros
- Apple takes 15% after year 1 (vs 30% initially, or 15% under Small Business Program)
- Recurring revenue stream
- Apple promotes subscription apps in search results
- Free trial capability built into StoreKit
- Family Sharing support

#### Subscription Cons
- Users have subscription fatigue -- especially for study apps (students are price-sensitive)
- Must continuously deliver new value to justify recurring payments
- App Review requires clear disclosure of subscription terms
- Complex StoreKit implementation (receipt validation, server notifications, grace periods)

#### One-Time Purchase Pros
- Simple to understand and implement
- No churn -- once purchased, it's purchased
- Appeals to the BYOK audience (technical users who dislike subscriptions)
- Anki has proven this works at $24.99 for a study app
- Under Small Business Program: only 15% commission

#### One-Time Purchase Cons
- No recurring revenue (must sell to new users or release paid major versions)
- Apple does not support "paid upgrades" natively -- must use new App Store listing or IAP for major version upgrades
- Lower lifetime value per user compared to subscriptions

### Key App Store Review Guidelines for BYOK Apps

**3.1.1 In-App Purchase**: "If you want to unlock features or functionality within your app, you must use in-app purchase."

However, there are critical exceptions:

1. **Mac App Store exception**: "Apps distributed via the Mac App Store may host plug-ins or extensions that are enabled with mechanisms other than the App Store." This is significant -- Mac apps have more flexibility.

2. **3.1.1(a) Link to Other Purchase Methods**: Developers can apply for entitlements to link to external purchase methods (the "reader app" entitlement). This was introduced post-Epic Games lawsuit.

3. **BYOK is not an IAP issue**: When a user enters their own API key, they are not purchasing content through your app. They are configuring the app to use their existing service. This is analogous to entering IMAP credentials in a mail client. Apple has not historically required IAP for user-provided API keys. Apps like BoltAI exist on the Mac App Store with BYOK functionality.

4. **Important caveat for iOS**: If the app has a free tier with limited features and a "premium" tier unlocked by... anything other than IAP (including "bring your own key to unlock AI features"), Apple may reject it. The safest approach: the app should work without an API key (as a flashcard/study app), and the API key simply enables AI features as a user-configured integration, not as "unlocking premium content."

### Recommended App Store Strategy

**Mac App Store**: One-time purchase ($24.99-49.99 range, matching Anki's precedent). BYOK as a feature, not a paywall gate. Under Small Business Program = 15% commission.

**iOS App Store**: If pursued later, same one-time purchase model. BYOK for AI features. Core flashcard/study features work without AI.

---

## 5. Homebrew Cask Distribution

### Can Homebrew Cask Coexist with App Store?

**Yes, absolutely.** This is a well-established pattern. Many apps distribute via both channels:

- **Homebrew cask**: Direct download from developer website, no Apple commission, full system access (no sandbox), can include CLI tools
- **App Store**: Sandboxed, Apple commission applies, but gains discoverability, trust, and automatic updates

#### Real-World Examples of Dual Distribution

- **iTerm2**: Homebrew cask only (no App Store) -- free, open-source
- **1Password**: Both App Store and direct download (Homebrew cask)
- **VS Code**: Homebrew cask (not on App Store due to sandbox limitations)
- **Raycast**: Homebrew cask (free) with premium subscription managed outside App Store
- **Alfred**: Direct download (Homebrew cask) with Powerpack license sold on website

### Homebrew Cask Acceptance Criteria

From the official Homebrew documentation:

#### Notability Thresholds
- **Third-party submissions**: 30+ forks, 30+ watchers, 75+ stars on the GitHub repo
- **Self-submitted casks**: 90+ forks, 90+ watchers, 225+ stars (higher bar)
- **Exceptions**: Popular apps with their own website, apps with significant buzz, submissions by Homebrew maintainers

#### Key Requirements
- Must be a stable release (not beta/nightly)
- Must work with GateKeeper enabled (must be signed -- critical for Apple Silicon)
- Must not be a trial version where the only full version is Mac App Store exclusive
- Freemium versions ARE accepted ("Gratis version that works indefinitely but with limitations that can be removed by paying")
- App must have a public presence beyond just `brew install`
- Must not require SIP to be disabled

#### Rejected If
- App is too obscure (below notability thresholds)
- App is unmaintained
- Trial-only where full version is Mac App Store exclusive
- Unsigned/GateKeeper incompatible on supported macOS versions
- CLI-only AND open-source (these go in homebrew-core as formulae, not casks)

### Dual Distribution Strategy

**Homebrew cask build** (direct download):
- Full system access (no sandbox)
- CLI integration (`studyctl` command)
- Can auto-update via Sparkle or homebrew upgrade
- No Apple commission
- License validation via your own system (Gumroad, Paddle, LemonSqueezy)
- Can include features impossible in sandboxed App Store builds (filesystem watchers, global hotkeys, etc.)

**App Store build** (if pursued):
- Sandboxed (may limit some features)
- Apple handles payments, receipts, refunds
- 15% commission (Small Business Program)
- Automatic updates, family sharing
- Discoverability for non-technical users

**Key consideration**: Homebrew cask policy explicitly rejects apps where "the only way to acquire the full version is through the Mac App Store." So the Homebrew version must be a complete, non-trial build. This means: same features, different distribution channel. You can differentiate on _how_ the license is purchased (direct vs App Store IAP) but not on what features are available.

---

## 6. Legal/Terms Implications of Proxying User API Keys

### Critical Question: Does the User's Key Go Through Your Server?

This is the most important architectural decision. There are two models:

#### Model A: Direct Client-Side Calls (Recommended)

The app calls the LLM API directly from the user's device using their key. Your server never sees the key.

**Legal implications**: Minimal. You are a tool/client, like a web browser. The user has their own relationship with Anthropic/Google/OpenRouter. Your app is not an intermediary.

**Privacy implications**: Excellent. You never handle user API keys or their conversation data.

#### Model B: Server-Side Proxy

The user's key is sent to your server, which proxies requests to the LLM API.

**Legal implications**: Significant. You are now processing user credentials and conversation data. This triggers:
- GDPR/data protection obligations (you are a data processor)
- Need for a Data Processing Agreement
- Liability for key security (breach = your problem)
- Potential violation of LLM provider terms if you are seen as "reselling" access

**Recommendation**: Use Model A (direct client-side calls) exclusively.

### Anthropic API Terms Analysis

From Anthropic's Commercial Terms of Service:

- **Section A.1**: "Anthropic gives Customer permission to use the Services, including to power products and services Customer makes available to its own customers and end users ('Users')." This explicitly permits building apps that use the API.
- **Section B**: Customer retains rights to Inputs and owns Outputs. Anthropic may not train on Customer Content.
- **Section D.1**: Must comply with the Acceptable Use Policy. Importantly, the AUP has specific requirements for "consumer-facing" applications and "products serving minors."
- **Acceptable Use Policy - Products Serving Minors**: If your study app could be used by under-18s, additional requirements apply. Anthropic requires additional safeguards for consumer-facing chatbots and products serving minors.

**Key takeaway for BYOK**: When the user provides their own API key, THEY are the "Customer" under Anthropic's Commercial Terms. Your app is simply a client/tool they use to access their own API account. You do not need a separate commercial agreement with Anthropic for BYOK usage. However, your app should not encourage or facilitate AUP violations.

### Google Gemini API Terms Analysis

From the Gemini API Additional Terms of Service:

- **Age requirement**: Users must be 18+ to use the APIs. Apps ("API Clients") must not be directed toward or likely accessed by individuals under 18. **This is a hard restriction** -- if your study app targets students under 18, you cannot use Gemini API as the backend.
- **Use restrictions**: "Use of Google AI Studio and Gemini API is for developers building with Google AI models for professional or business purposes, not for consumer use." **This is ambiguous for BYOK** -- if the end user has their own API key, they are arguably a developer. But Google could argue that a study app is consumer use.
- **Regional restrictions**: Paid Services required for API Clients available to users in EEA, Switzerland, or UK. Free tier Gemini API cannot be used in production apps serving European users.
- **Competing models**: Cannot use the Services to develop models that compete with Gemini.

**Key takeaway**: Gemini's terms are more restrictive than Anthropic's. The age restriction (18+) and "not for consumer use" clause are potential issues for a study app. For BYOK, the user's own key usage should be governed by their own agreement with Google, but your app should clearly communicate these restrictions.

### OpenRouter Terms Analysis

From OpenRouter's Terms of Service (updated March 2026):

- **Section 1**: OpenRouter is an LLM aggregator. Users access third-party APIs through OpenRouter.
- **Age requirement**: Must be at least 13 to use the service (more permissive than Gemini).
- **Developer-friendly**: OpenRouter is explicitly designed for third-party app integration. Their business model is being an API aggregator -- they expect and encourage apps to use their service.

**Key takeaway**: OpenRouter is the most BYOK-friendly option. Their terms are designed for exactly this use case. They handle the relationship with upstream providers (Anthropic, Google, etc.), and the user's key is with OpenRouter, not directly with each provider.

### Summary of Legal Considerations

| Provider | BYOK Friendly? | Age Restriction | Key Risk |
|----------|----------------|-----------------|----------|
| Anthropic | Yes -- explicit support for building apps | 18+ (commercial terms) | AUP compliance for consumer-facing + minors |
| Google Gemini | Cautious -- "not for consumer use" clause | 18+ (hard restriction) | Consumer use restriction; EEA paid-only requirement |
| OpenRouter | Very friendly -- designed for this | 13+ | Aggregator risk (upstream provider changes) |

### Recommendations

1. **Use direct client-side API calls** -- never proxy keys through your server
2. **Support OpenRouter as the primary BYOK option** -- most developer-friendly terms, single key for multiple models, 13+ age restriction
3. **Support Anthropic direct as a secondary option** -- well-suited, but be mindful of AUP requirements for consumer-facing apps
4. **Be cautious with Gemini direct** -- the "not for consumer use" clause creates ambiguity. OpenRouter as an intermediary may be safer for Gemini model access
5. **Clearly communicate** in your app: "By entering your API key, you agree to the respective provider's terms of service"
6. **If targeting under-18 students**: OpenRouter (13+) is the only option. Cannot use Gemini or Anthropic direct for under-18 users
7. **Privacy policy**: Even with BYOK, you need a privacy policy that explains what data stays local, what (if anything) leaves the device, and that API keys are stored only on the user's device

---

## 7. Free CLI / Paid API Precedent

### Does "Free with Coding Plan CLI, Paid API Otherwise" Exist?

This is an emerging pattern, though not yet widespread. The closest precedents:

#### Cursor

- **Hobby (Free)**: Limited Agent requests, limited Tab completions, no credit card required
- **Pro ($20/month)**: Extended limits, frontier models, MCPs/skills/hooks, cloud agents
- **Pro+ ($60/month)**: 3x usage on all models
- **Ultra ($200/month)**: 20x usage on all models
- **Model**: Free tier exists but is usage-capped. All paid tiers include API access to frontier models (Claude, GPT-4, Gemini). The user does NOT need their own API key.
- **Relevance**: Cursor bundles API costs into the subscription. This is NOT a BYOK model. But it shows that "free tier + paid for more AI" works.

#### Claude Code (Anthropic)

- **Included with Claude Pro/Max subscription**: Claude Code CLI is available to subscribers
- **Also usable with API key**: Developers can use their own API key directly
- **Model**: "Free if you already pay for the subscription, bring your own key otherwise"
- **This is the closest precedent** to what you're describing. Anthropic bundles CLI access with their consumer subscription but also allows direct API key usage.

#### GitHub Copilot

- **Free tier**: Limited completions/month in VS Code
- **Individual ($10/month)**: Full completions + chat
- **Business ($19/user/month)**: Team features
- **Enterprise ($39/user/month)**: Customization
- **Model**: Subscription bundles API access. No BYOK option. But there IS a free tier.

### The Pattern You're Describing

"Free if you have a coding plan CLI, paid API otherwise" -- this would look like:

1. User has a Claude Max subscription -> Claude Code works -> your app detects this and works for free
2. User has no subscription -> your app asks for an API key -> user pays per-token

**This is technically feasible but has challenges**:

- **Detection**: How do you detect that the user has a Claude Pro/Max subscription? There is no public API for this. You could check for the presence of `claude` CLI and try to use it, but this is fragile.
- **Terms of service**: Using Claude Code (which is bundled with a consumer subscription) to power a third-party app may violate Anthropic's terms. The subscription is for the user's personal use of Claude, not for third-party apps to leverage.
- **Better alternative**: Simply support BYOK with an API key. Users with coding plans already have API keys. Users without can get one. The pricing is transparent.

### Recommended Approach

Rather than trying to detect existing subscriptions, offer a cleaner model:

1. **Free app** with full study features (flashcards, SRS, course management, TUI, web UI)
2. **AI features require an API key** (BYOK) -- user provides their own Claude, Gemini, or OpenRouter key
3. **Optional paid tier** (one-time purchase, $29-49) that adds premium non-AI features: cloud sync, advanced analytics, export formats, multi-device support
4. **No server costs for you** beyond distribution -- all AI calls are client-side with user's key

---

## 8. Recommended Strategy for Socratic Study Mentor

### Proposed Monetisation Model

Based on all research, the recommended approach is a **Freemium + BYOK + One-Time Premium Purchase** hybrid:

```
+--------------------------------------------------+
|              SOCRATIC STUDY MENTOR                |
+--------------------------------------------------+
|                                                    |
|  FREE TIER (Homebrew / Direct Download)            |
|  - Full TUI and Web interface                      |
|  - Flashcard creation and SRS scheduling           |
|  - Course management and progress tracking         |
|  - Basic study modes (no AI)                       |
|  - Import/export decks                             |
|  - BYOK: Enter your own API key for AI features   |
|    - Socratic questioning                          |
|    - AI-generated flashcards                       |
|    - Adaptive explanations                         |
|    - Voice study mode                              |
|                                                    |
+--------------------------------------------------+
|                                                    |
|  PREMIUM ($29.99 one-time)                         |
|  Everything in Free, plus:                         |
|  - Advanced analytics and learning insights        |
|  - Cross-device sync                               |
|  - Priority support                                |
|  - Offline AI model support (local LLMs)           |
|  - Advanced export (Obsidian, PDF, APKG)           |
|  - Pomodoro integration + study scheduling         |
|  - Custom prompt templates                         |
|                                                    |
+--------------------------------------------------+
```

### Why This Model

1. **Low barrier to entry**: Free tier is genuinely useful without AI. The Brainscape model (free creation, paid premium) proves this works.
2. **BYOK avoids server costs**: You have zero marginal cost per user for AI features. No need to manage API budgets, quotas, or billing.
3. **One-time purchase avoids subscription fatigue**: Students are price-sensitive. A one-time fee (like Anki at $24.99) matches the audience. BoltAI proves BYOK apps can charge $79-99 for a one-time license.
4. **Price point**: $29.99 is in the sweet spot -- more than Anki (justified by AI features), less than BoltAI (study app, not professional tool). Under Small Business Program, Apple takes $4.50, you keep $25.49.
5. **Technical users self-select**: The BYOK audience overlaps heavily with Homebrew users, developers, and power users who value privacy and control.

### Distribution Strategy

| Channel | Build Type | Pricing | Commission |
|---------|-----------|---------|------------|
| **Homebrew cask** | Full app, unsigned or self-signed | Free + paid license via Gumroad/LemonSqueezy | 0% (Gumroad takes ~5%) |
| **Direct download** | Full app, signed + notarised | Free + paid license via Gumroad/LemonSqueezy | 0% (Gumroad takes ~5%) |
| **Mac App Store** | Sandboxed build | Free download + one-time IAP for Premium | 15% (Small Business Program) |
| **iOS App Store** (future) | Sandboxed build | One-time purchase or Free + IAP | 15% (Small Business Program) |
| **PyPI / uv tool** | CLI only (`studyctl`) | Free, open-source | 0% |

### Implementation Priority

1. **Phase 1 (Now)**: Free + BYOK via `uv tool install` and direct download. Open-source core.
2. **Phase 2**: Homebrew cask distribution. Add premium features behind a license key (Gumroad/LemonSqueezy).
3. **Phase 3**: Mac App Store submission with IAP for premium tier.
4. **Phase 4 (Optional)**: iOS port if demand warrants.

### Revenue Projections (Conservative)

Assuming 1,000 downloads/month and 5% conversion:

| Metric | Monthly | Annual |
|--------|---------|--------|
| Downloads | 1,000 | 12,000 |
| Conversions (5%) | 50 | 600 |
| Revenue (@ $29.99) | $1,500 | $18,000 |
| After commission (~10% blended) | $1,350 | $16,200 |

This is sustainable indie developer revenue. With 10,000 downloads/month (achievable if the app gains traction in the study/developer community), revenue scales to $162K/year.

---

## Sources

| Source | Type | Key Finding |
|--------|------|-------------|
| Apple Small Business Program | Official docs | 15% commission for developers earning < $1M/year |
| Apple App Store Review Guidelines (Section 3.1.1) | Official docs | IAP required for digital content; Mac apps have plug-in exception; BYOK keys are user configuration, not content purchase |
| Apple Auto-renewable Subscriptions docs | Official docs | 15% commission after subscriber retains > 1 year |
| Anki (apps.apple.com) | Competitor | $24.99 one-time iOS purchase funds open-source development |
| Quizlet (quizlet.com/upgrade) | Competitor | Freemium with usage-capped free tier; GBP 36-45/year subscription |
| Brainscape (brainscape.com/pricing) | Competitor | Free creation + study; Pro at ~$48/year for premium content + analytics |
| Knowt (knowt.com/plans) | Competitor | Free basic; Ultra at $150/year reflecting AI feature costs |
| BoltAI (boltai.com/pricing) | BYOK precedent | One-time purchase $79-99 + BYOK; perpetual license with optional renewal |
| TypingMind (typingmind.com) | BYOK precedent | One-time license + BYOK; pioneered the model |
| ChatBox AI (chatboxai.app) | BYOK precedent | Free + BYOK; open-source core |
| Jan AI (jan.ai) | BYOK precedent | Free, open-source, 5.3M downloads, 41K GitHub stars |
| Cursor (cursor.com/pricing) | Free CLI precedent | Free Hobby tier with limits; $20-200/month subscription includes API |
| Anthropic Commercial Terms | Legal | Explicitly permits building apps for end users; customer owns outputs |
| Anthropic Acceptable Use Policy | Legal | Additional requirements for consumer-facing apps and products serving minors |
| Google Gemini API Terms | Legal | 18+ age restriction; "not for consumer use" clause; EEA requires paid tier |
| OpenRouter Terms of Service | Legal | Most BYOK-friendly; 13+ age minimum; designed for third-party app integration |
| Homebrew Acceptable Casks (docs.brew.sh) | Distribution | Notability thresholds (75/225 stars); freemium accepted; trial-only rejected |
