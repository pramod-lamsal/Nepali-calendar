
## 1. Overview

This document describes the implementation of the **Sūrya-Siddhānta** planetary longitude computation used in the Nepali Pañcāṅga engine. The implementation follows the classical epicycle system with **manda** (apsidal) and **śīghra** (conjunction) corrections, using the iterative **Asakṛt-vidhi** method for convergence and Nilakantha Somayāji's 16th-century revision for the interior planets.

**References:**
- *Sūrya-Siddhānta* — Burgess translation (1860)
- *Tantrasaṅgraha* of Nīlakaṇṭha Somayāji (1500 CE)
- *Pañcasiddhāntikā* of Varāhamihira (6th century CE)
- *Āryabhaṭīya* of Āryabhaṭa (499 CE)

---

## 2. Mean Motion Parameters

### 2.1 Revolutions per Mahāyuga

The Mahāyuga is 4,320,000 years. Civil days per Mahāyuga = **1,577,917,828** (Sūrya-Siddhānta I.29–34).

| Planet | Revolutions per Mahāyuga |
|---|---|
| Mangala (Mars) | 2,296,832 |
| Budha (Mercury, śīghra) | 17,937,060 |
| Bṛhaspati (Jupiter) | 364,220 |
| Śukra (Venus, śīghra) | 7,022,376 |
| Śani (Saturn) | 146,568 |
| Rāhu (mean node) | 232,238 |

**Mean longitude:**
```
L_mean = norm360( (rev × ahargana / CIVIL_DAYS_MAHAYUGA) × 360 )
```

### 2.2 Mandocca (Apogee) Motion

Revolutions per Kalpa (1000 Mahāyugas) — Sūrya-Siddhānta II.1–2:

| Planet | Rev. per Kalpa | Epoch Longitude (Kali Yuga) |
|---|---|---|
| Mangala | 204 | 110° |
| Budha | 368 | 220° |
| Bṛhaspati | 900 | 160° |
| Śukra | 535 | 80° |
| Śani | 39 | 240° |

```
θ_mandocca = norm360( epoch + (rev_per_kalpa × ahargana × 360) / (1000 × CIVIL_DAYS_MAHAYUGA) )
```

---

## 3. Epicycle Dimensions (Paridhi)

### 3.1 Manda Paridhi (Equation of Centre)

From Sūrya-Siddhānta II.35:

| Planet | Even Quadrant (0°/180°) | Odd Quadrant (90°/270°) |
|---|---|---|
| Sun | 14° 0′ | 13° 40′ |
| Moon | 32° 0′ | 31° 40′ |
| Mangala | 69° 0′ | 68° 0′ |
| Budha | 28° 0′ | 27° 0′ |
| Bṛhaspati | 33° 0′ | 32° 0′ |
| Śukra | 12° 0′ | 11° 0′ |
| Śani | 49° 0′ | 48° 0′ |

### 3.2 Śīghra Paridhi (Conjunction)

From Sūrya-Siddhānta II.36–37:

| Planet | Even Quadrant | Odd Quadrant |
|---|---|---|
| Mangala | 235° 0′ | 232° 0′ |
| Budha | 133° 0′ | 132° 0′ |
| Bṛhaspati | 70° 0′ | 69° 0′ |
| Śukra | 262° 0′ | 260° 0′ |
| Śani | 39° 0′ | 38° 0′ |

### 3.3 Corrected Paridhi (Sphuṭa Paridhi)

The epicycle dimension varies with the anomaly. The correction is proportional to the sine of the anomaly:

```javascript
function sphutaParidhi(kendra, evenVal, oddVal) {
    const bhuja = Math.abs(((kendra % 180) + 180) % 180); // 0..180
    const bhujajya = Math.abs(Jya.jya(bhuja)) / Jya.R;    // 0..1
    return evenVal + (oddVal - evenVal) * bhujajya;
}
```

---

## 4. The Asakṛt-Vidhi Iterative Method

The **Asakṛt-vidhi** (literally "not just once") is the iterative procedure used to solve the interdependence between the epicycle radius and the hypotenuse (karṇa).

### 4.1 Manda Correction (Equation of Centre)

**Step 1 — Manda Kendra:**
```
K_manda = norm360(θ_mandocca − θ_planet)
```

**Step 2 — Bhujajyā (sine of kendra):**
```
bhujajya = |Jya.jya(K_manda)| / R
```

**Step 3 — Corrected paridhi (sphuṭa paridhi):**
```
P = P_even + (P_odd − P_even) × bhujajya
```

**Step 4 — Karṇa (hypotenuse):**
```
r₀ = (P / 360) × R
K = √(R² + r₀² + 2Rr₀·cos(K_manda))
```

**Step 5 — Manda-phala (correction):**
```
phala = arcsin( (r₀ × sin(K_manda)) / K )
```

**Step 6 — Apply correction:**
```
θ_manda-sphuṭa = norm360(θ_planet + phala)
```

### 4.2 Śīghra Correction

**Step 1 — Śīghra Kendra:**
```
K_śīghra = norm360(θ_śīghrocca − θ_manda-sphuṭa)
```

**Step 2 — Corrected śīghra paridhi:**
```
P = P_even + (P_odd − P_even) × bhujajya(K_śīghra)
```

**Step 3 — Karṇa:**
```
r₀ = (P / 360) × R
K = √(R² + r₀² + 2Rr₀·cos(K_śīghra))
```

**Step 4 — Śīghra-phala:**
```
phala = arcsin( (r₀ × sin(K_śīghra)) / K )
```

**Step 5 — True longitude:**
```
θ_true = norm360(θ_manda-sphuṭa + phala)
```

---

## 5. Four-Step Process for Exterior Planets

For Mars, Jupiter, and Saturn, the classical texts prescribe a **four-step process**: half-manda → half-śīghra → full-manda → full-śīghra.

**Rationale:** The manda-sphuṭa corresponds to the heliocentric longitude of the exterior planet.

### Step-by-Step Algorithm

```javascript
function planetLongitudeExterior(planetKey, jd) {
    const meanLon   = meanPlanetSidereal(planetKey, jd);
    const mandocca  = meanMandoccaSidereal(planetKey, jd);
    const meanSun   = trueSunSS(jd).true;
    const R = Jya.R;

    // Step 1: Half-manda correction
    const k1 = norm360(mandocca - meanLon);
    const m1 = mandaPhala(k1, MANDA_PARIDHI[planetKey]);
    const θ1 = norm360(meanLon + m1 / 2);

    // Step 2: Half-śīghra correction
    const s1 = sighraPhala(θ1, meanSun, SIGHRA_PARIDHI[planetKey]);
    const θ2 = norm360(θ1 + s1 / 2);

    // Step 3: Full-manda correction
    const k3 = norm360(mandocca - θ2);
    const m3 = mandaPhala(k3, MANDA_PARIDHI[planetKey]);
    const θ_manda = norm360(meanLon + m3);

    // Step 4: Full-śīghra correction
    const s4 = sighraPhala(θ_manda, meanSun, SIGHRA_PARIDHI[planetKey]);
    const trueLon = norm360(θ_manda + s4);

    return trueLon;
}
```

**Note:** The four-step process has the effect of changing the arc MA from the mean planet to the true planet, and the first two operations approximate the heliocentric orbit.

---

## 6. Interior Planets — Nilakantha's Revision

Nilakantha Somayāji (c. 1444–1550 CE) proposed a fundamental revision for Mercury and Venus in his *Tantrasaṅgraha* (1500 CE).

**Traditional model:** Manda correction applied to the mean Sun.

**Nilakantha's revision:** Manda correction applied to **what was traditionally called the śīghrocca of the planet** — which for interior planets corresponds to the mean heliocentric planet.

This was a radical departure: the manda correction for an interior planet was previously applied to the mean Sun, but Nilakantha argued it should be applied to the planet itself.

### Algorithm for Interior Planets

```javascript
function planetLongitudeInterior(planetKey, jd) {
    const meanLon  = meanPlanetSidereal(planetKey, jd);
    const mandocca = meanMandoccaSidereal(planetKey, jd);
    const meanSun  = trueSunSS(jd).true;
    const R = Jya.R;

    // Step 1: Manda correction applied to mean planet (Nilakantha)
    const kM = norm360(mandocca - meanLon);
    const mM = mandaPhala(kM, MANDA_PARIDHI[planetKey]);
    const mandaSphuta = norm360(meanLon + mM);

    // Step 2: Śīghra correction with mean Sun as śīghrocca
    const kS = norm360(meanSun - mandaSphuta);
    const sS = sighraPhala(mandaSphuta, meanSun, SIGHRA_PARIDHI[planetKey]);
    let trueLon = norm360(mandaSphuta + sS);

    // Special corrections
    if (planetKey === 'budh') {
        // Mercury: additional Sun's apogee correction
        // Sūrya-Siddhānta II.44
        const sunApogee = trueSunSS(jd).mandocca;
        const diff = norm360(meanSun - sunApogee);
        const corr = mandaPhala(diff, SUN_MANDA_PARIDHI.even);
        trueLon = norm360(trueLon + corr);
    } else if (planetKey === 'shukra') {
        // Venus: subtract 1°07' (67 arc-minutes)
        // Sūrya-Siddhānta II.45
        trueLon = norm360(trueLon - 1.1167);
    }

    return trueLon;
}
```

**Result:** Nilakantha's revised planetary model closely approximates the Keplerian model, except that the planets go in eccentric orbits around the mean Sun rather than the true Sun.

---

## 7. Rāhu and Ketu

**Mean node:** 232,238 revolutions per Mahāyuga (Sūrya-Siddhānta I.34).

```javascript
function rahuLongitude(jd) {
    const REV_RAHU = 232238;
    const ah = ahargana(jd);
    const totalRev = REV_RAHU * ah / CIVIL_DAYS_MAHAYUGA;
    const frac = totalRev - Math.floor(totalRev);
    return norm360(180 - frac * 360); // retrograde
}
// Ketu = Rāhu + 180°
```

---

## 8. Complete Parameter Reference

### 8.1 Mean Motion (Revolutions per Mahāyuga)

```javascript
const REV_PER_MAHAYUGA = {
    mangal:     2296832,
    budh:      17937060,
    brihaspati:  364220,
    shukra:     7022376,
    shani:       146568,
};
const CIVIL_DAYS_MAHAYUGA = 1577917828;
```

### 8.2 Mandocca Motion (Revolutions per Kalpa)

```javascript
const MANDOCCA_REV_PER_KALPA = {
    mangal:     204,
    budh:       368,
    brihaspati: 900,
    shukra:     535,
    shani:       39,
};
const MANDOCCA_EPOCH = {
    mangal:     110.0,
    budh:       220.0,
    brihaspati: 160.0,
    shukra:      80.0,
    shani:      240.0,
};
```

### 8.3 Manda Paridhi (degrees)

```javascript
const MANDA_PARIDHI = {
    mangal:     69.0,
    budh:       27.0,  // 28° even, 27° odd
    brihaspati: 34.5,  // 33° even, 32° odd → 34.5° average
    shukra:     12.0,
    shani:      39.0,  // 49° even, 48° odd → 39° average
};
```

### 8.4 Śīghra Paridhi (degrees)

```javascript
const SIGHRA_PARIDHI = {
    mangal:     { even: 238, odd: 237 },
    budh:       { even: 139, odd: 140 },
    brihaspati: { even:  70, odd:  69 },
    shukra:     { even: 261, odd: 262 },
    shani:      { even:  39, odd:  38 },
};
```

---

## 9. Summary Table

| Planet | Manda Paridhi | Śīghra Even | Śīghra Odd | Rev/Mahāyuga | Mandocca Rev/Kalpa |
|---|---|---|---|---|---|
| Mars | 69° | 235° | 232° | 2,296,832 | 204 |
| Mercury | 28°/27° | 133° | 132° | 17,937,060 | 368 |
| Jupiter | 33°/32° | 70° | 69° | 364,220 | 900 |
| Venus | 12° | 262° | 260° | 7,022,376 | 535 |
| Saturn | 49°/48° | 39° | 38° | 146,568 | 39 |

---

## 10. License and Attribution

This implementation is based on public-domain translations of classical Sanskrit astronomical texts. The algorithms and parameters are documented from:

- *Sūrya-Siddhānta* — translated by Rev. Ebenezer Burgess (1860)
- *Tantrasaṅgraha* of Nīlakaṇṭha Somayāji — translated by K. Ramasubramanian & M.S. Sriram (Springer, 2011)
- *Pañcasiddhāntikā* of Varāhamihira — translated by Otto Neugebauer & David Pingree (1970–71)
- *Āryabhaṭīya* of Āryabhaṭa — translated by Kripa Shankar Shukla (1976)

The code implementation is original and may be licensed under MIT or any compatible open-source license.
