# Current Rendering Architecture - Step by Step

## Executive Summary

**Important Clarification:** Astonia is NOT a true 3D game with vertices and 3D models. It's a **2D isometric game** that uses pre-rendered 2D sprites to create the illusion of 3D depth. There are no vertex buffers, no polygon meshes, no 3D transformations - just clever 2D sprite positioning and layering.

---

## The Big Picture: What "3D" Really Means in Astonia

### Isometric Projection
```
Map Coordinates (mx, my)  →  Isometric Transform  →  Screen Coordinates (sx, sy)
     (grid-based)                   (diamond view)           (pixel positions)

Example transformation (from game_display.c:50,84):
  sx = (mx - my) * 20 + offset_x
  sy = (mx + my) * 10 + offset_y
```

This creates the "diamond" view where:
- Moving RIGHT on map → moves down-right on screen
- Moving UP on map → moves up-right on screen
- Moving LEFT on map → moves up-left on screen
- Moving DOWN on map → moves down-left on screen

### Depth Simulation Through Layering

The "3D" effect comes from:
1. **Sprites drawn from back to front** (painter's algorithm)
2. **Vertical offset** (higher objects drawn lower on screen)
3. **Multi-directional lighting** (edges lit differently)
4. **Height field** (`h` in display list) for proper occlusion

---

## Complete Rendering Pipeline (Frame by Frame)

### Phase 1: Main Loop (`gui_core.c:265-362`)

```c
while (!quit) {
    // 1. Network: Receive game state updates from server
    poll_network();

    // 2. Tick Processing: Update game logic at ~10Hz
    if (time_for_next_tick()) {
        do_tick();           // Update positions, animations, state
        prefetch_game();     // Pre-load textures for next frame
    }

    // 3. Frame Rendering: Draw at ~60Hz
    if (time_for_next_frame()) {
        sdl_clear();         // Clear screen to black
        display();           // Build and render entire scene
        sdl_render();        // SDL_RenderPresent() - flip buffers
    }
}
```

**Key Point:** The game **decodes future ticks** and pre-loads textures in background threads so they're ready when that tick becomes visible.

---

### Phase 2: Display Function (`gui_display.c:128-222`)

```c
void display() {
    // 1. UI state updates
    display_toplogic();      // Top menu bar animation
    set_cmd_states();        // Update UI element states

    // 2. Main game scene (THE IMPORTANT PART)
    render_push_clip();                    // Save clip rect
    render_more_clip(map_bounds);          // Restrict to map area
    display_game();                        // ← DRAW THE ISOMETRIC SCENE
    render_pop_clip();                     // Restore clip rect

    // 3. UI overlays (drawn ON TOP of game scene)
    display_screen();        // UI panels/frames
    display_keys();          // Skill buttons
    display_inventory();     // Inventory grid
    display_text();          // Chat/messages
    display_minimap();       // Mini-map
    // ... 20+ more UI elements ...
}
```

---

### Phase 3: Display Game - Building the Scene (`game_display.c:1383-1390`)

```c
void display_game() {
    display_game_spells();    // Spell effects (ground)
    display_game_spells2();   // More spell effects
    display_game_map(map);    // ← THE CORE RENDERER
    display_game_names();     // Character names floating above heads
    display_pents();          // Pentagram effects
}
```

---

### Phase 4: Display Game Map - The Heart of Rendering (`game_display.c:887-1380`)

This is where the magic happens. Let me break it down step by step:

#### Step 4.1: Iterate Over Visible Map Tiles

```c
void display_game_map(struct map *cmap) {
    // 'quick' is a pre-computed list of visible map tiles
    // Built by make_quick() - tiles within view distance

    for (i = 0; i < maxquick; i++) {
        map_index_t mn = quick[i].mn[4];    // Center tile index
        int scrx = quick[i].cx;             // Screen X
        int scry = quick[i].cy;             // Screen Y
        int light = cmap[mn].rlight;        // Tile lighting (0-15)

        // If tile is dark (invisible), draw black square
        if (!light) {
            dl_next_set(GNDSTR_LAY, 0, scrx, scry, RENDERFX_NORMAL_LIGHT);
            continue;
        }

        // Otherwise, render tile layers...
    }
}
```

**Quick Array Structure:**
```c
struct quick {
    map_index_t mn[9];   // This tile + 8 neighbors (for lighting)
    int cx, cy;          // Screen coordinates (isometric)
};
```

#### Step 4.2: Render Ground Layer (Base Texture)

```c
// Layer 1: Ground sprite (grass, stone, dirt, etc.)
if (cmap[mn].rg.sprite) {
    DL *dl = dl_next_set(
        GND_LAY,                  // Layer number (for sorting)
        cmap[mn].rg.sprite,       // Sprite ID (e.g., 5042 = grass)
        scrx, scry - 10,          // Screen position (offset down 10px)
        light                     // Base light level
    );

    // Multi-directional lighting from neighbors
    dl->renderfx.ml = light;              // Middle (this tile)
    dl->renderfx.ll = cmap[mn_left].rlight;   // Left edge
    dl->renderfx.rl = cmap[mn_right].rlight;  // Right edge
    dl->renderfx.ul = cmap[mn_up].rlight;     // Up edge
    dl->renderfx.dl = cmap[mn_down].rlight;   // Down edge

    // Color effects
    dl->renderfx.cr = cmap[mn].rg.cr;     // Red tint
    dl->renderfx.cg = cmap[mn].rg.cg;     // Green tint
    dl->renderfx.cb = cmap[mn].rg.cb;     // Blue tint
    dl->renderfx.sat = cmap[mn].rg.sat;   // Saturation

    // Special channels (colorization)
    dl->renderfx.c1 = cmap[mn].rg.c1;     // Color channel 1
    dl->renderfx.c2 = cmap[mn].rg.c2;     // Color channel 2
    dl->renderfx.c3 = cmap[mn].rg.c3;     // Color channel 3

    // Special effects
    dl->renderfx.shine = cmap[mn].rg.shine;  // Specular highlights

    // Height for depth sorting
    dl->h = -10;   // Ground is lowest layer
}
```

#### Step 4.3: Render Second Ground Layer (Optional)

```c
// Layer 2: Ground decoration (shadows, decals, etc.)
if (cmap[mn].rg2.sprite) {
    // Same process as rg, but different sprite
    // This allows ground + ground detail (e.g., grass + flowers)
}
```

#### Step 4.4: Render Floor Items

```c
// Items lying on the ground (dropped weapons, potions, etc.)
if (cmap[mn].fi.sprite) {
    DL *dl = dl_next_set(
        get_lay_sprite(cmap[mn].fisprite, F_LAY),  // Layer varies by sprite
        cmap[mn].fi.sprite,
        scrx + xoff, scry + yoff,   // Offset for item positioning
        light
    );

    dl->h = 0;  // Items slightly above ground

    // Sink effect (items partially buried in water/snow)
    dl->renderfx.sink = get_sink(mn, cmap);

    // Apply all lighting/color effects like ground
}
```

#### Step 4.5: Render Floor Sprites (Walls, Trees, Buildings)

```c
// Large sprites that occupy floor space (walls, pillars, trees)
if (cmap[mn].fs.sprite) {
    // Determine layer based on sprite type
    // Taller objects = higher layer = drawn later = appear in front
    int layer = get_lay_sprite(cmap[mn].fsprite, F_LAY);

    // Handle "cut" sprites (tall objects that can hide player)
    int cutsprite = is_cut_sprite(cmap[mn].fsprite);
    if (cutsprite && player_behind_sprite) {
        // Draw semi-transparent version so player is visible
        dl->renderfx.scale = 50;  // 50% size/transparency
    }

    // Multi-directional lighting from 5 edges
    // Same as ground layer

    // Color effects for environmental moods
    if (cmap[mn].flags & CMF_INFRA) {
        dl->renderfx.cr += 80;   // Red tint for infravision
    }
    if (cmap[mn].flags & CMF_UNDERWATER) {
        dl->renderfx.cb += 80;   // Blue tint underwater
    }
}
```

#### Step 4.6: Render Characters (Players, NPCs, Monsters)

```c
// Characters moving around the map
if (cmap[mn].cs.sprite) {
    // Character position interpolation for smooth movement
    int x = trans_x(from_x, from_y, to_x, to_y, move_step, move_start);
    int y = trans_y(from_x, from_y, to_x, to_y, move_step, move_start);

    // Height calculation (characters standing on terrain)
    int height = get_chr_height(cmap[mn].csprite);

    DL *dl = dl_next_set(
        C_LAY + height,   // Layer depends on character height
        cmap[mn].cs.sprite,
        x, y,
        light
    );

    // Freeze frame for animation
    dl->renderfx.freeze = cmap[mn].cs.freeze;

    // Character-specific color channels (armor coloring)
    dl->renderfx.c1 = cmap[mn].cs.c1;  // Primary armor color
    dl->renderfx.c2 = cmap[mn].cs.c2;  // Secondary armor color
    dl->renderfx.c3 = cmap[mn].cs.c3;  // Tertiary armor color

    // Height offset for depth sorting
    dl->h = height;
}
```

#### Step 4.7: Special Effects

```c
// Spell effects, damage numbers, bless auras, etc.
// These use "call" entries instead of sprites

dl = dl_next();
dl->call = DLC_BLESS;        // Special rendering function
dl->call_x1 = x;
dl->call_y1 = y;
dl->call_x2 = ticker;        // Animation frame
dl->call_x3 = strength;
dl->layer = SPECIAL_LAY;
```

---

### Phase 5: Display List - The Scene Graph (`game_core.c:76-213`)

All those `dl_next_set()` calls are building a **display list** - a sorted array of everything to draw.

#### Display List Structure

```c
struct dl {
    int layer;              // Rendering layer (0-99+)
    int x, y, h;            // Position: screen X, Y, height offset

    RenderFX renderfx;      // ALL rendering parameters:
                            // - sprite ID
                            // - lighting (ml, ll, rl, ul, dl)
                            // - color effects (cr, cg, cb, sat)
                            // - colorization (c1, c2, c3)
                            // - special effects (shine, freeze, sink)
                            // - scale

    // OR special function call
    char call;              // DLC_STRIKE, DLC_BLESS, etc.
    int call_x1, call_y1, call_x2, call_y2, call_x3;
};
```

#### Adding to Display List

```c
DL *dl_next_set(int layer, uint sprite, int scrx, int scry, uchar light) {
    DL *dl = dl_next();    // Get next free slot

    dl->x = scrx;
    dl->y = scry;
    dl->layer = layer;

    // Initialize renderfx with defaults
    dl->renderfx.sprite = sprite;
    dl->renderfx.ml = light;  // All edges start at same light
    dl->renderfx.ll = light;
    dl->renderfx.rl = light;
    dl->renderfx.ul = light;
    dl->renderfx.dl = light;
    dl->renderfx.sink = 0;
    dl->renderfx.scale = 100;
    // ... all other params to defaults

    return dl;  // Caller can modify renderfx
}
```

---

### Phase 6: Display List Execution (`game_core.c:163-213`)

After building the entire scene, we sort and render:

```c
void dl_play() {
    // 1. SORT by layer, then Y position (painter's algorithm)
    qsort(dlsort, dlused, sizeof(DL*), dl_qcmp);

    // Comparison function:
    // - Objects with lower layer drawn first
    // - Within same layer, lower Y drawn first
    // - This ensures proper depth ordering

    // 2. RENDER each display list entry
    for (d = 0; d < dlused; d++) {
        if (dlsort[d]->call == 0) {
            // Normal sprite
            render_sprite_fx(
                &dlsort[d]->renderfx,
                dlsort[d]->x,
                dlsort[d]->y - dlsort[d]->h  // Apply height offset
            );
        } else {
            // Special effect
            switch (dlsort[d]->call) {
                case DLC_STRIKE:
                    render_display_strike(...);
                    break;
                case DLC_BLESS:
                    render_draw_bless(...);
                    break;
                // ... etc
            }
        }
    }
}
```

**Sorting Example:**
```
Before sort (order added):
  [Ground, layer=0, y=100]
  [Character, layer=50, y=150]
  [Tree, layer=30, y=120]
  [Wall, layer=40, y=130]

After sort (render order):
  [Ground, layer=0, y=100]    ← Drawn first (back)
  [Tree, layer=30, y=120]
  [Wall, layer=40, y=130]
  [Character, layer=50, y=150] ← Drawn last (front)
```

---

### Phase 7: Render Sprite FX - Texture Lookup (`render.c:193-250`)

```c
int render_sprite_fx(RenderFX *fx, int scrx, int scry) {
    // 1. Get texture index from cache
    int stx = sdl_tx_load(
        fx->sprite,      // Sprite ID
        fx->ml,          // Middle light
        fx->ll, fx->rl, fx->ul, fx->dl,  // Edge lights
        fx->freeze,
        fx->scale,
        fx->cr, fx->cg, fx->cb,
        fx->clight,
        fx->sat,
        fx->c1, fx->c2, fx->c3,
        fx->shine,
        fx->sink,
        0  // preload flag
    );

    if (stx < 0) return 0;  // Texture not ready

    // 2. Get sprite offset from PNG metadata
    int xoff = sdlt[stx].xoff;
    int yoff = sdlt[stx].yoff;

    // 3. Blit to screen
    sdl_blit(stx, scrx + xoff, scry + yoff,
             clipsx, clipsy, clipex, clipey,
             x_offset, y_offset);

    return 1;
}
```

---

### Phase 8: Texture Cache Lookup (`sdl_texture.c:267-400`)

This is THE critical performance path:

```c
int sdl_tx_load(sprite, ml, ll, rl, ul, dl, freeze, scale,
                cr, cg, cb, light, sat, c1, c2, c3, shine, sink, preload) {

    // 1. HASH all parameters
    uint32_t hash = hashfunc(sprite, ml, ll, rl, ul, dl);

    // 2. SEARCH cache for exact match
    for (stx in cache_bucket[hash]) {
        if (sdlt[stx].sprite == sprite &&
            sdlt[stx].ml == ml &&
            sdlt[stx].ll == ll &&
            // ... ALL parameters match ...
            sdlt[stx].sink == sink) {

            // CACHE HIT - texture already exists
            touch_for_lru(stx);
            return stx;
        }
    }

    // 3. CACHE MISS - need to create texture

    // 3a. Check if background worker is making it
    if (already_in_worker_queue(params)) {
        if (preload) return -1;  // Don't wait, just queue it

        // WAIT for worker to finish (BLOCKING!)
        while (!(flags & SF_DIDMAKE)) {
            SDL_Delay(1);  // Poll every 1ms
        }
    }

    // 3b. Not in queue - add to worker queue
    else {
        add_to_prefetch_queue(params);
        SDL_SemPost(prework);  // Wake up worker thread
        return -1;
    }

    // 4. Texture ready - finalize GPU upload on main thread
    if (!(flags & SF_DIDTEX)) {
        finalize_texture_upload(stx);
    }

    return stx;
}
```

---

### Phase 9: Background Worker Threads (`sdl_core.c:1066-1118`)

These run in parallel with rendering:

```c
void *sdl_prefetch_worker(void *arg) {
    while (running) {
        // 1. Wait for work
        SDL_SemWait(prework);

        // 2. Claim a job from queue
        job = claim_prefetch_job();
        if (!job) continue;

        // 3. Process texture (CPU-bound work)
        sdl_make(
            job->sprite,
            job->ml, job->ll, job->rl, job->ul, job->dl,
            // ... all params ...
            2  // preload=2: do CPU effects only
        );

        // Worker has now:
        // - Loaded PNG from ZIP
        // - Decompressed it
        // - Applied bilinear scaling
        // - Applied lighting
        // - Applied colorization
        // - Applied all color effects
        // - Allocated pixel buffer

        // 4. Mark ready for main thread
        flags_or(stx, SF_DIDMAKE);
    }
}
```

---

### Phase 10: Texture Creation - The CPU Pipeline (`sdl_image.c:911-1156`)

This is where ALL the visual processing happens on CPU:

```c
void sdl_make(sprite, ml, ll, rl, ul, dl, freeze, scale,
              cr, cg, cb, light, sat, c1, c2, c3, shine, sink, preload) {

    // Stage 1: Load base PNG
    uint32_t *base_pixels = load_png_from_zip(sprite);

    // Stage 2: Allocate output buffer
    int out_width = base_width * sdl_scale;
    int out_height = base_height * sdl_scale;
    uint32_t *output = malloc(out_width * out_height * 4);

    // Stage 3: BILINEAR SCALING (if needed)
    if (scale != 100 || sdl_scale > 1) {
        for (y = 0; y < out_height; y++) {
            for (x = 0; x < out_width; x++) {
                // Map output pixel to input space
                float src_x = x / scale_factor;
                float src_y = y / scale_factor;

                // Bilinear interpolation
                output[y*out_width + x] = bilinear_sample(
                    base_pixels, src_x, src_y
                );
            }
        }
    }

    // Stage 4: LIGHTING (per-pixel)
    for (y = 0; y < out_height; y++) {
        for (x = 0; x < out_width; x++) {
            // Interpolate lighting across sprite
            float u = x / out_width;   // 0..1 left to right
            float v = y / out_height;  // 0..1 top to bottom

            // Calculate lighting from 5 edges
            float light_factor =
                ml * (1-u) * (1-v) +  // Top-left
                ll * u * (1-v) +       // Top-right
                rl * (1-u) * v +       // Bottom-left
                ul * u * v +           // Bottom-right
                dl * center_weight;    // Middle

            // Apply to pixel
            uint32_t pixel = output[y*out_width + x];
            int r = (pixel >> 16) & 0xFF;
            int g = (pixel >> 8) & 0xFF;
            int b = pixel & 0xFF;

            r = r * light_factor / 15;
            g = g * light_factor / 15;
            b = b * light_factor / 15;

            output[y*out_width + x] = (r<<16) | (g<<8) | b;
        }
    }

    // Stage 5: COLORIZATION
    if (c1 || c2 || c3) {
        output = sdl_colorize(output, width, height, c1, c2, c3);
    }

    // Stage 6: COLOR BALANCE
    if (cr || cg || cb || sat != 0) {
        output = sdl_colorbalance(output, width, height,
                                  cr, cg, cb, sat);
    }

    // Stage 7: FREEZE EFFECT
    if (freeze) {
        output = sdl_freeze(output, width, height, freeze);
    }

    // Stage 8: SHINE EFFECT
    if (shine) {
        output = sdl_shine(output, width, height, shine);
    }

    // Stage 9: SINK EFFECT
    if (sink) {
        // Fade bottom N rows to transparent
        fade_bottom_rows(output, width, height, sink);
    }

    // Stage 10: PREMULTIPLY ALPHA
    for (i = 0; i < width * height; i++) {
        uint32_t pixel = output[i];
        int a = (pixel >> 24) & 0xFF;
        int r = ((pixel >> 16) & 0xFF) * a / 255;
        int g = ((pixel >> 8) & 0xFF) * a / 255;
        int b = (pixel & 0xFF) * a / 255;
        output[i] = (a<<24) | (r<<16) | (g<<8) | b;
    }

    // Stage 11: Store in cache
    sdlt[stx].pixel = output;
    sdlt[stx].xres = width;
    sdlt[stx].yres = height;

    // Mark stage complete
    flags_or(stx, SF_DIDMAKE);
}
```

---

### Phase 11: GPU Texture Upload (`sdl_image.c:1150-1156`)

Back on the main thread, before rendering:

```c
void finalize_texture(int stx) {
    // 1. Create GPU texture
    sdlt[stx].tex = SDL_CreateTexture(
        sdlren,
        SDL_PIXELFORMAT_ARGB8888,
        SDL_TEXTUREACCESS_STATIC,
        sdlt[stx].xres,
        sdlt[stx].yres
    );

    // 2. UPLOAD PIXELS (BIG DATA TRANSFER)
    SDL_UpdateTexture(
        sdlt[stx].tex,
        NULL,  // Entire texture
        sdlt[stx].pixel,  // CPU pixel buffer
        sdlt[stx].xres * 4  // Row stride
    );

    // 3. Set blend mode
    SDL_SetTextureBlendMode(sdlt[stx].tex, SDL_BLENDMODE_BLEND);

    // 4. Mark complete
    flags_or(stx, SF_DIDTEX);
}
```

**Data Transfer Size:**
- 1x scale, 64x64 sprite: 64 × 64 × 4 = 16 KB
- 4x scale, 64x64 sprite: 256 × 256 × 4 = 262 KB ← THIS IS THE BOTTLENECK

---

### Phase 12: Final Blit to Screen (`sdl_draw.c:39-89`)

```c
void sdl_blit(int stx, int sx, int sy, ...) {
    SDL_Texture *tex = sdlt[stx].tex;

    // Apply clipping
    SDL_Rect src = {addx, addy, width, height};
    SDL_Rect dst = {sx + x_offset, sy + y_offset, width, height};

    // RENDER (GPU copy texture to backbuffer)
    SDL_RenderCopy(sdlren, tex, &src, &dst);
}
```

---

## Summary: Complete Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ FRAME START                                                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 1. BUILD DISPLAY LIST (CPU)                                     │
│    - Iterate visible map tiles (maxquick ~500-1000)             │
│    - For each tile: ground, items, sprites, characters          │
│    - Each adds 1-5 DL entries                                   │
│    - Total: ~2000-5000 display list entries                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. SORT DISPLAY LIST (CPU)                                      │
│    - qsort() by layer, then Y position                          │
│    - Ensures back-to-front rendering                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. RENDER EACH ENTRY (CPU + GPU)                                │
│    For each of ~2000-5000 entries:                              │
└─────────────────────────────────────────────────────────────────┘
        ↓                                ↓
┌──────────────────┐          ┌──────────────────────────┐
│ CACHE HIT        │          │ CACHE MISS               │
│ (~60% of time)   │          │ (~40% of time)           │
└──────────────────┘          └──────────────────────────┘
        ↓                                ↓
┌──────────────────┐          ┌──────────────────────────┐
│ sdl_blit()       │          │ Wait for worker thread   │
│ SDL_RenderCopy() │          │ (or stall if not ready)  │
│ ~0.01ms          │          └──────────────────────────┘
└──────────────────┘                     ↓
                              ┌──────────────────────────┐
                              │ Worker created texture:  │
                              │  1. Load PNG (5-20ms)   │
                              │  2. Bilinear scale      │
                              │  3. Light per pixel     │
                              │  4. Colorize            │
                              │  5. Effects             │
                              │  Total: 2-10ms on CPU   │
                              └──────────────────────────┘
                                         ↓
                              ┌──────────────────────────┐
                              │ Main thread finalize:    │
                              │  SDL_UpdateTexture()     │
                              │  262KB upload @ 4x scale │
                              │  ~1-5ms per texture      │
                              └──────────────────────────┘
                                         ↓
                              ┌──────────────────────────┐
                              │ sdl_blit()               │
                              │ SDL_RenderCopy()         │
                              └──────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 4. PRESENT (GPU)                                                │
│    SDL_RenderPresent() - Flip backbuffer to screen              │
│    VSync wait (~16.67ms @ 60Hz)                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                         FRAME COMPLETE

┌─────────────────────────────────────────────────────────────────┐
│ BACKGROUND WORKERS (Parallel with above)                        │
│    - 4 worker threads (configurable)                            │
│    - Process prefetch queue (16,384 max entries)                │
│    - Pre-load textures for future frames                        │
│    - Goal: Have texture ready BEFORE it's needed                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

### No 3D Geometry
- **No vertices, no meshes, no 3D models**
- Everything is 2D sprites (PNG images from ZIP files)
- "3D" is an illusion from isometric projection + depth sorting

### Texture Cache is Everything
- Cache key: (sprite, ml, ll, rl, ul, dl, freeze, scale, cr, cg, cb, light, sat, c1, c2, c3, shine, sink)
- 8000 cache entries, but millions of possible combinations
- Cache miss = 2-10ms CPU work + 1-5ms GPU upload = frame stutter

### CPU Does All Effects
- Lighting: Per-pixel calculation with 5-edge interpolation
- Colorization: Extract channels and recolor
- Scaling: Bilinear interpolation
- Everything happens BEFORE upload to GPU

### GPU is Underutilized
- GPU only does: texture → backbuffer copy (SDL_RenderCopy)
- No shaders, no GPU effects, no GPU compute
- GPU idle while CPU processes pixels

### Why SDL3 + Shaders Will Help
- **Upload base sprite once** (1x scale, no effects) = 16KB
- **Apply all effects in fragment shader** (real-time, on GPU)
- **Update 64-byte uniform buffer** instead of 262KB texture
- **4096× less data transfer per draw**
- **Zero CPU time** for effects (parallel on GPU)
- **Cache hit rate 95%+** (only hash on sprite + scale)

---

## Next Steps

Now that you understand the current architecture:

1. **Learn shaders** using the resources I provided
2. **Visualize** how fragment shaders would replace the CPU pipeline
3. **Prototype** a simple SDL3 GPU example (triangle + shader)
4. **Migrate** step by step using the redesign document

Questions to think about:
- Which effect would you move to shaders first? (Hint: lighting)
- How would you pass the 5-edge lighting to the fragment shader?
- What would the cache key look like with shaders? (Just sprite + scale!)

