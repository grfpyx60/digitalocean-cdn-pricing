# DigitalOcean CDN Pricing Explained: How the Built-In Spaces CDN Is Billed, How Much Bandwidth You Really Get, and How It Compares to CloudFront and Bunny CDN — With Full Plan Breakdown and Setup Walkthrough

If you've ever stared at a cloud bill that ballooned because of egress charges, you already know why "CDN pricing" is one of the most-Googled phrases in the infrastructure world. It's also why **DigitalOcean CDN pricing** keeps showing up in developer forums, Reddit threads, and side-by-side comparisons: the model is unusually simple, and for a lot of small-to-mid projects, it's genuinely hard to beat. This guide breaks down how DigitalOcean's built-in CDN is actually billed, what's included, where the limits bite, and how it stacks up against alternatives — so you can decide with real numbers instead of marketing copy.

## What the DigitalOcean CDN Actually Is

Here's the part that confuses people first: DigitalOcean doesn't sell a standalone CDN product the way CloudFront or Bunny CDN does. The CDN is baked into **Spaces Object Storage**, their S3-compatible storage service. Every Spaces bucket in a Standard Storage region can have the CDN enabled with a single toggle, and the edge layer caches your objects across a network of 200+ points of presence (PoPs) spanning North America, Europe, Asia, Latin America, the Middle East, Africa, and Oceania.

The practical implication: you don't pick a "CDN plan." You pick a Spaces storage plan, and the CDN comes along with it. That single fact shapes everything about how **DigitalOcean CDN pricing** works — and why it reads so differently from competitors' line-item bills.

## How DigitalOcean CDN Pricing Works: The Core Model

The headline number is $5 per month. That's the base subscription for **Spaces Standard Storage**, and it's the floor of what you'll pay regardless of whether you serve one byte or one terabyte through the CDN.

What that $5 gets you:

- **250 GiB of storage** across all your buckets
- **1,024 GiB (1 TiB) of outbound data transfer** per month — and this allowance covers both direct origin traffic *and* CDN-served traffic
- A built-in CDN with automatic Let's Encrypt TLS provisioning
- Custom subdomain support (e.g., `assets.yourdomain.com`)
- S3-compatible API access, so your existing tooling works unchanged

The critical detail: **the CDN itself adds no separate fee.** Traffic that flows through the CDN edge nodes counts against the same 1 TiB outbound transfer allowance that direct origin requests would. There's no "CDN request pricing," no per-HTTP-request surcharge, no separate edge-egress tier. This is the single biggest reason the bill stays low for typical workloads.

Once you blow past the included allowance, two overage rates apply:

- **Additional storage:** $0.02 per GiB per month
- **Additional outbound transfer:** $0.01 per GiB (roughly $10 per TiB)

Inbound bandwidth — uploads *into* Spaces — is always free and never counts against the allowance. That's a meaningful perk if you're regularly pushing backups or large datasets upstream.

### Full Plan Comparison

DigitalOcean's Spaces product line has two storage classes. Here's the complete breakdown as currently listed on the official pricing page:

| Plan | Storage Included | Outbound Transfer Included | Base Price | Overage Storage | Overage Transfer | CDN Support | Get Started |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Spaces Standard Storage** | 250 GiB | 1,024 GiB (1 TiB) | $5/month | $0.02/GiB/mo | $0.01/GiB | Yes — full CDN, custom subdomains, auto TLS | [Start with Spaces Standard](https://bit.ly/DigitaLocean) |
| **Spaces Cold Storage** | Pay-per-GiB (no base allotment) | Pay-per-GiB retrieval | $0.007/GiB/mo storage | — | $0.01/GiB retrieved | No — CDN integration not supported | [Start with Spaces Cold Storage](https://bit.ly/DigitaLocean) |

A few things worth noting on the Cold Storage row: it's designed for infrequently accessed data with a 30-day minimum retention period. Retrieval fees are waived for data retrieved up to your average daily Cold Storage usage for the month, and the first 250 GiB per month of early-deletion or update charges are free. Objects smaller than 128 KiB are billed as 128 KiB. Cold Storage buckets **cannot** have CDN enabled — if you need edge caching, that data needs to live in a Standard Storage bucket.

You can have up to 100 buckets per subscription, and billing is prorated hourly, so spinning one up for a quick test costs cents, not five dollars.

## A Real-World Cost Example

The flat-rate model only shines when you plug in actual numbers. Let's say you run a media-heavy site serving **5 TiB of assets per month** through the Spaces CDN, with 300 GiB stored.

Your monthly bill:

- Base subscription: **$5**
- Additional storage (300 GiB − 250 GiB = 50 GiB × $0.02): **$1**
- Additional transfer (5 TiB − 1 TiB = 4 TiB × $0.01/GiB ≈ 4,096 GiB × $0.01): **~$41**
- **Total: ~$47/month**

Run the same workload through AWS S3 + CloudFront in us-east-1 and you're looking at roughly $0.023/GB for storage plus $0.085/GB for the first 10 TB of CloudFront transfer — which lands somewhere around **$460/month** at 5 TB. That order-of-magnitude gap is the reason teams migrate static asset delivery to Spaces in the first place.

The math flips at higher volumes. Beyond roughly 10–15 TB/month, Spaces' flat $0.01/GB overage rate never volume-discounts, while dedicated CDN providers and CloudFront both introduce tiered pricing that drops the per-GB cost as you commit to more. At 50 TB/month and above, a tiered-CDN provider will usually win on delivery cost — though Spaces may still be the right *origin* layer, with a dedicated CDN layered in front.

## Bandwidth Allowance Nuances Most Guides Skip

The 1 TiB outbound allowance is shared across all your buckets and applies to **both** Standard and Cold Storage classes. A few edge cases worth knowing:

- **Traffic from Spaces to Droplets in the same region is free.** If your app server and your bucket both live in NYC3, that traffic doesn't touch your allowance. The same applies to same-region-group pairs like NYC1↔NYC3, AMS2↔AMS3, and SFO1↔SFO3.
- **VPC-local DNS resolver traffic is private.** Droplets configured to use the VPC-local DNS resolver access Spaces over DigitalOcean's internal network, bypassing public-endpoint billing entirely. Droplets using external DNS resolvers (Google, Cloudflare) resolve to the public endpoint and are billed normally.
- **Droplet outbound transfer is separate from Spaces transfer.** Traffic from a Droplet *to* Spaces doesn't count against Spaces (inbound is free), but it does count against the Droplet's own outbound allowance.

These details matter because they let you architect around the allowance. If your app pulls large objects from Spaces on every request, co-locating the app and the bucket in the same region can effectively make that traffic free — and the CDN handles the user-facing leg.

## How DigitalOcean CDN Pricing Compares to the Alternatives

The CDN market has a few distinct pricing philosophies. Here's how Spaces CDN fits in:

| Provider | Pricing Model | ~Cost at 1 TB/mo | ~Cost at 10 TB/mo | Notable Tradeoff |
| --- | --- | --- | --- | --- |
| **DigitalOcean Spaces CDN** | Flat $5/mo + $0.01/GB overage | $5 | ~$86 | Simple, no per-request fees; thin APAC/LATAM edge density |
| **AWS CloudFront** | Per-request + per-GB, tiered | ~$85 | ~$850 | Deep feature set, origin shield, Lambda@Edge; complex billing |
| **Bunny CDN** | Per-GB, region-tiered | ~$1–$5 | ~$10–$50 | Cheapest at scale; separate account/storage needed |
| **Cloudflare** | Free tier, Pro $20/mo, R2 storage $0.015/GB | $0 (free) or $20 | $0 (free) or $20 | No egress fees on R2; vendor lock-in concerns; fewer regions on free tier |

The pattern: at **low to mid volumes (under 10 TB/month)**, DigitalOcean Spaces CDN is dramatically cheaper than CloudFront and simpler to operate than stitching Bunny CDN in front of S3. At **high volumes (25 TB+/month)**, dedicated CDN providers with tiered pricing — Bunny, BlazingCDN, Cloudflare — start winning on raw delivery cost, and the architecture pattern becomes "use Spaces as origin, dedicated CDN as edge."

For hobby projects, static sites, and indie SaaS apps in the sub-1-TB range, the $5 flat fee is almost unbeatable. You'd have to work hard to spend more than the base rate.

## CDN Edge Network Coverage and Performance

Pricing only matters if the CDN actually delivers. As of the latest documentation, the Spaces CDN maintains PoPs across every inhabited continent:

- **North America:** 60+ locations across the US, Canada, and Mexico
- **Europe:** 58+ locations from Lisbon to Helsinki, including Istanbul and Tbilisi
- **Asia:** 90+ locations including mainland China, India, Southeast Asia, Japan, and Korea
- **Latin America & the Caribbean:** 50+ locations, heavy Brazil coverage
- **Middle East:** 20+ locations including Dubai, Riyadh, Tel Aviv
- **Africa:** 34 locations from Cape Town to Cairo
- **Oceania:** 13 locations across Australia, New Zealand, and Pacific islands

Independent benchmarks from 2026 testing report median TTFB of **28–38 ms to US East edge nodes**, **42–55 ms to EU West**, **65–80 ms to Singapore**, and **90–130 ms to South America** — with cache hit ratios of 92–96% in primary regions. Enabling CDN on a Space reduced median TTFB by **40–60%** for US and EU users compared to direct origin requests.

The honest caveat: edge density is thinner in Asia-Pacific and Latin America compared to providers with longer CDN pedigrees. If your audience skews heavily toward those regions, expect longer tail latencies and lower cache hit ratios — which is the scenario where layering a dedicated CDN in front of Spaces starts to make sense.

## Enabling the CDN: A Quick Walkthrough

Setting up the CDN on a Space takes under a minute:

1. **Create a Space** in your preferred region from the DigitalOcean control panel.
2. **Open the Space, click the Settings tab**, and find the CDN section.
3. **Click Enable CDN.** The endpoint provisions within seconds and issues a Let's Encrypt certificate automatically. Your CDN URL takes the form `<spacename>.<region>.cdn.digitaloceanspaces.com`.
4. **(Optional) Add a custom subdomain.** Create a DNS CNAME record (e.g., `assets`) pointing to the CDN subdomain. DigitalOcean handles TLS provisioning for the custom subdomain automatically. Note: apex-domain setups are awkward because the CDN requires a CNAME, not an ALIAS record.
5. **Set cache-control headers** on your objects. The default TTL is 3600 seconds; you can override per-object down to 600 seconds via `Cache-Control: max-age` headers.

The same operation works via the `doctl` CLI with a single command, which is handy for scripted setups.

### One Operational Constraint to Know

Purging is **per-file or full-flush only** — there's no purge-by-prefix or tag-based invalidation. If you deploy asset updates frequently and need granular cache busting, plan around this: use versioned filenames (`app.v123.js`) or accept full-cache flushes on deploy. This is a real limitation versus CloudFront, which supports path-based invalidation, and it's the operational detail most likely to bite teams used to more sophisticated CDN tooling.

## When DigitalOcean CDN Is the Right Pick — and When It Isn't

The decision mostly comes down to volume and audience geography.

**Spaces CDN is a strong fit when:**

- You're serving under 10 TB/month of static assets, media, or downloads
- Your primary audience is in North America or Europe
- You want one bill for storage + delivery instead of stitching multiple services
- You're already on DigitalOcean for Droplets, App Platform, or databases
- You value predictable flat pricing over micro-optimized per-request billing

**You should look elsewhere when:**

- You're delivering more than 25 TB/month and tiered CDN pricing would be cheaper
- Your audience is heavily APAC or Latin America and edge density matters
- You need origin shield, edge compute, or tag-based cache invalidation
- You require sub-50 ms TTFB to regions where DigitalOcean's PoP coverage is thin
- You need a standalone CDN in front of a non-Spaces origin

For the workloads in the first list, the value proposition is clean: 👉 [spin up a Space, flip the CDN toggle, and you're done for $5/month](https://bit.ly/DigitaLocean).

## Limitations Worth Stating Plainly

The Spaces CDN is a solid, simple product, but it's not pretending to be CloudFront. Here's what it doesn't offer:

- **No origin shield** — every edge miss goes back to the origin Space
- **No edge compute** — no Lambda@Edge equivalent, no edge-side includes
- **No tag-based or prefix-based cache invalidation** — per-file or full-flush only
- **No CDN on Cold Storage buckets** — CDN integration is Standard Storage only
- **Apex domain setups are awkward** — CNAME-only, no ALIAS support
- **Overage rates never volume-discount** — flat $0.01/GB regardless of how much you deliver

None of these are dealbreakers for the typical Spaces use case. They're the tradeoffs that come with a $5/month flat-fee model. If you're hitting one of these walls regularly, that's the signal to evaluate a dedicated CDN layered in front.

## Frequently Asked Questions

**Is the DigitalOcean CDN really free?**

The CDN layer itself adds no separate fee. You pay for Spaces storage ($5/month base, which includes 1 TiB of outbound transfer), and traffic through the CDN counts against that same transfer allowance. Above 1 TiB, you pay $0.01/GiB for additional transfer — same rate whether it goes through the CDN or directly from the origin.

**How much bandwidth do I get before overage charges kick in?**

1,024 GiB (1 TiB) of outbound transfer per month is included with the $5 Spaces Standard Storage subscription. The allowance is shared across all your buckets. Inbound bandwidth is always free.

**Does the CDN support custom domains?**

Yes. You can add a custom subdomain (e.g., `cdn.yourdomain.com`) by creating a CNAME record pointing to the CDN endpoint. DigitalOcean provisions TLS automatically via Let's Encrypt. Apex domains aren't well-supported because the CDN requires a CNAME rather than an ALIAS record.

**How does DigitalOcean CDN pricing compare to AWS CloudFront?**

At low volumes (under 1 TB/month), Spaces is dramatically cheaper — $5 flat versus $85+ on AWS for equivalent delivery. Between 1 and 10 TB, Spaces maintains a 5–8× cost advantage. Above 10 TB, the gap narrows because Spaces never volume-discounts while CloudFront introduces tiered pricing. At 50 TB/month and above, CloudFront (or a tiered dedicated CDN) will typically beat Spaces on per-GB delivery cost.

**Can I use the Spaces CDN with non-Spaces origins?**

No. The Spaces CDN is tied to Spaces Object Storage buckets. If you need a CDN in front of a Droplet, an external origin, or another storage provider, you'll need a dedicated CDN product (Bunny, CloudFront, Cloudflare, etc.).

**Does Cold Storage support the CDN?**

No. CDN integration and custom CDN endpoints are only available on Spaces Standard Storage buckets. Cold Storage is optimized for infrequently accessed data with a 30-day minimum retention period and isn't designed for edge-cached delivery.

**What's the cache hit ratio I can expect?**

Independent benchmarks report 92–96% cache hit ratios in US and EU primary regions for repeat-access static content. Ratios drop to 82–88% in APAC and 75–82% in South America due to thinner edge density in those regions.

**Are there per-request fees?**

No. Spaces billing is based on storage and outbound transfer only. There are no per-HTTP-request charges, no per-PUT/GET API request fees on the standard tier, and no separate CDN request pricing.

## The Bottom Line

**DigitalOcean CDN pricing** is one of those rare infrastructure stories where the simple version is also the true version: $5/month gets you 250 GiB of storage and 1 TiB of outbound transfer, the CDN is included at no extra cost, and overages are flat $0.01/GiB for transfer and $0.02/GiB for storage. No per-request fees, no tiered edge pricing, no spreadsheet required to estimate your bill.

For the majority of small-to-mid web projects — static sites, indie SaaS apps, media delivery under 10 TB/month, backups, and developer tooling — this model is hard to beat. It eliminates the egress-gouging problem that makes AWS bills unpredictable, and it keeps storage and delivery on a single invoice.

Where it stops being the right answer: high-volume delivery (25 TB+/month) where tiered CDN pricing wins on cost, and globally distributed audiences in APAC or Latin America where DigitalOcean's edge density is thinner than dedicated CDN competitors. In those cases, the architecture pattern is to keep Spaces as your origin and put a dedicated CDN in front — which is itself a perfectly reasonable setup that plenty of production teams run.

If you're starting from scratch or re-evaluating an overpriced CDN bill, 👉 [the $5/month Spaces plan is the cheapest real-world test you can run](https://bit.ly/DigitaLocean) — and unlike most cloud "free trials," the pricing model stays the same once you're past the trial. Spin one up, push your assets through the CDN, and let the actual bill tell you whether it fits your workload.
