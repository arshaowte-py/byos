# Frido · Make Your Own Shoe — Claude Code Handoff

*State as of 2 Oct 2026. The working file is `index.html` (single file, about 215 KB), live at **https://fridoxbyos.netlify.app**.*

---

## 0. Ground rules for Claude Code (read first)

1. **Never run any git command**: no checkout, add, commit, push, status, restore, stash or diff. The user runs all git commands and creates and switches branches before invoking Claude Code.
2. **Do not change the flow structure.** The 7 steps, their order, the sub-slides inside them, navigation and validation are fixed (signed off by the CEO and AVP). UI changes are welcome; flow changes must be asked about first.
3. **Do not change business logic silently**: product fit, recommendations, pricing, safety rules, report actions. Ask first if a request implies it.
4. **Naming:** always say **"scientifically personalised footwear"**, never "custom footwear".
5. Keep it a **single self-contained HTML file** unless the user asks to split it.
6. **Executing over planning:** the user prefers direct execution, and outputs that look human-made rather than over-engineered.
7. **Clarifying questions:** ask them as short multiple-choice options.

---

## 1. What the product is

A store portal for Frido's **scientifically personalised footwear**. Meta ads bring customers into Frido stores. In the store:

1. A podiatrist assesses the customer: conditions, a 3D Volumental scan, and a 2D gait / OHM 6000 pressure test.
2. The customer picks Frido shoe designs (up to 3 pairs).
3. The portal prescribes an insole (ready or custom) for those shoes.
4. The customer adds customisation and slipper/sandal cross-sell.
5. A **Frido Foot Profile** report goes out on WhatsApp/PDF/email.

Users:
- The **podiatrist** and the **retail sales officer (RSO)** share one tablet.
- **Customer view** hides staff-only detail; **Staff view** shows everything.
- **Admin** holds the assessment log, prices and brand assets.

**Stakeholders:**
- **Ganesh Sir (CEO):** product decisions.
- **Saiyed Abdal Sir (AVP):** UX reviews.

---

## 2. Flow (fixed)

`SUBS = [3,1,2,2,1,1,1]` gives the number of sub-slides per step. Render map (in `render()`):

```
[[sLand, sF0, sF1], [sF4], [sScan, sF2], [sF3, sBuild], [sCust], [sCross], [s6]]
```

| Step | Sub-slides | Function(s) | What happens |
|---|---|---|---|
| 1 Customer & conditions | Lander → Details → Conditions | `sLand`, `sF0`, `sF1` | **Lander:** "One size fits no one" pitch (a brand's UK 8 is the same for lakhs of people) with CTA "Get your personalised footwear today". **Details:** name\*, mobile\* (10 digits), age\* (<14 auto-selects Kid), email (to share the official report), gender\*, shoe category\* (Male / Female / Kid / Unisex, plus a blinking "unisex styles" toggle), podiatrist\* and store (staff). **Conditions** "What are you experiencing?": 6 image cards plus "I don't know about my feet" and "None of these"; "View example" drawer; severity and diabetic red flags are staff only. |
| 2 Design | — | `sF4` | Closed footwear (Open = coming soon: Sandals, Flip Flops & Slippers). Blinking "Broad toe box" / "Zero drop" pills. Style tabs: Formals / Sports / Sneakers / Slip-ons / Lace-ups / Barefoot. Product cards with real photos, colour circles, "Recommended for your feet" badge, **up to 3 pairs** (pair tray), "View details" opens the product detail view (colour slider, gallery, myfrido.com link). |
| 3 Scan reports | Scan status → Results | `sScan`, `sF2` | **Scan status:** Start scan → Scan in process (timer) → podiatrist taps **Scan completed**. **Results:** 1 **Customer perceived** · 2 **Scan led actual** (3D: STL+CSV only; gait analysis: OHM PDF only; 3D viewer; findings; technical data collapsed) · 3 **Podiatrist notes** (text, physical copy, voice note, feet photos). |
| 4 Finalise | 3-s "Making a personalised footwear for Mr …" animation → Prescription → Build | `showMaking`, `sF3`, `sBuild`/`sF5` | **Prescription:** perceived-vs-found comparison line, shoe-visual hero, findings (from list or free text), level and components each "Suggested by AI" vs "Accepted by podiatrist", podiatrist confirmation (required). **Build:** base shoe + insole, with pair tabs to customise individual pairs. |
| 5 Customisation | — | `sCust` | Engraving (≤12 chars; tongue / heel / insole), EVA or memory-foam footbed. One per order. |
| 6 Cross-sell | — | `sCross` | 3 open-footwear picks, each "Suggested because you have '…'". |
| 7 Report | — | `s6`, `reportHTML`, `waText` | Frido Foot Profile. Send PDF on WhatsApp (share sheet, or download + open chat), WhatsApp text, Email, Download PDF, Print, Save to log, next free consultation date (default +30 days). |

**Navigation:**
- `stepNext()` and `stepBack()` move through the sub-slides.
- `go(n, toSub)` jumps between steps.
- `subProblems(step, sub)` validates each slide; `problems(step)` validates the whole step.

---

## 3. Business logic (current — do not change without asking)

### Conditions
`CONDS` ids:
- `pf` Plantar fasciitis
- `pronation` Pronation & overpronation
- `flat` Flat Feet / Fallen Arch
- `corn` Corn & callus
- `diabetes`
- `bunion`
- `high` High Arch: `scanOnly:true`, not shown on the step-1 cards, offered from the scan when the Volumental arch type is "high"

How the condition data is stored:
- `S.perceived` holds what the customer selected in step 1.
- `S.conds` holds the actual / accepted findings. It drives all logic and can be edited by the podiatrist in Finalise.
- `S.xfind` holds free-text findings.

### Segments (`segment()`)
- **A:** no conditions.
- **B:** only flat and/or pronation, all mild.
- **C:** anything else.

### Build levels (`TIERS`)
Base shoe + insole only. The midsole, outsole and upper options were removed on the CEO's call.

| idx | Level | Price |
|---|---|---|
| 0 | Base shoe (no insole) | shoe MRP |
| 1 | Ready insole (tuned in store) | shoe + ₹3,000 |
| 2 | Custom insole (from scan) | shoe + ₹4,000 |

How the level is picked:
- `recTier()`: A → 0, B → 1, C → 2. Diabetes means the minimum level is 2.
- Insole layers (`ADDONS`, all `g:'insole'`): arch_basic, arch_custom, heel_cup, met_pad, fascia, offload, total_contact (locked on for diabetes), lateral_wedge, heel_gel (paid), top_cover (paid).
- `recs()` maps conditions to recommended layers.

### Multiple pairs
- `S.base`/`S.color` is pair 1 (main).
- `S.extra = [{uid, base, color, custom, tier, addons, dropped}]` holds pairs 2–3. `MAX_PAIRS = 3`. The same shoe in another colour counts as its own pair.
- Non-custom pairs inherit `S.tier`/`S.addons`. Custom pairs use their own.
- `withTarget(fn)` swaps the custom pair's build into `S` while the build screen edits it.
- **Price:** `pairPrice(pr) = shoe + level price + paid layers`, and `shoeTotal()` is the sum over `pairs()`.
- **Order total:** `grand()` = `shoeTotal()` + customisation + cross-sell.

### Product fit
`productFit(p)`:
- Diabetes blocks barefoot and sock shoes; plantar fasciitis blocks sock shoes.
- "I don't know" with no findings yet shows every design.

### After the scan
`shoeAdvice()`:
- If a chosen pair doesn't suit the actual findings, it shows "Sorry, we won't recommend …" and staff must change that pair before continuing.
- If no condition is found, it recommends barefoot (with a switch button).

### Safety
Diabetic red flags (open wound, numbness, poor circulation, amputation) put the order **On hold**: sticky banner, report note, saved status.

---

## 4. Data and images

- **`PRODUCTS`:** closed shoes, with `cat`, `line`, `good`/`avoid`, `note`, `tech`, `price`.
- **`OPEN`:** cross-sell open footwear. Prices are in `P.products`, editable in Admin → Prices.
- **`CATALOG`:**
  - Scraped from myfrido.com on 2 Oct 2026: `{id: {h: handle, c: [[colour, file]], g: [gallery files]}}`.
  - Images load from `https://cdn.shopify.com/s/files/1/0553/0419/2034/files/<file>?width=800`.
  - The Shopify CDN sends `Access-Control-Allow-Origin: *`, so the photos also appear in the PDF.
  - Product page link: `https://myfrido.com/products/frido-<handle>`.
- **Refreshing the catalogue:** on a myfrido.com product page, each variant has `"selectedOptions":[…Color…],"image":{"url":…}`, and the product has an `"images":[…]` list.
- **`COND_IMG`:** the 6 condition illustrations embedded as base64 webp.
- **Colour circles:** dot shades come from `P.colors` (approximate hexes, editable in Admin); the colour names come from `CATALOG`.

---

## 5. Runtime: Netlify vs claude.ai

- `ON_CLAUDE` = the page is running inside a claude.ai artifact.
  - On **claude.ai**, external images are blocked by CSP, so the portal falls back to uploaded brand assets or drawn silhouettes.
  - **Netlify is the production target.**
- Storage uses the claude.ai runtime (`window.claude.use('db' | 'assets' | 'downloads' | 'user')`). **On Netlify these are absent**, so the log, shared prices, stored files and brand assets are off; the flow still works fully. Next step if needed: a backend (Firebase / Supabase / Netlify Blobs).
- Functions to swap for a backend: `save`, `openRecord`, `startLog`, `savePrices`, `addFiles` (upload part), `uploadAsset`, `removeAsset`, `importPayload`, and the boot block's `db.doc('settings/…').onSnapshot`.
- **CDN libraries, loaded lazily:**
  - three.js r128 for the 3D viewer
  - pdf.js 3.11.174 for gait report thumbnails
  - html2canvas 1.4.1 and jsPDF 2.5.1 for the PDF report
- Voice notes use `MediaRecorder`; if the microphone is blocked, the portal shows a message instead.

---

## 6. Scan inputs

- **Volumental CSV:** the last data row is used.
  - Fields: left/right length, width, arch height, instep height, heel width, ball girth, instep girth (metres → mm), hallux valgus angle (degrees), arch type, hallux valgus type.
  - Scan suggestions: a low arch suggests Flat Feet / Fallen Arch, a high arch suggests High Arch, and a big-toe angle ≥15° suggests Bunion.
  - The scan's email is not stored.
- **STL:** left/right files, binary or ASCII, in mm (Y = length, Z = up). Parsed by `parseSTL`, rendered by `mount3D`.
- **OHM 6000 S.T.E.P.P. PDF:** 5 image pages, turned into thumbnails by `pdfThumbs`.

---

## 7. Design system (consolidated, v4)

- **Brand:** Frido Brand Book 2024 and the reference site fcevp.netlify.app.
  - Yellow #FFD100, Black #101820 (also the dark canvas), White.
  - Secondary colours as small accents only: mint #80E0A7, coral #FF585D, blue #6CACE4.
  - Typeface: Outfit (stand-in for Gilroy).
- **Tokens** at the top of `<style>`:
  - Surfaces: `--bg`, `--s1`, `--s2`, `--s3`.
  - Lines: `--line`, `--line2`.
  - Text: `--text`, `--muted`, `--faint`.
  - Radii: `--r` 14px, `--r-sm` 10px, `--pill`, `--sig` (signature corner, 32px top-left).
  - Type scale `--f-*` and spacing `--sp1`–`--sp6`.
- **Light theme:** `:root[data-theme="light"]`. The theme toggle reads `--bg === '#101820'`.
- **Rules:**
  - Signature corner only on hero components: lander panels, scan box, prescription card, shoe visual, product detail view, report.
  - Pills only for metadata tags and buttons.
  - Selection controls use the rounded-rect `.chip`.
  - Surfaces are soft fills without borders; the selected state is a yellow border.
- **Headings:** bold word + light italic word (`.lead em`); uppercase yellow eyebrow labels.
- **Motion:** slide transitions only when the step or sub-slide changes (`lastSlide`), the scan beam, the "making" overlay, toggles. Respect reduced motion.
- **Customer mode:** `body.m-cust` hides `.staff`.

---

## 8. Testing

Headless Playwright works well. Route `three.min.js`, `pdf.min.js`, `html2canvas`, `jspdf` and the Shopify CDN as needed, or abort external requests. Walk the full flow:
- lander → details → conditions → design (add 2–3 pairs) → scan (start, completed, or skip) → finalise (wait 3.5 s for the animation, confirm) → build (pair tabs, customise one pair) → customisation → cross-sell → report → Download PDF
- also check: Customer/Staff toggle, theme toggle, "I don't know" path, diabetes + barefoot "sorry" path

Check `pageerror` is empty. The sample scan files are Volumental `v-export.csv`, `left.stl`, `right.stl`, and the OHM PDF.

---

## 9. Open items

- **Insole prices:** confirm ₹3,000 (ready insole) and ₹4,000 (custom insole) now that the ₹5k/₹10k levels are gone.
- **Customisation prices:** engraving ₹299, EVA ₹499 and memory foam ₹799 are placeholders. They are currently per order, not per pair.
- **Product data gaps:**
  - Colour dot hexes are approximate.
  - The HRX sock shoe and School Shoes list no colours on myfrido.
  - Women's Barefoot uses the same colour photos as the men's.
- **Backend for Netlify:** the assessment log, shared settings and file storage.
- **Report title:** the audit suggested "Your Frido Foot Assessment"; it currently stays "Your Frido Foot Profile".
