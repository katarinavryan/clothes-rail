# Clothes Rail 👗

**A virtual try-on tool that solves the £25 billion returns problem — using AI to show you how something will fit before you buy it.**

🔗 **[Try it live](https://sleek-barge-51657936.figma.site/)**

> **Status: Working prototype.** Measurement input, body avatar morphing, fit predictions, and size recommendations are all working. The garment overlay layer is a documented next iteration — see "What I learned building it" below.

---

## The problem

Online clothing returns cost the global retail industry an estimated **$25 billion per year**. The number one reason people return clothes is fit. Not quality. Not colour. **Fit.**

Current solutions don't work:
- **Size guides** — every brand sizes differently; a UK 12 at ASOS is not a UK 12 at Reiss
- **"True to size" reviews** — subjective, inconsistent, not your body
- **Virtual mannequins** — show the garment, not the person wearing it
- **AR try-on tools** — exist, but are typically brand-specific, not cross-retailer, and don't combine items into outfits

Clothes Rail is a personal, brand-agnostic try-on layer that works across any retailer.

---

## How it works

### What the user does

1. **Enter measurements** — chest, waist, hips, inseam, height. Takes 2 minutes.
2. **Add your usual UK size** (optional) — helps calibrate fit predictions across brands
3. **Add a photo** (optional) — for visual representation; stored locally, never uploaded to any server
4. **Browse and add items** — paste a product URL or upload an image of a garment
5. **See your fit** — the tool generates a visual showing the item sized to your measurements, with a fit confidence score and a plain-English note ("this will be loose across the shoulders but fitted at the waist")
6. **Build an outfit** — combine multiple items to see how they work together
7. **Own and share your image** — the generated image belongs to you; share with friends, save to notes, screenshot for your basket

### What the AI does

- Extracts garment data from product images (shape, cut, fabric weight where stated)
- Interprets size chart data from the retailer if available
- Applies your measurements to generate a fit prediction
- Creates a visual representation showing the garment proportioned to your body shape
- Generates a plain-English fit note — not a score, but an explanation

---

## Feedback & evaluation loop

The fit prediction is only as good as what it learns from. Clothes Rail closes the loop with three distinct feedback signals — two explicit, one behavioural.

### Signal 1 — Prototype vote (pre-purchase)

When a user views their generated fit preview, they can vote thumbs up or down.

> *"Does this look right to you?"*

This captures **perceived fit confidence** at the point of decision. It's a fast, low-friction signal that tells the system whether the user trusts the prediction enough to buy — and if not, why not (optional short prompt: *too big / too small / doesn't look like me*).

### Signal 2 — Post-receive vote (follow-up email)

After the estimated delivery window, the user receives a short follow-up email with the item name and a single question:

> *"Did it fit as expected?"* 👍 👎

One click. No login required. The response is tied back to the original prediction via a unique item token in the email link. This captures the **ground truth of the actual fit** — the moment between "I bought it" and "I returned it."

### Signal 3 — Return rate (behavioural ground truth)

The hardest signal and the most honest one. If a predicted fit leads to a return, the prediction was wrong regardless of what the user voted. Return events are tracked per item per customer and fed back into the evaluation layer.

### How the evaluation works

Each prediction is scored against three baselines:

| Dimension | What it measures |
|---|---|
| **This customer, past returns** | Is the tool improving fit accuracy for this specific person over time? If their return rate is falling, the model is learning their body. If it's flat or rising, the calibration is off. |
| **This item, all returns** | Is the fit prediction for this specific garment reliable? High return rates on a specific item signal a systematic mismatch — the size chart may be misleading, the cut may not match the stated measurements, or the garment data extraction needs work. |
| **vs average return rate** | Is Clothes Rail actually moving the needle? Baseline return rate for online fashion is 30–40%. A user who consistently sees lower return rates on items they've run through Clothes Rail is getting measurable value. |

### What this enables

- **Model improvement:** Low-confidence predictions (where prototype vote and post-receive vote diverge, or where return rates are high) are flagged for review and used to retrain fit logic
- **Item-level alerts:** Items with above-average return rates across users trigger a flag — *"Our fit prediction for this item has lower confidence than usual. Here's why."*
- **Customer fit profile:** Over time, each user's vote history and return pattern builds a richer fit profile, improving future predictions without them having to re-enter measurements
- **Retailer-facing reporting (B2B):** For the white-label use case, retailers see a dashboard showing return rate reduction attributed to Clothes Rail predictions — the core commercial proof point

---

## The privacy model (this matters)

Clothes Rail is built on a **local-first** principle:

- Your photo never leaves your device
- Your measurements are stored locally, not in a user database
- Generated images are yours — not used for training, not retained by the service
- No account required to use the core features

This is a deliberate product decision, not just a technical constraint. Asking someone to upload a photo of their body to a retail platform requires a level of trust that most platforms haven't earned. Keeping that data local removes the barrier.

---

## What's under the hood

- **Frontend:** Built with [Figma Make](https://figma.com/make) — the UI and user flow are fully functional
- **Garment analysis:** Claude API (Anthropic) — extracts cut, shape, and sizing signals from product images and descriptions
- **Fit prediction:** Measurement-to-garment mapping logic, calibrated against size guide data
- **Image generation (design target):** Stable Diffusion / DALL-E API — generating photorealistic fit visualisations from measurements and garment data. This layer is documented as a technical spec; the prototype uses illustrated representations.
- **Data:** All personal data (measurements, photos) stored in browser localStorage only

---

## What I learned building it

Building the prototype forced decisions I hadn't anticipated. This is where the interesting stuff happened.

### The avatar baseline problem

My first instinct was to prompt generative AI to create a figure — "create a model at these measurements." That didn't work. The output was too variable, too unpredictable, and impossible to build consistent logic on top of. You can't adjust something that shifts every time you look at it.

The fix: I found a reference drawing, defined it as the baseline avatar with known measurements, and treated it as the fixed foundation. Everything else is relative to that. This also meant converting the avatar to a vector format, so measurement adjustments could be applied mathematically rather than visually guessed.

It took longer than it should have to reach this conclusion — because it felt like cheating to use a fixed reference rather than a generated one. It wasn't cheating. It was the right call.

### The garment fitting problem

Each garment also needed its own baseline — a size 12 starting position using standard high-street measurements — before it could be overlaid on the avatar.

The hard part was proportions. When I scaled a garment image to fit the avatar, the AI consistently stretched width without scaling height. Clothes ended up looking wider and shorter than they should — wrong in a way that would make every item look unflattering. I had to establish explicit constraints: scale proportionally from a fixed anchor point, centred on the figure, with transparency preserved so the avatar showed through correctly.

The images also need to be centred and on a transparent background. This sounds obvious in hindsight. It wasn't obvious when garments kept drifting off-centre or sitting on white boxes that obscured the avatar underneath.

### Guardrails mattered more than I expected

The guardrails are essentially the size 12 contract: the avatar is size 12, the garment baseline is size 12, and every adjustment — larger or smaller — is calculated relative to that shared reference. Without this fixed anchor, the AI drifted constantly: wrong proportions, inconsistent positioning, clothing that floated above the avatar or merged into it.

Establishing the guardrails wasn't a nice-to-have. It was what made the output usable rather than broken. And it's the part of AI product development that most demos skip — the failure modes are invisible until you build the constraints that prevent them.

### The output

The result is a size recommendation with a plain-English explanation — not a percentage confidence score, but a note a human would actually understand. "This will fit across the shoulders but run long in the body." That's what I'd want to read before deciding whether to buy something.

### Post-MVP

The full outfit creator — combining multiple garments and seeing how they work together — is planned for the next phase. The single-item flow needed to work reliably first.

---

## Why "concept prototype" is the right call

The image generation piece — producing photorealistic, accurate depictions of specific garments on specific body measurements — is a genuinely hard computer vision problem. Companies like Snap, Amazon, and several well-funded startups are working on it. Getting it wrong doesn't just produce bad UX; it produces images that could make people feel worse about their bodies. That's not a technical risk I want to rush.

This prototype builds and validates everything around that core: the measurement input, the garment data extraction, the fit logic, the privacy model, the outfit builder, the sharing flow. The image generation layer has a clear technical spec and is ready to be connected when the right model exists or becomes affordable to call via API.

Shipping a prototype that works for what it can do, and is honest about what it can't, is better product management than shipping something that overpromises.

---

## The market case (in brief)

- UK online clothing returns: ~£7 billion annually
- Average return rate for online fashion: 30–40%
- Retailers absorb ~60% of return costs; the rest falls on consumers and the environment
- ASOS, Next, and Zara have all publicly cited returns as a top strategic priority
- No current cross-retailer solution exists that is privacy-preserving and measurement-based

A tool that reduces a retailer's return rate by even 10% pays for itself many times over. The monetisation path is B2B2C: Clothes Rail as a white-label layer that retailers embed into their product pages.

---

## Status

✅ Measurement input and profile builder
✅ Avatar baseline (size 12 reference figure, vectorised for measurement-driven adjustment)
✅ Garment baseline system (size 12 starting point, proportional scaling to user measurements)
✅ Fit prediction logic and plain-English fit notes
✅ Size recommendation output — plain-English sizing advice based on measurement delta
✅ Share flow — export and share generated look
🔄 Cross-retailer size calibration (in progress)
📋 Planned: prototype thumbs up/down vote on fit preview
📋 Planned: post-receive follow-up email with one-click fit vote
📋 Planned: return rate tracking and eval against customer history, item aggregate, and average baseline
📋 Planned: photorealistic image generation when API capability matures
📋 Post-MVP: outfit builder — combine multiple garments into a single look

---

## What I'd build next

- **Eval pipeline** — ship the three-signal feedback loop (prototype vote → post-receive email → return tracking) and wire it into a dashboard showing prediction accuracy over time, per customer and per item
- **Browser extension** — run a fit check directly on any retail product page without leaving the site
- **Retailer API** — brands feed their size chart data directly, improving predict