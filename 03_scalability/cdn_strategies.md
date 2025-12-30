
# CDN Strategies for Mobile Apps
*Gentle-Networking-Notes · 03_scalability*

> Designing a CDN strategy that balances **binary size, network cost, flexibility, reliability, and Apple-native distribution advantages**.

---

## The Mobile CDN Paradox

Mobile teams want:

* Tiny App Store download size
* Massive, rich content
* Instant first-launch experience
* Minimal backend & CDN spend

These goals conflict. Your CDN architecture determines whether users love your app — or uninstall it on first launch.

---

## Apple's “Free CDN” You Are Probably Ignoring

Apple already runs one of the largest CDNs in the world:

| Feature | Tool |
|-------|------|
| On-demand assets | **On-Demand Resources (ODR)** |
| Background updates | **App Store background asset delivery** |
| Differential updates | **App thinning / slicing** |
| System prioritization | OS-managed scheduling |

ODR traffic is delivered via Apple's infrastructure at **zero incremental CDN cost**.

---

## When to Prefer Apple ODR vs Third-Party CDN

| Use Case | Apple ODR | Cloudflare / Fastly |
|---------|-----------|--------------------|
| App-bundled art | ✅ Ideal |
| Localized media packs | ✅ Ideal |
| User-generated content | ❌ No |
| Personalized payloads | ❌ No |
| Rapid hotfix updates | ⚠ Limited |

Rule: **If content can ship with the app, let Apple pay for the CDN.**

---

## The Fallback Content Principle

Never require the network for first launch.

Ship a **minimal essential content pack** inside the binary:

| Asset | Purpose |
|-------|--------|
| Placeholder images | Avoid empty UI |
| Default config | Prevent crashes |
| Offline onboarding | First-time UX |

If CDN fetch fails, users should not notice.

---

## Progressive Content Quality

Offer tiered payloads:

| Network | Asset Tier |
|--------|------------|
| Offline / Low Data | Ultra-light placeholders |
| Cellular | Medium-res compressed |
| WiFi + Charging | Full fidelity |

This avoids the “10MB hero image over LTE” disaster.

---

## Failure Modes You Must Design For

| Failure | Symptom |
|--------|--------|
| CDN region outage | Blank screens |
| Partial downloads | Corrupted cache |
| Captive portals | Infinite spinners |
| OS background kill | Stuck updates |
| Disk pressure | Silent asset purge |

Every CDN fetch must be:

* Idempotent
* Resume-capable
* Versioned
* Verifiable (checksum / ETag)

---

## CDN Cost vs App Size

| Strategy | App Size | Flexibility | Cost |
|---------|----------|-------------|------|
| Bundle everything | Huge | None | $0 |
| ODR tiered packs | Small | Medium | $0 |
| External CDN | Tiny | High | $$$ |

For most apps, the winning formula is:

**Binary + ODR for baseline → CDN for dynamic content only.**

---

## Cache Headers Matter

```
Cache-Control: public, max-age=86400, stale-if-error=604800
ETag: "asset-v18"
```

This prevents redownload storms after transient failures.

---

## Observability

Track:

* % of sessions blocked by content fetch
* Mean time to interactive (TTI)
* CDN error codes by carrier

If >3% of users wait on content, you have a product problem.

---

## Key Concepts

> “Your CDN should be invisible. If users notice loading, your strategy is wrong.”

---

*Part of Gentle-Networking-Notes · 03_scalability*
