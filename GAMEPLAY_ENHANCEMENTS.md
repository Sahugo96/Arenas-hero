# Arena's Hero - Sonar Vision & Gameplay Ideas

---

## 0. ARENA DESIGN - "MEGABONK FINAL BOSS ARENA STYLE"

### 0.1 Core Concept
Circular arena inspired by **Megabonk final boss arena** landscape principles:
- **Shape**: Perfect circle, no edges/ledges (cannot fall off map)
- **Geometry**: Pillars, uneven elevations, mounds, depressions
- **Cover**: Pillars and elevation provide natural cover/LOS blockers
- **Symmetry**: Rotationally symmetric for competitive fairness
- **No Environmental Hazards**: No lava, pits, death zones

### 0.2 1v1 vs 3v3 Scaling

| Parameter | 1v1 Duel | 3v3 Team Arena |
|-----------|----------|----------------|
| **Radius** | 30m | 50m |
| **Pillar Count** | 8 (symmetrical) | 16 (symmetrical) |
| **Elevation Variance** | ±2m | ±3m |
| **Center Zone Radius** | N/A | 9m (Control objective) |
| **Spawn Points** | 2 (opposite ends) | 6 (3 per team, spaced) |

### 0.3 Layout Schematic (Top-Down)
```
        1v1 (30m radius)          3v3 (50m radius)
        
           ████                       ████████████
        ████████████              ████████████████
      ████  ██  ██  ████         ████  ██  ██  ██  ████
    ████  ██      ██  ████     ████  ██      ██  ██  ████
   ████  ██  ●●  ██  ████    ████  ██  ●●●●●●●●●  ██  ████
    ████  ██      ██  ████     ████  ██  ●●●●●●●●●  ██  ████
      ████  ██  ██  ████         ████  ██  ██  ██  ████
        ████████████              ████████████████
           ████                       ████████████

● = Pillars (cover/LOS blockers)     ● = Center Control Zone 9m radius (3v3 only)
■ = Player Spawns                    ■ = Team Spawns (3 per side)
```

### 0.4 Technical Implementation
```cpp
// AArenaLayout.h
UCLASS()
class AArenaLayout : public AActor
{
    GENERATED_BODY()

public:
    enum class EArenaSize : uint8 { Duel_1v1, Team_3v3 };

    UPROPERTY(EditDefaultsOnly) EArenaSize CurrentSize = EArenaSize::Duel_1v1;
    
    // Generated at runtime based on size
    UPROPERTY() TArray<FVector> PillarLocations;
    UPROPERTY() TArray<FVector> SpawnLocations;
    UPROPERTY() FVector CenterZoneLocation;
    UPROPERTY() float CenterZoneRadius;
    UPROPERTY() float ArenaRadius;
    UPROPERTY() TArray<FVector> RuneSpawnLocations; // 4 fixed corners

    void GenerateLayout(EArenaSize Size);
    bool IsLocationInArena(FVector Location) const; // Always true (no fall-off)
    TArray<FVector> GetCoverPositions() const;      // For AI/navigation
};
```

---

## 1. SONAR VISION / SEISMIC SENSE - COMPLETE IMPLEMENTATION

### 1.1 Core Concept: "The Blind Master"
> **Design Pillar**: Playing a Seismic Sense character means **voluntarily choosing blindness** for omnidirectional awareness through obstacles.

---

### 1.2 Three Implementation Variants

#### Variant 1: Full Screen Blackout (Hardcore/Immersive)
```
┌─────────────────────────────────────────────┐
│  ████████████████████████████████████████   │  <- Entire world = black
│  ████████████████████████████████████████   │
│  █████████  ● ENEMY  ███████████████████   │  <- Only seismic "pings" visible
│  ████████████████████████████████████████   │
│  ████████████████████████████████████████   │
├─────────────────────────────────────────────┤
│  [HP] [MANA] [RESERVE]    [Q][W][E][R][F]  │  <- UI ONLY visible
└─────────────────────────────────────────────┘
```
**Mechanics**:
- **Passive**: 360° seismic pulse every 0.5s (radius 15m, through walls)
- **Active (Q)**: Focused pulse - 30m radius, reveals enemy facing/animation state
- **Ultimate (F)**: "Earth's Memory" - reveals ALL enemy positions on map for 3s (100% reserve)
- **Tradeoff**: Cannot see projectiles, traps, environmental hazards visually
- **Audio**: Critical - footsteps, ability casts, movement have distinct seismic signatures

#### Variant 2: "Seismic Overlay" (Accessible/Competitive)
```
┌─────────────────────────────────────────────┐
│  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   │  <- World desaturated/grey
│  ░░░░░░░░░░  ● ENEMY  ░░░░░░░░░░░░░░░░░░   │  <- Seismic highlights in color
│  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   │
│  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   │
│  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   │
├─────────────────────────────────────────────┤
│  [HP] [MANA] [RESERVE]    [Q][W][E][R][F]  │
└─────────────────────────────────────────────┘
```
**Mechanics**:
- World rendered desaturated (post-process)
- Seismic contacts = glowing outlines + velocity vectors
- Can still see projectiles/environment (reduced clarity)
- **Default for ranked**, Variant 1 = "Mastery Challenge" mode

#### Variant 3: Hybrid - "Pulse Vision" (RECOMMENDED DEFAULT)
- **Passive**: World is dark but not black. Silhouettes of geometry visible (low ambient)
- **Active Pulse (Cooldown)**: Full "sonar ping" - reveals everything in radius with detail
- **Seismic Sense Passive**: Enemies moving on ground = constant ripple outlines
- **Air/Water movement**: 50% reduced detection (silent)
- **Earth/Fire movement**: 150% detection (heavy footsteps)

---

### 1.3 Technical Implementation: Seismic Sense Component

```cpp
// USeismicSenseComponent.h
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class USeismicSenseComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    enum class ESeismicMode : uint8
    {
        Passive,      // Continuous low-res detection
        ActivePulse,  // High-res, cooldown, reserve cost
        Ultimate      // Global, 100% reserve
    };

    struct FSeismicContact
    {
        FVector WorldLocation;
        FVector Velocity;
        float Confidence;           // 0.0-1.0
        ESeismicContactType Type;   // Enemy, Projectile, Trap, Structure
        float TimeSinceUpdate;
        bool bIsMoving;
    };

    UPROPERTY(ReplicatedUsing=OnRep_Contacts)
    TArray<FSeismicContact> ActiveContacts;

    // Configuration
    UPROPERTY(EditDefaultsOnly) float PassiveRadius = 1500.f;
    UPROPERTY(EditDefaultsOnly) float ActiveRadius = 3000.f;
    UPROPERTY(EditDefaultsOnly) float PassiveUpdateRate = 0.5f;
    UPROPERTY(EditDefaultsOnly) float ActiveCooldown = 8.f;
    UPROPERTY(EditDefaultsOnly) int32 ActiveReserveCost = 50;
    UPROPERTY(EditDefaultsOnly) int32 UltimateReserveCost = 500;

    UPROPERTY(EditDefaultsOnly) float AirWaterDetectionMultiplier = 0.5f;
    UPROPERTY(EditDefaultsOnly) float EarthFireDetectionMultiplier = 1.5f;
    UPROPERTY(EditDefaultsOnly) float StillTargetMultiplier = 0.2f;

    virtual void TickComponent(float DeltaTime, ELevelTick, FActorComponentTickFunction*) override;

    UFUNCTION(BlueprintCallable) void RequestActivePulse();
    UFUNCTION(BlueprintCallable) void RequestUltimate();

protected:
    void PerformSeismicScan(float Radius, bool bHighRes);
    float CalculateConfidence(ADuelCharacter* Target, const FVector& ScanOrigin);
    void UpdateClientVisualization();
};

// Server Tick Logic
void USeismicSenseComponent::TickComponent(float DeltaTime, ...)
{
    if (!GetOwner()->HasAuthority()) return;

    PassiveTimer += DeltaTime;
    if (PassiveTimer >= PassiveUpdateRate)
    {
        PassiveTimer = 0.f;
        PerformSeismicScan(PassiveRadius, false);
    }
}

void USeismicSenseComponent::PerformSeismicScan(float Radius, bool bHighRes)
{
    TArray<FOverlapResult> Overlaps;
    FCollisionShape Sphere = FCollisionShape::MakeSphere(Radius);
    
    GetWorld()->OverlapMultiByChannel(Overlaps, GetOwner()->GetActorLocation(), 
        FQuat::Identity, ECC_Pawn, Sphere);

    TArray<FSeismicContact> NewContacts;
    
    for (auto& Overlap : Overlaps)
    {
        if (ADuelCharacter* Target = Cast<ADuelCharacter>(Overlap.GetActor()))
        {
            if (Target == GetOwner()) continue;
            
            FSeismicContact Contact;
            Contact.WorldLocation = Target->GetActorLocation();
            Contact.Velocity = Target->GetVelocity();
            Contact.bIsMoving = Contact.Velocity.Size() > 50.f;
            Contact.Confidence = CalculateConfidence(Target, GetOwner()->GetActorLocation());
            Contact.Type = ESeismicContactType::Enemy;
            Contact.TimeSinceUpdate = 0.f;
            
            NewContacts.Add(Contact);
        }
    }

    ActiveContacts = MergeContacts(ActiveContacts, NewContacts);
    MarkArrayDirty();
}

float USeismicSenseComponent::CalculateConfidence(ADuelCharacter* Target, const FVector& Origin)
{
    float Confidence = 1.0f;
    
    // Element modifiers (configurable per character)
    if (Target->HasTag("SilentMovement")) Confidence *= AirWaterDetectionMultiplier;
    else if (Target->HasTag("HeavyMovement")) Confidence *= EarthFireDetectionMultiplier;

    // Distance falloff
    float Dist = FVector::Dist(Origin, Target->GetActorLocation());
    Confidence *= FMath::Clamp(1.f - Dist / PassiveRadius, 0.3f, 1.f);

    // Movement speed factor
    float Speed = Target->GetVelocity().Size();
    Confidence *= FMath::Clamp(Speed / 1000.f, 0.2f, 1.5f);

    // Still = nearly invisible
    if (!Contact.bIsMoving) Confidence *= StillTargetMultiplier;

    // Stealth abilities
    if (Target->HasActiveGameplayEffect("Stealth")) Confidence *= 0.1f;

    return FMath::Clamp(Confidence, 0.f, 1.f);
}
```

---

### 1.4 Client-Side Visualization: Sonar Post-Process Material

```hlsl
// SonarVision.ush - Post-process material

// Uniforms
float ScanTime;           // Pulse animation timer
float WorldRadius;        // Current scan radius in world units
Texture2D SonarContacts;  // Render target with contact points
float3 PlayerWorldPos;    // Player position for distance calc

float3 SonarVision(float2 UV, float3 WorldPos)
{
    float3 BaseColor = SceneTexture::PostProcessInput0.Sample(UV).rgb;
    
    // Desaturate world to dark ambient
    float Lum = dot(BaseColor, float3(0.299, 0.587, 0.114));
    float3 Desaturated = float3(Lum, Lum, Lum) * 0.25;
    
    // Sonar sweep line (rotating)
    float2 CenterUV = float2(0.5, 0.5);
    float DistFromCenter = length(UV - CenterUV) * 2.0;
    float SweepAngle = atan2(UV.y - 0.5, UV.x - 0.5);
    float SweepProgress = frac(ScanTime * 0.5); // 2s per revolution
    float SweepWidth = 0.05;
    float SweepIntensity = smoothstep(SweepProgress - SweepWidth, SweepProgress, 
        (SweepAngle + PI) / (2 * PI)) * (1.0 - DistFromCenter);
    
    // Active pulse ring (expanding circle)
    float WorldDist = distance(WorldPos, PlayerWorldPos);
    float PulseRing = smoothstep(WorldRadius - 100, WorldRadius + 100, WorldDist);
    PulseRing *= (1.0 - DistFromCenter) * 2.0;
    
    // Contact highlights from render target
    float4 ContactData = SonarContacts.Sample(Sampler, UV);
    float ContactStrength = ContactData.r;
    float2 VelocityDir = ContactData.gb * 2.0 - 1.0;
    float ContactVelocity = length(VelocityDir);
    
    // Build final color
    float3 Result = Desaturated;
    
    // Sweep line
    Result += float3(0.0, 0.9, 1.0) * SweepIntensity * 3.0;
    
    // Pulse ring
    Result += float3(0.2, 1.0, 0.5) * PulseRing;
    
    // Contacts - cyan for enemies, yellow for projectiles
    if (ContactStrength > 0.1)
    {
        float3 ContactColor = lerp(float3(1.0, 1.0, 0.0), float3(0.0, 1.0, 1.0), ContactData.a);
        Result += ContactColor * ContactStrength * (1.0 + ContactVelocity * 0.5);
        
        // Velocity indicator line
        float2 ToContact = UV - CenterUV;
        float VelDot = dot(normalize(ToContact), normalize(VelocityDir));
        if (VelDot > 0.7) // Moving toward/away indicator
        {
            Result += float3(1.0, 1.0, 1.0) * ContactStrength * VelDot * 0.3;
        }
    }
    
    // Vignette (tunnel vision)
    Result *= 1.0 - smoothstep(0.65, 1.0, DistFromCenter);
    
    return saturate(Result);
}
```

---

### 1.5 Audio Design for Blind Play (MetaSounds)

| Event | Seismic Audio Cue | Spatialization |
|-------|-------------------|----------------|
| **Enemy Footstep** | Low freq "thump" 60-80Hz | Binaural, distance-attenuated |
| **Enemy Dash/Blink** | Sharp "crack" + doppler | Directional, high priority |
| **Projectile Launch** | Element-specific whoosh | Fire=hiss, Ice=crack, Air=flutter |
| **Ability Cast** | Chord: Major=ally, Minor=enemy | Spatialized to caster |
| **Active Pulse** | Descending sweep tone | Centered on player |
| **Ultimate** | Deep resonant hum + reverb | Omnidirectional |

**MetaSound Parameters**:
- Distance → LPF cutoff, volume, reverb send
- Confidence → Clarity vs noise floor
- Element type → Waveform, harmonic content

---

### 1.6 Accessibility Options

| Mode | Visual | Audio | Target Audience |
|------|--------|-------|-----------------|
| **Full Blind** | Black screen, only UI | Full 3D spatial | Mastery challenge |
| **Overlay** | Desaturated + highlights | Full 3D spatial | Ranked default |
| **Pulse** | Dark ambient + active pings | Full 3D spatial | New players (recommended) |
| **Audio-Only** | Normal vision | Seismic audio only | Screen reader users |
| **High Contrast** | Neon outlines, configurable | Standard | Low vision |
| **Reduced Motion** | No sweep, no vignette, no shake | Standard | Vestibular disorders |

---

## 2. FOG OF WAR - SERVER AUTHORITATIVE (KEPT FOR CONTEXT)

### Approach A: Grid-Based (RECOMMENDED FOR 1v1)
- 2D grid over walkable area, 100-200 units/cell
- States: 0=unexplored, 1=explored, 2=visible
- Server rasterizes vision sources each frame
- Replicates dirty cells only to relevant clients
- Client: Post-process material sampling 512x512 fog texture

### Key Principle: Server-Side Vision Culling
```cpp
// NetDriver override - hidden actors NEVER replicated
bool ADuelNetDriver::ShouldReplicateActor(AActor* Actor, const FNetViewer& Viewer)
{
    if (!Super::ShouldReplicateActor(Actor, Viewer)) return false;
    return FoWManager->IsVisibleTo(Viewer.InViewer, Actor);
}
```

---

## 3. 150 "TRASH IDEAS WITH A TWIST" (Cross-Genre Inspiration)

*Each takes a mechanic from an unrelated genre/game and twists it for 1v1 arena duel*

### Movement & Traversal (15)
2. **Mirror's Edge Runner Vision** → "Flow Path": Holding Shift highlights optimal dodge/attack paths in world geometry, following grants 15% speed
8. **Portal Momentum** → "Blink Portal": Place entrance/exit portals (cooldown), fall through = conserve momentum, launch at angles
10. **Apex Legends Wall Run** → "Airbender Wall Kick": Kick off walls for verticality, chain 3 = brief flight, cooldown per wall

### Combat & Abilities (25)
17. **Bayonetta Witch Time** → "Perfect Dodge": Dodge at exact impact frame = 3s time dilation (enemy 0.25x, you 1x), 20s CD
22. **Marvel vs Capcom Assist** → "Spirit Assist": Summon AI spirit (30s CD) that uses one ability, shares your reserve
26. **Melty Blood Moon Drive** → "Lunar Phase": Match timer phases (Day/Night/Twilight), each buffs different elements
39. **Osu! Sliders** → "Channeled Aim": Hold ability = slider path appears, follow with mouse for max damage

### Strategy & Mind Games (20)
41. **Poker Bluffing** → "False Tell": Channel ultimate (visual/audio) but cancel early = enemy wasted cooldown
45. **Mastermind Code** → "Elemental Weakness": Enemy has hidden elemental weakness, discover via trial
55. **Battle Royale Circle** → "Closing Mists": Map shrinks over 5min, outside = rapid reserve drain
56. **Extraction Shooter** → "Loot & Extract": Win = extract with artifacts, death = lose everything
57. **Roguelike Relics** → "Per-Run Artifacts": Draft 3 artifacts at match start, persist for tournament set
58. **Deckbuilder** → "Ability Deck": Start with 5 abilities, draw 3/turn, build deck between rounds

### Progression & Meta (Archived)

### Social & Spectacle (Archived)

### Simulation/Management (10)
150. **SimCity** → "Zone Control": Capture zones = passive resource generation

---

## 4. 30 GENUINELY WELL-THOUGHT-OUT IDEAS

### 4.1 Core Combat & Mechanics (8)

#### 2. **Elemental Residue System** (From Divinity: Original Sin + fighting game okizeme)
**Concept**: Abilities leave persistent "residue" on ground/walls for 8s.
- **Fire**: Burning ground (DoT), ignites other residues
- **Water**: Wet ground (conducts lightning, extinguishes fire)
- **Earth**: Dust cloud (blocks vision, slows)
- **Air**: Updraft (launches, extends jumps)
- **Lightning**: Charged ground (arcs to wet targets)
- **Ice**: Slippery ground (uncontrollable slide)
**Interactions**: Fire + Water = Steam (vision block), Lightning + Water = Chain, Earth + Air = Dust Devil
**Why**: Creates battlefield memory, rewards combo planning, zone control without permanent structures.

#### 3. **Adaptive Cooldowns** (From MOBA CDR + RPG rest mechanics)
**Concept**: Cooldowns dynamically adjust based on context.
- **Hit Enemy**: -15% current CDs (aggression rewarded)
- **Perfect Dodge**: -30% current CDs (skill rewarded)
- **Take Damage**: +20% current CDs (punishment)
- **Low HP (<30%)**: All CDs -40% (comeback mechanic)
- **High Reserve (>80%)**: CDs -10% (resource management)
- **Out of Combat 5s**: CDs -50% (recovery)
**Why**: Creates natural pacing, rewards skill expression, prevents stale standoffs.

#### 5. **Ability Modification Slots** (From Path of Exile support gems + LoL runes)
**Concept**: Each ability has 2 modification slots. Insert "Mod Chips" found/crafted.
- **Damage Chips**: +Damage, +Crit, +Penetration, +AoE
- **Utility Chips**: +Range, +Speed, +Pierce, +Homing, +Bounce
- **Defensive Chips**: +Shield, +Heal, +Cleanse, +Block, +Reflect
- **Weird Chips**: +Delay (trap), +Echo (repeat), +Split (3-way), +Swap (positions)
**Rarity**: Common/Uncommon/Rare/Epic/Legendary (cosmetic only, power balanced)
**Why**: Build diversity without power creep, collection meta, theorycrafting.

#### 6. **Environmental Hazard Synergy** (From Divinity + Magicka)
**Concept**: Map hazards have elemental states. Abilities change them.
- **Water Pool** → Freeze (Ice) = Solid bridge → Melt (Fire) = Steam cloud
- **Lava** → Cool (Water) = Obsidian wall → Heat (Fire) = Lava again
- **Metal Debris** → Magnetize (Lightning) = Pull enemies → Demagnetize = Explode
- **Gas Vent** → Ignite (Fire) = Explosion → Extinguish = Safe zone
- **Crystals** → Resonate (Sound) = Shield → Shatter (Earth) = Shrapnel
**Why**: Map mastery = advantage, dynamic battlefield, encourages exploration.

---

### 4.2 Strategic Depth & Meta (8)

#### 9. **Draft Phase (Pre-Match)** (From MOBA draft + card games)
**Concept**: 30s draft before each match (BO3).
- **Ban Phase**: Each bans 1 character (2 total banned)
- **Pick Phase**: Alternating picks (1-2-2-1), see enemy picks
- **Counter-Pick**: Last pick sees full enemy team
- **Blind Draft Option**: For casual (simultaneous reveal)
**Why**: Strategic depth, counter-play, reduces hard-counter frustration.

#### 10. **Dynamic Map Events** (From battle royale circles + MOBA objectives)
**Concept**: Scripted events every 60-90s that change the map.
- **Meteor Shower**: Random impacts create craters (cover) + fire patches
- **Tidal Surge**: Water rises 2m, swim speed +50%, fire -50%
- **Earthquake**: Walls crack (new paths), dust clouds (vision -50%)
- **Solar Flare**: Vision range +100%, fire +30%, stealth disabled
- **Void Rift**: Random teleport zones appear, reserve regen +200%
- **Aurora**: All elements +15% damage, reactions +50%
**Why**: Breaks stalemates, map knowledge tested, spectator excitement.

---

## 4. 30 GENUINELY WELL-THOUGHT-OUT IDEAS

### 4.1 Core Combat & Mechanics (8)

#### 2. **Elemental Residue System** (From Divinity: Original Sin + fighting game okizeme)
- **AI Prediction**: "Player likely to: Dash → Fireball → Parry" (based on patterns)
- **Probability**: Shows % for top 3 predicted actions
- **Accuracy Tracking**: Public stats on prediction accuracy
- **Player Opt-Out**: Can disable for own matches

#### 26. **Interactive Spectator Betting** (From Twitch predictions + salary cap)
- **Currency**: "Arena Coins" (earned by watching, no real money)
- **Markets**: Round winner, first blood, most damage, perfect parries
- **Odds**: Dynamic based on bets
- **Rewards**: Cosmetics, titles, profile flair

#### 27. **Multi-Language Commentary Support** (From Dota 2 / LoL)
- **Audio Tracks**: Multiple commentator languages
- **Auto-Translate**: Chat/voice → text in viewer language
- **Terminology Glossary**: Hover any term → definition

#### 28. **Tournament Administration Suite** (From Smash.gg + Battlefy)
- **Bracket Generator**: Single/Double elim, Swiss, Round Robin
- **Auto-Scheduling**: Matches auto-assign, notify players
- **Pause/Resume**: Admin control, crash recovery
- **Rule Enforcement**: Auto-verify settings, character locks
- **Stream Integration**: OBS plugin for auto-scenes

#### 29. **Historical Match Archive** (From chess databases + sports)
- **Search**: By player, character, date, tournament, outcome
- **Meta Analysis**: "Earth vs Fire winrate Patch 1.3: 52%"
- **Legendary Matches**: Curated "Greatest Duels" with commentary
- **Player Profiles**: Career stats, signature moments, rivalries

#### 30. **Custom Game Scripting** (From WC3/SC2 custom maps + Roblox)
- **Visual Scripting**: Node-based (Blueprint-style) for custom modes
- **API**: Access to all gameplay systems (mana, FoW, runes, camera)
- **Share**: Workshop/Steam Workshop integration
- **Monetization**: Creators earn from popular modes (cosmetic revenue share)
- **Examples**: "Gun Game", "King of the Hill", "Zombie Survival", "Ability Draft"

---

## 5. CORE MECHANICS & ABILITIES (New Systems)

### 5.1 Basic Auto-Attack
**Zero-mana basic attack** available to all characters.
- **Input**: Left Mouse Button (configurable)
- **Cost**: 0 Primary Mana, 0 Reserve Mana
- **Damage**: Base 35-45 (scales with Spell Power)
- **Speed**: 0.6s attack speed (1.67 attacks/sec base)
- **Range**: Melee (200 units) or Projectile (800 units) per character
- **Properties**: 
  - Cannot be Enhanced (Shift) or Weakened (Ctrl)
  - Generates 2 Primary Mana on hit
  - Does not break Combat Rush combo
  - Can be cancelled into any ability (animation cancel window: 0.15s)

---

### 5.2 Shadow Dash (Double-Tap Dodge)
**Double-tap movement for instant directional dash** - the Attack on Titan gas reference was only for conceptual understanding of the "burst" feel.

| Property | Value |
|----------|-------|
| **Input** | Double-tap WASD (or movement keys) within 0.25s |
| **Cooldown** | 4 seconds (shared across all directions) |
| **Distance** | 12 meters (1200 units) |
| **Duration** | 0.3 seconds (300ms) |
| **Cost** | 0 Mana (free) |
| **Invulnerability** | Full (0.3s) - immune to damage, CC, projectiles |
| **Visual** | Shadow/void particles from character, motion blur, screen shake |
| **Audio** | Sharp *WHOOSH* + void tear sound |
| **Direction Lock** | Dash direction = last tapped direction |
| **Mid-Air** | Usable in air (preserves momentum) |
| **Collision** | **Passes through players** during dash animation (ghost), **stops on walls/textures** |

**Mechanics**:
- **Double-tap detection**: 250ms window between taps
- **No mana cost** - purely cooldown based
- **Passes through enemies/ally** during 0.3s dash (ghost collision)
- **Cannot pass through walls, pillars, map geometry** - stops on collision
- **End lag**: 0.1s recovery (cannot act, can move)

```cpp
// UShadowDashComponent.h
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class UShadowDashComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly) float DashDistance = 1200.f;
    UPROPERTY(EditDefaultsOnly) float DashDuration = 0.3f;
    UPROPERTY(EditDefaultsOnly) float Cooldown = 4.0f;
    UPROPERTY(EditDefaultsOnly) float DoubleTapWindow = 0.25f;
    UPROPERTY(EditDefaultsOnly) float EndLag = 0.1f;

    // Per-direction tracking
    TMap<FKey, float> LastPressTime;
    TMap<FKey, int32> PressCount;

    virtual void TickComponent(float DeltaTime, ...) override;

    void OnMovementKeyPressed(FKey Key)
    {
        float Now = GetWorld()->GetTimeSeconds();
        float& LastTime = LastPressTime.FindOrAdd(Key);
        int32& Count = PressCount.FindOrAdd(Key);
        
        if (Now - LastTime <= DoubleTapWindow)
        {
            Count++;
            if (Count >= 2)
            {
                TryDash(Key);
                Count = 0; // Reset
            }
        }
        else
        {
            Count = 1;
        }
        LastTime = Now;
    }

    void TryDash(FKey DirectionKey)
    {
        if (bOnCooldown || bDashing) return;
        
        FVector DashDir = GetDirectionFromKey(DirectionKey);
        if (DashDir.IsNearlyZero()) return;

        StartDash(DashDir);
    }

    void StartDash(FVector Direction)
    {
        bDashing = true;
        bOnCooldown = true;
        CooldownTimer = Cooldown;
        
        ADuelCharacter* Owner = Cast<ADuelCharacter>(GetOwner());
        Owner->SetInvulnerable(true);
        Owner->LaunchCharacter(Direction * DashDistance / DashDuration, true, true);
        
        // Spawn gas burst effect
        SpawnDashEffect(Direction);
        
        // Timer to end dash
        FTimerHandle DashEndTimer;
        GetWorld()->GetTimerManager().SetTimer(DashEndTimer, [this, Owner]()
        {
            EndDash(Owner);
        }, DashDuration, false);
    }

    void EndDash(ADuelCharacter* Owner)
    {
        bDashing = false;
        Owner->SetInvulnerable(false);
        
        // End lag timer
        FTimerHandle EndLagTimer;
        GetWorld()->GetTimerManager().SetTimer(EndLagTimer, [this]()
        {
            bCanAct = true;
        }, EndLag, false);
    }
};
```

---

### 5.3 Player Collision (CS-Style Body Blocking)
**All characters have solid collision with each other** - no passing through during normal movement.

| Property | Value |
|----------|-------|
| **Collision Type** | Capsule vs Capsule (CharacterMovementComponent) |
| **Behavior** | Solid blocking - cannot walk through enemies or allies |
| **Push Force** | None (no pushing, hard stop) |
| **Dash Interaction** | **Shadow Dash passes through players** (ghost during 0.3s), stops on walls |
| **Projectile Interaction** | Players block projectiles (friendly fire off, but collision on) |
| **3v3 Stacking** | No stacking - 3 players cannot occupy same space |

**Implementation**:
```cpp
// In ACharacter or ADuelCharacter
GetCapsuleComponent()->SetCollisionResponseToChannel(ECC_Pawn, ECR_Block);
GetCharacterMovement()->bEnablePhysicsInteraction = true;
// Shadow Dash: temporarily set ECC_Pawn to ECR_Overlap for 0.3s
```

---

### 5.4 Trap Ability (Bear Trap / Snare)
**Placeable immobilizing trap** - affects anyone who steps on it (including placer). **No enhanced/weakened variants.**

| Property | Value |
|----------|-------|
| **Input** | G (configurable) |
| **Type** | Placeable actor (trap) |
| **Placement** | At feet, instant (no channel) |
| **Max Active** | 2 traps per player (placing 3rd destroys oldest) |
| **Arming Time** | 1.5 seconds (visible, can be avoided) |
| **Trigger Radius** | 100 units (center of trap) |
| **Effect** | **Root 1.5s** (cannot move, can cast/attack/turn) |
| **Damage** | 0 (pure CC) |
| **Visibility** | Visible to placer always; invisible to enemies until armed (then faint outline) |
| **Seismic Sense** | Detects as "Trap" contact type |
| **Friendly Fire** | **YES** - placer can trigger own trap |
| **Destruction** | 1 HP (any attack destroys), 500 HP if armed |
| **Duration** | 60 seconds or until triggered/destroyed |
| **Cost** | 20 Primary Mana |
| **Cooldown** | 12 seconds (per trap slot) |

```cpp
// ATrapActor.h
UCLASS()
class ATrapActor : public AActor
{
    GENERATED_BODY()

public:
    UPROPERTY(VisibleAnywhere) UStaticMeshComponent* TrapMesh;
    UPROPERTY(VisibleAnywhere) UBoxComponent* TriggerVolume;
    UPROPERTY(VisibleAnywhere) UParticleSystemComponent* ArmedEffect;
    UPROPERTY(VisibleAnywhere) UHealthComponent* HealthComponent;

    UPROPERTY(Replicated) ADuelCharacter* Placer;
    UPROPERTY(Replicated) bool bIsArmed = false;
    UPROPERTY(Replicated) float ArmTime = 0.f;
    UPROPERTY(Replicated) float MaxDuration = 60.f;

    virtual void BeginPlay() override
    {
        // Initially unarmed - no collision for enemies
        TriggerVolume->SetCollisionEnabled(ECollisionEnabled::NoCollision);
        ArmedEffect->SetVisibility(false);
        HealthComponent->Health = 1.f; // 1 HP unarmed
        
        // Arm timer
        GetWorld()->GetTimerManager().SetTimer(ArmTimer, this, &ATrapActor::ArmTrap, ArmDelay, false);
    }

    void ArmTrap()
    {
        bIsArmed = true;
        TriggerVolume->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
        ArmedEffect->SetVisibility(true);
        HealthComponent->Health = 500.f; // 500 HP armed
        
        // Auto-destruct after max duration
        GetWorld()->GetTimerManager().SetTimer(DestructTimer, this, &ATrapActor::Destroy, MaxDuration, false);
    }

    UFUNCTION()
    void OnOverlapBegin(UPrimitiveComponent* Overlapped, AActor* Other, ...)
    {
        if (!bIsArmed) return;
        if (ADuelCharacter* Char = Cast<ADuelCharacter>(Other))
        {
            // ROOT for 1.5s - affects EVERYONE including placer
            Char->ApplyRoot(1.5f);
            
            // Visual/audio feedback
            TriggerEffect(Char);
            
            // Trap consumed
            Destroy();
        }
    }
};
```

---

### 5.5 Madness on the Edge (Бешенство на грани)
**High-risk, high-reward toggle stance** - burns resources for power.

| Property | Value |
|----------|-------|
| **Type** | Toggle ability (press to activate, press again to deactivate) |
| **Input** | V (configurable) |
| **Activation Cost** | 2% Max Reserve Mana + 2% Max Health |
| **Maintenance Cost** | 1% Max Reserve Mana/sec + 1% Max Health/sec |
| **Attack Buff** | **+30% damage** on all offensive abilities |
| **Buff Cost** | **+5% Max Reserve Mana + 5% Max Health** per ability cast (on top of normal cost) |
| **Auto-Attack** | Also buffed (+30% dmg), costs +5% Reserve+Health per hit |
| **Deactivation** | Press V again OR resource insufficient |
| **Cooldown** | **30 seconds** after deactivation (cannot re-enable) |
| **Visual** | Character glows red/black, veins pulse, screen vignette darkens |
| **Audio** | Heartbeat + distorted breathing, intensifies over time |
| **UI** | Resource bars show "burning" animation, stance indicator |

**Mechanics**:
- **Resource Check**: Every tick + on ability cast - if insufficient → auto-deactivate
- **Minimum Resources**: Cannot activate if Reserve < 10% OR Health < 15%
- **Healing Blocked**: No health/mana regen while active (including runes)
- **Death Risk**: Can kill self from maintenance cost if not careful
- **Counterplay**: Enemy sees visual/audio, knows you're burning resources - burst you down
- **Synergy**: Works with Combat Rush (mana reduction applies to base cost, not buff surcharge)

**Resource Math Example** (Max Reserve 500, Max Health 1000):
```
Activate: -10 Reserve, -20 Health
Tick (1s): -5 Reserve, -10 Health
Cast Fireball (base 50 Reserve): -50 Reserve - 5 Reserve (buff) - 5 Health (buff) = -60 Reserve, -5 Health
Auto-attack: -5 Reserve, -5 Health per hit
Deactivate: 30s CD starts
```

```cpp
// UMadnessStanceComponent.h
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class UMadnessStanceComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly) float ActivationReservePct = 0.02f;  // 2%
    UPROPERTY(EditDefaultsOnly) float ActivationHealthPct = 0.02f;   // 2%
    UPROPERTY(EditDefaultsOnly) float MaintenanceReservePctPerSec = 0.01f; // 1%/s
    UPROPERTY(EditDefaultsOnly) float MaintenanceHealthPctPerSec = 0.01f;  // 1%/s
    UPROPERTY(EditDefaultsOnly) float BuffDamageMultiplier = 1.3f;   // +30%
    UPROPERTY(EditDefaultsOnly) float BuffCastReservePct = 0.05f;    // +5% per cast
    UPROPERTY(EditDefaultsOnly) float BuffCastHealthPct = 0.05f;     // +5% per cast
    UPROPERTY(EditDefaultsOnly) float Cooldown = 30.0f;
    UPROPERTY(EditDefaultsOnly) float MinReservePctToActivate = 0.10f;
    UPROPERTY(EditDefaultsOnly) float MinHealthPctToActivate = 0.15f;

    UPROPERTY(Replicated) bool bIsActive = false;
    UPROPERTY(Replicated) float CooldownRemaining = 0.f;
    float TickTimer = 0.f;

    UFUNCTION(Server, Reliable) void ServerToggleStance()
    {
        if (bIsActive)
        {
            DeactivateStance();
        }
        else if (CanActivate())
        {
            ActivateStance();
        }
    }

    bool CanActivate() const
    {
        ADuelCharacter* Owner = Cast<ADuelCharacter>(GetOwner());
        if (!Owner) return false;
        if (CooldownRemaining > 0) return false;
        if (Owner->GetReserveMana() / Owner->GetMaxReserveMana() < MinReservePctToActivate) return false;
        if (Owner->GetHealth() / Owner->GetMaxHealth() < MinHealthPctToActivate) return false;
        return true;
    }

    void ActivateStance()
    {
        ADuelCharacter* Owner = Cast<ADuelCharacter>(GetOwner());
        Owner->AddReserveMana(-Owner->GetMaxReserveMana() * ActivationReservePct);
        Owner->AddHealth(-Owner->GetMaxHealth() * ActivationHealthPct);
        
        bIsActive = true;
        TickTimer = 0.f;
        UpdateVisuals(true);
    }

    void DeactivateStance()
    {
        bIsActive = false;
        CooldownRemaining = Cooldown;
        UpdateVisuals(false);
    }

    virtual void TickComponent(float DeltaTime, ...) override
    {
        if (bIsActive)
        {
            TickTimer += DeltaTime;
            
            // Maintenance cost every second
            if (TickTimer >= 1.0f)
            {
                TickTimer = 0.f;
                ApplyMaintenanceCost();
            }
        }
        else if (CooldownRemaining > 0)
        {
            CooldownRemaining -= DeltaTime;
        }
    }

    void ApplyMaintenanceCost()
    {
        ADuelCharacter* Owner = Cast<ADuelCharacter>(GetOwner());
        Owner->AddReserveMana(-Owner->GetMaxReserveMana() * MaintenanceReservePctPerSec);
        Owner->AddHealth(-Owner->GetMaxHealth() * MaintenanceHealthPctPerSec);
        
        // Auto-deactivate if resources too low
        if (Owner->GetReserveMana() <= 0 || Owner->GetHealth() <= 0)
        {
            DeactivateStance();
        }
    }

    void OnAbilityCast(UGameplayAbility* Ability)
    {
        if (!bIsActive) return;
        if (!Ability->IsOffensive()) return;
        
        ADuelCharacter* Owner = Cast<ADuelCharacter>(GetOwner());
        Owner->AddReserveMana(-Owner->GetMaxReserveMana() * BuffCastReservePct);
        Owner->AddHealth(-Owner->GetMaxHealth() * BuffCastHealthPct);
    }
};
```

---

### 5.7 Directional Damage & Armor System
**Positional combat depth** - damage varies by hit angle (inspired by Chivalry/Mordhau + Pokémon type effectiveness).

| Property | Value |
|----------|-------|
| **Armor Zones** | 4 zones: Front, Back, Left Side, Right Side |
| **Front** | 100% damage taken (baseline) |
| **Back** | **150% damage taken** (punish turning away/retreating blindly) |
| **Sides (Left/Right)** | **120% damage taken** (flanking rewarded) |
| **Abilities** | Can target specific zones (e.g., Backstab = only from behind) |
| **Visual** | Armor cracks / impact decals show hit direction |
| **UI** | Directional damage indicator on HUD (brief arrow) |

**Mechanics**:
- **Zone Detection**: Based on angle between attacker→victim vector and victim's forward vector
  - Front: ±60° from forward
  - Sides: 60°-120° left/right
  - Back: ±60° from backward
- **Projectiles/Abilities**: Use impact angle at moment of hit
- **Melee/Auto-Attack**: Use attacker position at hit time
- **Backstab Abilities**: Only trigger if attacker in Back zone (e.g., Assassin-type abilities)
- **Directional Shields**: Earthbender shield blocks Front 100%, Sides 50%, Back 0%

**Why**: Adds positional depth, makes movement/positioning meaningful, enables "get behind me" callouts, rewards flanking.

```cpp
// UDirectionalDamageComponent.h
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class UDirectionalDamageComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    enum class EHitZone : uint8 { Front, Back, LeftSide, RightSide };

    UPROPERTY(EditDefaultsOnly) float FrontMultiplier = 1.0f;
    UPROPERTY(EditDefaultsOnly) float BackMultiplier = 1.5f;
    UPROPERTY(EditDefaultsOnly) float SideMultiplier = 1.2f;

    EHitZone GetHitZone(const FVector& AttackerLocation, const FVector& VictimLocation, const FRotator& VictimRotation) const
    {
        FVector ToAttacker = (AttackerLocation - VictimLocation).GetSafeNormal();
        FVector VictimForward = VictimRotation.Vector();
        float Dot = FVector::DotProduct(ToAttacker, VictimForward);
        float Angle = FMath::Acos(FMath::Clamp(Dot, -1.0f, 1.0f)) * 180.0f / PI;
        
        // Determine left/right
        FVector Right = VictimRotation.RotateVector(FVector::RightVector);
        float SideDot = FVector::DotProduct(ToAttacker, Right);
        
        if (Angle <= 60.0f) return EHitZone::Front;
        if (Angle >= 120.0f) return EHitZone::Back;
        return (SideDot > 0) ? EHitZone::RightSide : EHitZone::LeftSide;
    }

    float GetDamageMultiplier(EHitZone Zone) const
    {
        switch (Zone)
        {
            case EHitZone::Front: return FrontMultiplier;
            case EHitZone::Back: return BackMultiplier;
            case EHitZone::LeftSide:
            case EHitZone::RightSide: return SideMultiplier;
        }
        return 1.0f;
    }
};
```

---

### 5.6 Chat System (CS-Style)

| Channel | Access | Key | Features |
|---------|--------|-----|----------|
| **Team Voice** | Team only | Push-to-talk (V default) / Voice activity | Proximity 3D audio, volume slider |
| **Team Text** | Team only | Enter (team) | Green text, shows in HUD + console |
| **All Text** | Everyone | Shift+Enter | White text, shows to both teams |
| **Spectator** | Spectators only | Enter | Gray text, not visible to players |

**Features**:
- **Profanity Filter**: Optional, client-side
- **Quick Messages**: F1-F4 (GG, Nice, Sorry, Lag)
- **Mute**: Per-player (voice + text separately)
- **History**: Last 50 messages in console
- **Tournament Mode**: All chat disabled except team voice
**Stacking efficiency bonus** for maintaining offensive pressure.

| Property | Value |
|----------|-------|
| **Trigger** | Landing any damaging ability or auto-attack |
| **Stack Gain** | +1 stack per hit (max 10 stacks) |
| **Stack Bonus** | -7% mana cost per stack (additive: 7%, 14%, 21%... 70%) |
| **Duration** | 5 seconds since LAST hit |
| **Decay** | All stacks lost if 5s pass without landing a hit |
| **Visual** | Character glows intensifies per stack, UI counter |
| **Audio** | Rising pitch "combo" sound each stack |

**Stack Benefits**:
| Stacks | Mana Cost Reduction | Visual Intensity |
|--------|---------------------|------------------|
| 1 | 7% | Faint glow |
| 3 | 21% | Visible aura |
| 5 | 35% | Strong glow + particles |
| 7 | 49% | Intense aura + screen pulse |
| 10 | **70%** | **Maximum - blinding aura, unique sound** |

**What Counts as "Hit"**:
- ✅ Auto-attack landing
- ✅ Ability dealing damage (direct or DoT tick)
- ✅ Elemental reaction damage
- ❌ Shield/heal abilities
- ❌ Missed/dodged/blocked attacks
- ❌ Damage over time ticks (only initial application)

**Interaction with Enhanced/Weakened**:
- Enhanced (Shift): Base cost × 1.5, then -70% = net -55% from normal
- Weakened (Ctrl): Base cost × 0.5, then -70% = net -85% from normal
- **Minimum cost floor**: 10% of base (cannot go free)

```cpp
// UCombatRushComponent.h
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class UCombatRushComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly) int32 MaxStacks = 10;
    UPROPERTY(EditDefaultsOnly) float StackDuration = 5.0f;
    UPROPERTY(EditDefaultsOnly) float ManaReductionPerStack = 0.07f; // 7%
    UPROPERTY(EditDefaultsOnly) float MinCostFloor = 0.10f; // 10% minimum

    int32 CurrentStacks = 0;
    float LastHitTime = 0.f;

    virtual void TickComponent(float DeltaTime, ...) override
    {
        if (CurrentStacks > 0)
        {
            float TimeSinceHit = GetWorld()->GetTimeSeconds() - LastHitTime;
            if (TimeSinceHit >= StackDuration)
            {
                ResetStacks();
            }
        }
    }

    void OnHitLanded(float Damage, bool bWasAbility)
    {
        if (Damage <= 0) return;
        
        CurrentStacks = FMath::Min(CurrentStacks + 1, MaxStacks);
        LastHitTime = GetWorld()->GetTimeSeconds();
        
        UpdateManaCostModifiers();
        UpdateVisuals();
    }

    float GetManaCostMultiplier() const
    {
        float Reduction = CurrentStacks * ManaReductionPerStack;
        return FMath::Max(1.0f - Reduction, MinCostFloor);
    }

    void ResetStacks()
    {
        CurrentStacks = 0;
        UpdateManaCostModifiers();
        UpdateVisuals();
    }
};
```

---

### 5.4 Deployable Cover (Earthbender Wall) - 3 Modes
**Expanded from Halo Equipment concept** - Earthbender creates destructible/indestructible walls.

#### Base Properties (All Modes)
| Property | Value |
|----------|-------|
| **Height** | 4 meters (400 units) |
| **Thickness** | 1 meter (100 units) |
| **Duration** | See mode specifics |
| **Health** | See mode specifics |
| **Placement** | Ground only, snaps to grid |

#### Mode 1: Normal (Default - No Modifier)
| Property | Value |
|----------|-------|
| **Input** | E (tap) |
| **Placement** | 3 meters in front of player, facing player |
| **Width** | 5 player models (~5 meters / 500 units) |
| **Health** | 2000 HP (destructible) |
| **Duration** | 15 seconds or until destroyed |
| **Cost** | 30 Primary Mana |
| **Cooldown** | 12 seconds |
| **Properties** | Blocks projectiles, blocks movement, climbable (Air), swimmable (Water) |

#### Mode 2: Enhanced (Hold Shift + E)
| Property | Value |
|----------|-------|
| **Input** | Shift + E (hold) |
| **Placement** | **Anywhere in LOS up to 15m**, mouse-targeted |
| **Width** | 5 player models (~5 meters / 500 units) |
| **Health** | 2000 HP (destructible) |
| **Duration** | 15 seconds or until destroyed |
| **Cost** | 45 Primary Mana + 15 Reserve Mana |
| **Cooldown** | 18 seconds |
| **Strategic Uses** | Trap enemy in corridor, block retreat path, create cover for ally |
| **Counterplay** | Enemy sees preview, can dodge placement, destroy wall (time/mana cost) |

#### Mode 3: Weakened (Hold Ctrl + E)
| Property | Value |
|----------|-------|
| **Input** | Ctrl + E (hold) |
| **Placement** | 2 meters in front, facing player |
| **Width** | 4 player models (~4 meters / 400 units) |
| **Health** | **INDESTRUCTIBLE** (cannot be damaged) |
| **Duration** | **Channel 4 seconds** (must hold key) |
| **Cost** | 15 Primary Mana (no cost to maintain) |
| **Cooldown** | 8 seconds |
| **Mechanic** | **Must hold key for 4s** - wall exists only while channeling |
| **Early Cancel** | Release key → wall instantly vanishes |
| **Interrupt** | Stun/silence/knockback → wall vanishes |
| **No Mana Drain** | Channeling costs no additional mana |

```cpp
// UDeployableCoverComponent.h
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class UDeployableCoverComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    enum class ECoverMode : uint8 { Normal, Enhanced, Weakened };

    struct FCoverConfig
    {
        float Width;
        float Health;
        float Duration;
        float PrimaryCost;
        float ReserveCost;
        float Cooldown;
        bool bDestructible;
        bool bRequiresChannel;
        float MaxRange; // 0 = self-only
    };

    static const TMap<ECoverMode, FCoverMode> CoverConfigs;

    UPROPERTY(Replicated) ECoverMode CurrentMode = ECoverMode::Normal;
    UPROPERTY(Replicated) AActor* ActiveWall = nullptr;
    float ChannelTime = 0.f;
    bool bIsChanneling = false;

    void TryDeployCover(ECoverMode Mode, FVector TargetLocation = FVector::ZeroVector)
    {
        const FCoverConfig& Config = CoverConfigs[Mode];
        
        // Check cooldown, mana
        if (!CanAfford(Config)) return;
        if (bOnCooldown[Mode]) return;

        // Deduct cost
        DeductCost(Config);

        if (Mode == ECoverMode::Weakened)
        {
            StartChannel(Config, TargetLocation);
        }
        else
        {
            SpawnWall(Config, TargetLocation);
            StartCooldown(Mode, Config.Cooldown);
        }
    }

    void StartChannel(const FCoverConfig& Config, FVector Location)
    {
        bIsChanneling = true;
        ChannelTime = 0.f;
        CurrentMode = ECoverMode::Weakened;
        
        // Spawn wall immediately
        SpawnWall(Config, Location);
    }

    virtual void TickComponent(float DeltaTime, ...) override
    {
        if (bIsChanneling && CurrentMode == ECoverMode::Weakened)
        {
            ChannelTime += DeltaTime;
            
            // Check cancel conditions
            if (!IsInputHeld(CtrlKey) || ChannelTime >= 4.0f || Owner->IsStunned())
            {
                EndChannel();
            }
        }
    }

    void EndChannel()
    {
        bIsChanneling = false;
        if (ActiveWall) ActiveWall->Destroy();
        ActiveWall = nullptr;
        StartCooldown(ECoverMode::Weakened, CoverConfigs[ECoverMode::Weakened].Cooldown);
    }

    void SpawnWall(const FCoverConfig& Config, FVector Location)
    {
        FVector SpawnLoc = (Config.MaxRange > 0) ? Location : GetOwner()->GetActorLocation() + GetOwner()->GetActorForwardVector() * 300.f;
        FRotator Rot = (Config.MaxRange > 0) ? (GetOwner()->GetActorLocation() - SpawnLoc).Rotation() : GetOwner()->GetActorRotation();
        
        ActiveWall = GetWorld()->SpawnActor<ACoverWall>(WallClass, SpawnLoc, Rot);
        ActiveWall->Initialize(Config.Width, Config.Health, Config.Duration, Config.bDestructible);
    }
};
```

---

### 5.5 Channeled Aim (Osu! Sliders Adaptation)
**Precision aiming for channeled abilities** - hold key, follow path with mouse for maximum effect.

| Property | Value |
|----------|-------|
| **Trigger** | Hold ability key (configurable per ability) |
| **Visual** | Path appears from player to cursor (Bezier curve) |
| **Mechanic** | Mouse must stay near path center (±50 units tolerance) |
| **Scoring** | Accuracy % = time on path / total channel time |
| **Reward** | Damage/Effect scales: 50% (min) → 100% (perfect) → 150% (perfect + speed) |
| **Cancel** | Release key early = fires at current accuracy |
| **Interrupt** | Stun/silence = fires at current accuracy |
| **Cooldown** | Starts on RELEASE, not on press |

**Path Types**:
- **Straight Line**: Basic projectiles (Fireball, Rock Throw)
- **Curve**: Arcing abilities (Meteor, Water Wave)
- **Loop**: Area abilities (Tornado, Blizzard center)
- **Zigzag**: Unpredictable paths (Lightning Chain)

**Visual Feedback**:
- Path glows character's element color
- "Cursor" indicator shows ideal position
- Accuracy ring around crosshair (fills as you track)
- Perfect = gold ring + sound cue

```cpp
// UChanneledAimComponent.h
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class UChanneledAimComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    struct FChannelPath
    {
        TArray<FVector> Points;  // World space points
        float Tolerance = 50.f;  // Units from center line
        float Length = 0.f;
    };

    UPROPERTY(EditDefaultsOnly) float MinAccuracy = 0.5f;  // 50%
    UPROPERTY(EditDefaultsOnly) float MaxAccuracy = 1.5f;  // 150%
    UPROPERTY(EditDefaultsOnly) float PathTolerance = 50.f;

    FChannelPath CurrentPath;
    float ChannelStartTime = 0.f;
    float AccurateTime = 0.f;
    bool bIsChanneling = false;

    void StartChannel(UGameplayAbility* Ability, FVector TargetLocation)
    {
        CurrentPath = GeneratePathForAbility(Ability, TargetLocation);
        ChannelStartTime = GetWorld()->GetTimeSeconds();
        AccurateTime = 0.f;
        bIsChanneling = true;
        
        // Show path visual to player
        ShowPathVisual(CurrentPath);
    }

    virtual void TickComponent(float DeltaTime, ...) override
    {
        if (!bIsChanneling) return;

        // Check mouse position vs path
        FVector MouseWorldPos = GetMouseWorldPosition();
        float DistToPath = DistanceToPath(MouseWorldPos, CurrentPath);
        
        if (DistToPath <= PathTolerance)
        {
            AccurateTime += DeltaTime;
        }

        // Update visual feedback
        UpdateAccuracyRing(GetCurrentAccuracy());
    }

    float GetCurrentAccuracy() const
    {
        float TotalTime = GetWorld()->GetTimeSeconds() - ChannelStartTime;
        if (TotalTime <= 0) return 1.0f;
        return AccurateTime / TotalTime;
    }

    void EndChannel(bool bInterrupted)
    {
        float Accuracy = GetCurrentAccuracy();
        float FinalMultiplier = FMath::Lerp(MinAccuracy, MaxAccuracy, Accuracy);
        
        // Fire ability with multiplier
        FireChanneledAbility(FinalMultiplier);
        
        bIsChanneling = false;
        HidePathVisual();
    }
};
```

---

### 5.6 Match Pause System (F9)

**Mutual pause with strategic cooldown warfare** - both players have access to pause, but cancelling has consequences.

#### Rules
| Parameter | Value |
|-----------|-------|
| **Input** | F9 (configurable) |
| **Pause Duration** | Unlimited (until cancelled) |
| **Minimum Pause Time** | 10 seconds (cannot be cancelled before this) |
| **Initiator Cooldown** | 180 seconds (3 minutes) after pause ENDS |
| **Canceller Cooldown** | 180 seconds (3 minutes) for the player who CANCELS |
| **Visual** | "PAUSED" banner, desaturated screen, timer showing pause duration |
| **Audio** | Low hum, muted game sounds, distinct "pause" jingle |

#### Mechanics
1. **Player A presses F9** → Match pauses immediately
2. **10-second lockout** → Neither player can cancel for 10 seconds
3. **After 10s**: Either player can press F9 to **cancel** the pause
4. **Cancellation consequences**:
   - **If Player A (initiator) cancels**: Player A gets 180s cooldown on pause
   - **If Player B (opponent) cancels**: Player B gets 180s cooldown on pause
5. **Cooldown applies to PAUSE ability**, not global
6. **During pause**: No inputs processed, game logic frozen, network state maintained
7. **Spectators**: See "PAUSED" overlay, can still use free camera

#### Strategic Implications
- **Tactical pause**: Buy time to think, drink water, adjust posture
- **Mind games**: Pause at critical moment → opponent must wait 10s
- **Cooldown warfare**: Force opponent to burn their pause cooldown
- **Comeback tool**: Losing player can pause to break momentum
- **Anti-grief**: 10s minimum prevents spam, 180s cooldown prevents abuse

#### Tournament/Competitive Rules
| Setting | Value |
|---------|-------|
| **Ranked** | Enabled (both players have pause) |
| **Tournament** | Enabled, but **max 2 pauses per player per match** |
| **Custom** | Configurable: disable, limit count, adjust cooldowns |
| **Spectator** | Can see pause cooldowns in overlay |

#### Technical Implementation

```cpp
// UMatchPauseComponent.h
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class UMatchPauseComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly) float MinPauseDuration = 10.0f;   // 10s lockout
    UPROPERTY(EditDefaultsOnly) float PauseCooldown = 180.0f;     // 3 minutes
    UPROPERTY(EditDefaultsOnly) int32 MaxPausesPerMatch = 2;      // Tournament limit

    // State (per player)
    struct FPlayerPauseState
    {
        bool bPauseAvailable = true;
        float PauseCooldownRemaining = 0.f;
        int32 PausesUsedThisMatch = 0;
        bool bIsPauseInitiator = false;
        float PauseStartTime = 0.f;
    };

    UPROPERTY(Replicated)
    FPlayerPauseState PlayerStates[2];  // Index 0 = Player 1, 1 = Player 2

    UPROPERTY(Replicated)
    bool bIsPaused = false;
    UPROPERTY(Replicated)
    int32 PauseInitiatorIndex = -1;  // 0 or 1, -1 = none
    float PauseLockoutTimer = 0.f;

    virtual void TickComponent(float DeltaTime, ...) override
    {
        // Update cooldowns
        for (int32 i = 0; i < 2; i++)
        {
            if (PlayerStates[i].PauseCooldownRemaining > 0)
            {
                PlayerStates[i].PauseCooldownRemaining -= DeltaTime;
                if (PlayerStates[i].PauseCooldownRemaining <= 0)
                {
                    PlayerStates[i].PauseAvailable = true;
                    PlayerStates[i].PauseCooldownRemaining = 0.f;
                }
            }
        }

        // Update lockout timer
        if (bIsPaused)
        {
            PauseLockoutTimer += DeltaTime;
        }
    }

    // Called by PlayerController on F9 press
    UFUNCTION(Server, Reliable)
    void ServerRequestPause(int32 PlayerIndex)
    {
        if (bIsPaused)
        {
            // Request to CANCEL pause
            TryCancelPause(PlayerIndex);
        }
        else
        {
            // Request to START pause
            TryStartPause(PlayerIndex);
        }
    }

    void TryStartPause(int32 PlayerIndex)
    {
        FPlayerPauseState& State = PlayerStates[PlayerIndex];
        
        // Check availability
        if (!State.bPauseAvailable) return;
        if (State.PausesUsedThisMatch >= MaxPausesPerMatch) return;
        if (bIsPaused) return;

        // Start pause
        bIsPaused = true;
        PauseInitiatorIndex = PlayerIndex;
        PauseLockoutTimer = 0.f;
        State.bIsPauseInitiator = true;
        State.PauseStartTime = GetWorld()->GetTimeSeconds();
        State.PausesUsedThisMatch++;
        
        // Apply pause to game
        GetWorld()->GetGameInstance()->SetGamePaused(true);
        
        // Visual/audio
        BroadcastPauseStarted(PlayerIndex);
    }

    void TryCancelPause(int32 PlayerIndex)
    {
        if (!bIsPaused) return;
        if (PauseLockoutTimer < MinPauseDuration) return;  // 10s lockout

        FPlayerPauseState& InitiatorState = PlayerStates[PauseInitiatorIndex];
        FPlayerPauseState& CancellerState = PlayerStates[PlayerIndex];

        // Determine who gets cooldown
        if (PlayerIndex == PauseInitiatorIndex)
        {
            // Initiator cancels their own pause
            InitiatorState.PauseCooldownRemaining = PauseCooldown;
            InitiatorState.bPauseAvailable = false;
        }
        else
        {
            // Opponent cancels
            CancellerState.PauseCooldownRemaining = PauseCooldown;
            CancellerState.bPauseAvailable = false;
        }

        // End pause
        EndPause();
    }

    void EndPause()
    {
        bIsPaused = false;
        PauseInitiatorIndex = -1;
        PauseLockoutTimer = 0.f;
        
        // Clear initiator flag
        for (int32 i = 0; i < 2; i++)
        {
            PlayerStates[i].bIsPauseInitiator = false;
        }

        // Resume game
        GetWorld()->GetGameInstance()->SetGamePaused(false);
        
        BroadcastPauseEnded();
    }
};
```

---

## 6. IMPLEMENTATION PRIORITY MATRIX

| Priority | System | Effort | Impact | Dependencies |
|----------|--------|--------|--------|--------------|
| **P0** | Arena Layout (1v1/3v3) | Medium | Critical | Foundation |
| **P0** | Sonar Vision (Passive/Active) | High | Critical | FoW, Post-process, Audio |
| **P0** | Server-Authoritative FoW | High | Critical | NetDriver, Replication |
| **P0** | Rune System (with destruction) | High | High | Core loop |
| **P1** | Basic Auto-Attack | Low | High | Animation, GAS |
| **P1** | Shadow Dash (Double-Tap) | Medium | High | Movement, Input |
| **P1** | Combat Rush (Combo Passive) | Medium | High | GAS, UI |
| **P1** | Deployable Cover (3 Modes) | High | High | Physics, GAS, Input |
| **P1** | Channeled Aim (Osu! Sliders) | High | High | Input, UI, Abilities |
| **P1** | Match Pause (F9) | Medium | High | GameState, Networking |
| **P1** | Victory Conditions (Elim + Control) | Medium | High | GameState |
| **P1** | Spectator Mode (No FoW, Overlay) | High | Medium | Esports ready |
| **P2** | Momentum Casting | Medium | High | Animation, Ability System |
| **P2** | Elemental Residue | Medium | High | Niagara, GAS |
| **P2** | Adaptive Cooldowns | Low | High | GAS |
| **P2** | Directional Armor | Medium | Medium | Hit Detection, UI |
| **P2** | Ability Mod Slots | High | High | Data System, UI |
| **P2** | Environmental Hazards | Medium | Medium | Map Design, Niagara |
| **P2** | Hype Meter | Low | Medium | Spectator, Announcer |
| **P2** | Input Ghost System | Medium | High | Replay, Input Recording |
| **P3** | Draft Phase | Medium | High | Matchmaking, UI |
| **P3** | Dynamic Map Events | High | Medium | Map Logic, Sequencer |
| **P3** | Rune Crafting | Medium | Medium | Inventory, Economy |
| **P3** | Sideboard System | Low | Medium | Loadout UI |
| **P3** | Mentor Queue | Medium | Medium | Matchmaking, Social |
| **P4** | Seasonal Rulesets | Low | High | Config, Build System |
| **P4** | Crowdsourced Anti-Cheat | High | Medium | Replay, Web Backend |
| **P4** | Mastery Expression | Low | High | Cosmetics, Progression |
| **P5** | Accessibility Suite | Medium | Critical | Platform Cert |
| **P5** | Training Mode | High | High | Bot AI, UI |
| **P5** | Match Analytics | Medium | Medium | Telemetry, UI |
| **P5** | Social Features | Medium | Medium | Backend, Voice |
| **P5** | LAN Support | Low | Critical | Networking |
| **P6** | Director AI | High | Low | Spectator, Camera |
| **P6** | Predictive Display | High | Low | ML, Telemetry |
| **P6** | Spectator Betting | Medium | Low | Economy, Web |
| **P6** | Multi-Lang Commentary | Low | Medium | Audio, Localization |
| **P6** | Tournament Admin | High | Low | Backend, UI |
| **P6** | Match Archive | Medium | Low | Database, Web |
| **P6** | Custom Scripting | Very High | High | Engine, Security |

---

*Document Version: 2.6*  
*Added: Directional Damage & Armor, Shadow Dash collision fix, Trap (no variants), Madness Stance, Chat System*  
*Updated: Shadow Dash (pass through players), Trap (normal only), Player Collision*  
*Archived: Anti-Cheat Crowdsourcing, QoL/Accessibility (7), Spectator/Esports (7)*  
*Status: Living Design Doc*

---

## 11. RUNE SYSTEM - "CYCLIC GUARANTEED DISTRIBUTION WITH ESCALATION"

### 11.1 Core Design

| Parameter | Value |
|-----------|-------|
| **Spawn Points** | 4 fixed locations (symmetrical corners) |
| **Spawn Interval** | Every 60 seconds (1:00, 2:00, 3:00...) |
| **Active Duration** | 20 seconds (despawns if uncollected) |
| **Base Pool** | 5 rune types |
| **Cycle 1 (Min 1-5)** | Guaranteed all 5 types, 1 per minute |
| **Cycle 2 (Min 6-9)** | 2 runes/min, **1 type excluded each minute** (tracked) |
| **Minute 10** | **4 runes at all 4 points** = the **single remaining type** from Cycle 2 exclusions |
| **Minute 11+** | 4 runes at all 4 points, **no exclusions** (5 types compete for 4 slots) |

---

### 11.2 The 5 Rune Types

| Rune | Base Effect | Stacking Behavior | Refresh on Re-pickup |
|------|-------------|-------------------|---------------------|
| **Double Damage** | ×2 outgoing damage | **Multiplicative**: ×2ⁿ (×2, ×4, ×8, ×16) | ✅ Refreshes 20s timer |
| **Double Defense** | +50% max HP as damage reduction | **Custom multiplicative**: 50% → 125% → 237.5% → 406.25% | ✅ Refreshes 20s timer |
| **Haste** | ×1.5 move speed, ×1.3 cast speed | **Multiplicative**: 1.5ⁿ / 1.3ⁿ | ✅ Refreshes 20s timer |
| **Accelerated Regeneration** | +10% max HP/Primary Mana/Reserve per second for 10s | **Timer refresh only** (no stack increase) | ✅ Refreshes 10s timer |
| **Instant Restoration** | Immediate +50% max HP, Primary Mana, Reserve | **No stacking** (instant, one-time) | N/A (consumed instantly) |

#### Stacking Tables

**Haste (Multiplicative)**
| Stacks | Move Speed | Cast Speed |
|--------|------------|------------|
| 1 | 1.50× | 1.30× |
| 2 | 2.25× | 1.69× |
| 3 | 3.375× | 2.197× |
| 4 | **5.0625×** | **2.8561×** |

**Double Defense (Custom Multiplicative)**
| Stacks | Damage Reduction | Effective HP Multiplier |
|--------|------------------|------------------------|
| 1 | 50% | 1.5× |
| 2 | 125% | 2.25× |
| 3 | 237.5% | 3.375× |
| 4 | **406.25%** | **5.0625×** |

---

### 11.3 Distribution Logic - Corrected Algorithm

```cpp
// ARuneSpawnManager.h - Core Logic

UCLASS()
class ARuneSpawnManager : public AActor
{
    GENERATED_BODY()

public:
    enum class ERuneType : uint8
    {
        DoubleDamage,
        DoubleDefense,
        Haste,
        AcceleratedRegen,
        InstantRestore,
        None
    };

    // Spawn Point State (for destruction mechanic)
    struct FSpawnPointState
    {
        FVector Location;
        bool bIsDestroyed = false;           // Destroyed this cycle?
        bool bDestroyedByTeamA = false;      // Track which team destroyed
        bool bDestroyedByTeamB = false;
        float DestroyedCycleMinute = -1.f;   // Which minute it was destroyed
    };

    struct FRuneSpawnState
    {
        // Cycle 1 (Min 1-5): Pool of remaining runes
        TArray<ERuneType> Cycle1Pool = {
            ERuneType::DoubleDamage,
            ERuneType::DoubleDefense,
            ERuneType::Haste,
            ERuneType::AcceleratedRegen,
            ERuneType::InstantRestore
        };
        
        // Cycle 2 (Min 6-9): Track excluded types each minute
        // At minute 10, the ONE type NOT excluded in min 6-9 spawns at all 4 points
        TArray<ERuneType> Cycle2ExcludedTypes;  // Max 4 entries (one per minute 6-9)
        
        int32 CurrentMinute = 0;
        int32 SpawnsThisMinute = 0;
        float LastSpawnTime = 0.f;
    };

    UPROPERTY(Replicated)
    FRuneSpawnState SpawnState;

    // 4 fixed spawn locations (set in editor per map) - WITH DESTRUCTIBLE STATES
    UPROPERTY(EditDefaultsOnly)
    TArray<FSpawnPointState> SpawnPoints;  // Must be exactly 4

    // Rune Actors currently spawned
    UPROPERTY()
    TArray<ARuneActor*> ActiveRunes;

    virtual void BeginPlay() override;
    void Tick(float DeltaTime) override;
    
    void TrySpawnRune();
    ERuneType SelectRuneForCurrentMinute();
    void SpawnRuneAtLocation(int32 SpawnPointIndex, ERuneType Type);
    void OnRuneCollected(ERuneType Type, ADuelCharacter* Collector);
    void OnRuneExpired(ERuneType Type);
    
    // DESTRUCTION MECHANICS
    UFUNCTION(Server, Reliable) void ServerDestroyRune(ARuneActor* RuneActor, ADuelCharacter* Destroyer);
    UFUNCTION(Server, Reliable) void ServerDestroySpawnPoint(int32 SpawnPointIndex, ADuelCharacter* Destroyer);
    bool CanPlayerDestroySpawnPoint(ADuelCharacter* Player, int32 SpawnPointIndex) const;
    int32 GetAvailableSpawnPointsCount() const;

protected:
    enum class EMatchPhase : uint8 
    { 
        Cycle1_Guaranteed,    // Min 1-5: 1 rune, guaranteed distribution
        Cycle2_Exclusions,    // Min 6-9: 2 runes, 1 excluded per minute
        Minute10_Resolution,  // Min 10: 4 runes, the 1 non-excluded type
        OpenCompetition       // Min 11+: 4 runes, 5 types compete for 4 slots
    };
    
    EMatchPhase CurrentPhase = EMatchPhase::Cycle1_Guaranteed;
    
    void UpdateMatchPhase();
    ERuneType SelectFromPool(TArray<ERuneType>& Pool);
    ERuneType GetMinute10RuneType() const;
    void MarkTypeExcludedThisMinute(ERuneType Type);
    TArray<int32> GetAvailableSpawnPointIndices() const;
};
```

#### Destruction Mechanics

```cpp
// Rune Destruction: Any attack can destroy a rune (500 HP, no armor)
// Spawn Point Destruction: Channel 2s (hold F on spawn point), 1 per player per cycle

bool ARuneSpawnManager::CanPlayerDestroySpawnPoint(ADuelCharacter* Player, int32 SpawnPointIndex) const
{
    if (SpawnPointIndex < 0 || SpawnPointIndex >= SpawnPoints.Num()) return false;
    
    const FSpawnPointState& Point = SpawnPoints[SpawnPointIndex];
    if (Point.bIsDestroyed) return false;  // Already destroyed this cycle
    
    // Check if THIS player/team already destroyed a spawn point this cycle
    int32 TeamID = Player->GetTeamID(); // 0 or 1
    if (TeamID == 0)
    {
        // Check if any spawn point was destroyed by Team A this cycle
        for (const auto& P : SpawnPoints)
        {
            if (P.bDestroyedByTeamA && P.DestroyedCycleMinute == SpawnState.CurrentMinute)
                return false;
        }
    }
    else
    {
        for (const auto& P : SpawnPoints)
        {
            if (P.bDestroyedByTeamB && P.DestroyedCycleMinute == SpawnState.CurrentMinute)
                return false;
        }
    }
    
    return true;
}

void ARuneSpawnManager::ServerDestroySpawnPoint_Implementation(int32 SpawnPointIndex, ADuelCharacter* Destroyer)
{
    if (!CanPlayerDestroySpawnPoint(Destroyer, SpawnPointIndex)) return;
    
    FSpawnPointState& Point = SpawnPoints[SpawnPointIndex];
    Point.bIsDestroyed = true;
    Point.DestroyedCycleMinute = SpawnState.CurrentMinute;
    
    if (Destroyer->GetTeamID() == 0)
        Point.bDestroyedByTeamA = true;
    else
        Point.bDestroyedByTeamB = true;
    
    // Visual feedback: Spawn point crumbles, particle effect
    // Audio: Distinct "shatter" sound, global announcement
    AnnounceSpawnPointDestroyed(SpawnPointIndex, Destroyer->GetTeamID());
}

TArray<int32> ARuneSpawnManager::GetAvailableSpawnPointIndices() const
{
    TArray<int32> Available;
    for (int32 i = 0; i < SpawnPoints.Num(); i++)
    {
        if (!SpawnPoints[i].bIsDestroyed)
            Available.Add(i);
    }
    return Available;
}
```

#### Modified Spawn Logic (Handles Destroyed Points)

```cpp
void ARuneSpawnManager::TrySpawnRune()
{
    UpdateMatchPhase();
    
    int32 RunesThisMinute = 0;
    switch (CurrentPhase)
    {
        case EMatchPhase::Cycle1_Guaranteed: RunesThisMinute = 1; break;
        case EMatchPhase::Cycle2_Exclusions: RunesThisMinute = 2; break;
        case EMatchPhase::Minute10_Resolution: RunesThisMinute = 4; break;
        case EMatchPhase::OpenCompetition: RunesThisMinute = 4; break;
    }
    
    SpawnState.SpawnsThisMinute = 0;
    
    // Get AVAILABLE spawn points (not destroyed this cycle)
    TArray<int32> AvailableIndices = GetAvailableSpawnPointIndices();
    int32 AvailableCount = AvailableIndices.Num();
    
    // If not enough spawn points, spawn what we can (exclusion still happens, cycle advances)
    int32 ActualSpawns = FMath::Min(RunesThisMinute, AvailableCount);
    
    if (ActualSpawns == 0)
    {
        // No spawn points available! Still process exclusion for cycle integrity
        if (CurrentPhase == EMatchPhase::Cycle2_Exclusions)
        {
            // Pick a type to exclude even though nothing spawns
            TArray<ERuneType> Pool = {ERuneType::DoubleDamage, ERuneType::DoubleDefense, 
                ERuneType::Haste, ERuneType::AcceleratedRegen, ERuneType::InstantRestore};
            ERuneType Excluded = SelectFromPool(Pool);
            MarkTypeExcludedThisMinute(Excluded);
        }
        return;
    }
    
    // Shuffle available indices for randomness
    AvailableIndices.Shuffle();
    
    for (int32 i = 0; i < ActualSpawns; i++)
    {
        int32 SpawnPointIndex = AvailableIndices[i];
        ERuneType Type = SelectRuneForCurrentMinute();
        SpawnRuneAtLocation(SpawnPointIndex, Type);
    }
    
    // Reset destroyed points at minute boundary (next cycle)
    // Handled in UpdateMatchPhase when minute changes
    
    SpawnState.LastSpawnTime = GetWorld()->GetTimeSeconds();
}

void ARuneSpawnManager::UpdateMatchPhase()
{
    int32 Minute = FMath::FloorToInt(GetWorld()->GetTimeSeconds() / 60.f) + 1;
    
    // Reset spawn point destruction at new minute
    if (Minute != SpawnState.CurrentMinute)
    {
        for (auto& Point : SpawnPoints)
        {
            Point.bIsDestroyed = false;
            Point.bDestroyedByTeamA = false;
            Point.bDestroyedByTeamB = false;
        }
    }
    
    SpawnState.CurrentMinute = Minute;
    // ... phase logic same as before
}
```

#### Rune Actor - Instant Pickup & Destructible

```cpp
// ARuneActor.h
UCLASS()
class ARuneActor : public AActor
{
    GENERATED_BODY()

public:
    UPROPERTY(VisibleAnywhere) UStaticMeshComponent* RuneMesh;
    UPROPERTY(VisibleAnywhere) UNiagaraComponent* AmbientParticles;
    UPROPERTY(VisibleAnywhere) UBoxComponent* PickupVolume;  // Instant pickup on overlap
    UPROPERTY(VisibleAnywhere) UHealthComponent* HealthComponent; // 500 HP, destructible

    UPROPERTY(Replicated) ERuneType RuneType;
    UPROPERTY(Replicated) int32 SpawnPointIndex;  // Which spawn point created this

    virtual void BeginPlay() override
    {
        // Instant pickup on contact
        PickupVolume->OnComponentBeginOverlap.AddDynamic(this, &ARuneActor::OnOverlapBegin);
        
        // Health component for destruction
        HealthComponent->MaxHealth = 500.f;
        HealthComponent->Health = 500.f;
        HealthComponent->OnDeath.AddDynamic(this, &ARuneActor::OnDestroyed);
    }

    UFUNCTION()
    void OnOverlapBegin(UPrimitiveComponent* Overlapped, AActor* Other, ...)
    {
        if (ADuelCharacter* Char = Cast<ADuelCharacter>(Other))
        {
            if (Char->HasAuthority())
            {
                // INSTANT PICKUP - no channeling
                Char->GetRuneEffectComponent()->ServerCollectRune(RuneType);
                
                // Notify spawn manager
                if (ARuneSpawnManager* Manager = GetWorld()->GetGameState<ADuelGameState>()->GetRuneSpawnManager())
                {
                    Manager->OnRuneCollected(RuneType, Char);
                }
                
                Destroy();
            }
        }
    }

    void OnDestroyed()
    {
        // Visual: Shatter effect, particles
        // Audio: Break sound
        // Notify spawn manager rune was destroyed (not collected)
        if (ARuneSpawnManager* Manager = GetWorld()->GetGameState<ADuelGameState>()->GetRuneSpawnManager())
        {
            Manager->OnRuneDestroyed(RuneType, SpawnPointIndex);
        }
        Destroy();
    }
};
```

#### Selection Logic

```cpp
void ARuneSpawnManager::UpdateMatchPhase()
{
    int32 Minute = FMath::FloorToInt(GetWorld()->GetTimeSeconds() / 60.f) + 1; // 1-indexed
    SpawnState.CurrentMinute = Minute;
    
    if (Minute >= 11)
        CurrentPhase = EMatchPhase::OpenCompetition;
    else if (Minute == 10)
        CurrentPhase = EMatchPhase::Minute10_Resolution;
    else if (Minute >= 6)
        CurrentPhase = EMatchPhase::Cycle2_Exclusions;
    else
        CurrentPhase = EMatchPhase::Cycle1_Guaranteed;
}

ERuneType ARuneSpawnManager::SelectRuneForCurrentMinute()
{
    switch (CurrentPhase)
    {
        case EMatchPhase::Cycle1_Guaranteed:
        {
            // Minutes 1-5: Pick from remaining pool, remove it
            TArray<ERuneType>& Pool = SpawnState.Cycle1Pool;
            if (Pool.Num() == 0) 
            {
                // Reset for next cycle (shouldn't happen in min 1-5)
                Pool = {ERuneType::DoubleDamage, ERuneType::DoubleDefense, 
                        ERuneType::Haste, ERuneType::AcceleratedRegen, ERuneType::InstantRestore};
            }
            ERuneType Selected = SelectFromPool(Pool);
            Pool.RemoveSingle(Selected); // Guaranteed distribution
            return Selected;
        }
        
        case EMatchPhase::Cycle2_Exclusions:
        {
            // Minutes 6-9: 2 runes spawn per minute
            // 1 type is EXCLUDED from the 5 for this minute (tracked for minute 10)
            // Pick 2 from the remaining 4
            
            TArray<ERuneType> AvailablePool = {
                ERuneType::DoubleDamage, ERuneType::DoubleDefense, ERuneType::Haste,
                ERuneType::AcceleratedRegen, ERuneType::InstantRestore
            };
            
            // Pick 1 to exclude for THIS minute (if not already excluded this minute)
            ERuneType ExcludedType = SelectFromPool(AvailablePool);
            MarkTypeExcludedThisMinute(ExcludedType);
            AvailablePool.RemoveSingle(ExcludedType);
            
            // Now pick 2 from remaining 4 for the 2 spawn points
            // (Caller will call this twice, or we return array - simplified here)
            // For 2 spawns: shuffle and take first 2
            AvailablePool.Shuffle();
            
            // Return first for first spawn, second for second spawn
            // (Implementation detail: SpawnState.SpawnsThisMinute tracks which of 2)
            int32 SpawnIndex = SpawnState.SpawnsThisMinute % 2;
            ERuneType Selected = AvailablePool[SpawnIndex];
            SpawnState.SpawnsThisMinute++;
            
            return Selected;
        }
        
        case EMatchPhase::Minute10_Resolution:
        {
            // Minute 10: The ONE type NOT excluded in minutes 6-9 spawns at ALL 4 points
            // All 4 spawn points get the same rune type
            return GetMinute10RuneType();
        }
        
        case EMatchPhase::OpenCompetition:
        {
            // Minute 11+: 4 runes at 4 points, 5 types compete for 4 slots (no exclusions)
            TArray<ERuneType> AvailablePool = {
                ERuneType::DoubleDamage, ERuneType::DoubleDefense, ERuneType::Haste,
                ERuneType::AcceleratedRegen, ERuneType::InstantRestore
            };
            AvailablePool.Shuffle();
            AvailablePool.SetNum(4); // 4 slots for 4 spawn points
            
            int32 SpawnIndex = SpawnState.SpawnsThisMinute % 4;
            ERuneType Selected = AvailablePool[SpawnIndex];
            SpawnState.SpawnsThisMinute++;
            
            return Selected;
        }
    }
    return ERuneType::None;
}

void ARuneSpawnManager::MarkTypeExcludedThisMinute(ERuneType Type)
{
    // Only add if not already excluded this minute (prevent duplicates within same minute)
    if (!SpawnState.Cycle2ExcludedTypes.Contains(Type))
    {
        SpawnState.Cycle2ExcludedTypes.Add(Type);
    }
}

ERuneType ARuneSpawnManager::GetMinute10RuneType() const
{
    // The type that was NEVER excluded in minutes 6-9
    TArray<ERuneType> AllTypes = {
        ERuneType::DoubleDamage, ERuneType::DoubleDefense, ERuneType::Haste,
        ERuneType::AcceleratedRegen, ERuneType::InstantRestore
    };
    
    for (ERuneType Type : AllTypes)
    {
        if (!SpawnState.Cycle2ExcludedTypes.Contains(Type))
        {
            return Type; // This is the one!
        }
    }
    
    // Fallback (shouldn't happen if logic correct)
    return ERuneType::DoubleDamage;
}

ERuneType ARuneSpawnManager::SelectFromPool(TArray<ERuneType>& Pool)
{
    if (Pool.Num() == 0) return ERuneType::None;
    int32 Index = FMath::RandRange(0, Pool.Num() - 1);
    return Pool[Index];
}

void ARuneSpawnManager::TrySpawnRune()
{
    UpdateMatchPhase();
    
    int32 RunesThisMinute = 0;
    switch (CurrentPhase)
    {
        case EMatchPhase::Cycle1_Guaranteed: RunesThisMinute = 1; break;
        case EMatchPhase::Cycle2_Exclusions: RunesThisMinute = 2; break;
        case EMatchPhase::Minute10_Resolution: RunesThisMinute = 4; break;
        case EMatchPhase::OpenCompetition: RunesThisMinute = 4; break;
    }
    
    SpawnState.SpawnsThisMinute = 0; // Reset counter for this minute
    
    TArray<int32> UsedIndices;
    for (int32 i = 0; i < RunesThisMinute; i++)
    {
        // Pick random unused spawn point
        int32 SpawnIndex;
        do { SpawnIndex = FMath::RandRange(0, SpawnLocations.Num() - 1); }
        while (UsedIndices.Contains(SpawnIndex));
        UsedIndices.Add(SpawnIndex);
        
        ERuneType Type = SelectRuneForCurrentMinute();
        SpawnRuneAtLocation(SpawnLocations[SpawnIndex], Type);
    }
    
    SpawnState.LastSpawnTime = GetWorld()->GetTimeSeconds();
}
```

---

### 11.4 Rune Effects Implementation (Corrected Stacking)

```cpp
// URuneEffectComponent.h - Applied to character on pickup

UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class URuneEffectComponent : public UActorComponent
{
    GENERATED_BODY()

public:
    struct FActiveRuneEffect
    {
        ERuneType Type;
        float StartTime;
        float Duration;  // 20s base (10s for Regen)
        int32 StackCount = 1;  // For stacking runes
    };
    
    UPROPERTY(Replicated)
    TArray<FActiveRuneEffect> ActiveEffects;

    // Collection (server only)
    UFUNCTION(Server, Reliable)
    void ServerCollectRune(ERuneType Type)
    {
        ApplyOrRefreshRuneEffect(Type);
    }

    void ApplyOrRefreshRuneEffect(ERuneType Type)
    {
        // Find existing effect of same type
        FActiveRuneEffect* ExistingEffect = nullptr;
        for (auto& Effect : ActiveEffects)
        {
            if (Effect.Type == Type)
            {
                ExistingEffect = &Effect;
                break;
            }
        }
        
        float CurrentTime = GetWorld()->GetTimeSeconds();
        
        if (ExistingEffect)
        {
            // REFRESH: Update start time (extends duration), increment stack if applicable
            ExistingEffect->StartTime = CurrentTime;
            
            switch (Type)
            {
                case ERuneType::DoubleDamage:
                case ERuneType::DoubleDefense:
                case ERuneType::Haste:
                    ExistingEffect->StackCount = FMath::Min(ExistingEffect->StackCount + 1, 4);
                    break;
                case ERuneType::AcceleratedRegen:
                    // No stack increase, just timer refresh
                    break;
                case ERuneType::InstantRestore:
                    // Handled separately, doesn't persist
                    break;
            }
            
            UpdateAllMultipliers();
            return;
        }
        
        // NEW effect
        FActiveRuneEffect NewEffect;
        NewEffect.Type = Type;
        NewEffect.StartTime = CurrentTime;
        NewEffect.StackCount = 1;
        
        switch (Type)
        {
            case ERuneType::DoubleDamage:
            case ERuneType::DoubleDefense:
            case ERuneType::Haste:
                NewEffect.Duration = 20.f;
                break;
            case ERuneType::AcceleratedRegen:
                NewEffect.Duration = 10.f;
                break;
            case ERuneType::InstantRestore:
                ApplyInstantRestore();
                return; // Don't add to ActiveEffects
        }
        
        ActiveEffects.Add(NewEffect);
        UpdateAllMultipliers();
    }

    void UpdateAllMultipliers()
    {
        // Calculate combined multipliers from all active effects
        float DamageMultiplier = 1.0f;
        float DamageReduction = 0.0f;  // As percentage of max HP
        float MoveSpeedMultiplier = 1.0f;
        float CastSpeedMultiplier = 1.0f;
        
        for (auto& Effect : ActiveEffects)
        {
            switch (Effect.Type)
            {
                case ERuneType::DoubleDamage:
                    // ×2 per stack: 2^stacks
                    DamageMultiplier *= FMath::Pow(2.0f, Effect.StackCount);
                    break;
                    
                case ERuneType::DoubleDefense:
                    // Custom: 50% → 125% → 237.5% → 406.25%
                    switch (Effect.StackCount)
                    {
                        case 1: DamageReduction += 0.50f; break;
                        case 2: DamageReduction += 1.25f; break;
                        case 3: DamageReduction += 2.375f; break;
                        case 4: DamageReduction += 4.0625f; break;
                    }
                    break;
                    
                case ERuneType::Haste:
                    // Multiplicative: 1.5^stacks, 1.3^stacks
                    MoveSpeedMultiplier *= FMath::Pow(1.5f, Effect.StackCount);
                    CastSpeedMultiplier *= FMath::Pow(1.3f, Effect.StackCount);
                    break;
            }
        }
        
        // Apply to character
        ADuelCharacter* Char = Cast<ADuelCharacter>(GetOwner());
        if (Char)
        {
            Char->SetDamageMultiplier(DamageMultiplier);
            Char->SetDamageReductionPct(DamageReduction); // Applied as % of max HP
            Char->SetMoveSpeedMultiplier(MoveSpeedMultiplier);
            Char->SetCastSpeedMultiplier(CastSpeedMultiplier);
        }
    }

    // Called every tick
    void TickComponent(float DeltaTime, ELevelTick, FActorComponentTickFunction*) override
    {
        float CurrentTime = GetWorld()->GetTimeSeconds();
        ADuelCharacter* Char = Cast<ADuelCharacter>(GetOwner());
        
        // Process regeneration
        for (auto& Effect : ActiveEffects)
        {
            if (Effect.Type != ERuneType::AcceleratedRegen) continue;
            if (CurrentTime - Effect.StartTime > Effect.Duration) continue;
            
            // Check interruption: full resources OR took damage this tick
            bool bFullResources = (Char->GetHealth() >= Char->GetMaxHealth() &&
                                   Char->GetPrimaryMana() >= Char->GetMaxPrimaryMana() &&
                                   Char->GetReserveMana() >= Char->GetMaxReserveMana());
            
            if (bFullResources || Char->bTookDamageThisTick)
            {
                RemoveEffect(Effect);
                UpdateAllMultipliers();
                continue;
            }
            
            // Apply 10% per second of max values
            float RestoreRate = DeltaTime * 0.1f;
            Char->AddHealth(Char->GetMaxHealth() * RestoreRate);
            Char->AddPrimaryMana(Char->GetMaxPrimaryMana() * RestoreRate);
            Char->AddReserveMana(Char->GetMaxReserveMana() * RestoreRate);
        }
        
        // Clean expired effects
        ActiveEffects.RemoveAll([&](const FActiveRuneEffect& E) {
            return CurrentTime - E.StartTime > E.Duration;
        });
        
        // Recalculate if any expired
        UpdateAllMultipliers();
    }

    void ApplyInstantRestore()
    {
        ADuelCharacter* Char = Cast<ADuelCharacter>(GetOwner());
        if (!Char) return;
        
        Char->AddHealth(Char->GetMaxHealth() * 0.5f);
        Char->AddPrimaryMana(Char->GetMaxPrimaryMana() * 0.5f);
        Char->AddReserveMana(Char->GetMaxReserveMana() * 0.5f);
    }

    void RemoveEffect(FActiveRuneEffect& Effect)
    {
        ActiveEffects.Remove(Effect);
    }
};
```

---

### 11.5 Timeline & Escalation Summary (Corrected)

| Minute | Phase | Spawns | Pool Logic | Result |
|--------|-------|--------|------------|--------|
| **1** | Cycle 1 | 1 | Random from 5 | 1 type revealed |
| **2** | Cycle 1 | 1 | Random from 4 | 2 types revealed |
| **3** | Cycle 1 | 1 | Random from 3 | 3 types revealed |
| **4** | Cycle 1 | 1 | Random from 2 | 4 types revealed |
| **5** | Cycle 1 | 1 | **Last remaining** | Cycle 1 complete, all 5 seen |
| **6** | Cycle 2 | 2 | **1 excluded** (tracked), pick 2 from 4 | First exclusion recorded |
| **7** | Cycle 2 | 2 | **1 excluded**, pick 2 from 4 | Second exclusion |
| **8** | Cycle 2 | 2 | **1 excluded**, pick 2 from 4 | Third exclusion |
| **9** | Cycle 2 | 2 | **1 excluded**, pick 2 from 4 | Fourth exclusion |
| **10** | Resolution | **4 (all points)** | **The 1 type NEVER excluded in 6-9** | **All 4 = same type** |
| **11+** | Open | 4 (all points) | 5 types compete for 4 slots (no exclusions) | Random 4 of 5 each minute |

#### Minute 10 Scenarios (The "Resolution")
Since 4 types were excluded in minutes 6-9 (one per minute), **exactly 1 type remains unexcluded**. That type spawns at all 4 points.

| If Excluded in 6-9 Were: | Minute 10 Spawns (×4) |
|--------------------------|----------------------|
| DoubleDamage, DoubleDefense, Haste, AccelRegen | **Instant Restore** ×4 |
| DoubleDamage, DoubleDefense, Haste, InstantRestore | **Accelerated Regen** ×4 |
| DoubleDamage, DoubleDefense, AccelRegen, InstantRestore | **Haste** ×4 (→ 5.06× move, 2.86× cast) |
| DoubleDamage, Haste, AccelRegen, InstantRestore | **Double Defense** ×4 (→ 406% DR) |
| DoubleDefense, Haste, AccelRegen, InstantRestore | **Double Damage** ×4 (→ **×16 damage**) |

#### Minute 11+ (Open Competition)
- 5 types, 4 spawn points
- Each minute: 4 random distinct types (1 type doesn't spawn)
- All stacking rules apply
- Possible to get 4x Double Damage again (if RNG allows)

---

### 11.6 Strategic Implications (Corrected)

#### Information Warfare Evolution
- **Min 1-5**: Perfect information by minute 5 (all 5 seen once)
- **Min 6-9**: **Deduction game** - each exclusion narrows minute 10
  - See Double Damage at min 6 → it's excluded → **won't be minute 10**
  - Track all 4 exclusions → **know minute 10 with certainty by min 9**
- **Min 10**: **Known quantity** - both players know exactly what's spawning
  - Becomes pure **positioning/contest** fight
- **Min 11+**: **RNG returns** - 5 choose 4, less predictable

#### Stacking Consequences
| Rune | 4-Stack Result (Minute 10) | Counterplay |
|------|---------------------------|-------------|
| **Double Damage** | **×16 damage** | Burst down carrier, Double Defense, kite |
| **Double Defense** | **406% DR** (5× EHP) | True damage, %HP damage, wait it out |
| **Haste** | **5.06× move, 2.86× cast** | Root/stun, predict movement, area denial |
| **Accel Regen** | 10%/s for 10s (refreshed ×4 = 40s) | Damage interrupt, burst through |
| **Instant Restore** | 4 × 50% = **200% total resources** | Pressure before they collect all |

#### Refresh Mechanic Impact
- **Contest extends duration**: Fighting over a rune keeps it alive longer
- **Denial by damage**: Hitting carrier prevents Accel Regen, forces Double Defense refresh
- **Stack management**: Do you grab 2nd Double Damage now (×4) or wait for 3rd/4th?

---

### 11.7 Visual & Audio Design (Updated for Stacking)

| Stack Count | Double Damage Visual | Double Defense Visual | Haste Visual |
|-------------|---------------------|----------------------|--------------|
| 1 | Red glow | Blue armor shimmer | Green trails |
| 2 | Red lightning | Crystalline armor | Wind vortex |
| 3 | Fire aura | Golden plates | Sonic boom lines |
| 4 | **Nuclear red pillar** | **Diamond skin** | **Reality warp** |

**Screen Flash on Pickup**: Intensity × stack count
**Audio Pitch**: Rises with each stack (audible warning to enemy)

---

### 11.8 Balance Config (Hot-Reloadable, Corrected)

```ini
; RuneSystem.ini

[General]
SpawnInterval=60.0
RuneDuration=20.0
RegenDuration=10.0
SpawnPoints=4
PoolSize=5

[Phases]
Cycle1_EndMinute=5
Cycle2_StartMinute=6
Cycle2_EndMinute=9
ResolutionMinute=10
OpenCompetitionStartMinute=11

[Cycle2]
RunesPerMinute=2
ExclusionsPerMinute=1  ; Tracked for minute 10 resolution

[Resolution]
RunesAtAllPoints=4
AllSameType=true

[OpenCompetition]
RunesPerMinute=4
TypesCompeting=5
SlotsAvailable=4

[Stacking]
MaxStacks=4
RefreshOnPickup=true

[DoubleDamage]
BaseMultiplier=2.0
StackFormula=Pow  ; 2^stacks

[DoubleDefense]
; Custom values per stack (as % of max HP)
Stack1_Reduction=0.50    ; 50%
Stack2_Reduction=1.25    ; 125%
Stack3_Reduction=2.375   ; 237.5%
Stack4_Reduction=4.0625  ; 406.25%

[Haste]
MoveSpeedBase=1.5
CastSpeedBase=1.3
StackFormula=Pow  ; 1.5^stacks, 1.3^stacks

[AcceleratedRegen]
PercentPerSecond=0.10  ; 10%
InterruptOnDamage=true
InterruptOnFullResources=true
TimerRefreshOnly=true  ; No stack increase

[InstantRestore]
PercentOfMax=0.50  ; 50%
NoStacking=true
InstantConsume=true
```

### 11.9 Destruction Mechanics Config

```ini
; RuneSystem.ini (additions)

[Destruction]
RuneHealth=1.0
RuneArmor=0.0
SpawnPointChannelTime=2.0
MaxSpawnPointDestroysPerPlayerPerCycle=1
AnnounceDestructionGlobally=true

[Visuals]
SpawnPointDestroyedEffect=CrumbleParticles
SpawnPointDestroyedSound=ShatterGlobal
RuneDestroyedEffect=ShatterParticles
RuneDestroyedSound=BreakSound
```

---

## 12. VICTORY CONDITIONS

### 12.1 Universal: Elimination
**Primary win condition for ALL modes**: Eliminate all enemy players.
- 1v1: Kill opponent → Round win
- 3v3: Kill all 3 enemies → Round win
- Best of 3 rounds (configurable)

### 12.2 3v3 Alternative: Center Control
**Secondary win condition ONLY in 3v3 mode**.

| Parameter | Value |
|-----------|-------|
| **Zone** | Center circle, 9m radius (marked on ground) |
| **Target Score** | 400 points |
| **Scoring Rate** | 1 point per second per player in zone |
| **Max Rate** | 3 points/sec (all 3 teammates in zone) |
| **Contested** | If ANY enemy in zone → **NO points for either team** |
| **Progress Persistence** | Score persists between deaths, resets per round |
| **Visual** | Team-colored ring fills toward center, UI progress bar |

#### Scoring Examples
```
Team A: 2 players in zone, Team B: 0 → Team A gains 2 pts/sec
Team A: 3 players in zone, Team B: 0 → Team A gains 3 pts/sec  
Team A: 2 players, Team B: 1 player → CONTESTED → 0 pts/sec for both
Team A: 1 player, Team B: 1 player → CONTESTED → 0 pts/sec
```

#### Strategic Implications
- **Force fights**: Team behind on eliminations can win via control
- **Defensive play**: Contesting zone denies enemy win condition
- **Split decisions**: Send 1 to contest, 2 to hunt / or 3 to dominate zone
- **Comeback mechanic**: Losing team can stall by contesting

---

## 13. BOSS MECHANICS - FUTURE IDEA (NOT IMPLEMENTED)

> **NOTE**: The following was designed for a "Boss Pit" / "Elemental Colossus" at minutes 2:30, 5:00, 7:30 in 3v3. **This is REMOVED from current scope** but preserved as a future expansion idea.

### 13.1 Concept (Archived)
- **Spawn**: Neutral boss in center at fixed intervals
- **Phases**: 3 phases (Fire → Earth → Air/Void)
- **Reward**: Avatar Rune effect (45s) + resources + center vision
- **Enrage**: 3 minutes or both teams die

### 13.2 Why Removed
- Adds PvE complexity to pure PvP game
- Balancing boss vs player power is nightmare
- Distracts from core duel/arena fantasy
- Can be added later as "Event Mode" or "Seasonal Ruleset"

---

## 14. SPECTATOR / OBSERVER MODE - ENHANCED

### 14.1 Core Requirements
| Feature | Implementation |
|---------|----------------|
| **Fog of War** | **DISABLED** for spectators - absolute vision of entire arena |
| **Camera Modes** | Player Follow (1-6) / Free Flight / Strategic Overview |
| **Player Overlay** | All abilities, HP, Primary Mana, Reserve Mana for all players |
| **Team View** | Toggle Team A / Team B / Both vision |

### 14.2 UI Overlay Layout (Spectator HUD)

```
┌─────────────────────────────────────────────────────────────────────┐
│  TEAM A (Blue)                                    TEAM B (Red)      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ P1 [Earth]  HP:████████░░  PM:████████░░  RM:████░░░░░░░  │   │
│  │     [Q]● [W]● [E]○ [R]● [F]○  CD: 2.3s  5.1s  0s  8.2s 12s │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │ P2 [Fire]   HP:██████░░░░  PM:██████████  RM:██████░░░░  │   │
│  │     [Q]○ [W]● [E]● [R]○ [F]●  CD: 0s   1.2s  3.4s 0s  0s  │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │ P3 [Water]  HP:██████████  PM:████░░░░░░  RM:██████████  │   │
│  │     [Q]● [W]○ [E]● [R]● [F]○  CD: 4.1s  0s   2.8s 6.0s 0s  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  [CENTER CONTROL]  Team A: 247 / 400  |  Team B: 189 / 400        │
│  ████████████████████░░░░░░░░░░  vs  ██████████████░░░░░░░░░░░░  │
│                                                                     │
│  [TIME]  3:42  |  [ROUND]  2/3  |  [MODE]  3v3 Ranked             │
└─────────────────────────────────────────────────────────────────────┘
```

### 14.3 Camera Controls

| Input | Action |
|-------|--------|
| **1-6** | Snap to Player 1-6 (1v1: 1-2) |
| **Tab** | Cycle through players |
| **Space** | Toggle Free Flight mode |
| **Shift + Space** | Strategic Overview (top-down, 5000 units up) |
| **Mouse Wheel** | Zoom (in follow mode) |
| **Middle Mouse** | Pan (in free flight) |
| **F1-F4** | Toggle overlays: Abilities / Resources / Cooldowns / Minimap |

### 14.4 Free Flight Mode
- **No Fog of War**: Sees everything
- **Movement**: WASD + Q/E (up/down), Shift = sprint
- **Speed**: 5000 units/sec base, 15000 with Shift
- **Collision**: None (noclip through geometry)
- **Focus**: Double-click player → snap to follow mode

### 14.5 Strategic Overview Mode
- **Camera**: Fixed top-down, orthographic, 5000 units height
- **Shows**: Full arena, all players, all runes, center zone progress
- **Icons**: Player models as colored circles with element symbols
- **Runes**: Pulsing icons at spawn points with type indicator
- **Click**: Click any player → enter follow mode for that player

### 14.6 Technical Implementation

```cpp
// ASpectatorPawn.h
UCLASS()
class ASpectatorPawn : public APawn
{
    GENERATED_BODY()

public:
    enum class ECameraMode : uint8
    {
        FollowPlayer,      // Locked behind player (Battlerite style)
        FreeFlight,        // Noclip, full control
        StrategicOverview  // Top-down orthographic
    };

    UPROPERTY(Replicated) ECameraMode CurrentMode = ECameraMode::FollowPlayer;
    UPROPERTY(Replicated) int32 FollowedPlayerIndex = 0;  // 0-5
    
    // No Fog of War for spectator
    virtual void Tick(float DeltaTime) override
    {
        // Override FoW: Spectator sees ALL actors
        if (APlayerController* PC = Cast<APlayerController>(GetController()))
        {
            PC->bShowAllActors = true;  // Custom flag to bypass FoW culling
        }
        
        switch (CurrentMode)
        {
            case ECameraMode::FollowPlayer:
                UpdateFollowCamera(DeltaTime);
                break;
            case ECameraMode::FreeFlight:
                UpdateFreeFlight(DeltaTime);
                break;
            case ECameraMode::StrategicOverview:
                // Fixed position, no update needed
                break;
        }
    }

    void SetFollowTarget(int32 PlayerIndex)
    {
        FollowedPlayerIndex = PlayerIndex;
        CurrentMode = ECameraMode::FollowPlayer;
    }

    void ToggleFreeFlight()
    {
        CurrentMode = (CurrentMode == ECameraMode::FreeFlight) 
            ? ECameraMode::FollowPlayer 
            : ECameraMode::FreeFlight;
    }

    void EnterStrategicOverview()
    {
        CurrentMode = ECameraMode::StrategicOverview;
        SetActorLocation(FVector(0, 0, 5000));
        SetActorRotation(FRotator(-90, 0, 0));
    }
};
```

```cpp
// Spectator HUD Widget - Receives data from GameState
class USpectatorOverlayWidget : public UUserWidget
{
    // Binds to ADuelGameState::OnPlayerStateChanged
    // Receives: HP, PrimaryMana, ReserveMana, AbilityCooldowns[5], Element, RuneEffects
    // Updates 2 or 6 player panels automatically
};
```

### 14.7 Data Provided to Spectator (GameState Replication)

```cpp
// ADuelGameState.h - Additional for Spectator
UCLASS()
class ADuelGameState : public AGameStateBase
{
    GENERATED_BODY()

public:
    // Full state for ALL players (spectator only, not replicated to players)
    UPROPERTY(ReplicatedUsing=OnRep_SpectatorData)
    TArray<FSpectatorPlayerData> SpectatorPlayerData;  // Size = 2 (1v1) or 6 (3v3)

    struct FSpectatorPlayerData
    {
        FString PlayerName;
        int32 TeamID;  // 0 or 1
        EElement Element;
        float Health, MaxHealth;
        float PrimaryMana, MaxPrimaryMana;
        float ReserveMana, MaxReserveMana;
        TArray<FAbilityCooldown> AbilityCooldowns;  // 5 abilities
        TArray<FRuneEffect> ActiveRunes;
        int32 Kills, Deaths, Assists;
    };

    // Center Control (3v3)
    UPROPERTY(Replicated) float TeamAControlScore;
    UPROPERTY(Replicated) float TeamBControlScore;
    UPROPERTY(Replicated) bool bZoneContested;
    UPROPERTY(Replicated) int32 PlayersInZone_TeamA;
    UPROPERTY(Replicated) int32 PlayersInZone_TeamB;

    // Rune Info
    UPROPERTY(Replicated) TArray<FSpectatorRuneData> ActiveRunes;
    
    struct FSpectatorRuneData
    {
        ERuneType Type;
        FVector Location;
        float TimeRemaining;
        int32 SpawnPointIndex;
    };
};
```

---

## 15. UPDATED IMPLEMENTATION PRIORITY

| Priority | System | Effort | Impact | Notes |
|----------|--------|--------|--------|-------|
| **P0** | Arena Layout (1v1/3v3 scaling) | Medium | Critical | Foundation |
| **P0** | Sonar Vision (Passive/Active) | High | Critical | Core mechanic |
| **P0** | Server-Authoritative FoW | High | Critical | Competitive integrity |
| **P0** | Rune System (with destruction) | High | High | Core loop |
| **P1** | Victory Conditions (Elim + Control) | Medium | High | 3v3 win states |
| **P1** | Spectator Mode (No FoW, Overlay) | High | Medium | Esports ready |
| **P2** | Center Control Scoring | Low | High | 3v3 alt win |
| **P2** | Rune Destruction Mechanics | Medium | Medium | Strategic depth |
| **P3** | Spawn Point Destruction | Medium | Low | Denial gameplay |
| **P4** | Boss Mechanics | Very High | Low | **FUTURE - Not now** |

---

## 16. ARCHIVE - FUTURE IMPLEMENTATION (Post-Launch)

*The following ideas are archived for potential future implementation if project succeeds. Removed from active scope per design review.*

### 16.1 Progression & Meta (Archived)

**63. Battle Pass** → "Seasonal Duel Pass": Win matches = tiers, rewards = cosmetics only

**66. Artifact Relics** → "Equip 3 Relics": Passive effects (on hit, on dodge, on ult), rarity tiers

**67. Socketed Gems** → "Rune Slots": Abilities have 2 slots, insert gems (Fire/DoT, Ice/Slow, etc.)

**68. Enchantment** → "Weapon Enchant": Each character has weapon, enchant for elemental damage

**70. Achievement Hunting** → "Feats of Strength": "Win 100 matches as Earth", "Perfect parry 50 times"

**71. Leaderboard Seasons** → "Monthly Ladder": Top 0.1% = "Avatar" title, unique cosmetic

**72. Replay Analysis** → "Ghost Data": Download top player ghost, practice against their inputs

**74. Tournament Mode** → "Bracket Runner": Built-in 8/16/32 player bracket, auto-schedule

**75. Clan Wars** → "Guild Duel": 5v5 best-of-9, guild ranking, shared bank

### 16.2 Social & Spectacle (Archived)

**76. Taunt System** → "Respect/Disrespect": Post-round, honor opponent = +rep, tea-bag = -rep

**77. Emote Wheel** → "In-Game Communication": Ping, "Good game", "Nice dodge", "Lag"

**79. Replay Theater** → "Director Mode": Edit replays, add camera shakes, slow-mo, export video

### 16.3 Additional Archived (This Session)

**83. Announcer Packs** → "Voice Packs": Buy/unlock different announcers (serious, meme, lore)

**84. MVP Screen** → "Play of the Game": Auto-detect best moment, replay for all

**85. Post-Match Stats** → "Heatmap": Damage dealt/taken zones, movement paths, ability usage

**86. Friend Spectate** → "Watch Friend": Join friend's match as spectator, voice chat

**87. Mentor System** → "Student/Teacher": High rank mentors low rank, bonus rewards

**88. Challenge System** → "Duel Request": Right-click profile → challenge, custom rules

**89. Tournament Spectate** → "Multi-POV": Watch all 4 quarterfinals simultaneously

**95. Untitled Goose Game** → "Honk Mechanic": Honk stuns enemies in cone, cooldown, pure chaos

**98. Gang Beasts** → "Grab & Throw": Melee grab, throw enemy into hazards/off map

### 16.4 Anti-Cheat & QoL & Spectator (This Session)

**15. Replay-Based "Anti-Cheat" Crowdsourcing** (From CS:GO Overwatch + chess)
**Concept**: Suspicious matches flagged → Community reviews replays.
- **Reviewers**: Top 1% players, verified identity
- **Tools**: Frame-perfect input display, mana tracking, aim analysis
- **Verdict**: "Clean" / "Suspicious" / "Cheating"
- **Consensus**: 5 reviewers, 4/5 = action
- **Rewards**: Reviewers get "Judicator" cosmetics, XP
**Why**: Scales anti-cheat, community ownership, deters cheaters.

**17. Comprehensive Input Remapping + Macros** (From fighting game standards)
- **Per-Ability**: Bind any key, modifier combos (Shift+Q, Ctrl+Alt+E)
- **Macro Slots**: 4 slots, record 5-input sequences (legal: combo practice only)
- **Profile System**: Save/load/share binding profiles (JSON)
- **Controller Support**: Full remap, stick curves, dead zones, vibration patterns
- **One-Handed Mode**: All actions on one side + modifiers
- **Eye Tracking**: Optional aim assist for accessibility

**18. Visual Clarity Suite** (From competitive FPS standards)
- **Colorblind Modes**: Protanope/Deuteranope/Tritanope + custom
- **Outline Thickness**: 1-5px, color per team/element
- **Projectile Trails**: Toggle, thickness, opacity
- **Screen Shake**: 0-100% intensity, directional only
- **Motion Blur**: Off / Low / High
- **UI Scale**: 80-150%, per-element (mana bars, cooldowns, text)
- **High Contrast Mode**: Flat colors, no gradients, thick outlines

**19. Audio Accessibility Suite** (From The Last of Us Part II)
- **Visual Audio Cues**: Subtitles for all sounds (footsteps, casts, pulses)
- **Directional Indicators**: On-screen arrows for off-screen sounds
- **Frequency Sliders**: Separate volume for footsteps, abilities, music, VO
- **Mono Audio Option**: Collapse stereo for single-ear hearing
- **Dynamic Range Compressor**: Normalize loud/quiet sounds
- **Screen Reader Support**: All menus narrated, keyboard nav

**20. Training Mode Suite** (From fighting game lab mode)
- **Frame Data Display**: Startup/Active/Recovery, advantage on hit/block
- **Hitbox Viewer**: Real-time hurtbox/hitbox visualization
- **Record/Playback**: Record dummy actions, loop playback
- **Scenario Editor**: "Enemy at 30% HP, Fire ultimate ready, you have 500 reserve"
- **Replay Takeover**: Load replay, take control at any frame
- **Bot AI Director**: Script behavior trees (aggressive/defensive/mixup)
- **Combo Counter**: Damage, hits, optimal route suggestion

**21. Match History & Analytics** (From chess.com + LoL client)
- **Filterable History**: By character, map, result, duration, date
- **Trend Graphs**: Winrate over time, avg damage, mana efficiency
- **Matchup Table**: vs each character (W/L, avg duration, common causes)
- **Heatmaps**: Your damage dealt/taken by map zone
- **Ability Efficiency**: % hit, damage per cast, mana per damage
- **Export**: CSV/JSON for external analysis

**22. Social & Communication** (From modern multiplayer standards)
- **Ping System**: 8-wheel (Enemy, Going, Danger, Assist, Rune, Ward, Retreat, Look)
- **Voice Chat**: Proximity / Team / Party / Push-to-talk / Voice activity
- **Text Chat**: Quick messages (GG, Nice, Sorry, Lag, Rematch)
- **Replay Share**: Generate link, expires 7 days, view in browser
- **Clan/Guild**: Shared bank, clan matches, clan ladder
- **Mentor Request**: "Looking for mentor" flag in profile

**23. Offline/LAN Support** (From fighting game tournament standard)
- **LAN Mode**: No internet required, direct IP connect
- **Tournament Build**: Deterministic, no auto-updates, fixed version
- **Offline Practice**: Full bot AI, all modes, no account needed
- **Replay Local**: Save/load without server
- **Custom Game Server**: Host dedicated on LAN

**24. Director AI for Auto-Spectate** (From CS:GO GOTV + sports broadcasting)
- **Rules**: Always show player with lowest HP, or ult ready, or in combat
- **Transitions**: Smooth blend, no hard cuts during action
- **Picture-in-Picture**: Mini-map + secondary POV
- **Replay Buffer**: 30s instant replay on key moments
- **Graph Overlay**: Real-time mana/health/cooldown graphs

**25. Predictive "Next Play" Display** (From poker broadcasts + NFL)
- **AI Prediction**: "Player likely to: Dash → Fireball → Parry" (based on patterns)
- **Probability**: Shows % for top 3 predicted actions
- **Accuracy Tracking**: Public stats on prediction accuracy
- **Player Opt-Out**: Can disable for own matches

**26. Interactive Spectator Betting** (From Twitch predictions + salary cap)
- **Currency**: "Arena Coins" (earned by watching, no real money)
- **Markets**: Round winner, first blood, most damage, perfect parries
- **Odds**: Dynamic based on bets
- **Rewards**: Cosmetics, titles, profile flair

**27. Multi-Language Commentary Support** (From Dota 2 / LoL)
- **Audio Tracks**: Multiple commentator languages
- **Auto-Translate**: Chat/voice → text in viewer language
- **Terminology Glossary**: Hover any term → definition

**28. Tournament Administration Suite** (From Smash.gg + Battlefy)
- **Bracket Generator**: Single/Double elim, Swiss, Round Robin
- **Auto-Scheduling**: Matches auto-assign, notify players
- **Pause/Resume**: Admin control, crash recovery
- **Rule Enforcement**: Auto-verify settings, character locks
- **Stream Integration**: OBS plugin for auto-scenes

**29. Historical Match Archive** (From chess databases + sports)
- **Search**: By player, character, date, tournament, outcome
- **Meta Analysis**: "Earth vs Fire winrate Patch 1.3: 52%"
- **Legendary Matches**: Curated "Greatest Duels" with commentary
- **Player Profiles**: Career stats, signature moments, rivalries

**30. Custom Game Scripting** (From WC3/SC2 custom maps + Roblox)
- **Visual Scripting**: Node-based (Blueprint-style) for custom modes
- **API**: Access to all gameplay systems (mana, FoW, runes, camera)
- **Share**: Workshop/Steam Workshop integration
- **Monetization**: Creators earn from popular modes (cosmetic revenue share)
- **Examples**: "Gun Game", "King of the Hill", "Zombie Survival", "Ability Draft"

---

*Document Version: 2.6*  
*Added: Directional Damage & Armor, Shadow Dash collision fix, Trap (no variants), Madness Stance, Chat System*  
*Updated: Shadow Dash (pass through players), Trap (normal only), Player Collision*  
*Archived: Anti-Cheat Crowdsourcing, QoL/Accessibility (7), Spectator/Esports (7)*  
*Status: Living Design Doc*