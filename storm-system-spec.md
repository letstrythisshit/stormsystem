# StormCore — Unity HDRP Weather Phenomena System
## Package Architecture Specification

---

## 1. Design Philosophy

**One data model. One renderer. Everything else reads from it.**

The fundamental problem with layering VFX Graph, HDRP volumetric clouds, particle systems,
and destruction together is that you end up with five systems that look vaguely similar but
aren't actually synchronized. A tornado particle effect swirls at one speed, the cloud map
rotates at another, the destruction system samples a third wind curve, and none of them
agree on where the eyewall actually is.

This spec takes a different approach: a single authoritative storm simulation runs on the
CPU (with GPU-accelerated density generation), and every visual and gameplay system is a
**consumer** of that data. Nothing generates its own idea of where the storm is or how
strong it is.

### What we build in code vs. what we delegate

| System | Implementation | Rationale |
|--------|---------------|-----------|
| Storm simulation | C# + Compute Shaders | Must be authoritative, queryable, deterministic |
| Cloud rendering | Custom HDRP CloudRenderer + HLSL raymarcher | No existing system handles vertical storm structures |
| Precipitation | Compute shader particle sim + custom DrawProceduralIndirect | VFX Graph can't read our wind field efficiently at scale |
| Debris | Compute shader particle sim | Same — needs direct wind field coupling |
| Lightning | Compute shader bolt generation + mesh builder | Needs to follow storm geometry exactly |
| Ground effects | Shader Graph + custom passes | Wetness, puddles, dust — these are surface shaders |
| Destruction query | C# API reading storm model | CPU-side, no GPU readback needed |
| Audio | C# spatial audio driver | Reads storm model for positioning and intensity |

---

## 2. Package Structure

```
com.studio.stormcore/
├── Runtime/
│   ├── Core/
│   │   ├── StormManager.cs              — Singleton, owns all active storms
│   │   ├── StormInstance.cs              — Single storm: type, position, path, intensity
│   │   ├── StormProfile.asset           — ScriptableObject preset (Category 5, EF4, etc.)
│   │   ├── WindField.cs                 — CPU-side wind query API
│   │   └── StormPath.cs                 — Spline-based storm movement over time
│   │
│   ├── Simulation/
│   │   ├── StormSimulation.cs           — Ticks storm state each frame
│   │   ├── VortexModels.cs              — Rankine, Lamb-Oseen, Holland wind profiles
│   │   ├── WindFieldCompute.compute     — GPU wind field texture generation
│   │   └── DensityFieldCompute.compute  — GPU cloud density generation
│   │
│   ├── Rendering/
│   │   ├── Clouds/
│   │   │   ├── StormCloudSettings.cs    — CloudSettings implementation (Volume component)
│   │   │   ├── StormCloudRenderer.cs    — CloudRenderer implementation
│   │   │   ├── StormClouds.hlsl         — Raymarching shader
│   │   │   ├── NoiseLibrary.hlsl        — FBM, curl noise, Worley
│   │   │   └── StormShapes.hlsl         — Analytical density functions per storm type
│   │   │
│   │   ├── Precipitation/
│   │   │   ├── RainSystem.cs            — Manages rain compute + rendering
│   │   │   ├── RainSimulation.compute   — Particle simulation (position, velocity, life)
│   │   │   ├── RainRenderer.cs          — DrawProceduralIndirect for rain streaks
│   │   │   ├── Rain.shader              — Stretched billboard rain drops
│   │   │   ├── HailSystem.cs            — Same pattern, different particle behavior
│   │   │   └── SnowSystem.cs            — Same pattern, different particle behavior
│   │   │
│   │   ├── Debris/
│   │   │   ├── DebrisSystem.cs          — Manages debris compute + rendering
│   │   │   ├── DebrisSimulation.compute — Heavier particles with collision
│   │   │   ├── DebrisRenderer.cs        — Instanced mesh rendering for debris chunks
│   │   │   └── DebrisCatalog.asset      — ScriptableObject: mesh + weight + drag per type
│   │   │
│   │   ├── Lightning/
│   │   │   ├── LightningSystem.cs       — Bolt timing, branching, targeting
│   │   │   ├── BoltGenerator.compute    — L-system or DLA bolt path generation
│   │   │   ├── LightningMeshBuilder.cs  — Builds mesh from bolt path each strike
│   │   │   └── Lightning.shader         — Emissive, animated fade
│   │   │
│   │   ├── Atmosphere/
│   │   │   ├── StormFogOverride.cs      — Modifies HDRP Fog volume per storm proximity
│   │   │   ├── StormExposureOverride.cs — Darkens exposure under storm
│   │   │   └── StormLightOverride.cs    — Modulates directional light color/intensity
│   │   │
│   │   └── Surface/
│   │       ├── WetnessManager.cs        — Writes wetness to a screen-space or world-space map
│   │       ├── PuddleManager.cs         — Spawns/grows puddle decals based on rainfall
│   │       └── Wetness.shadergraph      — Subgraph: darkens albedo, increases smoothness
│   │
│   ├── Destruction/
│   │   ├── IStormDamageable.cs          — Interface for destructible objects
│   │   ├── WindForceApplicator.cs       — Queries WindField, applies forces to rigidbodies
│   │   ├── StructuralStressTracker.cs   — Accumulates wind load over time
│   │   └── StormDamageZone.cs           — Trigger volume that activates destruction
│   │
│   ├── Audio/
│   │   ├── StormAudioDriver.cs          — Positions audio sources, crossfades layers
│   │   ├── StormAudioProfile.asset      — ScriptableObject: clips per intensity band
│   │   └── ThunderScheduler.cs          — Syncs thunder with lightning system
│   │
│   └── Quality/
│       ├── StormQualitySettings.cs      — ScriptableObject per quality level
│       └── StormQualityManager.cs       — Applies settings, handles dynamic scaling
│
├── Editor/
│   ├── StormManagerEditor.cs            — Scene view gizmos showing storm structure
│   ├── StormProfileEditor.cs            — Custom inspector with wind curve preview
│   ├── StormPathEditor.cs               — Scene view spline editing
│   └── StormDebugWindow.cs              — Density field visualizer, perf counters
│
└── Shaders/
    └── Include/
        ├── StormCommon.hlsl             — Shared constants, utility functions
        ├── WindSampling.hlsl            — Sample wind field texture from any shader
        └── AtmosphericScattering.hlsl   — Mie/Rayleigh for storm atmosphere tinting
```

---

## 3. Core Simulation — The Single Source of Truth

### 3.1 StormInstance

Each active storm is a `StormInstance` — a plain C# class (not MonoBehaviour) owned by
`StormManager`. It holds:

```
StormInstance
├── StormType             enum: Hurricane, Tornado, Supercell, Derecho, Cyclone
├── Position              Vector3 (world space, updated each tick)
├── MovementPath          StormPath (spline + speed curve)
├── Intensity             float 0-1 (normalized, mapped to category via profile)
├── IntensityCurve        AnimationCurve (intensity over storm lifetime)
├── Phase                 enum: Forming, Mature, Dissipating
│
├── Profile (ScriptableObject reference)
│   ├── EyeRadius         float (meters) — 0 for tornado, 20-60km for hurricane
│   ├── MaxWindRadius     float (meters) — radius of maximum winds
│   ├── OuterRadius       float (meters) — total affected radius
│   ├── MaxWindSpeed      float (m/s at intensity=1)
│   ├── WindProfile       enum: Rankine, Holland, Custom
│   ├── HollandB          float (Holland profile shape parameter)
│   ├── CloudBaseAlt      float (meters)
│   ├── CloudTopAlt       float (meters)
│   ├── RotationDir       float (+1 or -1, hemisphere dependent)
│   ├── TiltAngle         float (tornado lean)
│   ├── PrecipRate        float (mm/hr at intensity=1)
│   └── DebrisThreshold   float (wind speed that starts picking up debris)
│
├── Derived (computed each tick, cached)
│   ├── CurrentMaxWind    float (MaxWindSpeed × Intensity)
│   ├── BoundingSphere    float (OuterRadius for broad-phase culling)
│   └── ActiveBands[]     struct { innerR, outerR, density, windMult }
```

### 3.2 WindField Query API

This is the critical interface that both visuals and destruction use:

```csharp
// CPU-side, no GPU involved. Pure math. Sub-microsecond per call.
public struct WindQuery
{
    public Vector3 windVelocity;     // direction + magnitude (m/s)
    public float   windSpeed;        // magnitude only
    public float   turbulence;       // 0-1, high near eyewall
    public float   precipitation;    // 0-1, rain/hail intensity at this point
    public float   distToEye;        // useful for destruction thresholds
}

public static class WindField
{
    // Single point query — for destruction, audio, individual objects
    public static WindQuery SampleAt(Vector3 worldPos);

    // Batch query — for particle systems that want CPU wind
    public static void SampleBatch(NativeArray<Vector3> positions,
                                   NativeArray<WindQuery> results);

    // GPU texture — for shaders and compute to sample
    // Updated once per frame by WindFieldCompute.compute
    public static RenderTexture WindFieldTexture { get; }
    // RG = wind XZ direction, B = wind speed, A = turbulence
    // World-space mapping: covers BoundingRect of all active storms + margin
}
```

### 3.3 How the wind math works

For a hurricane (Holland model):

```
windSpeed(r) = sqrt( (MaxWind² × (MaxWindR / r)^B × exp(1 - (MaxWindR/r)^B)) + (r × f/2)² ) - r×f/2

where:
  r = distance from storm center
  B = Holland shape parameter (1.0-2.5, controls eyewall sharpness)
  f = Coriolis parameter (latitude dependent, can be hardcoded)
```

Wind direction is tangential (perpendicular to radial direction) with an inflow angle
(~20° inward at surface, decreasing with altitude). This gives you the spiral.

For a tornado (Rankine vortex):

```
windSpeed(r) = MaxWind × (r / MaxWindR)           when r < MaxWindR  (solid body core)
windSpeed(r) = MaxWind × (MaxWindR / r)            when r > MaxWindR  (irrotational outer)
```

Plus a vertical component (updraft) that increases toward the center.

These are pure functions — no state, no simulation, just evaluate at any point. This is
why CPU queries are essentially free.

---

## 4. GPU Density Field — What the Raymarcher Reads

### 4.1 Why not a 3D texture?

A 3D texture covering a hurricane at reasonable resolution would be enormous. A Category 5
hurricane is ~500km across. At even 500m per texel that's a 1000³ texture — 4GB at float4.
Completely impractical.

### 4.2 The actual approach: Analytical density in the raymarcher

The raymarching shader evaluates cloud density **analytically** at each sample point. No
precomputed 3D texture for the storm shape. Instead:

```hlsl
float SampleStormDensity(float3 worldPos, StormData storm)
{
    float3 relPos = worldPos - storm.center;
    float r = length(relPos.xz);                    // radial distance
    float theta = atan2(relPos.z, relPos.x);         // angle
    float alt = worldPos.y;

    // 1. Vertical profile — cloud exists between base and top altitude
    float verticalMask = smoothstep(storm.cloudBase, storm.cloudBase + 500, alt)
                       * smoothstep(storm.cloudTop, storm.cloudTop - 1000, alt);

    // 2. Radial profile — high density at eyewall, spiral arms outward
    float eyewallDensity = exp(-sqr(r - storm.maxWindR) / (2 * sqr(storm.eyewallWidth)));

    // 3. Spiral arms — logarithmic spiral modulation
    float spiralPhase = theta - storm.spiralRate * log(max(r, 1.0)) + _Time.y * storm.rotSpeed;
    float spiralDensity = saturate(sin(spiralPhase * storm.armCount) * 0.5 + 0.3);

    // 4. Combine
    float baseDensity = max(eyewallDensity, spiralDensity * storm.outerDensity);
    baseDensity *= verticalMask;

    // 5. Add detail noise (THIS is where the texture sampling happens)
    float3 noiseUV = worldPos * storm.noiseScale + _Time.y * storm.noiseWind;
    float detailNoise = SampleNoiseOctaves(noiseUV, OCTAVE_COUNT);  // FBM
    baseDensity += (detailNoise - 0.5) * storm.noiseStrength;

    return saturate(baseDensity);
}
```

The noise textures are small (128³ or 256³ tiling 3D textures for FBM), same as what
HDRP's built-in volumetric clouds use. The storm shape is pure math — free to evaluate,
infinite resolution, no memory cost.

### 4.3 What IS stored in textures

| Texture | Format | Size | Updated | Purpose |
|---------|--------|------|---------|---------|
| Wind Field 2D | RGBA16f | 512×512 | Every frame | Wind XZ + speed + turbulence, world-space atlas |
| Noise 3D (detail) | R8 | 128³ | Static | Tiling Perlin/Worley FBM for cloud detail |
| Noise 3D (shape) | R8 | 32³ | Static | Low-freq shape noise |
| Curl Noise 3D | RG8 | 64³ | Static | For wispy edge displacement |
| Cloud Map 2D | RGBA8 | 1024×1024 | Every frame | Top-down density for shadow casting + ambient clouds |

The Cloud Map 2D is the only per-frame texture write for visuals. It's generated by a
simple compute shader that evaluates the storm density function from above, looking
straight down — essentially a "satellite view." This is used for cloud shadows on the
ground and for the ambient (non-storm) cloud coverage.

---

## 5. Rendering — The Custom CloudRenderer

### 5.1 Integration with HDRP

We implement `CloudSettings` (the Volume component) and `CloudRenderer` (the actual
renderer). HDRP calls our renderer at the right point in the frame:

```
StormCloudSettings : CloudSettings
├── Exposes: global cloud coverage, ambient wind, noise params
├── Plus: reference to StormManager for active storm data
└── GetCloudRendererType() → returns typeof(StormCloudRenderer)

StormCloudRenderer : CloudRenderer
├── RenderClouds(builtinParams, hdCamera, cmd)
│   ├── 1. Upload storm data to GPU (StructuredBuffer<StormData>)
│   ├── 2. Dispatch cloud map compute (for shadows)
│   ├── 3. Execute fullscreen raymarching pass
│   └── 4. Output: cloudColor + cloudTransmittance (composited by HDRP)
│
├── RenderCloudsIntoLightingCubemap(...)
│   └── Same raymarcher, lower quality, for ambient lighting
│
└── GetCloudHashCode() → changes when storm data changes (triggers re-bake)
```

### 5.2 Raymarcher structure

```hlsl
// Executed as a fullscreen pass by HDRP's cloud system
float4 RenderStormClouds(Varyings input) : SV_Target
{
    float3 rayOrigin = _WorldSpaceCameraPos;
    float3 rayDir = GetSkyViewDirWS(input.positionCS);

    // Intersect ray with cloud volume (atmosphere shell)
    float2 hits = IntersectCloudVolume(rayOrigin, rayDir, _CloudBaseAlt, _CloudTopAlt);
    if (hits.x > hits.y) return float4(0, 0, 0, 1); // miss

    // Adaptive step count based on quality setting
    int stepCount = _QualityStepCount;  // 32 / 64 / 128
    float stepSize = (hits.y - hits.x) / stepCount;

    float transmittance = 1.0;
    float3 scatteredLight = 0;

    // March through cloud volume
    for (int i = 0; i < stepCount; i++)
    {
        float t = hits.x + (i + BlueNoiseDither(input.positionCS)) * stepSize;
        float3 samplePos = rayOrigin + rayDir * t;

        // Evaluate density — ambient clouds + all active storms
        float density = SampleAmbientCloudDensity(samplePos);
        for (int s = 0; s < _StormCount; s++)
            density = max(density, SampleStormDensity(samplePos, _Storms[s]));

        if (density > 0.001)
        {
            // Light marching toward sun (cheaper inner loop)
            float lightTransmittance = MarchToLight(samplePos, _SunDirection, 6);

            // Phase function (Henyey-Greenstein for forward scattering)
            float phase = HGPhase(dot(rayDir, _SunDirection), _PhaseG);

            // Accumulate
            float3 luminance = _SunColor * lightTransmittance * phase
                             + _AmbientTop * saturate(density);

            float extinction = density * _ExtinctionCoeff * stepSize;
            scatteredLight += transmittance * luminance * (1 - exp(-extinction));
            transmittance *= exp(-extinction);

            if (transmittance < 0.01) break;  // early exit
        }
    }

    return float4(scatteredLight, transmittance);
}
```

### 5.3 Tornado funnel rendering

Tornadoes are tricky because the funnel is a narrow vertical structure that the main
cloud-volume raymarcher will undersample. Two options:

**Option A (recommended): Dedicated funnel pass.**
A second raymarching pass that only runs in screen-space pixels near the tornado (use a
screen-space bounding rect from the tornado's world-space cylinder projected). This pass
uses much smaller step sizes and evaluates the tornado vortex density at high resolution.
The result composites with the main cloud pass.

**Option B: Mesh-based funnel with shader distortion.**
A tapered cylinder mesh positioned at the tornado location. The vertex shader applies
rotational displacement and wobble. The fragment shader does short-range raymarching within
the mesh's bounds only (a "shell raymarcher"). This is cheaper and more art-directable but
doesn't integrate as seamlessly with the cloud layer above.

In practice, combine both: the main raymarcher draws the rotating supercell cloud mass
above, and a dedicated funnel pass or mesh handles the narrow funnel below. They share
density parameters from the same StormInstance, so they look unified.

---

## 6. Precipitation — Compute Shader Particle System

### 6.1 Why not VFX Graph?

VFX Graph is excellent for isolated effects, but for storm precipitation you need:

- Particles to be driven by the exact same wind field texture the clouds use
- Hundreds of thousands of particles covering a large area around the camera
- Particles to exist only where the storm model says there's precipitation
- Particle density to scale precisely with storm intensity
- All of this to be deterministic and synced with the simulation tick

VFX Graph can read textures and exposed properties, but the data flow is indirect and
fragile — you're binding properties by name, hoping the graph samples the right UV
coordinates, and debugging through a visual node graph when things desync. A compute
shader does the same work with direct, auditable code.

### 6.2 Architecture

```
RainSystem.cs (MonoBehaviour)
├── Manages two ComputeBuffers:
│   ├── ParticleBuffer (float4 pos + float4 vel+life × MAX_PARTICLES)
│   └── IndirectArgsBuffer (for DrawProceduralIndirect)
│
├── Each frame:
│   1. Dispatch RainSimulation.compute
│      ├── SpawnKernel: emit new particles in a disc around camera
│      │   ├── Position: random XZ within spawn radius, Y = cloud base altitude
│      │   ├── Only spawn where SampleWindField().precipitation > threshold
│      │   └── Spawn rate = precipitationIntensity × MAX_SPAWN_PER_FRAME
│      │
│      ├── SimulateKernel: update existing particles
│      │   ├── Sample wind field texture at particle position → get wind vector
│      │   ├── velocity += wind + gravity
│      │   ├── position += velocity × dt
│      │   ├── Kill if: Y < terrain height, or life expired, or outside cull radius
│      │   └── Write to append buffer for alive particles
│      │
│      └── IndirectArgsKernel: count alive particles → write draw args
│
│   2. Call Graphics.DrawProceduralIndirect
│      ├── Rain.shader reads ParticleBuffer via vertex ID
│      ├── Each particle → stretched billboard quad (2 triangles)
│      ├── Length/alpha based on velocity magnitude
│      └── Depth tested against scene, no shadow casting
│
└── Quality scaling:
    ├── MAX_PARTICLES: 50k / 200k / 500k
    ├── Spawn radius: 50m / 150m / 300m
    └── Billboard size: affects visual density at same particle count
```

### 6.3 Debris works identically but heavier

Same compute pattern but:
- Far fewer particles (500 / 2000 / 5000)
- Rendered as instanced meshes instead of billboards
- Have angular velocity (spin)
- Only spawn when wind speed > debris threshold from StormProfile
- Collide with scene depth buffer (read depth, kill or bounce on hit)
- Heavier drag, affected more by turbulence than laminar wind

---

## 7. Lightning

```
LightningSystem.cs
├── StrikeScheduler
│   ├── Poisson process: rate = f(storm.intensity, storm.type)
│   ├── Cloud-to-ground vs cloud-to-cloud ratio from profile
│   └── Strike position: random within eyewall band, biased by density
│
├── BoltGenerator.compute
│   ├── Input: start point (cloud base), end point (ground or cloud)
│   ├── Algorithm: recursive midpoint displacement
│   │   ├── Subdivide segment, displace midpoint by random perpendicular offset
│   │   ├── Recurse N levels (3-6 based on quality)
│   │   ├── At each level, chance to spawn branch (copy & continue with decay)
│   │   └── Output: StructuredBuffer<float3> of path vertices
│   │
│   └── Output: bolt path buffer + branch path buffers
│
├── LightningMeshBuilder.cs
│   ├── Reads bolt path buffer (GPU→CPU or build mesh in compute)
│   ├── Builds ribbon mesh: each segment → camera-facing quad
│   ├── Width tapers along path length and per-branch
│   └── Mesh is temporary — exists for ~0.1-0.3 seconds
│
├── Visual
│   ├── Lightning.shader: Unlit emissive, animated intensity curve
│   ├── Flash: momentary point light at strike point
│   ├── Glow: bloom from HDRP post-process handles this naturally
│   └── Ground impact: decal at strike position (scorch mark)
│
└── Audio
    ├── Thunder delay = distance / 343 (speed of sound)
    └── ThunderScheduler queues audio events with delay
```

---

## 8. Atmosphere & Lighting Integration

This is about making the world FEEL like it's in a storm, not just having clouds overhead.

```
StormAtmosphereController.cs
├── Reads: nearest storm intensity, distance, phase
│
├── Modifies HDRP Volumes (via Volume scripting API):
│   ├── Fog
│   │   ├── Increase mean free path as storm approaches
│   │   ├── Shift fog color toward grey-green (tornado) or grey (hurricane)
│   │   └── Increase max fog distance
│   │
│   ├── Exposure
│   │   ├── Decrease fixed exposure under storm
│   │   ├── Ramp: clear sky +14 EV → heavy storm +10 EV
│   │   └── Momentary spike during lightning flash
│   │
│   ├── Volumetric Lighting (HDRP's built-in)
│   │   ├── Increase scattering under storm (god rays through cloud breaks)
│   │   └── Anisotropy shift
│   │
│   ├── Directional Light
│   │   ├── Reduce intensity (cloud coverage blocks sunlight)
│   │   ├── Shift color toward desaturated blue
│   │   └── Shadow strength reduction (diffuse cloud lighting)
│   │
│   └── Indirect Lighting
│       ├── Tint ambient probe darker
│       └── Reduce reflection probe intensity
│
└── All transitions use Mathf.SmoothDamp with configurable speeds
    └── Storm approach feels gradual, not switched
```

---

## 9. Destruction Interface

```csharp
// Objects that want to react to storms implement this:
public interface IStormDamageable
{
    Transform Transform { get; }
    float     WindResistance { get; }  // m/s threshold before damage starts
    void      OnWindDamage(float magnitude, Vector3 windDir, float duration);
    void      OnDebrisImpact(float impactForce, Vector3 impactPoint);
    void      OnLightningStrike();     // if targeted
}

// This runs on FixedUpdate, queries the CPU wind field, applies forces:
public class WindForceApplicator : MonoBehaviour
{
    // Operates in two modes:
    //
    // Mode 1 — Rigidbody force application (physics objects)
    //   Query WindField.SampleAt(transform.position)
    //   Apply force = windDir * windSpeed² * dragCoeff * surfaceArea
    //   (wind force is proportional to velocity SQUARED, not linear)
    //
    // Mode 2 — Structural stress accumulation (buildings, trees)
    //   Accumulate stress += windSpeed² * dt
    //   When stress > threshold → trigger IStormDamageable.OnWindDamage()
    //   Allows progressive destruction: shutters break, then roof, then walls
    //
    // Optimization: only queries wind field if storm bounding sphere
    // overlaps object's position (broad-phase check via StormManager)
}
```

### 9.1 Destruction data flow

```
StormInstance (CPU)
    ↓
WindField.SampleAt(position)            ← pure math, ~0.2μs per call
    ↓
WindForceApplicator (FixedUpdate)       ← queries per-object
    ↓
├── Rigidbody.AddForce()               ← physics objects
├── IStormDamageable.OnWindDamage()     ← custom destruction
└── StructuralStressTracker.Accumulate() ← progressive damage

No GPU readback anywhere in this path.
The same StormInstance drives the clouds, rain, and destruction.
They are synchronized by construction.
```

---

## 10. Quality Settings — The Part That Actually Ships

### 10.1 Quality tiers

```csharp
[CreateAssetMenu]
public class StormQualitySettings : ScriptableObject
{
    [Header("Cloud Raymarching")]
    public int    rayStepCount;          // 32 / 64 / 128
    public int    lightStepCount;        // 3 / 6 / 8
    public float  renderScale;           // 0.5 / 0.75 / 1.0 (of screen res)
    public int    noiseOctaves;          // 2 / 4 / 6
    public bool   enableCurlErosion;     // false / true / true
    public int    temporalFrames;        // 4 / 8 / 16  (temporal reprojection)

    [Header("Precipitation")]
    public int    maxRainParticles;      // 50000 / 200000 / 500000
    public float  rainSpawnRadius;       // 50 / 150 / 300 meters
    public int    maxDebrisParticles;    // 500 / 2000 / 5000

    [Header("Lightning")]
    public int    boltSubdivisions;      // 3 / 5 / 6
    public bool   enableBranching;       // false / true / true
    public bool   enableGroundDecal;     // false / true / true

    [Header("Atmosphere")]
    public bool   enableVolumetricFog;   // false / true / true
    public bool   enableWetSurfaces;     // false / true / true
    public bool   enablePuddles;         // false / false / true

    [Header("Global")]
    public int    maxSimultaneousStorms; // 1 / 2 / 4
    public float  windFieldResolution;   // 256 / 512 / 1024 (texels)
}
```

### 10.2 What each setting actually costs

| Setting | GPU cost | Why it matters |
|---------|----------|---------------|
| rayStepCount 32→128 | ~2-6ms | More samples = better density resolve, fewer banding artifacts |
| renderScale 0.5→1.0 | ~1-4ms | Half-res raymarching + bilateral upscale is the single biggest perf win |
| lightStepCount 3→8 | ~0.5-2ms | Nested loop inside main march, multiplied by ray step count |
| noiseOctaves 2→6 | ~0.3-1ms | Each octave = one 3D texture sample per ray step |
| maxRainParticles 50k→500k | ~0.5-2ms | Compute dispatch + draw call, mostly fill-rate bound |
| temporalFrames 4→16 | quality only | More frames = smoother result, but more ghosting on fast camera moves |

### 10.3 Half-resolution rendering (the critical optimization)

The raymarcher renders to a half-res (or quarter-res) target, then upscales. This is the
same strategy HDRP's built-in volumetric clouds use. Implementation:

```
1. Render raymarcher at renderScale resolution → halfResCloudColor + halfResTransmittance
2. Bilateral upscale using scene depth as edge guide
   - For each full-res pixel, sample 4 nearest half-res pixels
   - Weight by depth similarity (reject samples with depth discontinuity)
   - This preserves sharp edges at cloud-vs-sky boundaries
3. Composite full-res result over scene
```

At 0.5 scale you're raymarching 1/4 the pixels — this alone is 4× faster.

### 10.4 Temporal reprojection (the second critical optimization)

Spread the raymarching cost across multiple frames:

```
Frame 0: march every 4th pixel (checkerboard pattern A)
Frame 1: march every 4th pixel (checkerboard pattern B)
Frame 2: march every 4th pixel (checkerboard pattern C)
Frame 3: march every 4th pixel (checkerboard pattern D)

Each frame: reproject previous frames' results using motion vectors
Result: 4× cheaper per frame, slight ghosting on fast motion
```

Combined with half-res: you're raymarching 1/16th of the original work. A 128-step
march that would cost ~8ms becomes ~0.5ms.

### 10.5 Dynamic quality scaling

```csharp
// StormQualityManager checks GPU frame time each frame
// If frame time exceeds budget, reduce quality incrementally:
//
// Level 0 (target met):     use configured quality preset
// Level 1 (slight overrun): reduce ray steps by 25%
// Level 2 (moderate):       reduce render scale one notch
// Level 3 (heavy):          reduce rain particles by 50%
// Level 4 (critical):       skip temporal frames, reduce to minimum
//
// Recovery: when headroom returns, step back up one level per second
// Uses hysteresis to prevent oscillation (different up/down thresholds)
```

---

## 11. Data Flow Diagram — One Frame

```
                         ┌───────────────────────────┐
                         │     StormSimulation.cs     │
                         │   (CPU, start of frame)    │
                         │                            │
                         │  Tick each StormInstance:   │
                         │  - advance along path      │
                         │  - update intensity curve   │
                         │  - update phase             │
                         │  - recompute derived data   │
                         └─────────┬─────────────────┘
                                   │
                     StormInstance data (CPU struct)
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                     │
              ▼                    ▼                     ▼
   ┌──────────────────┐  ┌─────────────────┐  ┌─────────────────────┐
   │   WindField.cs    │  │ Upload to GPU   │  │ StormAtmosphere     │
   │  (CPU queries)    │  │ StructuredBuf   │  │ Controller.cs       │
   │                   │  │ <StormData>     │  │                     │
   │  Used by:         │  └────────┬────────┘  │ Modifies HDRP       │
   │  - Destruction    │           │           │ volumes:             │
   │  - Audio          │           │           │ fog, exposure,       │
   │  - AI             │           │           │ light color          │
   └──────────────────┘           │           └─────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                     │
              ▼                    ▼                     ▼
   ┌──────────────────┐  ┌─────────────────┐  ┌─────────────────────┐
   │ WindFieldCompute  │  │ StormCloud      │  │ RainSimulation      │
   │ .compute          │  │ Renderer        │  │ .compute            │
   │                   │  │                 │  │                     │
   │ Writes 2D wind    │  │ Fullscreen      │  │ Spawn + simulate    │
   │ field texture     │  │ raymarching     │  │ particles using     │
   │ (for rain/debris  │  │ using storm     │  │ wind field texture  │
   │  compute to read) │  │ data + noise    │  │                     │
   └──────────────────┘  └─────────────────┘  └──────────┬──────────┘
                                                          │
                                                          ▼
                                               ┌─────────────────────┐
                                               │ DrawProcedural      │
                                               │ Indirect            │
                                               │                     │
                                               │ Rain streaks        │
                                               │ rendered as         │
                                               │ stretched quads     │
                                               └─────────────────────┘

    All GPU work reads from the same StormData buffer.
    All CPU work reads from the same StormInstance objects.
    There is exactly one place where storm state is defined.
```

---

## 12. What You Get vs. What You're Building

**Things that are genuinely hard to build (budget accordingly):**

- The raymarching shader with good lighting (~2-3 weeks for a senior graphics programmer)
- Temporal reprojection + bilateral upscale that doesn't ghost (~1 week)
- The tornado funnel rendering that blends with the cloud layer (~1-2 weeks)
- Tuning noise parameters to look convincing (~ongoing, art-dependent)

**Things that are straightforward (days, not weeks):**

- StormInstance / StormManager / WindField (~2-3 days)
- Compute shader rain/debris particles (~3-4 days)
- Lightning generation and rendering (~2-3 days)
- Atmosphere/lighting overrides (~1-2 days)
- Destruction interface (~1 day)
- Quality settings and dynamic scaling (~2-3 days)

**Things you absolutely don't need to build:**

- A fluid dynamics solver (analytical wind profiles are better for games)
- A custom atmospheric scattering model (HDRP's built-in works)
- Custom shadow mapping (HDRP handles this)
- Custom tone mapping / post-processing (HDRP handles this)
- Audio engine (use Unity's built-in spatial audio or FMOD/Wwise)

---

## 13. Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Raymarcher too expensive on target hardware | High | Half-res + temporal is the baseline, not optional. Build it first. |
| Tornado funnel looks disconnected from clouds | Medium | Share noise textures and density parameters. Match lighting model. |
| Wind field desync between CPU and GPU | High | CPU wind uses exact same math as GPU. Unit test both paths. |
| Temporal reprojection ghosting during fast camera motion | Medium | Use HDRP's motion vectors. Reduce temporal weight near screen edges. |
| Multiple storms overlapping causes perf cliff | Medium | MaxSimultaneousStorms cap. LOD distant storms to cloud-map-only. |
| Rain particles pop in/out at spawn boundary | Low | Large spawn radius with density falloff at edges. |

---

## 14. Recommended Build Order

```
Phase 1 — Foundation (weeks 1-2)
├── StormInstance + StormProfile + StormManager
├── WindField CPU query API
├── Basic analytical wind models (Rankine, Holland)
└── Editor gizmos showing storm structure

Phase 2 — Cloud Rendering (weeks 3-5)
├── Custom CloudSettings + CloudRenderer scaffolding
├── Raymarcher with ambient clouds only (no storms)
├── Add storm density functions to raymarcher
├── Half-res rendering + bilateral upscale
├── Temporal reprojection
└── Light marching + phase function

Phase 3 — Weather Effects (weeks 5-7)
├── Compute shader rain system
├── Wind field texture generation
├── Rain reads wind field for particle advection
├── Debris system (same pattern)
├── Lightning system
└── Atmosphere / lighting overrides

Phase 4 — Integration (weeks 7-8)
├── Destruction interface + WindForceApplicator
├── Quality settings + dynamic scaling
├── Tornado funnel rendering
├── Audio integration
└── Surface wetness / puddles

Phase 5 — Polish (weeks 8-10)
├── Noise tuning for visual quality
├── Per-storm-type visual profiles
├── Performance profiling + optimization
├── Edge cases (storm overlap, horizon rendering, etc.)
└── Debug visualization tools
```
