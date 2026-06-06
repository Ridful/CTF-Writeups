# Culinary Circles — Writeup

> **CTF:** GPN CTF
> **Category:** Miscellaneous (OSINT + geometry)
> **Author:** Alkalem
> **Flag format:** `GPNCTF{.*}`
>
> *"Why do some people care so much about social circles? Let us focus on cuisine and peoples tastes instead. Talk with others about food and in rare cases your culinary circles might intersect."*
> *Note: Use OpenStreetMap as source of truth for the locations and their names. The solution is the location of a restaurant, formatted as flag, e.g. `GPNCTF{The French Laundry}`. Precision may be required.* **Takeout order.**

## TL;DR

The archive contains **nine** restaurant screenshots. Geolocate every one, then notice the file names group cleanly by first letter into **three triplets** (A / B / C). Three points uniquely define a circle, so each triplet draws one **circle on the globe**. The flavor text — *"in rare cases your culinary circles might intersect"* — is literal: the three circles all pass through a **single shared point** on the Isle of Skye, Scotland. The only restaurant sitting at that point on OpenStreetMap is **The Three Chimneys**.

```
GPNCTF{The Three Chimneys}
```

The *"Takeout order"* tag at the end is the author's wink — Skye is about as remote a place as you could possibly order food from.

---

## 1. Recon: what's in the box

The handout `culinary-circles.tar.gz` unpacks to nine PNGs:

```
adam.png  alice.png  alkalem.png  beatrice.png  bob.png
bruno.png  carol.png  catherine.png  cristoph.png
```

Each is a screenshot of a food place — some are Google Maps "place" cards with a photo, some are Street View frames, and two are deliberately awkward (an aerial shot and a cropped photo) to make the geolocation harder. There is no other data: the whole challenge is *find these nine spots, then figure out what to do with the coordinates.*

## 2. Step one — geolocate all nine (the OSINT grind)

Standard geo-OSINT: reverse image search, Google Lens, reading signage, license-plate/road-marking conventions, and matching architecture. A few were quick place-card lookups; three needed real work.

| Person | Restaurant | City / Country | Lat | Lon |
|---|---|---|---|---|
| **Alkalem** | Kippe 23 | Karlsruhe, DE | 49.00748 | 8.41930 |
| **Adam** | Hofu (sushi) | Køge, DK | 55.45690 | 12.17853 |
| **Alice** | The Peppermill | Nenagh, IE | 52.86181 | −8.19826 |
| **Beatrice** | BCHEF Burgers | Dijon, FR | 47.32267 | 5.03815 |
| **Bob** | Pastéis de Belém | Lisbon, PT | 38.69755 | −9.20324 |
| **Bruno** | Mairie Restaurant & Events | Berkel en Rodenrijs, NL | 51.99574 | 4.47555 |
| **Carol** | Skeppsbro Bakery | Stockholm, SE | 59.32471 | 18.07612 |
| **Catherine** | (high-street unit) | Armadale / Livingston area, UK | 55.89847 | −3.70242 |
| **Christoph** | Café Marina | Sønderborg, DK | 54.89936 | 9.79558 |

### The tricky three (and one trap)

- **bob** — the screenshot had its top cropped off to hide the trademark Lisbon azulejo storefront and the "blue" branding. Reverse image search on the remaining facade pinned it to **Pastéis de Belém**.
- **bruno** — a Dutch harbour-town Street View with construction gear (a "Boels" toilet cabin) in front. The faint road name at the edge plus the marina layout led to **Westersingel**, and the restaurant pin resolves to **Mairie Restaurant & Events** in Berkel en Rodenrijs. (Early guesses of Medemblik / Sittard were red herrings — trust the actual pin, not the vibe.)
- **catherine** — **the satellite view is a decoy.** The big aerial image only shows green countryside on a firth/estuary; the *real* pin is the tiny Street View thumbnail in the bottom-right corner (red-brick terraced high street, double-yellow lines, cars on the left → UK). The countryside + Firth of Forth narrowed it to the Edinburgh/Livingston area, and the thumbnail matched a shop unit there. In the screenshot, the mouse cursor was also conveniently placed over the location which helped narrow down the search for the streetview dramatically.
- **cristoph** — a marina café with an **Albani** parasol. Albani is a Danish brewery (Fyn/Odense), and the sign reads *"MARINA · Mad · Kaffe…"* — but following the harbour signage actually lands it at **Café Marina in Sønderborg**, southern Denmark.

> **Source-of-truth note:** the challenge explicitly says *use OpenStreetMap*. The Google pin coordinates above are good enough to get the circles into the right neighbourhood, but for a clean intersection you want the OSM node/way coordinate of each venue. Small offsets here are exactly why the final intersection is a tight *cluster* rather than a perfect point (see §5).

## 3. Step two — the "circles" insight

The flavor text hammers on *circles*, and the file names are the giveaway. Sort by first letter:

- **A:** **A**lkalem, **A**dam, **A**lice
- **B:** **B**eatrice, **B**ob, **B**runo
- **C:** **C**arol, **C**atherine, **C**hristoph

Three groups of exactly three. In geometry, **three points uniquely determine a circle** (the circumcircle). So each name-group defines one circle, and *"in rare cases your culinary circles might intersect"* tells you the three circles were placed to **share a common intersection point** — and that point is the answer.

This also kills the tempting wrong turn: the points are *not* pairs for trilateration, and the "radius" is *not* the distance between alphabetical pairs (Adam→Alice). They're **triplets → circumcircles**.

## 4. Step three — do it on a sphere, not a flat map

These points span ~2000 km across Europe. If you fit *planar* circles to `(lon, lat)` you'll get visibly wrong shapes and the circles won't concur. You must compute the circle as the intersection of a **plane with the unit sphere** — i.e. the spherical circumcircle.

For three points (as unit vectors `v1, v2, v3`), the plane through them has normal

```
n = normalize( (v2 − v1) × (v3 − v1) )
```

That normalized normal is the **pole** of the small circle (the spherical circumcenter), and the circle's angular radius is `r = arccos(n · v1)`. Every point `X` on that circle satisfies the single linear constraint

```
n · X = cos(r)
```

### Finding the shared point exactly

Each circle is a plane `n_i · X = cos(r_i)`. **Three planes meet at one point in 3-D.** So stack them and solve, then project back onto the sphere:

```
[ n_A ]        [ cos r_A ]
[ n_B ] · X =  [ cos r_B ]      →  X0 = M⁻¹ b ,   P = X0 / ‖X0‖
[ n_C ]        [ cos r_C ]
```

If the circles genuinely concur, the solution `X0` already lies on the sphere (`‖X0‖ ≈ 1`) and `P` is the intersection. That single check, `‖X0‖`, doubles as a confidence test that you have the right interpretation.

### Numbers from the solver

```
Circle A (Alkalem, Adam, Alice)         pole 53.683, 2.006   r ≈ 6.15°  (684 km)
Circle B (Beatrice, Bob, Bruno)         pole 48.123, −8.945  r ≈ 9.43°  (1048 km)
Circle C (Carol, Catherine, Christoph)  pole 61.252, 4.679   r ≈ 6.90°  (767 km)

‖X0‖ = 0.999973            ← circles really do concur
Triple intersection ≈ 57.4356, −6.5975  (Isle of Skye)
```

`‖X0‖` essentially equal to 1 is the "aha": this isn't a coincidence, the author placed nine restaurants so three globe-spanning circles would cross at one spot.

## 5. Step four — from a point to a flag

The computed intersection (and the cluster of pairwise A/B, B/C, A/C crossings) all land within a couple of kilometres of **57.443, −6.642** on the **Isle of Skye**. The residual spread (~1–2 km, and ~2.8 km from the venue) is purely OSINT precision — Google pin centres vs. exact OSM nodes — which is why the prompt nudges you toward OSM and warns *"precision may be required."*

Look that spot up on OpenStreetMap and there is exactly one restaurant there:

```
amenity=restaurant
name=The Three Chimneys
OSM: way/244132959   (≈ 57.443422, −6.641956)
```

<img width="1143" height="734" alt="image" src="https://github.com/user-attachments/assets/a262dcc7-f1aa-40f6-a4b3-2c0bc7f7b793" />


A famous fine-dining restaurant in Colbost, on Skye — and given how far it is from anywhere, the challenge's closing **"Takeout order"** is the punchline.

## Flag

```
GPNCTF{The Three Chimneys}
```

---

## Appendix A — gotchas / lessons

- **Don't fit flat circles.** At continental scale you must use spherical circumcircles, or nothing lines up.
- **It's triplets, not pairs.** The first-letter grouping is the entire mechanic; chasing pairwise distances/trilateration is a dead end.
- **Aerial = decoy (catherine).** When a screenshot shows a giant unhelpful satellite view, check the Street-View corner thumbnail — that's the real pin.
- **Trust the pin, not the first guess (bruno).** The town "vibe" sent people to Medemblik/Sittard; the actual answer was Berkel en Rodenrijs.
- **Use OSM for the final lookup.** Google coordinates get you to the right cluster; the named restaurant at the intersection comes from OSM, exactly as the prompt demands.

## Appendix B — solve script

This is the full, self-contained solver: spherical circumcircles, the exact triple-intersection via the three-plane system, and a sanity check against the venue. (A `folium` map render is included at the bottom to eyeball the intersection.)

```python
import math
import numpy as np

# --- spherical helpers -------------------------------------------------
def to_xyz(lat, lon):
    la, lo = math.radians(lat), math.radians(lon)
    return (math.cos(la)*math.cos(lo), math.cos(la)*math.sin(lo), math.sin(la))

def to_ll(v):
    x, y, z = v
    return (math.degrees(math.asin(z)), math.degrees(math.atan2(y, x)))

def sub(a, b):   return (a[0]-b[0], a[1]-b[1], a[2]-b[2])
def cross(a, b): return (a[1]*b[2]-a[2]*b[1], a[2]*b[0]-a[0]*b[2], a[0]*b[1]-a[1]*b[0])
def dot(a, b):   return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
def norm(a):
    m = math.sqrt(dot(a, a)); return (a[0]/m, a[1]/m, a[2]/m)

def spherical_circle(p1, p2, p3):
    """Return (pole n, angular radius r) of the small circle through 3 points."""
    v1, v2, v3 = to_xyz(*p1), to_xyz(*p2), to_xyz(*p3)
    n = norm(cross(sub(v2, v1), sub(v3, v1)))
    if dot(n, v1) < 0:                       # keep pole on the near hemisphere
        n = (-n[0], -n[1], -n[2])
    r = math.acos(max(-1, min(1, dot(n, v1))))
    return n, r

# --- the nine converged coordinates ------------------------------------
groups = {
    "A": [(49.00748, 8.41930), (55.45690, 12.17853), (52.86181, -8.19826)],   # Alkalem, Adam, Alice
    "B": [(47.32267, 5.03815), (38.69755, -9.20324), (51.99574, 4.47555)],    # Beatrice, Bob, Bruno
    "C": [(59.32471, 18.07612), (55.89847, -3.70242), (54.89936, 9.79558)],   # Carol, Catherine, Christoph
}

poles, coss = [], []
for name, pts in groups.items():
    n, r = spherical_circle(*pts)
    poles.append(n); coss.append(math.cos(r))
    print(f"Circle {name}: pole {to_ll(n)[0]:.3f},{to_ll(n)[1]:.3f}  r={math.degrees(r):.3f} deg")

# --- exact triple intersection: three planes  n_i . X = cos(r_i) -------
X0 = np.linalg.solve(np.array(poles), np.array(coss))
P  = X0 / np.linalg.norm(X0)
lat, lon = to_ll(tuple(P))
print(f"\n||X0|| = {np.linalg.norm(X0):.6f}  (==1 means the circles concur)")
print(f"Intersection ~ {lat:.6f}, {lon:.6f}  ->  Isle of Skye")
print("Nearest OSM restaurant: The Three Chimneys  =>  GPNCTF{The Three Chimneys}")

# --- optional: render the circles on a slippy map ----------------------
# import folium
# m = folium.Map(location=[55, 2], zoom_start=5, tiles="CartoDB positron")
# colors = {"A": "#e74c3c", "B": "#3498db", "C": "#2ecc71"}
# def hav(a, b, R=6371008.8):
#     la1, lo1 = map(math.radians, a); la2, lo2 = map(math.radians, b)
#     h = math.sin((la2-la1)/2)**2 + math.cos(la1)*math.cos(la2)*math.sin((lo2-lo1)/2)**2
#     return R*2*math.atan2(math.sqrt(h), math.sqrt(1-h))
# for name, pts in groups.items():
#     n, r = spherical_circle(*pts); c = to_ll(n)
#     folium.Circle(c, radius=hav(c, pts[0]), color=colors[name], fill=False).add_to(m)
#     for p in pts:
#         folium.Marker(p, icon=folium.Icon(color="black", icon="cutlery", prefix="fa")).add_to(m)
# m.save("culinary_intersection_map.html")
```
