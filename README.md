<h1 align="center">🌴 BALI 2: Kook gets an upgrade</h1>

<p align="center">
  <b>When Uluwatu turns on, this books the trip, sorts everyone's visa, arrival card and Bali tax, and has Ketut waiting with your names on a sign.</b>
</p>

<p align="center">
  <img alt="n8n" src="https://img.shields.io/badge/built%20in-n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white">
  <img alt="Claude" src="https://img.shields.io/badge/vision-Claude-D97757?style=for-the-badge">
  <img alt="Browser Use" src="https://img.shields.io/badge/agent-Browser%20Use-111111?style=for-the-badge">
  <img alt="Telegram" src="https://img.shields.io/badge/chat-Telegram-26A5E4?style=for-the-badge&logo=telegram&logoColor=white">
  <img alt="WhatsApp" src="https://img.shields.io/badge/driver-WhatsApp-25D366?style=for-the-badge&logo=whatsapp&logoColor=white">
  <img alt="MIT" src="https://img.shields.io/badge/licence-MIT-blue?style=for-the-badge">
</p>

---

One workflow, nine stages, one long chain. It runs for weeks: n8n's Wait nodes park
the execution between steps (lock-in deadline, arrival card window, landing day,
every 6am), so the whole trip lives in a single run.

| # | Stage | What actually happens |
|---|---|---|
| 1 | 🌊 **Watch the swell** | Surfline every 6 hours. Fires when the break is **6ft+ and offshore for 3 days straight**, starting tomorrow or later. |
| 2 | 🔎 **The hunt** | Cheapest flights (Apify, Google Flights), the villa **closest to the break within budget** (Apify, Airbnb), then every cost per person at today's live rupiah rate: flights, villa, scooters, driver, visa + tax, estimated beers. Posted to the group chat. |
| 3 | 🔒 **Lock in or miss out** | Each mate fills in the Lock-in form with a photo of their passport. Claude reads the MRZ, and the code checks **all 5 ICAO check digits** and the **6-month validity rule** before accepting it. 24 hours, then whoever's in, is in. |
| 4 | ✈️ **Book it** | Duffel **holds** the cheapest seats with every passport attached. A browser agent requests the villa on your logged-in Airbnb. |
| 5 | 🛂 **The paperwork** | One **single-use virtual card per person** (Airwallex), capped at the visa + tax. A browser agent applies for each e-VOA and pays the Bali tourist levy on the **official .go.id sites only**, then fills the All Indonesia arrival card the moment it opens (3 days out). Everyone gets **three QR codes** by email. |
| 6 | 🛬 **Land → Ketut** | Live flight tracking (AeroDataBox). Wheels down: your driver gets a WhatsApp with a **name sign** to hold up at Arrivals. |
| 7 | 🛵 **Scooters** | Booked on WhatsApp. At handover you snap every bike; Claude logs **every scratch**, and the owner gets the report for a timestamped OK. |
| 8 | 🌅 **6am surf check** | Every morning: live Surfline rating, size and wind for 5 breaks. The cleanest wins, and your driver gets the call. |
| 9 | 🧹 **Wrap** | Every passport and scan is deleted the day the trip ends. |

## How the "impossible" bits work

**There's no API for an Indonesian visa.** So a browser agent (Browser Use) does it the
way you would, on the real government sites:

- **It only ever touches the official sites:** `evisa.imigrasi.go.id`, `lovebali.baliprov.go.id`,
  `allindonesia.imigrasi.go.id`. Look-alike agent sites charge triple; the task prompt forbids them.
- **Payment without handing an AI your card:** each traveller gets a fresh Airwallex
  single-use virtual card, capped at the visa + tax. Its number goes to the agent as a
  **domain-locked secret**: it can only be typed into those two payment pages, it never
  appears in the prompt, and it dies after one charge.
- **Email verification codes:** the agent gets its own inbox (Browser Use AgentMail) and
  reads the code itself.
- **CAPTCHAs and surprises:** the agent stops and the workflow tells you which run to
  open. You finish one click, it carries on.
- **A hard spend cap** on every agent run (`AGENT_MAX_USD`).

**Never trust a vision model with a passport number.** Claude reads the MRZ (the two lines
at the bottom), then the code verifies it against the five check digits printed on every
passport. A single misread character fails. In testing, Claude read every character right
but miscounted the `<` fillers, so the code repairs filler runs and keeps the version
where the digits agree. It also fixes OCR's classic O-for-0 in number-only fields.

**WhatsApp's 24-hour rule.** A business number can only send free text to someone who
messaged it in the last 24 hours. Your driver won't have, so every driver message is an
**approved template** (below).

## Setup

1. **Import** `workflows/bali2-kook.json` into n8n (1.113 or later, for Data tables).
2. **Credentials.** Open each red node and create:

   | Service | n8n credential type | Header / setting |
   |---|---|---|
   | Telegram bot | Telegram API | your bot token, and add the bot to the group chat |
   | Claude | Anthropic API | your API key |
   | Gmail | Gmail OAuth2 | the account that sends the entry packs |
   | Apify | Header Auth | `Authorization: Bearer <apify token>` |
   | Browser Use | Header Auth | `X-Browser-Use-API-Key: <key>` |
   | Duffel | Header Auth | `Authorization: Bearer <duffel token>` |
   | Airwallex | Custom Auth | `{"headers":{"x-client-id":"...","x-api-key":"..."}}` |
   | AeroDataBox | Header Auth | `X-RapidAPI-Key: <key>` |
   | WhatsApp Cloud API | Header Auth | `Authorization: Bearer <system user token>` |

3. **Two Data tables** (n8n → Data tables), all columns text:
   - `kook_trips`: `trip_id`, `workspace_id`, `status`, `depart_date`
   - `kook_lockins`: `trip_id`, `name`, `email`, `phone`, `given`, `surname`, `nationality`, `number`, `sex`, `dob`, `expiry`
4. **Four WhatsApp templates** (WhatsApp Manager, category Utility, language English):
   - `kook_landed`: image header. `{{1}} has landed. {{2}} of us, visas all done. Heading to {{3}}. Sign attached, see you at Arrivals!`
   - `kook_scooters`: `Hi! Could we get {{1}} scooters delivered to {{2}} for {{3}} days, with surf racks? We'll photograph them at handover.`
   - `kook_handover`: `Scooter handover report: {{1}} Please reply OK to confirm.`
   - `kook_surf_call`: `Morning! Today's break is {{1}}. Pick up 6:30 at the villa please, {{2}} boards on the roof.`
5. **Browser Use profile:** create one and log it in to your Airbnb, so the villa agent can book.
6. **Fill in `Kook settings`** (and the four lines in `Lock-in settings`), then **activate**.

## Honest limits

- The browser agent is fast, not magic: when a government site changes its layout or
  throws a CAPTCHA, it stops and asks you. That's by design.
- e-VOA eligibility depends on your passport. Australians, Kiwis, Brits, Americans and most
  of Europe qualify; check yours.
- Duffel's airline coverage varies, and not every fare can be held. If none can, it says so.
- Airbnb has no booking API, so the villa is a browser agent on your own account.
- Card numbers pass through the run. This workflow ships with successful-execution saving
  **off**; keep it that way.

MIT licence. Built in n8n by Aaron Parton.
