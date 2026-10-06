## 👤 About the Developer
- **Nama:** Rozak Subagja
- **Role:** Roblox Luau Programmer  
- **Email:** Babehbhinay31@gmail.com
- **CV:** [Download CV (PDF)](https://drive.google.com/file/d/1Z5-IZraEJA6CvJuy5nZkjJP_XZvoW13O/view?usp=drive_link)
- **Game:** [Thangkuban Expedition](https://www.roblox.com/id/games/110963911512584/Ekspedisi-Thangkuban)
# Thangkuban Expedition - Technical Portfolio

> **Role:** Lead Programmer  
> **Platform:** Roblox (Luau)  
> **Genre:** Mountain Climbing / Expedition Simulation  
> **Players:** Multiplayer (up to 30 per server)  
> **Data Layer:** ProfileService + DataStore  
> **Anti-Cheat:** Server-authoritative with Discord webhook logging  

---

## Table of Contents

| # | System | Description |
|---|--------|-------------|
| 1 | [ProfileService / DataStore](#1-profileservice--datastore) | Data persistence, transaction locks, ProcessReceipt, daily rewards |
| 2 | [Checkpoint](#2-checkpoint) | Camp checkpoints, tent placement, revive system, teleport handshake |
| 3 | [Death / Life](#3-death--life) | Kill bricks, fall damage, water damage, anti-regen, lives UI |
| 4 | [VIP / VVVIP](#4-vip--vvvip) | Helmet welding, role-based doors, live group membership, patrol teleport |
| 5 | [Inventory](#5-inventory) | Custom backpack UI, batch saving, currency transfer, trading |
| 6 | [Teleport](#6-teleport) | Camera orbit cutscene, letterbox, title cards, waypoint walking |
| 7 | [Mission / Quest](#7-mission--quest) | Summit mission (31 wins = Tuyul pet), sandboxed remotes |
| 8 | [Pet](#8-pet) | 5 skills, stat system, starvation lock, CariDuit load balancing, name filtering |
| 9 | [Leaderboard](#9-leaderboard) | 4 leaderboard types, role tiers, miss counting, MessagingService |
| 10 | [Remote Events / Architecture](#10-remote-events--architecture) | 57 RemoteEvents, 9 RemoteFunctions, server init, cross-server config sync |
| 11 | [UI Scripting](#11-ui-scripting) | 9 UI scripts, notification system, hysteresis warnings, death screen |
| 12 | [Gamepass / DevProduct / Shop](#12-gamepass--devproduct--shop) | Monetization, anti-regift, VIP discount, bank recovery, Sawer donations |
| 13 | [Optimization / Anti-Cheat](#13-optimization--anti-cheat) | Speed detection, strike system, Discord webhook, checkpoint validation |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        SERVER (Authoritative)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │ ProfileService│  │ DataManager  │  │ AntiCheat + Strike     │  │
│  │ (Session Lock)│  │ (Transaction │  │ (Speed/Position Check) │  │
│  └──────┬───────┘  │  Locks)      │  └───────────┬───────────┘  │
│         │          └──────┬───────┘               │              │
│  ┌──────┴───────┐  ┌──────┴───────┐  ┌───────────┴───────────┐  │
│  │ Checkpoint   │  │ Pet System   │  │ ValidasiWin             │  │
│  │ + Tent        │  │ (5 Skills)   │  │ (Checkpoint Validation) │  │
│  │ + Revive      │  │ (Stat Decay)  │  └────────────────────────┘  │
│  └──────────────┘  └──────────────┘                               │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │ Shop/Bank    │  │ Leaderboard   │  │ Helmet/Door/Patrol     │  │
│  │ (Anti-Regift) │  │ (4 Types)     │  │ (Role-Based Access)    │  │
│  └──────────────┘  └──────────────┘  └───────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                    REMOTESTORAGE (Bridge)                        │
│  57 RemoteEvents · 9 RemoteFunctions · Constants Module           │
├─────────────────────────────────────────────────────────────────┤
│                        CLIENT (Presentation)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │ UI Scripts(9) │  │ Teleport FX   │  │ Inventory Controller  │  │
│  │ (Notifications│  │ (Cutscene)    │  │ (Multi-Input)         │  │
│  │  HUD, Deaths) │  │              │  │                       │  │
│  └──────────────┘  └──────────────┘  └───────────────────────┘  │
│  ┌──────────────┐  ┌──────────────┐                               │
│  │ Pet Panel    │  │ Shop Client   │                               │
│  └──────────────┘  └──────────────┘                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## System Summaries

### 1. ProfileService / DataStore

**Scope:** Data persistence for all player data (currency, checkpoints, deaths, pets, gamepasses, daily rewards).

**Key Features:**
- ProfileService with session locking, auto-saving, and corruption handling
- Transaction lock system prevents race conditions on currency operations
- Currency hard-capped at 1 billion to prevent overflow exploits
- Unified ProcessReceipt with deduplication (5-min cleanup cycle)
- Daily login with 7-day streak tracking and UTC normalization
- DataStoreManager with exponential backoff retry (3 attempts)
- BindToClose handler saves all profiles on server shutdown

**Design Patterns:**
- Singleton DataManager as the single API surface for all data operations
- Request-response for ProcessReceipt (deduplication table)
- Observer pattern for attribute-based UI sync

**Security:**
- Server-authoritative: All data operations on server only
- pcall-wrapped DataStore operations with retry logic
- Session ownership validation before every save
- Corruption detection and silent recovery
- Receipt deduplication prevents double-granting

---

### 2. Checkpoint

**Scope:** Camp-based checkpoint system with tent placement, respawn management, and revive.

**Key Features:**
- Client-server teleport acknowledgment handshake (timeout-based)
- Revive system: 20-second window with Robux purchase callback
- Tent placement with 13-point server-side validation (type, NaN, profile, carrying, patrol, cooldown, tool, distance, raycast, slope, spacing, obstruction)
- Patrol mode blocking prevents checkpoint updates
- Late revive grant handling for purchases after window expiry

**Design Patterns:**
- Handshake with graceful degradation (continues without ack)
- Defense-in-depth validation chain (13 sequential checks)
- Attribute-based state machine (ReviveWindow, PatrolMode, CarryingTarget)

**Security:**
- Server-side raycast verification (not trusting client Y position)
- Cooldowns on teleport (2s) and touch (1s)
- Carrying state check prevents teleport while carrying/being carried
- CollectionService tag checks for restricted zones

---

### 3. Death / Life

**Scope:** Comprehensive death handling including kill bricks, water damage, fall damage, and life counter.

**Key Features:**
- Three-phase fall detection: air -> falling -> landing (with water splash)
- Multi-source fall immunity: potion, carrying, teleport, parachute, rappel
- Anti-regen health locking with attribute-based override for legitimate healing
- Water damage: 5 DPS while swimming (Heartbeat-based)
- Kill bricks with 0.2s cooldown (folder-based)
- Lives counter UI with critical blinking at 1 life

**Design Patterns:**
- State machine for fall detection (3 phases with distinct transitions)
- Health lock with override flag (IsBeingHealed attribute)
- Heartbeat loop with elapsed-time accumulator for water damage

**Security:**
- Server-authoritative: All damage applied on server
- os.clock() for accurate short-duration timing
- Health check before applying damage (humanoid.Health > 0)
- Connection cleanup on character/player removal

---

### 4. VIP / VVVIP

**Scope:** Multi-tier access system with helmet equipping, role-based doors, and patrol teleport.

**Key Features:**
- 5 helmet variants with dynamic discovery (ChildAdded) and Motor6D attachment
- Clone-and-weld pattern: Weld all parts to Middle, Motor6D to character Head
- Three-tier role-based access: VIP, Navigator, Group
- Live group membership double-check (bypasses Roblox 30-min cache via GroupService)
- Server-authoritative teleport with live HRP refresh

**Design Patterns:**
- Factory pattern for helmet cloning and welding
- Double-check pattern for group membership (cached -> live API)
- Role-based access control (RBAC) with attribute sync

**Security:**
- Server-authoritative: All helmet equipping on server
- Role validation on every door trigger (not cached)
- Carrying state check blocks all door types
- Server-side cooldown per player per door (0.5s)
- Massless attribute preservation (KeepMassless flag)

---

### 5. Inventory

**Scope:** Custom inventory UI replacing default backpack with hotbar, trading, and persistence.

**Key Features:**
- Multi-input: keyboard (1-9, numpad), mouse wheel, gamepad, touch
- Chat-aware input detection prevents hotkey conflicts
- Mobile-aware layout with dynamic cell size calculation
- Batch save system with parallel processing (5 at a time, 15s intervals)
- Server-authoritative currency transfer with daily limit (5,000,000)
- Multi-layer tradeability check (banned items, event rods, untradeable attribute)
- Stack-safe non-consumable transfer (amount attribute management)

**Design Patterns:**
- Delegated logic pattern (UI controller delegates to settings module)
- Request-response for currency transfers (server-side pending state)
- Batch processing with throttling for DataStore saves

**Security:**
- Server-authoritative save/load (clients cannot modify saved data)
- Tool blacklist prevents saving exploit items
- Save locks prevent concurrent saves for same player
- Transfer pending state is server-side (prevents client spoofing)
- Transfer expiration (6 seconds) prevents stale transfers
- Min save interval (10s) prevents spam

---

### 6. Teleport

**Scope:** Cinematic teleport system with camera cutscene, letterbox, title cards, and waypoint walking.

**Key Features:**
- Smoothstep easing for camera orbit animation (RenderStepped)
- Dynamic GUI hiding during cutscene (including ChildAdded listener)
- Player control lock during cutscene with re-enable on completion
- Waypoint walking with timeout protection per waypoint (5s max)
- Movie-style title card: fade-in -> slide-up -> underline reveal -> fade-out

**Design Patterns:**
- finally pattern for guaranteed GUI restoration
- Observer pattern for ChildAdded (catches new GUIs during cutscene)
- Sequence pattern for title card animation (chained tweens)

**Security:**
- Client-side only (LocalScript) - no game logic exposed
- Graceful degradation: pcall on PlayerModule access
- Timeout protection per waypoint prevents infinite walking
- Control lock re-enabled on completion regardless of errors

---

### 7. Mission / Quest

**Scope:** Tiered mission system where players summit 31 times to earn a special pet (Tuyul).

**Key Features:**
- Dynamic RemoteEvent creation with Sandboxed = true
- Profile-ready attribute sync on player join (once-pattern)
- Server-side validation of wins count before granting rewards
- Duplicate claim prevention (attribute + profile check)
- Input sanitization with math.max(0, count) in QuestModule

**Design Patterns:**
- Once-pattern (disconnect after first fire for ProfileReady sync)
- Guard clause chain (profile, claimed, wins, pet ownership)
- Pure utility module (QuestModule) with type annotations

**Security:**
- RemoteEvents are Sandboxed = true (prevents cross-script access)
- Server-side validation of wins count before granting reward
- Checks if pet is already owned before granting (duplicate prevention)
- No direct data modification - always goes through attributes

---

### 8. Pet

**Scope:** Full pet companion system with 5 skills, stat management, dance mechanics, feeding economy, and naming.

**Key Skills:**
- **Bother:** Pet punches nearby players
- **Jail:** Pet chases and harasses target with periodic punching
- **Guard:** Pet defends against NPCs
- **GiveLife:** Pet transfers lives to low-life players
- **CariDuit:** Pet searches trees for money (load-balanced tree selection)

**Key Features:**
- Heartbeat-based follow loop with lerp movement and mode switching
- State machine pattern per skill (Approach -> Action -> Return)
- Starvation lock system (hunger < 70 blocks skills and leveling)
- Stat decay loop (every 30s) with reduced decay during CariDuit (75% reduction)
- Level scaling for CariDuit rewards
- Collision group reassertion every 0.2s
- TextService filtering for pet names
- Pet feed with currency charge (DataManager.TryPurchase)
- Separate SearchingPetInstance system for CariDuit

**Design Patterns:**
- Module-based architecture (15+ modules for separate concerns)
- State machine per skill with distinct phases
- Load-balanced selection for CariDuit (fewest players per tree)
- Observer pattern for stat decay and attribute sync

**Security:**
- Sandboxed RemoteEvents for all pet communication
- Type checking on all remote inputs
- Debounce on all remote events (0.4s)
- Server-side validation of pet ownership before equipping
- Server-authoritative stat management and decay
- Level cap at 70 with akrab overflow wrapping
- DataManager integration for atomic currency deduction

---

### 9. Leaderboard

**Scope:** Multi-tier leaderboard system with 4 types, role tiers, and real-time updates.

**Leaderboard Types:**
- **Global Wins** (30s refresh): Multi-page DataStore fetch, role tiers (gold crown, gold, silver)
- **Top Spender** (60s refresh): Miss-counting system for graceful rank removal (2-cycle threshold)
- **Local Expedition** (120s refresh): Per-server ranking
- **Anniversary** (MessagingService + 60s polling): Cross-server special event

**Key Features:**
- Name/thumbnail caching with fallback values ("User" + userId)
- Rank attribute sync only fires on actual change (prevents event spam)
- Hidden IDs filter prevents admin accounts from appearing on leaderboard
- Random initial delay (5-20s) to stagger server loads
- Exponential backoff retry (2^attempt * 0.2s)
- sanitizeIdList ensures only valid numbers processed

**Design Patterns:**
- Module-based architecture with shared utility module
- Singleton guard (attribute-based prevention of duplicate execution)
- Dual strategy: MessagingService (real-time) + polling (fallback)
- Miss-counting pattern for graceful state transitions

**Security:**
- pcall on all DataStore operations
- lastGoodPage fallback: uses cached data if DataStore fetch fails
- Connection tracking and cleanup on shutdown
- Rank attribute sync only fires on actual change

---

### 10. Remote Events / Architecture

**Scope:** Comprehensive client-server architecture with 57 RemoteEvents and 9 RemoteFunctions.

**Remote Events (57):**
- Notifications (7): CreateNotification, ScreenFade, TopJoinNotify, DonationAnnounce, CariDuitNotif, RefillNotification, BankFullWarning
- Gameplay (6): SprintToggle, RequestRevive, ShowRevivePrompt, ToggleAura, EquipHelmet, MaskToggleEvent
- Movement (4): ClimbRequest, TeleportToSPEAG, TeleportComplete, StartTeleportLoading
- Inventory/Storage (8): GiveMedkitTool, StoreItem, RetrieveItem, UpdateStorageUI, OpenStorageUI, SetMaxUses, UpdateUsesServer
- Banking (4): BankDeposit, BankWithdraw, SyncBankItems, SyncBankCapacity
- Transfers (3): TransferDuit, TransferRequest, TransferResponse
- Quests (3): SetMissionActive, StartIstanaQuest, QuizPassed
- Chests (8): ClaimChest, SpawnChestEvent, RequestSpawnChest, ClaimTreasureEvent, ApplyChestCollisionGroup, LootChestClaim, ChestBlipUpdate, GetChestPositions
- Admin/Utility (14): ChooseTitle, BadgeClaim, MusicControl, GroupJoinEvent, PlayerKicked, MiahriPrivate, RefreshTopEffect, GetMapCount, KorekApiState, SupplyGiveSelf, SendingStatus, HelmetSelected, UpdateDeathsUI, ToggleHelmetCam

**Remote Functions (9):**
- RequestItemFromBank, TryDepositItem, GetBankCapacity, CheckTerrainValidation, GetChestData, ClaimMap, GetCollectedMaps, RequestStreamAt, GetBankSnapshot

**Key Features:**
- ServerInit: join time tracking, cross-server config sync (MessagingService), FPS broadcast (0.5s), country detection (5 retries)
- PlayerSetup: leaderstats, gamepass with legacy ID support, group rewards, navigator role
- Gamepass ID aliasing for migrations (old IDs map to new IDs)
- Navigator role with heli-activation guard (prevents switching mid-activity)
- Role sync ensures client always matches server state

**Design Patterns:**
- Server-authoritative architecture
- Event-driven communication (RemoteEvents for one-way, RemoteFunctions for request-response)
- Config sync via MessagingService with validation

**Security:**
- Server-authoritative design: All game logic on server
- pcall-wrapped API calls with retry logic
- Role sync on every state change prevents desync
- Heli-activation guard prevents role switching mid-activity
- Cross-server config sync with validation

---

### 11. UI Scripting

**Scope:** Professional UI system with 9 client-side scripts.

**UI Scripts:**
- Lives counter (Nyawa) with critical blinking at 1 life
- Death screen with revive integration
- Hunger/Thirst/Health HUD with weighted color interpolation
- Sprint system with stamina bar
- Money display with Indonesian currency formatting (Rp123,456.78)
- Checkpoint notification
- In-game clock
- Notification system (5 types: INFO, ERROR, GIFT, WARN, SUCCESS)
- Screen fade handler

**Key Features:**
- Hysteresis-based warning system (5% buffer prevents toast spam)
- Dynamic UI creation (no pre-placed elements for notifications)
- TweenService for all animations
- Attribute-driven data binding
- Weighted color interpolation for stamina bar (inverse square weighting)
- Max 5 notifications with auto-removal and progress bar
- Mobile-responsive inventory with dynamic cell sizing

**Design Patterns:**
- Singleton pattern for notification system (_G.ShowClientNotification)
- Observer pattern for attribute-driven UI updates
- Hysteresis pattern for threshold-based warnings

**Security:**
- Client-only UI (LocalScripts) - no game logic exposed
- Fail-safe label lookup with FindFirstChildWhichIsA fallback
- Race condition prevention (single CharacterAdded listener)
- Type-safe attribute access with fallbacks
- Tween cancellation on destroy (memory-safe)

---

### 12. Gamepass / DevProduct / Shop

**Scope:** Complete monetization system including gamepass shop, tool shop, donations, banking, and anti-regift.

**Key Features:**
- Anti-regift system via persistent DataStore tracking (GiftGamepasses_V1)
- Gamepass ID aliasing for migrations (old IDs -> new IDs)
- VIP discount (15%) with cached verification
- Purchase debounce and rollback on grant failure
- Per-life claim tracking for money giver parts (prevents farming)
- Pending transaction system for crash recovery (BankServer)
- Text filtering via TextService for donation messages
- Admin gift system with live attribute application (ForceGrantGamepass)
- Live regional price fetch for donation products
- Sawer (donation) system with queue, batch, and announcement

**Design Patterns:**
- Transaction pattern with rollback (refund on grant failure)
- Pending transaction pattern for crash recovery (write pending -> modify -> clear)
- Cache pattern for VIP verification and price lookup
- Queue pattern for donation announcements (FIFO with max size)

**Security:**
- Whitelist validation prevents buying arbitrary items
- Purchase debounce (0.5s) prevents duplicate transactions
- Per-life claim tracking prevents money farming exploits
- FORBIDDEN_ITEMS list prevents storing admin/exploit items in bank
- withLock pattern prevents concurrent bank transactions
- Multi-retry DataStore saves (3 attempts) for bank data
- Backup DataStore for disaster recovery (every 300s)
- Rejoin cooldown (8s) prevents rapid rejoin exploits
- Anti-regift: permanent DataStore tracking of gifted gamepasses

---

### 13. Optimization / Anti-Cheat

**Scope:** Multi-layered anti-cheat with speed detection, strike tracking, Discord logging, and checkpoint validation.

**Key Features:**
- TrailRunAntiCheat: Heartbeat-based dual speed detection
  - Sustained speed (1s threshold at 1.05x legal speed)
  - Instant speed (position delta at 2.0x legal speed)
- CheatStrikeHandler: Persistent DataStore strike tracking (3 strikes = kick + checkpoint reset)
- AntiCheatLogger: Batched Discord webhook logging (15 events/batch, 8s intervals, per-player 3s cooldown)
- ValidasiWin: Flexible validation modes (sequential or minimal count)
- Speed validation at checkpoint touch (max 110 studs/s)
- Air gap detection via raycast (max 12 studs from ground)
- Carried player progress sync (gendong mechanic)
- Rappel ending buffer prevents false positives
- Admin bypass system with controlled modes (instant/step)

**Design Patterns:**
- Dual detection pattern (sustained + instant) for robustness
- Strike accumulation pattern with DataStore persistence
- Batch processing with throttling for webhook delivery
- Flexible validation strategy (mode-based: sequential vs minimal)

**Security:**
- Server-authoritative: All validation on server, no client trust
- Instant kick on speed hack detection
- Persistent DataStore strike tracking (survives rejoins)
- Auto-kick at 3 strikes with checkpoint reset
- Strike reset after kick prevents cross-session accumulation
- Speed thresholds with tolerance factors
- Air gap detection via raycast prevents fly hacking
- Queue size cap (200 events) with FIFO eviction
- Heli/patrol detection for validation bypass

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Language | Luau (Roblox) |
| Data Persistence | ProfileService + DataStore |
| Client-Server Communication | 57 RemoteEvents + 9 RemoteFunctions |
| Cross-Server Sync | MessagingService |
| Text Filtering | TextService |
| Group Membership | GroupService (live API) |
| Analytics/Logging | Discord Webhook (HttpService) |
| Monetization | Gamepass + Developer Products + Sawer |
| UI Framework | TweenService + ProximityPrompts |
| Anti-Cheat | Server-authoritative + Speed/Position Validation |

---

## Key Design Principles

1. **Server-Authoritative:** All game state, data, and validation run on the server. Clients are presentation-only.
2. **Defense in Depth:** Multiple validation layers (type checks, profile checks, state checks, distance checks, raycast verification).
3. **Graceful Degradation:** Systems continue functioning even when optional components fail (timeouts, fallbacks, cached data).
4. **Atomic Operations:** Currency and data operations use locks and rollback patterns to prevent race conditions.
5. **Modular Architecture:** Systems are split into focused modules with clear responsibilities (15+ pet modules, 4 leaderboard modules).
6. **Connection Safety:** All event connections are tracked and cleaned up on shutdown/player removal.
7. **Anti-Exploit by Design:** Input sanitization, debouncing, deduplication, and server-side validation throughout.

---

## File Structure

```
ServerScriptService/
├── ProfileService              # Base module (session locking)
├── DataManager                 # Business logic + ProcessReceipt
├── DataStoreManager            # Leaderboard DataStores
├── PlayerDataLoader             # Init wrapper
├── PlayerLoadCoordinator        # Sequential equipment restore
├── CampCheckpointSpawn          # Core checkpoint logic
├── CampCheckpointSpawnTenda     # Tent placement validation
├── FallDamageHandler            # Fall damage calculation (ModuleScript)
├── FallDamageController          # Fall detection loop
├── WaterDamage                  # Water 5 DPS loop
├── KillBrickSetup               # Kill zones setup
├── AntiRegen                    # Health regen blocker
├── HelmetHandler                # Helmet equipping
├── DoorService                  # Role-based door access
├── BackpackPersist              # Inventory save/load
├── TransferHandler              # Currency transfer
├── SimpleTradeServer            # Item trading
├── SummitMissionHandler         # 31-summit mission
├── PetFollowService/            # Pet system (15+ modules)
│   ├── Skills/                  # 5 skill modules
│   └── ...                      # Stats, movement, config, etc.
├── LeaderboardManager/          # 4 leaderboard modules
│   └── LeaderboardModules/      # Shared, Global, Sawer, Local, Anniversary
├── ServerInit                   # Cross-server config + FPS broadcast
├── PlayerSetup                  # Leaderstats + gamepass + roles
├── ShopServer                   # Gamepass gifts + anti-regift
├── ItemPurchaseAndMoneyGiverFIX # Item purchases
├── SawerServer                  # Donation system
├── ATMServer                    # ATM transactions
├── BankServer                   # Item bank + crash recovery
├── TrailRunAntiCheat            # Speed detection (ModuleScript)
├── AntiCheatLogger              # Discord webhook (ModuleScript)
├── CheatStrikeHandler           # Strike system (ModuleScript)
└── ValidasiWin                  # Checkpoint validation (ModuleScript)

ReplicatedStorage/
├── RemoteEvents/                # 57 RemoteEvents
├── RemoteFunctions/             # 9 RemoteFunctions
├── Constants                    # Centralized attribute names
├── QuestModule                  # Quest utility module
└── Portfolio/                   # ← This documentation
    └── Systems/                 # 13 ModuleScripts

StarterGui/
├── Nyawa/                       # Lives counter UI
├── Death Effect/                # Death screen
├── HungerAndHealthBar/          # HUD
├── Sprint/                      # Sprint system
├── UI/                          # Money display
├── CheckpointUI/                # Checkpoint notification
├── Clock/                       # In-game clock
├── Custom Inventory/            # Custom backpack UI
├── DonasiGui/                   # Donation UI
└── ToolShop/                    # Tool shop UI

StarterPlayer/StarterPlayerScripts/
├── TeleportFX                  # Cutscene system
├── PetPanel                    # Pet UI
├── NotificationHandler         # Global notification system
├── ScreenFadeHandler           # Screen fade effects
├── CheckpointClient             # Checkpoint client logic
└── ShopClientFIX               # Modern shop UI
```

---

## Metrics

| Metric | Value |
|--------|-------|
| Total Systems | 13 |
| RemoteEvents | 57 |
| RemoteFunctions | 9 |
| Server Scripts | 30+ |
| Client Scripts | 15+ |
| ModuleScripts | 20+ |
| DataStores | 5+ (Profile, Bank, Strikes, Gifts, Leaderboards) |
| Pet Skills | 5 |
| Leaderboard Types | 4 |
| Anti-Cheat Layers | 7+ |

---

## License

This portfolio documentation is provided for technical showcase purposes. The game code itself is proprietary.

---

*Generated as a Roblox portfolio package. All code examples are extracted from the live production game.*
]]
