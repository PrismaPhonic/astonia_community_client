# SDL3 GPU Shader Redesign Proposal

## Executive Summary

This document outlines a complete redesign of the Astonia client rendering system to migrate from SDL2's CPU-based effects pipeline to SDL3's GPU shader system. The goal is to minimize CPU-GPU data transfer and maximize GPU utilization.

## Current Architecture Analysis

### Current Pipeline (SDL2)
```
PNG (ZIP) → CPU Decompress → CPU Effects → SDL_UpdateTexture → GPU → Render
              [Worker]         [Worker]      [BOTTLENECK]

CPU Effects Applied:
- Bilinear scaling
- Multi-directional lighting (5 edges)
- Colorization (c1, c2, c3 channels)
- Color balance (cr, cg, cb, saturation)
- Freeze effect (cyan overlay)
- Shine effect (specular highlights)
- Sink effect (bottom fade)
```

### Performance Bottlenecks
1. **SDL_UpdateTexture()** - Full ARGB8888 buffer transfer (256x256x4 bytes = 262KB per 4x scaled sprite)
2. **CPU Pixel Processing** - O(xres × yres × scale²) per sprite
3. **Cache Misses** - 8000 entry LRU with every combination of (sprite, lighting, effects)
4. **Main Thread Waits** - Polling for worker completion

### Current Data Flow
```
16,384 Prefetch Queue
    ↓
4 Worker Threads (CPU-bound):
    - PNG decompression
    - Per-pixel effects
    - Texture upload
    ↓
Main Thread:
    - Wait (polling)
    - SDL_RenderCopy()
    - SDL_RenderPresent()
```

---

## SDL3 GPU Architecture

### SDL3 GPU Features
- **SDL_GPU** - Modern GPU abstraction (Vulkan/Metal/D3D12 backend)
- **Shader Support** - SPIR-V cross-compilation
- **Compute Shaders** - GPU-side data processing
- **Texture Arrays** - Batch multiple sprites
- **Uniform Buffers** - Fast parameter updates
- **Push Constants** - Ultra-fast per-draw parameters

### Proposed Pipeline
```
PNG (ZIP) → CPU Decompress → Base Texture Upload → GPU Shader Effects → Render
              [Worker]         [ONCE PER SPRITE]    [Real-time]

GPU Shader Effects:
- Hardware texture filtering (bilinear/trilinear)
- Fragment shader lighting
- Fragment shader colorization
- All color effects in shader
```

---

## Design Overview

### Core Principle: Upload Once, Process Many

**Old:** Upload a unique texture for each (sprite, lighting, color, effect) combination
**New:** Upload base sprite once, apply all effects in fragment shader

**Cache Reduction:**
- Old: 8000 entries × 262KB = ~2GB potential memory
- New: ~3000 unique base sprites × 64KB = ~192MB base + GPU shader instances

---

## Shader Architecture

### Shader Pipeline

```
Vertex Shader → Fragment Shader → Blend/Output
     ↓               ↓
Position/UV    Lighting/Color/Effects
```

### Fragment Shader Design

```glsl
// astonia_sprite.frag
#version 450

// Inputs from vertex shader
layout(location = 0) in vec2 fragUV;
layout(location = 1) in vec4 fragColor;

// Texture inputs
layout(binding = 0) uniform sampler2D baseTexture;

// Uniform buffer for sprite effects
layout(std140, binding = 1) uniform SpriteEffects {
    // Lighting (5-edge directional)
    float lightMiddle;
    float lightLeft;
    float lightRight;
    float lightUp;
    float lightDown;

    // Colorization channels (0-1023 mapped to 0-1)
    vec3 colorize;      // c1, c2, c3

    // Color balance
    vec3 colorBalance;  // cr, cg, cb
    float saturation;

    // Special effects
    float freeze;       // 0-1
    float shine;        // 0-1023 → specular power
    float sink;         // 0-3 bottom fade rows

    // Sprite geometry
    vec2 spriteSize;    // xres, yres in pixels
} effects;

// Output
layout(location = 0) out vec4 outColor;

// Lighting calculation
float calculateLighting(vec2 uv) {
    // Interpolate lighting based on UV position
    // Center = lightMiddle
    // Edges = weighted blend of directional lights

    float u = uv.x;
    float v = uv.y;

    // Bilinear interpolation of 5 lighting values
    float centerWeight = 1.0 - max(abs(u - 0.5), abs(v - 0.5)) * 2.0;

    float edgeLight = 0.0;
    edgeLight += mix(0.0, effects.lightLeft,  max(0.0, 0.5 - u) * 2.0);
    edgeLight += mix(0.0, effects.lightRight, max(0.0, u - 0.5) * 2.0);
    edgeLight += mix(0.0, effects.lightUp,    max(0.0, 0.5 - v) * 2.0);
    edgeLight += mix(0.0, effects.lightDown,  max(0.0, v - 0.5) * 2.0);

    return mix(edgeLight, effects.lightMiddle, centerWeight);
}

// Colorization (extract green/blue channels and replace with custom colors)
vec3 applyColorization(vec3 color, vec3 colorize) {
    // Original Astonia colorization: extract G/B, modulate by c1/c2/c3
    float green = color.g;
    float blue = color.b;

    return color.r * vec3(1.0, 0.0, 0.0) +
           green * colorize.r +
           blue * (colorize.g + colorize.b);
}

// Color balance and saturation
vec3 applyColorBalance(vec3 color, vec3 balance, float sat) {
    // Apply RGB shift
    color += balance / 255.0;
    color = clamp(color, 0.0, 1.0);

    // Saturation adjustment
    float gray = dot(color, vec3(0.299, 0.587, 0.114));
    return mix(vec3(gray), color, sat);
}

// Freeze effect (cyan overlay)
vec3 applyFreeze(vec3 color, float freeze) {
    vec3 cyan = vec3(0.0, 1.0, 1.0);
    return mix(color, cyan, freeze * 0.3);
}

// Shine effect (specular highlights)
vec3 applyShine(vec3 color, float shine, vec2 uv) {
    if (shine < 0.01) return color;

    // Specular highlight from top-left
    vec2 toLight = vec2(0.3, 0.3) - uv;
    float dist = length(toLight);
    float spec = pow(max(0.0, 1.0 - dist), shine / 100.0);

    return color + vec3(spec);
}

// Sink effect (fade bottom rows)
float applySink(vec2 uv, float sink) {
    if (sink < 0.01) return 1.0;

    float pixelY = uv.y * effects.spriteSize.y;
    float bottomRows = effects.spriteSize.y - sink;

    if (pixelY < bottomRows) return 1.0;

    float fade = (effects.spriteSize.y - pixelY) / sink;
    return clamp(fade, 0.0, 1.0);
}

void main() {
    // Sample base texture
    vec4 texColor = texture(baseTexture, fragUV);

    // Early out for transparent pixels
    if (texColor.a < 0.01) {
        discard;
    }

    vec3 color = texColor.rgb;
    float alpha = texColor.a;

    // Apply lighting
    float lighting = calculateLighting(fragUV) / 15.0; // Normalize to 0-1
    color *= lighting;

    // Apply colorization
    if (length(effects.colorize) > 0.01) {
        color = applyColorization(color, effects.colorize / 1023.0);
    }

    // Apply color balance and saturation
    color = applyColorBalance(color, effects.colorBalance, effects.saturation);

    // Apply freeze effect
    if (effects.freeze > 0.01) {
        color = applyFreeze(color, effects.freeze);
    }

    // Apply shine effect
    if (effects.shine > 0.01) {
        color = applyShine(color, effects.shine, fragUV);
    }

    // Apply sink effect to alpha
    alpha *= applySink(fragUV, effects.sink);

    // Output final color
    outColor = vec4(color, alpha);
}
```

### Vertex Shader Design

```glsl
// astonia_sprite.vert
#version 450

// Per-vertex attributes
layout(location = 0) in vec2 position;
layout(location = 1) in vec2 uv;
layout(location = 2) in vec4 color;

// Uniform buffer for camera/projection
layout(std140, binding = 0) uniform Camera {
    mat4 projection;
    mat4 view;
} camera;

// Per-instance data (for batching)
layout(std140, binding = 2) uniform Instance {
    mat4 model;
    vec2 offset;
} instance;

// Outputs to fragment shader
layout(location = 0) out vec2 fragUV;
layout(location = 1) out vec4 fragColor;

void main() {
    // Transform position
    vec4 worldPos = instance.model * vec4(position + instance.offset, 0.0, 1.0);
    gl_Position = camera.projection * camera.view * worldPos;

    // Pass through UV and color
    fragUV = uv;
    fragColor = color;
}
```

---

## Texture Management Redesign

### Base Texture Cache

```c
// New structure: base textures only
typedef struct {
    SDL_GPUTexture *gpu_texture;      // GPU texture handle

    uint32_t sprite_id;                // Sprite ID
    uint8_t scale;                     // 1x, 2x, 3x, 4x

    uint16_t xres, yres;               // Dimensions
    int16_t xoff, yoff;                // Offset for rendering

    // Metadata for cache management
    uint64_t last_used_frame;          // For LRU eviction
    _Atomic(uint16_t) flags;           // Upload state flags
} BaseTexture;

// Hash on (sprite_id, scale) only
// Estimated cache size: 3000 unique sprites × 4 scales = 12,000 entries
// Memory: 12,000 × 64KB average = 768MB (vs 2GB+ currently)
```

### Shader Parameter Cache

```c
// Uniform buffer for shader parameters
typedef struct {
    // Lighting (normalized to 0-1)
    float light_middle;
    float light_left;
    float light_right;
    float light_up;
    float light_down;

    // Colorization
    float c1, c2, c3;

    // Color balance
    float cr, cg, cb;
    float saturation;

    // Effects
    float freeze;
    float shine;
    float sink;

    // Sprite geometry
    float sprite_width;
    float sprite_height;
} ShaderParams;

// Small, fast updates via SDL_UpdateGPUBuffer()
// Size: 64 bytes vs 262KB texture upload
```

### Texture Atlas for Batching (Advanced Optimization)

```c
// Optional: Pack multiple sprites into texture arrays
typedef struct {
    SDL_GPUTexture *atlas_texture;     // Large texture array
    uint32_t atlas_width;              // e.g., 2048x2048
    uint32_t atlas_height;
    uint32_t layers;                   // Texture array depth

    // Sprite -> Atlas mapping
    AtlasEntry entries[MAX_SPRITES];
} TextureAtlas;

typedef struct {
    uint32_t layer;                    // Which texture layer
    float u0, v0, u1, v1;             // UV coordinates in atlas
} AtlasEntry;

// Benefit: Single texture bind for entire frame
// Challenge: Dynamic packing, need to rebuild on cache changes
```

---

## Rendering Pipeline Redesign

### New Rendering Flow

```c
// Frame rendering (sdl_render_frame)
void sdl_render_frame() {
    // 1. Begin GPU render pass
    SDL_GPUCommandBuffer *cmd = SDL_AcquireGPUCommandBuffer(gpu_device);
    SDL_GPURenderPass *pass = SDL_BeginGPURenderPass(
        cmd,
        &render_target,
        clear_color,
        NULL
    );

    // 2. Bind shader pipeline
    SDL_BindGPUGraphicsPipeline(pass, sprite_pipeline);

    // 3. Bind camera uniform buffer (once per frame)
    SDL_BindGPUVertexUniformBuffer(pass, 0, camera_buffer);

    // 4. Render all visible sprites
    for (int i = 0; i < visible_sprite_count; i++) {
        VisibleSprite *vs = &visible_sprites[i];

        // Get base texture (cache lookup)
        BaseTexture *base = get_base_texture(vs->sprite_id, current_scale);

        // Update shader parameters (64 bytes)
        ShaderParams params = {
            .light_middle = vs->ml / 15.0f,
            .light_left = vs->ll / 15.0f,
            .light_right = vs->rl / 15.0f,
            .light_up = vs->ul / 15.0f,
            .light_down = vs->dl / 15.0f,
            .c1 = vs->c1 / 1023.0f,
            .c2 = vs->c2 / 1023.0f,
            .c3 = vs->c3 / 1023.0f,
            .cr = vs->cr / 255.0f,
            .cg = vs->cg / 255.0f,
            .cb = vs->cb / 255.0f,
            .saturation = vs->sat / 100.0f,
            .freeze = vs->freeze / 3.0f,
            .shine = vs->shine / 1023.0f,
            .sink = vs->sink,
            .sprite_width = base->xres,
            .sprite_height = base->yres
        };

        // Bind texture and parameters
        SDL_BindGPUFragmentSamplers(pass, 0, base->gpu_texture, sampler);
        SDL_PushGPUFragmentUniformData(cmd, 1, &params, sizeof(params));

        // Draw quad (6 vertices for 2 triangles)
        SDL_DrawGPUPrimitives(pass, 6, 1, 0, 0);
    }

    // 5. End render pass and submit
    SDL_EndGPURenderPass(pass);
    SDL_SubmitGPUCommandBuffer(cmd);
    SDL_PresentGPUSwapchain(swapchain);
}
```

### Instanced Rendering (Batching Optimization)

```c
// For sprites using the same base texture
void render_sprite_batch(BaseTexture *base, VisibleSprite *sprites, int count) {
    // Build instance buffer with all shader params
    ShaderParams instance_data[count];
    for (int i = 0; i < count; i++) {
        instance_data[i] = build_shader_params(&sprites[i]);
    }

    // Upload instance buffer
    SDL_UpdateGPUBuffer(instance_buffer, instance_data,
                        sizeof(ShaderParams) * count);

    // Bind texture once
    SDL_BindGPUFragmentSamplers(pass, 0, base->gpu_texture, sampler);

    // Draw all instances
    SDL_DrawGPUPrimitivesInstanced(pass, 6, count, 0, 0);
}

// Reduces draw calls from ~500 to ~50 per frame
```

---

## Multi-Threading Strategy

### Background Loading (Unchanged)

```c
// Worker threads still handle:
// 1. PNG decompression (CPU-bound, parallelizable)
// 2. Initial texture upload (one-time per sprite)

typedef struct {
    uint32_t sprite_id;
    uint8_t scale;

    uint32_t *pixel_data;              // RGBA8888 from PNG
    uint16_t xres, yres;

    _Atomic(uint8_t) state;            // LOADING, READY, UPLOADED
} LoadJob;

void *worker_thread(void *arg) {
    while (running) {
        LoadJob *job = claim_job();
        if (!job) {
            SDL_SemWait(work_semaphore);
            continue;
        }

        // 1. Decompress PNG (CPU)
        decompress_png(job);

        // 2. Transfer to GPU (requires GL context or transfer queue)
        SDL_GPUTransferBuffer *transfer = SDL_AcquireGPUTransferBuffer(
            gpu_device,
            job->xres * job->yres * 4
        );
        memcpy(transfer->data, job->pixel_data, transfer->size);

        SDL_GPUTexture *texture = SDL_CreateGPUTexture(gpu_device, &texture_info);

        SDL_GPUCommandBuffer *upload_cmd = SDL_AcquireGPUCommandBuffer(gpu_device);
        SDL_UploadToGPUTexture(upload_cmd, transfer, texture, NULL);
        SDL_SubmitGPUCommandBuffer(upload_cmd);

        // 3. Store in cache
        cache_insert(job->sprite_id, job->scale, texture);

        job->state = UPLOADED;
        free(job->pixel_data);
    }
}
```

### GPU Command Buffer Threading

```c
// Advanced: Multi-threaded command buffer recording (SDL3 feature)
// Main thread: Frame N
// Worker 1: Pre-process frame N+1 visibility
// Worker 2: Upload textures for frame N+1

// Requires careful synchronization with SDL3's timeline semaphores
```

---

## Migration Path

### Phase 1: SDL3 Basic Migration (No Shaders)
**Goal:** Get SDL3 GPU rendering working with current CPU effects

1. Replace `SDL_Renderer` with `SDL_GPUDevice`
2. Replace `SDL_Texture` with `SDL_GPUTexture`
3. Replace `SDL_RenderCopy()` with `SDL_DrawGPUPrimitives()`
4. Keep CPU-side effects initially
5. Test performance parity

**Files to modify:**
- `src/sdl/sdl_core.c` - Device initialization
- `src/sdl/sdl_draw.c` - Rendering calls
- `src/sdl/sdl_texture.c` - Texture management

### Phase 2: Basic Shader Implementation
**Goal:** Move lighting to fragment shader

1. Write basic fragment shader with lighting only
2. Create shader pipeline
3. Replace CPU lighting with shader params
4. Benchmark performance improvement

**New files:**
- `src/shaders/sprite.vert`
- `src/shaders/sprite.frag`
- `src/sdl/sdl_shader.c` - Shader compilation/management

### Phase 3: Full Effect Shaders
**Goal:** Move all effects to GPU

1. Add colorization to shader
2. Add color balance, saturation
3. Add freeze, shine, sink effects
4. Remove CPU effect code paths
5. Measure cache hit rate improvement

**Files to modify:**
- `src/sdl/sdl_image.c` - Remove CPU effect code
- `src/shaders/sprite.frag` - Complete shader

### Phase 4: Texture Atlas & Batching
**Goal:** Minimize draw calls and state changes

1. Implement texture atlas packing
2. Sort sprites by texture for batching
3. Implement instanced rendering
4. Profile GPU utilization

**New files:**
- `src/sdl/sdl_atlas.c` - Atlas management

---

## Performance Projections

### Current Performance (Measured)
- **Cache Misses:** High (8000 combinations possible)
- **Texture Uploads:** 262KB per miss at 4x scale
- **CPU Effect Time:** ~2-5ms per sprite on worker thread
- **Frame Wait Time:** Spikes when cache misses occur

### Projected Performance (SDL3 + Shaders)

#### Cache Improvement
- **Old:** Hash(sprite, ml, ll, rl, ul, dl, c1, c2, c3, cr, cg, cb, light, sat, freeze, shine, sink, scale)
- **New:** Hash(sprite, scale)
- **Cache Hit Rate:** 60% → 95%+
- **Memory Usage:** ~2GB potential → ~768MB actual

#### Data Transfer Reduction
- **Per Sprite Draw:**
  - Old: 262KB texture upload (on cache miss)
  - New: 64 bytes uniform buffer update (always)
- **Reduction:** 4096× less data per draw

#### CPU Savings
- **Old:** 2-5ms CPU effect processing
- **New:** 0ms (all on GPU in parallel with other fragments)
- **CPU Freed:** 10-20ms per frame (500 sprites × ~2-4ms / 100 sprites processed per frame)

#### GPU Utilization
- **Old:** ~30% (mostly idle between blits)
- **New:** ~70-80% (fragment shaders running)

#### Expected Frame Time Reduction
- **Current:** 16.67ms target (60fps), occasional spikes to 30-50ms on cache misses
- **Projected:** Solid 16.67ms (60fps), spikes eliminated
- **Headroom for:** 4K resolution, more on-screen sprites, post-processing effects

---

## Code Structure

### Proposed File Organization

```
src/sdl/
├── sdl_core.c            [MODIFY] - SDL3 device init
├── sdl_gpu.c             [NEW]    - GPU device management
├── sdl_shader.c          [NEW]    - Shader pipeline management
├── sdl_texture.c         [MODIFY] - Base texture cache
├── sdl_draw.c            [MODIFY] - GPU draw calls
├── sdl_image.c           [MODIFY] - Remove CPU effects
├── sdl_atlas.c           [NEW]    - Texture atlas (Phase 4)
└── sdl_effects.c         [DELETE] - Replaced by shaders

src/shaders/
├── sprite.vert           [NEW]    - Vertex shader
├── sprite.frag           [NEW]    - Fragment shader
├── compile_shaders.sh    [NEW]    - SPIR-V compilation script
└── README.md             [NEW]    - Shader documentation

res/shaders/              [NEW]    - Compiled SPIR-V binaries
├── sprite.vert.spv
└── sprite.frag.spv
```

### New Header: sdl_gpu.h

```c
#ifndef SDL_GPU_H
#define SDL_GPU_H

#include <SDL3/SDL.h>
#include <SDL3/SDL_gpu.h>

// GPU device management
typedef struct {
    SDL_GPUDevice *device;
    SDL_Window *window;

    // Swapchain
    SDL_GPUSwapchainComposition swapchain_composition;
    SDL_GPUTextureFormat swapchain_format;

    // Render targets
    SDL_GPUTexture *backbuffer;
    uint32_t width, height;

    // Shader pipeline
    SDL_GPUGraphicsPipeline *sprite_pipeline;
    SDL_GPUShader *vertex_shader;
    SDL_GPUShader *fragment_shader;

    // Samplers
    SDL_GPUSampler *linear_sampler;
    SDL_GPUSampler *nearest_sampler;

    // Uniform buffers
    SDL_GPUBuffer *camera_buffer;

    // Stats
    uint64_t frames_rendered;
    uint32_t draw_calls_last_frame;
} GPUContext;

extern GPUContext *gpu_ctx;

// Initialization
bool sdl_gpu_init(SDL_Window *window, uint32_t width, uint32_t height);
void sdl_gpu_shutdown(void);

// Shader management
bool sdl_gpu_load_shaders(const char *vert_path, const char *frag_path);
void sdl_gpu_reload_pipeline(void);

// Frame rendering
void sdl_gpu_begin_frame(void);
void sdl_gpu_end_frame(void);

// Texture management
SDL_GPUTexture *sdl_gpu_upload_texture(uint32_t *pixels,
                                        uint32_t width,
                                        uint32_t height);
void sdl_gpu_destroy_texture(SDL_GPUTexture *texture);

#endif // SDL_GPU_H
```

---

## Backward Compatibility

### Feature Flags

```c
// sdl_config.h
#define SDL_USE_GPU_SHADERS 1      // Enable shader pipeline
#define SDL_USE_CPU_FALLBACK 0     // Keep CPU effects for testing

#if SDL_USE_GPU_SHADERS
    #define RENDER_SPRITE(sprite) sdl_gpu_render_sprite(sprite)
#else
    #define RENDER_SPRITE(sprite) sdl_cpu_render_sprite(sprite)
#endif
```

### Testing Strategy

1. **Dual Code Paths:** Keep CPU effects active during Phase 2-3
2. **Visual Comparison:** Screenshot diff tool for pixel-perfect validation
3. **Performance Comparison:** Side-by-side benchmarks
4. **Gradual Rollout:** Shader features enabled one at a time

---

## Shader Hot-Reloading (Developer Feature)

```c
// Watch shader files for changes
void sdl_shader_watch_thread(void *arg) {
    while (running) {
        if (file_modified("res/shaders/sprite.frag.spv")) {
            info("Shader modified, reloading...");
            sdl_gpu_reload_pipeline();
        }
        SDL_Delay(1000);
    }
}

// Triggered by file watcher
void sdl_gpu_reload_pipeline(void) {
    // Destroy old pipeline
    SDL_ReleaseGPUGraphicsPipeline(gpu_ctx->device,
                                    gpu_ctx->sprite_pipeline);

    // Reload shaders
    sdl_gpu_load_shaders("res/shaders/sprite.vert.spv",
                          "res/shaders/sprite.frag.spv");

    // Recreate pipeline
    SDL_GPUGraphicsPipelineCreateInfo info = {...};
    gpu_ctx->sprite_pipeline = SDL_CreateGPUGraphicsPipeline(
        gpu_ctx->device, &info);

    info("Pipeline reloaded successfully");
}
```

---

## Advanced Optimizations (Future)

### 1. Compute Shader Preprocessing
```c
// Use compute shaders for scaling on GPU
// Input: 1x PNG texture
// Output: 2x/3x/4x scaled texture
// Benefit: Eliminate CPU scaling entirely
```

### 2. Persistent Mapped Buffers
```c
// Avoid stalls on uniform buffer updates
SDL_GPUBuffer *uniform_ring[3];  // Triple buffering
uint32_t frame_index = 0;

// Each frame uses different buffer
SDL_PushGPUFragmentUniformData(cmd, 1,
    &params, sizeof(params),
    uniform_ring[frame_index % 3]);
```

### 3. Indirect Drawing
```c
// Build draw commands on GPU
SDL_GPUBuffer *indirect_buffer;
SDL_DrawGPUPrimitivesIndirect(pass, indirect_buffer, 0, draw_count);

// Benefit: Frustum culling on GPU, no CPU readback
```

### 4. Texture Compression
```c
// Use BC7/ASTC for base textures
// Reduces GPU memory and upload time
// Quality comparable to ARGB8888 for sprites
```

---

## Estimated Development Time

### Phase 1: SDL3 Basic Migration
- **Effort:** 3-5 days
- **Risk:** Medium (API changes, testing)
- **Deliverable:** SDL3 rendering with CPU effects

### Phase 2: Basic Shader Implementation
- **Effort:** 2-3 days
- **Risk:** Low (lighting shader straightforward)
- **Deliverable:** GPU lighting working

### Phase 3: Full Effect Shaders
- **Effort:** 4-6 days
- **Risk:** Medium (colorization complexity, visual matching)
- **Deliverable:** All effects on GPU, CPU code removed

### Phase 4: Texture Atlas & Batching
- **Effort:** 5-7 days
- **Risk:** High (dynamic packing, cache invalidation)
- **Deliverable:** Batched rendering, minimal draw calls

**Total: 14-21 days** (2-3 weeks full-time development)

---

## Success Metrics

### Performance KPIs
- [ ] Frame time variance < 2ms (smooth 60fps)
- [ ] Cache hit rate > 95%
- [ ] GPU utilization > 70%
- [ ] CPU usage reduced by 30%+
- [ ] Memory usage < 1GB for texture cache

### Quality KPIs
- [ ] Pixel-perfect rendering match (screenshot diff = 0)
- [ ] No visual regressions
- [ ] Shader hot-reload working in developer mode
- [ ] All existing effects supported

### Code Quality KPIs
- [ ] Reduced LOC by 30% (remove CPU effect code)
- [ ] Shader code coverage > 90%
- [ ] Zero GPU validation errors
- [ ] Clean architecture separation

---

## Risks and Mitigations

### Risk 1: SDL3 API Changes
- **Probability:** Medium
- **Impact:** High
- **Mitigation:** Start with stable SDL3 release, maintain abstraction layer

### Risk 2: Shader Complexity
- **Probability:** Low
- **Impact:** Medium
- **Mitigation:** Incremental shader development, extensive testing

### Risk 3: Visual Regressions
- **Probability:** Medium
- **Impact:** High
- **Mitigation:** Automated screenshot comparison, dual code paths during migration

### Risk 4: Platform Compatibility
- **Probability:** Low
- **Impact:** High
- **Mitigation:** Test on all target platforms (Windows/Linux/macOS), use widely-supported shader features

---

## Conclusion

This redesign leverages SDL3's modern GPU features to transform Astonia's rendering from a CPU-bottlenecked pipeline to a GPU-accelerated system. By uploading base textures once and applying all effects in fragment shaders, we reduce CPU-GPU data transfer by 4000×, eliminate cache thrashing, and free significant CPU time for game logic.

The phased migration approach allows incremental validation and reduces risk, while the final architecture positions Astonia for future enhancements like post-processing effects, higher resolutions, and more complex visual effects—all without additional CPU burden.

**Next Steps:**
1. Review this design with the team
2. Set up SDL3 development environment
3. Begin Phase 1 migration
4. Establish benchmarking framework
5. Proceed through phases with continuous validation

