# Procedurality

An open-universe observatory that runs entirely in the browser. You don't pilot a ship or play a character. You drift through an infinite, seeded universe and watch it: stars, planets, moons, climates, rivers, alien ecosystems. The universe keeps running whether you're watching or not.

**Play online:** https://pirata385.github.io/procedurality/

**Play locally:** open `index.html` in any modern browser. No server, build step, install or network access is needed. The whole game is one self-contained file, and it also works from `file://`.

Every push to `main` redeploys the site through `.github/workflows/pages.yml`.

## What's in it

| Scale | What you see |
|---|---|
| **Universe** | An effectively infinite 2D starfield laid out in 50-ly sectors. Filaments, clusters and voids come from density noise, procedural nebulae from domain-warped Simplex fBm, and named galactic regions. Star classes run O–M, plus red giants, blue supergiants, white and brown dwarfs, pulsars and black holes. |
| **Star system** | Animated WebGL star surfaces (granulation, spots, corona) and Keplerian orbits with eccentricity. Also companion stars, asteroid and Kuiper belts, comets with tails, and habitable-zone and frost-line overlays. |
| **Planet** | A WebGL globe with bump-mapped relief, ocean specular, drifting clouds, atmospheric scattering, rings, moons, lava glow and city lights on the night side. There are 12 archetypes: terrestrial, ocean, desert, frozen, volcanic, toxic, hydrocarbon, carbon, crystalline, barren, gas giant and ice giant. |
| **Surface** | A zoomable, streaming map of the whole sphere, with detail refined as you zoom (up to 15 noise octaves). It shows biomes, rivers, lakes, salt flats, dunes, vegetation, resource deposits and landmarks. Creatures move around it, and day/night and weather follow real planetary time. |
| **Life** | Species carry 48-byte genomes (192 bases) grouped into clades. Genes are expressed into 13 body plans and adapted to gravity, climate and starlight. Each world has a food web and deterministic population cycles, including booms and crashes. |

### Other features

- **Autonomous time.** One real second is one universe hour, derived from the wall clock. Orbits, planet spin, seasons, day/night and animal populations are functions of absolute time, so they keep evolving while you're away. You can pause, speed up to 1 yr/s, or resync to live time.
- **Seeds and codes.** Every place has an exact location code, and codes work as share links (`index.html#CODE`).
- **Search and scan.** Spiral outward from where you are, looking for worlds that match any combination of 24 criteria (life, sapience, ruins, Earth-like, ringed, black holes and more) or a resource.
- **Seed lab.** Grow a whole star system from any word, and optionally keep trying variants until a target such as "sapient species" shows up.
- **Discoveries.** Bookmarks with names and notes, travel history, statistics, and JSON export/import via clipboard or file. Everything is stored in `localStorage`.
- **Xenobiology codex.** Species are catalogued automatically as you observe living worlds.
- **Genesis tab.** Shows the full seed chain and the noise, climate and hydrology parameters. A "Verify determinism" button rebuilds the world twice and compares checksums.

## Controls

| Action | Mouse / keyboard | Touch |
|---|---|---|
| Pan / rotate | drag, `WASD` / arrows | one-finger drag (with inertia) |
| Zoom | wheel, `+` / `−` | pinch |
| Inspect | click | tap |
| Go deeper | double-click, `Enter` | double-tap |
| Back | `Esc`, `Backspace`, breadcrumbs, browser back | ‹ button, breadcrumbs, system back |
| Search / seed lab / codex | `/` · `G` · `C` | dock buttons |
| Save current place | `B` | bookmark icon in the panel |
| Time | `Space` pause · `,` `.` slower/faster · `L` live | ▶ time menu |
| Toggle panel | `P` | swipe the bottom sheet |

## Location codes

```
ORIGIN@600,-200                  a point in deep space        (universe@x,y in light-years)
ORIGIN:12:-4:3                   a star system                (universe:sectorX:sectorY:star)
ORIGIN:12:-4:3/2                 its third planet             (0-based)
ORIGIN:12:-4:3/2.1               that planet's second moon
ORIGIN:12:-4:3/2@12.50,-44.20    a landing site               (latitude, longitude)
~HELLO-WORLD/1                   a planet in a system grown from the text seed "HELLO-WORLD"
```

Changing the universe name in Settings (e.g. `ANDROMEDA-7`) gives you a different infinite universe. Codes carry their universe name, so shared links always resolve correctly.

## How it works

The single file has two `<script>` blocks.

1. **Generation core** (`<script id="gen-src">`). This is pure, DOM-free code:
   - `CORE`: math, hashing, the sfc32 RNG seeded through splitmix32 with `fork()` streams, colour, a scheduler
   - `NOISE`: seeded Simplex 2D/3D, fBm, ridged noise
   - `NAMES`
   - `ASTRO`: stars, systems, planets, orbits, time
   - `BIOMES`
   - `SURFACE`: heightfields, climate, hydrology, textures, features, tiles

   The same source is also loaded into a **Web Worker** through a Blob URL, so planet surveys build off the main thread. If workers are unavailable, a time-sliced main-thread scheduler takes over.
2. **Presentation.** `LIFE`, `CREATURES`, `RENDER` (WebGL globe and star shaders plus a software fallback), `VIEWS` (Galaxy, System, Planet, Surface), `UI`, `STORE`, `SEARCH`, `APP`.

### Determinism

Every generator takes a 32-bit seed and forks independent RNG streams for each concern (star, orbits, moons, life, palette…). Adding a feature to one stream never shifts the others. The chain is: universe name → hash → sector hash → star/system seed → planet seed → terrain, climate and cloud noise seeds → life seed → clade and species genomes. Because terrain is sampled from 3D noise on the unit sphere, the globe texture, the survey map and every surface tile read the same underlying field at different resolutions.

### Planet pipeline (roughly 1 s in a worker for a 1024×512 survey)

1. Heightfield: domain-warped fBm continents, masked ridged mountains, cellular craters and volcanoes, terraces, ice fractures.
2. Sea level: the area-weighted height quantile that matches the planet's liquid fraction.
3. Climate: latitude and lapse-rate temperature (or an eyeball pattern on tidally locked worlds), plus moisture from BFS distance to the sea and Hadley-cell banding.
4. Hydrology: priority-flood depression filling finds lakes and salt pans. Flow accumulation along the flood tree traces rivers, which then get meander subdivision.
5. Biome classification, albedo, height/emissive/specular data textures, a cloud layer, and landmarks (volcanoes, reefs, ruins, cities…).

### Performance notes

- Sector, system, surface, tile and sprite caches are all LRU-bounded, and GPU textures are released when their surface is evicted.
- Surface tiles carry a 1-sample border and are pixel-snapped, which keeps them seamless. Vegetation glyphs sit on a global cell grid.
- Tiny stars are batched into per-class paths, and nebula and surface tiles are generated nearest-first within a per-frame time budget.
- Quality presets (auto/low/medium/high) scale device-pixel ratio, texture size, tile resolution, particle counts and agent counts.
- WebGL is optional. Without it, globes render with a bilinear software rasteriser and stars with gradients.

## Browser support

The game targets current Chrome, Edge, Firefox and Safari on desktop and mobile. It uses WebGL 1, Pointer Events, Web Workers and `localStorage`, and degrades gracefully if WebGL, workers or storage are unavailable.
