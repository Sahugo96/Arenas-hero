# Arena's Hero - 1v1 Duel Game Development Plan

## Project Overview
**Game Title**: Arena's Hero (working title)  
**Genre**: Competitive 1v1 Online Duel / Action Arena  
**Engine**: Unreal Engine 5.4+  
**Target Platform**: PC (Steam), potential console port later  
**Art Style**: Royal Quest aesthetic (stylized fantasy, vibrant colors, clear readability)  
**Core Inspiration**: Avatar: The Last Airbender/Korra (elements, seismic sense), Battlerite (camera), Dota 2 (fog of war), Royal Quest (combat/abilities)

---

## 1. HIGH-LEVEL ARCHITECTURE

### 1.1 Technical Stack
| Layer | Technology | Justification |
|-------|------------|---------------|
| **Engine** | Unreal Engine 5.4+ | Nanite/Lumen, Gameplay Ability System (GAS), Netcode |
| **Networking** | UE5 NetDriver + custom replication | Deterministic rollback for competitive integrity |
| **Backend** | Dedicated Game Server (C++/Rust) + Redis + PostgreSQL | Authoritative server, matchmaking, persistence |
| **Anti-Cheat** | Custom server-authoritative + kernel-level (optional) | Competitive 1v1 requires zero trust |
| **Build System** | Unreal Build Tool + GitHub Actions CI/CD | Automated testing, deployment |

### 1.2 Core Design Principles
- **Server-Authoritative Everything**: No client trust for gameplay-critical data
- **Deterministic Simulation**: Identical results given same inputs (rollback-ready)
- **Data-Driven Design**: All abilities, stats, progression in DataAssets/DataTables
- **Modular Architecture**: Plugin-based systems for easy iteration

---

## 2. DEVELOPMENT MODULES & PHASES

### PHASE 0: FOUNDATION (Weeks 1-4)
**Goal**: Project setup, core architecture, CI/CD, basic networking

#### Module 0.1: Project Infrastructure
- [ ] UE5.4 project setup with C++ plugin architecture
- [ ] Git repo + GitHub Actions CI (build, test, static analysis)
- [ ] Coding standards, clang-format, UnrealHeaderTool config
- [ ] Plugin structure: `Core`, `Gameplay`, `Network`, `UI`, `AntiCheat`, `Analytics`

#### Module 0.2: Networking Foundation
- [ ] Custom NetDriver with deterministic replication
- [ ] Replication Graph for 1v1 (simplified vs MMORPG)
- [ ] Input serialization + server reconciliation
- [ ] Lag compensation framework (rewind/resimulate)
- [ ] Connection quality metrics (RTT, jitter, packet loss)

#### Module 0.3: Core Gameplay Framework
- [ ] GameModeBase → `ADuelGameMode`
- [ ] GameStateBase → `ADuelGameState` (match timer, round state, scores)
- [ ] PlayerController → `ADuelPlayerController` (input handling, spectator)
- [ ] Pawn → `ADuelCharacter` (base class for all mages)
- [ ] GameInstance → `UDuelGameInstance` (session management, settings)

#### Module 0.4: Input & Settings System
- [ ] Enhanced Input system with configurable bindings
- [ ] Shift/Ctrl modifiers for enhanced/weakened abilities
- [ ] Settings save/load (JSON + binary for performance)
- [ ] Input buffering for combo execution

---

### PHASE 1: COMBAT CORE (Weeks 5-12)
**Goal**: Playable 1v1 duel with 2-3 elements, basic abilities

#### Module 1.1: Gameplay Ability System (GAS) Integration
- [ ] `UDuelAbilitySystemComponent` extending UAbilitySystemComponent
- [ ] AttributeSet: `Health`, `PrimaryMana`, `ReserveMana`, `Stamina`, `CooldownReduction`, `SpellPower`, `MoveSpeed`
- [ ] **Critical**: Custom Mana System Implementation:
  ```cpp
  // Primary Mana: regenerates in/out of combat (slower in combat)
  // Reserve Mana: regenerates ONLY out of combat, very slow
  // Rule: If ReserveMana < 50% MaxReserve → PrimaryManaRegen = 0
  // Enhanced abilities (Shift): Cost Primary + Reserve
  // Weakened abilities (Ctrl): Cost reduced Primary, no Reserve
  // Key Abilities: Cost ONLY Reserve (large amounts)
  ```
- [ ] GameplayEffects for mana costs, regen, modifiers
- [ ] GameplayCues for visual/audio feedback

#### Module 1.2: Element System (Avatar-inspired)
| Element | Subtypes | Core Mechanics |
|---------|----------|----------------|
| **Water** | Healing, Ice, Bloodbending (late) | Flow, adaptability, state changes |
| **Earth** | Metal, Lava, Sand, Seismic Sense | Defense, area control, vision |
| **Fire** | Lightning, Combustion, Blue Fire | Burst, DoT, execution |
| **Air** | Sound, Flight, Spiritual Projection | Mobility, displacement, utility |
| **Energy** (Avatar State) | Spiritbending | Ultimate, high reserve cost |

- [ ] `UElementComponent` managing element affinity, unlocked subtypes
- [ ] Element interaction matrix (water beats fire, earth beats air, etc.)
- [ ] Seismic Sense (Toph): Reveals enemies through obstacles in radius

#### Module 1.3: Ability Framework
- [ ] Base `UDuelGameplayAbility` with:
  - Mouse-world-targeting (direction/location)
  - Charging mechanics (hold to charge, release to cast)
  - Enhanced/Weakened variants via modifier keys
  - Cooldown, mana cost (primary/reserve), casting time
- [ ] Ability Types:
  - **Projectile**: Fireball, Rock Throw, Water Whip, Air Blast
  - **AoE Epicenter**: Meteor, Earthquake, Blizzard, Tornado
  - **Instant/Buff**: Shields, Heals, Mobility (blink, dash), Counters
  - **Channelled**: Beam, Sustained AoE, Summons
  - **Key Abilities** (Reserve-only): Teleport, Reflect, Global Meteor, Avatar State

#### Module 1.4: Targeting & Casting System
- [ ] **No Tab-Targeting**: All abilities use mouse world position/direction
- [ ] Targeting Component: `UDuelTargetingComponent`
  - Raycast from camera to world (ground/obstacles)
  - Preview indicators (decal/particle) for AoE epicenters
  - Range validation (server-authoritative)
  - Smart casting options (quick cast, normal cast, cast on release)

#### Module 1.5: Movement & Physics
- [ ] CharacterMovementComponent customization:
  - Elemental movement modes (Air glide, Water swim, Earth surf, Fire dash)
  - Momentum preservation for skill expression
  - Server-authoritative movement with client prediction
- [ ] Obstacle system for cover mechanics

---

### PHASE 2: CAMERA & FOG OF WAR (Weeks 13-18)
**Goal**: Battlerite-style camera + Dota 2 fog of war + Seismic Sense

#### Module 2.1: Camera System (Battlerite-style)
- [ ] `ADuelCameraActor` with:
  - Fixed offset behind character (configurable distance/angle)
  - Smooth follow with interpolation (not locked to pawn rotation)
  - Zoom based on combat state (wider during ultimates)
  - Screen shake, impact effects
  - Spectator mode with free camera + player follow

#### Module 2.2: Fog of War System (Dota 2-style)
- [ ] **FogOfWarManager** (server-authoritative):
  - Grid-based or navmesh-based vision calculation
  - Vision sources: Characters, wards/totems, abilities
  - Vision blockers: Walls, elevation, smoke, stealth
  - **Replication**: Only replicate actors within vision radius + recent memory
- [ ] **Client-side Fog Rendering**:
  - Post-process material with fog texture
  - Unexplored = black, Explored but not visible = dark/grey, Visible = clear
  - Smooth transitions (feathering)
- [ ] **Seismic Sense (Toph/Earth)**:
  - Reveals enemies through obstacles in radius
  - Shows enemy position as outline/silhouette
  - Audio cue + vibration feedback
  - Counter: Air/Water movement silent, Earth movement loud

#### Module 2.3: Vision & Stealth Mechanics
- [ ] Stealth system (Air/Water subtypes):
  - Invisibility + silence (no footstep audio)
  - Broken by: Attacking, taking damage, Seismic Sense, detection abilities
- [ ] Ward/Totem system (placeable vision sources)
- [ ] Elevation vision advantage (high ground sees further)

---

### PHASE 3: CHARACTERS & PROGRESSION (Weeks 19-28)
**Goal**: 8-12 playable mages, loadout system, progression

#### Module 3.1: Character Roster Design
| Archetype | Elements | Playstyle | Key Ability (Reserve) |
|-----------|----------|-----------|----------------------|
| **Waterbender** | Water/Ice/Healing | Control/Sustain | **Spirit Healing** (full heal + cleanse, 80% reserve) |
| **Earthbender** | Earth/Metal/Lava | Tank/Zone Control | **Seismic Slam** (global stun, 70% reserve) |
| **Firebender** | Fire/Lightning | Burst/Aggression | **Combustion Beam** (global execute, 90% reserve) |
| **Airbender** | Air/Sound/Spirit | Mobility/Evasion | **Spiritual Projection** (invuln + scout, 60% reserve) |
| **Avatar** | All 4 + Energy | Versatile/High Skill | **Avatar State** (all elements enhanced, 100% reserve) |
| **Metalbender** | Earth/Metal | Precision/Disarm | **Metal Cage** (unbreakable prison, 75% reserve) |
| **Bloodbender** | Water/Blood | Manipulation/Control | **Puppet Master** (control enemy, 85% reserve) |
| **Lavabender** | Earth/Fire/Lava | Area Denial | **Volcanic Eruption** (map-wide DoT, 80% reserve) |

#### Module 3.2: Loadout & Talent System
- [ ] **Base Kit**: 4 basic abilities (LMB, RMB, Q, E) + 1 Movement (Shift/Space)
- [ ] **Ultimate/Key Ability**: F (Reserve-only, game-changing)
- [ ] **Talent Tree** (per character): 3 tiers, 3 choices each → modifies abilities
- [ ] **Rune/Artifact System**: Pre-match loadout (3 slots), minor stat bonuses
- [ ] Data-driven: All in DataTables/DataAssets for live balance updates

#### Module 3.3: Animation & Visuals
- [ ] Animation Blueprint with GAS integration (Montages, AnimNotifies for GCEs)
- [ ] Procedural animation for elemental trails (Niagara ribbons)
- [ ] **Royal Quest Style**: Stylized, readable, colorful VFX
  - Clear telegraphing (windup → cast → impact)
  - Color-coded by element (Blue=Water, Green=Earth, Red=Fire, White=Air)
  - Screen readability > visual flair

---

### PHASE 4: MAPS & ENVIRONMENT (Weeks 29-36)
**Goal**: 4-6 competitive maps with varied geometry

#### Module 4.1: Map Design Principles
- **Symmetrical** or **Mirrored** for competitive fairness
- **Cover System**: Destructible/non-destructible obstacles
- **Verticality**: High ground, pits, jump pads
- **Elemental Zones**: Water pools (waterbender buff), Lava (fire/earth), Wind tunnels (air)
- **Vision Blockers**: Tall grass, smoke vents, pillars

#### Module 4.2: Map Technical Implementation
- [ ] Landscape + Foliage system optimized for 1v1
- [ ] Destructible environment (GAS + Chaos Physics)
- [ ] NavMesh for AI (training bots) + pathfinding for projectiles
- [ ] Map-specific lighting (Lumen) + post-process volumes
- [ ] Streaming for larger maps (not needed for 1v1 arena size)

#### Module 4.3: Map Pool & Rotation
- [ ] Map veto system (BO3/BO5)
- [ ] Map-specific balance tweaks (DataAsset per map)

---

### PHASE 5: NETWORKING & ANTI-CHEAT (Weeks 37-46) **CRITICAL**
**Goal**: Production-ready netcode, zero-cheat tolerance

#### Module 5.1: Advanced Netcode
- [ ] **Deterministic Rollback** (GGPO-style):
  - Fixed timestep simulation (60Hz)
  - Input delay + rollback frames (configurable 2-8 frames)
  - State serialization (deterministic, minimal)
  - Desync detection + automatic recovery
- [ ] **Server Rewind** for lag compensation:
  - Projectile/hitbox rewind based on shooter's ping
  - Server-side hit validation (no client trust)
- [ ] **Bandwidth Optimization**:
  - Delta compression, priority replication
  - Bit-packed input serialization
  - ~20-30 KB/s target per client

#### Module 5.2: Anti-Cheat Architecture (Server-Authoritative First)
| Layer | Implementation |
|-------|----------------|
| **Server-Authoritative** | All gameplay: movement, abilities, damage, mana, cooldowns |
| **Input Validation** | Rate limiting, impossible input detection (superhuman APM) |
| **State Verification** | Periodic checksums of critical game state |
| **Heuristics** | Aim-assist detection, movement pattern analysis |
| **Kernel/Client** | Optional: Easy Anti-Cheat / Vanguard integration (post-launch) |
| **Replay Verification** | Server records all matches, automated analysis |

#### Module 5.3: Matchmaking & Ranking
- [ ] **MMR System**: Glicko-2 or TrueSkill 2
- [ ] **Queue Types**: Ranked (BO3), Unranked (BO1), Tournament, Custom
- [ ] **Region Selection**: Ping-based, with manual override
- [ ] **Penalty System**: Leaver/AFK detection, escalating bans

---

### PHASE 6: UI/UX & POLISH (Weeks 47-56)
**Goal**: Production-quality menus, HUD, accessibility

#### Module 6.1: HUD & In-Game UI
- [ ] **Mana Bars**: Primary (blue) + Reserve (gold) with 50% threshold marker
- [ ] **Cooldown Display**: Radial on ability icons + keybind hints
- [ ] **Health/Shield**: Clear visual hierarchy
- [ ] **Combat Log**: Damage numbers, combat events (toggleable)
- [ ] **Spectator UI**: Player stats, ability cooldowns, mana, timeline

#### Module 6.2: Menus & Flow
- [ ] Main Menu → Play → Queue → Match Found → Load → Game → Results → Replay
- [ ] **Settings**: Input (rebind + modifier keys), Video, Audio, Gameplay, Accessibility
- [ ] **Profile**: Stats, Match History, Replay Library, Rank, Cosmetics
- [ ] **Character Select**: Preview abilities, talents, lore, pick/ban phase

#### Module 6.3: Replay System
- [ ] Server-authoritative replay recording (deterministic → small files)
- [ ] Replay playback: Timeline, player POV, free camera, graphs
- [ ] Export to video, share links

---

### PHASE 7: LIVE OPS & MONETIZATION (Weeks 57+)
**Goal**: Sustainable live service

#### Module 7.1: Cosmetic System
- [ ] Skins (character, abilities, projectiles, trails)
- [ ] Emotes, Sprays, Victory Poses
- [ ] Battle Pass (seasonal, no pay-to-win)
- [ ] **No Gameplay-Affecting MTX**

#### Module 7.2: Analytics & Telemetry
- [ ] Gameplay metrics: Win rates, pick rates, ability usage, mana curves
- [ ] Performance metrics: FPS, latency, desyncs, crashes
- [ ] Balance dashboard for designers

#### Module 7.3: Tournament Tools
- [ ] Custom game lobby with spectator slots
- [ ] Bracket integration, pause/resume, admin commands
- [ ] Observer overlay for streaming

---

## 3. CRITICAL TECHNICAL CHALLENGES & SOLUTIONS

### 3.1 Mana System Edge Cases
```cpp
// UDuelAttributeSet::PreAttributeChange
void UDuelAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    if (Attribute == GetPrimaryManaAttribute())
    {
        // Clamp to max
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxPrimaryMana());
    }
    else if (Attribute == GetReserveManaAttribute())
    {
        NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxReserveMana());
        
        // CRITICAL: If reserve drops below 50%, flag primary regen block
        if (NewValue < GetMaxReserveMana() * 0.5f)
        {
            SetPrimaryManaRegenBlocked(true);
        }
        else if (GetReserveMana() >= GetMaxReserveMana())
        {
            SetPrimaryManaRegenBlocked(false);
        }
    }
}

// Regen System (Server Only)
void ADuelCharacter::TickManaRegen(float DeltaTime)
{
    if (IsPrimaryManaRegenBlocked()) return;
    
    float RegenRate = bInCombat ? CombatManaRegen : OutOfCombatManaRegen;
    AddPrimaryMana(RegenRate * DeltaTime);
    
    // Reserve ONLY regens out of combat, very slowly
    if (!bInCombat)
    {
        AddReserveMana(ReserveManaRegen * DeltaTime);
    }
}
```

### 3.2 Fog of War Replication (Critical for Competitive Integrity)
```cpp
// Server: Only replicate actors client CAN see
class ADuelNetDriver : public UIpNetDriver
{
    virtual bool ShouldReplicateActor(AActor* Actor, const FNetViewer& Viewer) override
    {
        if (!Super::ShouldReplicateActor(Actor, Viewer)) return false;
        
        // Check FogOfWarManager
        if (AFogOfWarManager* FoW = GetWorld()->GetSubsystem<AFogOfWarManager>())
        {
            return FoW->IsVisibleTo(Viewer.InViewer, Actor);
        }
        return true;
    }
};
```

### 3.3 Seismic Sense Implementation
```cpp
// Server-side only - reveals through obstacles
void UEarthbenderComponent::ActivateSeismicSense()
{
    TArray<AActor*> DetectedActors;
    FCollisionShape Sphere = FCollisionShape::MakeSphere(SeismicRadius);
    
    GetWorld()->OverlapMultiByChannel(
        DetectedActors,
        GetOwner()->GetActorLocation(),
        FQuat::Identity,
        ECC_Pawn,
        Sphere,
        FCollisionQueryParams(SCENE_QUERY_STAT(SeismicSense), false, GetOwner())
    );
    
    for (AActor* Actor : DetectedActors)
    {
        if (ADuelCharacter* Char = Cast<ADuelCharacter>(Actor))
        {
            // Check line of sight IGNORED for seismic
            // But: Air/Water movement = silent (reduced radius)
            // Earth/Fire movement = loud (full radius)
            float EffectiveRadius = CalculateEffectiveRadius(Char);
            if (FVector::Dist(GetOwner()->GetActorLocation(), Char->GetActorLocation()) <= EffectiveRadius)
            {
                RevealToTeam(Char, ERevealType::Seismic);
            }
        }
    }
}
```

### 3.4 Enhanced/Weakened Ability System
```cpp
// UDuelGameplayAbility
enum class EAbilityVariant : uint8
{
    Normal,
    Enhanced,    // Shift held
    Weakened     // Ctrl held
};

void UDuelGameplayAbility::ActivateAbility(...)
{
    EAbilityVariant Variant = DetermineVariantFromInput();
    
    switch (Variant)
    {
        case EAbilityVariant::Enhanced:
            // Cost: PrimaryMana * 1.5 + ReserveMana * 0.3
            // Effect: Damage * 1.4, Radius * 1.2, Duration * 1.3
            // Cooldown: * 1.2
            break;
        case EAbilityVariant::Weakened:
            // Cost: PrimaryMana * 0.5, No Reserve
            // Effect: Damage * 0.6, Radius * 0.8
            // Cooldown: * 0.8
            break;
        default:
            // Normal costs/effects
            break;
    }
    
    // Server validates client has sufficient mana for CHOSEN variant
    Super::ActivateAbility(...);
}
```

---

## 4. DEVELOPMENT TIMELINE SUMMARY

| Phase | Duration | Focus | Key Deliverable |
|-------|----------|-------|-----------------|
| **0: Foundation** | 4 weeks | Infra, Netcode, Core | Playable empty arena |
| **1: Combat Core** | 8 weeks | GAS, Elements, Abilities | 2v2 playtest with 3 elements |
| **2: Camera/FoW** | 6 weeks | Vision, Camera, Seismic | Competitive-ready vision |
| **3: Characters** | 10 weeks | 8 chars, Talents, Loadouts | Full roster playable |
| **4: Maps** | 8 weeks | 4-6 maps, Destruction | Map pool ready |
| **5: Netcode/AntiCheat** | 10 weeks | Rollback, Security | Production netcode |
| **6: UI/Polish** | 10 weeks | Menus, HUD, Replays | Launch-ready client |
| **7: Live Ops** | Ongoing | Seasons, Cosmetics | Sustainable service |

**Total to Launch**: ~56 weeks (~13.5 months) with 3-4 engineers  
**MVP (Private Alpha)**: ~24 weeks (Phases 0-2 core)

---

## 5. TEAM STRUCTURE (Recommended)

| Role | Count | Responsibilities |
|------|-------|------------------|
| **Lead Engineer** | 1 | Architecture, netcode, anti-cheat, code review |
| **Gameplay Engineers** | 2 | GAS, abilities, characters, mana system |
| **Network Engineer** | 1 | Rollback, replication, server backend |
| **Tools/Build Engineer** | 1 | CI/CD, editor tools, data pipeline |
| **Technical Artist** | 1 | VFX, materials, optimization, Niagara |
| **Game Designer** | 1 | Balance, abilities, talents, map design |
| **UI/UX Designer** | 1 | HUD, menus, flow, accessibility |
| **QA/Playtest Lead** | 1 | Testing, automation, balance data |

---

## 6. RISK MITIGATION

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Netcode desyncs in rollback | High | Critical | Early prototype (Phase 0), extensive simulation testing |
| Mana system exploitation | Medium | High | Server-only validation, invariant checks |
| Fog of war info leaks | High | Critical | Server-side culling, penetration testing |
| Balance issues (elements) | High | High | Data-driven, telemetry, weekly balance patches |
| Scope creep | High | Medium | Strict phase gates, MVP definition |
| UE5 version upgrades | Medium | Medium | Lock engine version until major milestones |

---

## 7. IMMEDIATE NEXT STEPS (Week 1)

1. **Create UE5.4 Project** with C++ plugin structure
2. **Setup GitHub Repo** + Actions (Windows/Linux build)
3. **Implement Core Plugin Modules**: `DuelCore`, `DuelGameplay`, `DuelNet`
4. **Build AttributeSet** with Primary/Reserve Mana + Blocking Logic
5. **Prototype Input System** with Shift/Ctrl modifiers
6. **Basic Network Test**: 2 clients + server, character movement sync
7. **Document Architecture Decisions** in `ARCHITECTURE.md`

---

## 8. KEY CONFIGURATION VALUES (Tunable)

```ini
; DefaultGame.ini [DuelGameplaySettings]
; Mana System
BasePrimaryMana=1000
BaseReserveMana=500
PrimaryManaRegen_OutOfCombat=50/sec
PrimaryManaRegen_InCombat=10/sec
ReserveManaRegen_OutOfCombat=5/sec
ReserveManaRegen_InCombat=0
ReserveThreshold_Percent=0.50  ; 50% blocks primary regen

; Ability Variants
Enhanced_PrimaryManaMultiplier=1.5
Enhanced_ReserveManaCost=0.3   ; 30% of base reserve cost
Enhanced_DamageMultiplier=1.4
Enhanced_CooldownMultiplier=1.2

Weakened_PrimaryManaMultiplier=0.5
Weakened_ReserveManaCost=0.0
Weakened_DamageMultiplier=0.6
Weakened_CooldownMultiplier=0.8

; Key Abilities (Reserve Only)
Teleport_ReserveCost=150
Reflect_ReserveCost=200
GlobalMeteor_ReserveCost=400
AvatarState_ReserveCost=500  ; Full reserve

; Netcode
TargetTickRate=60
MaxRollbackFrames=8
InputDelayFrames=2
MaxPacketSize=1200

; Fog of War
VisionUpdateRate=10Hz
FogTextureResolution=512
SeismicSense_BaseRadius=1500
SeismicSense_AirWaterReduction=0.5
```

---

*Document Version: 1.0*  
*Last Updated: 2026-11-17*  
*Status: DRAFT - For Review & Iteration*