---
name: makephotos
description: Generate studio-quality product photos with AI. Upload images, generate professional shots, remove backgrounds, and manage credits via the MakePhotos API.
requiredEnv:
  - MAKEPHOTOS_API_KEY
permissions:
  - network: Access makephotos.ai API for photo generation, background removal, uploads, and credits
  - filesystem: Write generated scripts and output files
source:
  url: https://makephotos.ai
  author: MakePhotos (@makephotos)
  github: https://github.com/makephotos-ai/skill
  verified: true
security:
  note: API key is used only to authenticate with makephotos.ai. All processing happens server-side at makephotos.ai. Credentials never leave the local machine or get sent to third parties.
---

# MakePhotos — AI Product Photography

MakePhotos turns ordinary product images into studio-quality photos using AI. Upload a product image, pick a style, and get back a professional shot — no photographer, no studio, no editing. It also supports background removal and image uploads via signed Cloudinary URLs.

## When to Use This

Use this skill when the user wants to:

- Generate professional product photos from existing images
- Create studio, Amazon, luxury, or lifestyle shots for e-commerce
- Remove backgrounds from product images
- Upload local images for AI processing
- Check their MakePhotos credit balance
- Automate product photography workflows

Keywords: product photography, AI photos, studio shots, e-commerce images, background removal, product images, photo generation.

## Setup

### Get an API Key

1. Sign up at [makephotos.ai/signup](https://makephotos.ai/signup)
2. Subscribe to a plan (API calls consume credits from your subscription)
3. Go to the [developer dashboard](https://makephotos.ai/platform/developers) and copy your API key

### Set the Environment Variable

```bash
export MAKEPHOTOS_API_KEY=mk_live_...
```

The API key starts with `mk_live_`. Keep it server-side only — never expose it in client-side browser code.

## Usage: SDK (Recommended)

The `makephotos` npm package is the easiest way to interact with the API.

### Install

```bash
npm install makephotos
```

### Initialize the Client

```typescript
import MakePhotos from 'makephotos';

const client = new MakePhotos({
  apiKey: process.env.MAKEPHOTOS_API_KEY,
});
```

### Generate a Photo

Transform a product image into a professional studio shot.

```typescript
const result = await client.generate.create({
  imageUrl: 'https://example.com/product.jpg',
  type: 'product',
  style: 'studio',
  aspectRatio: '1:1',   // optional
  resolution: '1K',     // optional
  prompt: 'on a marble countertop with soft lighting', // optional
});

console.log(result.resultUrl);        // Cloudinary URL of the generated image
console.log(result.creditsUsed);      // e.g. 1
console.log(result.creditsRemaining); // { monthly, topup, total }
```

### Remove Background

```typescript
const result = await client.removeBg.create({
  imageUrl: 'https://res.cloudinary.com/.../image.jpg',
});

console.log(result.resultUrl);
```

### Check Credits

```typescript
const credits = await client.credits.get();

console.log(credits.total);            // total available credits
console.log(credits.monthlyRemaining); // monthly pool remaining
console.log(credits.topupRemaining);   // top-up pool remaining
console.log(credits.tier);             // starter | pro | max | ultra
```

### Upload a Local Image

Upload a file to get a URL you can pass to the generate endpoint. Uses signed uploads — no file size limit.

```typescript
import { readFileSync } from 'fs';

const uploaded = await client.upload.create({
  file: readFileSync('./photo.jpg'),
  filename: 'photo.jpg',
});

const result = await client.generate.create({
  imageUrl: uploaded.imageUrl,
  type: 'product',
  style: 'lifestyle',
});
```

## Usage: REST API (Alternative)

For quick one-off calls or non-Node.js environments, hit the REST API directly. Pass the API key as the `key` query parameter.

### Generate a Photo

```bash
curl -X POST "https://makephotos.ai/api/v1/generate?key=$MAKEPHOTOS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "image_url": "https://example.com/product.jpg",
    "type": "product",
    "style": "studio",
    "aspect_ratio": "1:1",
    "resolution": "1K"
  }'
```

Response:

```json
{
  "result_url": "https://res.cloudinary.com/.../generated.jpg",
  "credits_used": 1,
  "credits_remaining": { "monthly": 49, "topup": 10, "total": 59 }
}
```

Generation takes 10-20 seconds. The endpoint has a 60-second timeout.

### Remove Background

```bash
curl -X POST "https://makephotos.ai/api/v1/remove-bg?key=$MAKEPHOTOS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "image_url": "https://res.cloudinary.com/.../image.jpg" }'
```

Costs 10 credits per image. The `image_url` must be a Cloudinary URL.

### Check Credits

```bash
curl -s "https://makephotos.ai/api/v1/credits?key=$MAKEPHOTOS_API_KEY"
```

Response:

```json
{
  "monthly_remaining": 49,
  "topup_remaining": 10,
  "total": 59,
  "credits_limit": 50,
  "tier": "pro",
  "status": "active"
}
```

### Upload (Direct)

For small files (up to ~4.5 MB). Uses multipart/form-data, not JSON.

```bash
curl -X POST "https://makephotos.ai/api/v1/upload?key=$MAKEPHOTOS_API_KEY" \
  -F "file=@./photo.jpg"
```

Response:

```json
{
  "image_url": "https://res.cloudinary.com/.../photo.jpg"
}
```

### Upload (Signed, for Large Files)

For larger files, get signed Cloudinary upload params first, then upload directly to Cloudinary.

Step 1 -- Get signed params:

```bash
curl -s "https://makephotos.ai/api/v1/upload/signed?key=$MAKEPHOTOS_API_KEY"
```

Response:

```json
{
  "signature": "...",
  "timestamp": 1713200000,
  "api_key": "...",
  "cloud_name": "...",
  "folder": "makephotos/inputs/..."
}
```

Step 2 -- Upload to Cloudinary using those params:

```bash
curl -X POST "https://api.cloudinary.com/v1_1/{cloud_name}/image/upload" \
  -F "file=@./photo.jpg" \
  -F "api_key={api_key}" \
  -F "timestamp={timestamp}" \
  -F "signature={signature}" \
  -F "folder={folder}"
```

The `secure_url` from Cloudinary's response is your `image_url` for the generate endpoint. Uploads do not consume credits.

## Parameters Reference

### Generate Parameters

| Parameter | Required | Values |
|-----------|----------|--------|
| `imageUrl` / `image_url` | Yes | URL of the source image (must be publicly accessible) |
| `type` | Yes | `product` (only enabled type currently) |
| `style` | Yes | `studio`, `amazon`, `luxury`, `lifestyle` |
| `aspectRatio` / `aspect_ratio` | No | `1:1` (default), `4:3`, `3:4`, `16:9`, `9:16` |
| `resolution` | No | `1K` (default), `2K`, `4K` |
| `prompt` | No | Custom prompt to override the default style prompt |

### Styles

- **studio** — Clean white/neutral background, professional product shot
- **amazon** — Optimized for Amazon product listings (white background, centered)
- **luxury** — Premium feel with dramatic lighting and rich textures
- **lifestyle** — Product in a natural, lifestyle context

### Credit Costs

- **Generate:** 1 credit base cost, multiplied by resolution (1K = 1x, 2K = 2x, 4K = 4x). A 4K generation costs 4 credits.
- **Remove background:** 10 credits flat per image.
- **Upload:** Free (no credits consumed).

There are no rate limits. Credits are the throttle.

## Error Handling

All API errors include a `code`, `message`, `status`, and optional `details`.

```typescript
import MakePhotos, { MakePhotosError, InsufficientCreditsError } from 'makephotos';

try {
  await client.generate.create({ /* ... */ });
} catch (err) {
  if (err instanceof InsufficientCreditsError) {
    console.log('Need more credits:', err.details);
  } else if (err instanceof MakePhotosError) {
    console.log(`API error [${err.code}]: ${err.message}`);
  }
}
```

### Error Codes

| Code | Status | Description |
|------|--------|-------------|
| `missing_api_key` | 401 | No API key provided |
| `invalid_api_key` | 401 | Key is invalid or revoked |
| `no_active_subscription` | 403 | No active subscription |
| `insufficient_credits` | 402 | Not enough credits |
| `invalid_params` | 400 | Missing or invalid parameters |
| `generation_failed` | 502 | AI generation failed (credits are refunded) |
| `internal_error` | 500 | Server error |

## Credits System

MakePhotos uses a credit-based system. Each generation consumes credits from your subscription plan.

- **Tiers:** starter, pro, max, ultra
- **Pools:** monthly (resets each billing cycle) + top-up (purchased separately, never expire)
- Monthly credits are consumed first, then top-up credits
- **Subscription status:** `active`, `trialing`, `past_due`, `canceled`

Always check credits before running batch generation jobs. If credits run out mid-batch, the API returns `insufficient_credits` (402). If a generation fails server-side, credits are automatically refunded.

## Constraints

- An active MakePhotos subscription is required for all API calls
- API key must start with `mk_live_`
- The `imageUrl` for generate must be a publicly accessible URL
- The `imageUrl` for removeBg must be a Cloudinary URL
- Use the upload endpoint to convert local files into usable URLs
- Keep API keys server-side only — never expose in browser code

## Links

- [Website](https://makephotos.ai)
- [Developer Dashboard](https://makephotos.ai/platform/developers)
- [SDK on npm](https://www.npmjs.com/package/makephotos)
- [SDK on GitHub](https://github.com/makephotos-ai/sdk)
- [Sign Up](https://makephotos.ai/signup)
