# ASSETS.md — Asset Provenance & License Registry

> **Rule:** Every 3D model (`.glb`), texture/HDRI, 2D card illustration, shader, font, and audio file committed to `assets/` MUST be registered here at import time.

## 1. IP, Licensing & Hardware Policy

1. **Intellectual Property Boundaries:**
   - Prototype/learning builds may reference public-domain folklore archetypes (e.g., *King Arthur*, *Medusa*, *Alice*, *Sinbad*).
   - Any public build (itch.io, Steam) MUST use original game branding, original card artwork, original card wording, and public-domain or original characters. Never use Restoration Games trademarks, rulebook text verbatim, or copyrighted character likenesses.
2. **License Requirements:**
   - Preferred licenses for third-party assets: **CC0** (Poly Haven, ambientCG, Kenney).
   - For **CC-BY** assets (e.g., Freesound), record the exact attribution string below so it is included in the in-game Credits screen.
   - Avoid **CC-BY-NC** (non-commercial) and **CC-BY-SA** (share-alike) for any asset intended for commercial release.
   - For AI-generated assets (Tripo, Meshy, ComfyUI/Flux, ElevenLabs, Stable Audio), record the exact model/service, generation date, and subscription tier confirming commercial rights.
3. **4 GB VRAM Technical Gates:**
   - **Hero Miniatures (`.glb`):** Max `15,000` triangles after Blender decimation; vertex/1K AO cavity map + procedural `mini_wash.gdshader`.
   - **Sidekick Tokens & Dials (`.glb`):** Max `1,500` triangles (generated procedurally via `bpy`).
   - **Board & Table Textures:** Max `2048x2048` (2K), imported with Godot VRAM Compression enabled.
   - **Card Art Textures:** Max `1024x1024` (1K) clean illustration only (no baked frame/text).

---

## 2. 3D Models (`assets/models/`)

| Asset ID | File Path | Category | Generator / Source | Prompt / Script / Version | Date (YYYY-MM-DD) | Tri Count | Plan / Tier | License | Commercial OK? |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `tpl_hero_example` | `assets/models/heroes/example.glb` | Hero Miniature | ComfyUI (Flux) -> Tripo 3D -> Blender (`bpy`) | `"unpainted grey plastic board game miniature..."` | 2026-10-17 | 12,400 | Template | Pending | Pending |
| `tpl_sidekick_token` | `assets/models/tokens/sidekick_disc.glb` | Sidekick Token | Blender Python (`bpy`) procedural | `scripts/tools/build_tokens.py` | 2026-10-17 | 512 | Local Tool | CC0 / Own | Yes |

---

## 3. Environment Textures, Materials & HDRIs (`assets/textures/environment/`)

| Asset ID | File Path | Resolution | Maps Included | Source | Author / URL | Date (YYYY-MM-DD) | License | Attribution Required? |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `tpl_table_wood` | `assets/textures/environment/wood_table_2k/` | 2048x2048 | Albedo, Normal, Roughness | Poly Haven | `https://polyhaven.com/` | 2026-10-17 | CC0 | No |
| `tpl_studio_hdri` | `assets/textures/environment/tabletop_studio_1k.hdr` | 1024x512 | Panorama HDRI | Poly Haven | `https://polyhaven.com/` | 2026-10-17 | CC0 | No |

---

## 4. 2D Card Illustrations & UI Art (`assets/textures/cards/`, `assets/textures/ui/`)

| Asset ID | File Path | Resolution | Generator / Source | Model / Checkpoint / LoRA | Date (YYYY-MM-DD) | Plan / Tier | License | Commercial OK? |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `tpl_card_art` | `assets/textures/cards/strike_01.webp` | 1024x1024 | ComfyUI / Imagen 3 | Template entry | 2026-10-17 | Template | Pending | Pending |

---

## 5. Audio — Music & SFX (`assets/audio/`)

| Asset ID | File Path | Bus (`Music`/`SFX`/`UI`) | Generator / Source | Author / Track / Prompt | Date (YYYY-MM-DD) | Plan / Tier | License | Attribution String |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `tpl_sfx_card_slide` | `assets/audio/sfx/card_slide.ogg` | `UI` | Kenney Casino Audio | Kenney.nl | 2026-10-17 | Free | CC0 | None |
| `tpl_bgm_battle` | `assets/audio/music/battle_theme.ogg` | `Music` | ElevenLabs Music / Stable Audio | Template entry | 2026-10-17 | Template | Pending | None |

---

## 6. Shaders & Fonts (`assets/shaders/`, `assets/fonts/`)

| Asset ID | File Path | Type | Source / Author | Date (YYYY-MM-DD) | License |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `shd_mini_wash` | `assets/shaders/mini_wash.gdshader` | Spatial Shader | In-house (Antigravity) | 2026-10-17 | MIT / Own |
| `shd_space_ring` | `assets/shaders/space_ring.gdshader` | Spatial Shader | In-house (Antigravity) | 2026-10-17 | MIT / Own |

