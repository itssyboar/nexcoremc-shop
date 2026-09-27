# NexCoreSmartSpawners

A small companion plugin for a Paper 1.21.11 server that adds a
**Smart Spawners** category to an existing **DonutShop v3.0** `/shop` GUI,
selling real **SmartSpawner v1.5.8** spawner items.

It does **not** modify, repackage, or depend at compile time on
`Donut_Shop.jar`. DonutShop is only ever touched through its public,
generic `org.bukkit.plugin.Plugin` interface (`getConfig()`, `isEnabled()`)
and its own `/shop` command.

---

## 1. What was inspected, and what it means for this plugin

Both supplied jars were disassembled at the bytecode level (constant pool +
method bodies) before any code was written, because no decompiler or `javap`
was available offline. What was found:

**Donut_Shop.jar** (registers itself as plugin name `CredosShop`, not
`DonutShop` — see its `plugin.yml`):
- `MainShopGUI.open()` builds a 27-slot inventory with
  `Bukkit.createInventory(null, 27, title)` — a **plain inventory, no
  custom `InventoryHolder`**. Slots 11/12/13/14/15 = End/Nether/Gear/Food/
  Shard. All other slots are invisible `AIR` "placeholder" items.
- `InventoryClickListener` decides which shop is open by comparing
  `InventoryView.getTitle()` against strings read from DonutShop's own
  `config.yml` (`gui.main-shop-title`, `gui.end-shop-title`, etc.) — there
  is no holder or NBT tag to detect it by.
- Purchases: check `Economy.getBalance(player) >= price`, check inventory
  space, `Economy.withdrawPlayer(...)`, `PlayerInventory.addItem(...)`,
  send a message + play a sound.

This plugin therefore:
- Identifies the DonutShop main menu the same way DonutShop's own listener
  does — by title — but reads that title **live** from
  `Bukkit.getPluginManager().getPlugin("CredosShop").getConfig()` so the
  two can never drift out of sync, and never needs `Donut_Shop.jar` on its
  classpath.
- Injects its button into the already-populated `InventoryOpenEvent`
  inventory (slot 16 by default, configurable), rather than editing
  DonutShop's classes.
- Reopens the main menu on "Back" by calling `player.performCommand("shop")`
  — DonutShop's own public command — never any internal DonutShop class.

**SmartSpawner-1.5.8.jar** (plugin name `SmartSpawner`):
- Ships a public API: `github.nighter.smartspawner.api.SmartSpawnerProvider
  .getAPI()` → `SmartSpawnerAPI`, with
  `createSpawnerItem(EntityType type, int amount)`.
- Its implementation (`SpawnerItemFactory.createSmartSpawnerItem`) builds
  `new ItemStack(Material.SPAWNER, amount)`, gets `BlockStateMeta`, casts
  the block state to `CreatureSpawner`, calls
  `setSpawnedType(entityType)`, and writes the state back — the exact same
  code path SmartSpawner itself uses everywhere else. It does **not** set
  the `vanilla_spawner` or `item_spawner_material` PDC markers that flag
  the other two spawner kinds, so `SmartSpawnerAPI.isSmartSpawner(item)`
  returns `true` for it.
- `DynamicEntityValidator` allows every `EntityType` except `PLAYER`, so
  all 8 requested mobs (Skeleton, Zombie, Blaze, Iron Golem, Evoker,
  Creeper, Pig, Cow) are valid.

This plugin therefore uses `SmartSpawnerProvider.getAPI().createSpawnerItem(
entityType, 1)` directly — the officially supported method — instead of the
`/smartspawner give` command or hand-built NBT.

---

## 2. Files created

```
NexCoreSmartSpawners/
├── build.gradle
├── settings.gradle
├── libs/
│   └── SmartSpawner-1.5.8.jar        (your uploaded jar, compile-time only)
├── src/main/resources/
│   ├── plugin.yml
│   └── config.yml
└── src/main/java/com/nexcore/smartspawners/
    ├── NexCoreSmartSpawners.java      (main class: onEnable/onDisable, wiring)
    ├── PurchaseService.java           (balance/inventory checks, real item creation, duplicate-click guard)
    ├── SmartSpawnerShopCommand.java   (/smartspawnershop command)
    ├── config/
    │   ├── SpawnerEntry.java
    │   └── SpawnerCatalog.java        (loads & validates config.yml's 8 spawners)
    ├── gui/
    │   ├── SmartSpawnerShopGUI.java    (builds/opens the Smart Spawners menu)
    │   └── SmartSpawnerShopHolder.java (real InventoryHolder for OUR gui only)
    ├── integration/
    │   └── MainShopIntegration.java    (all DonutShop contact — config/title/command only)
    ├── listeners/
    │   ├── MainShopInjectListener.java (adds the button to DonutShop's menu, handles its click)
    │   └── SpawnerShopListener.java    (purchase clicks, back button, anti-dupe cancelling)
    └── util/
        └── TextFormatter.java          (& color codes + <#RRGGBB> hex tags)
```

None of DonutShop's own files were touched, read at build time, or bundled.

---

## 3. How to build the JAR

**Important limitation, stated plainly:** the sandbox this project was
written in has no `javac`/JDK-with-compiler and no network access (outbound
requests return `403`), so I am not able to actually invoke a compiler or
download Gradle/the Paper API/Vault API jars here. I reviewed every file by
hand — package declarations match their folder paths, every referenced
class exists, braces balance, and every Bukkit/Vault/SmartSpawner API call
was checked against the real method signatures I extracted from your two
jars' bytecode (see section 1) — but I have not run a compiler over it, so
I can't hand you a jar I fabricated without one. Rather than pretend to
compile it, here are two real ways to get an actual compiled jar, in order
of how little setup they need:

### Option A (recommended, no local Java install needed): GitHub Actions

This project includes `.github/workflows/build.yml`, which builds the jar
in the cloud on every push and lets you download the result — you don't
need Java, Gradle, or anything else installed on your own machine:

1. Create a new (can be private) GitHub repository and push the contents
   of this `NexCoreSmartSpawners/` folder to it.
2. Go to the repo's **Actions** tab. A "Build NexCoreSmartSpawners"
   workflow run should start automatically (or click **Run workflow** to
   trigger it manually).
3. Open the finished run, scroll to **Artifacts**, and download
   `NexCoreSmartSpawners-jar` — this is your real, compiled
   `NexCoreSmartSpawners-1.0.0.jar`.
4. The workflow also runs `jar tf` on the built jar and prints its full
   contents in the "List produced jar contents" step's log — check that
   log to confirm every class below (section 4) is actually present
   before you download it.

### Option B: build locally

1. Install a JDK 21 (e.g. Eclipse Temurin 21) — `java -version` should
   report 21.
2. `libs/SmartSpawner-1.5.8.jar` is already the exact jar you uploaded —
   leave it there. If you ever update SmartSpawner, replace this file with
   the new version and rebuild.
3. If there's no `gradlew`/`gradlew.bat` in the folder yet (this project
   doesn't ship the wrapper jar, since generating it also requires
   network access), install Gradle once yourself, then from inside
   `NexCoreSmartSpawners/` run `gradle wrapper --gradle-version 8.10` to
   generate it, or just run `gradle clean build` directly.
4. `./gradlew clean build` (or `gradle clean build`).
5. The compiled jar appears at `build/libs/NexCoreSmartSpawners-1.0.0.jar`.

The Paper API version in `build.gradle` is now
`io.papermc.paper:paper-api:1.21.11-R0.1-SNAPSHOT` — confirmed against
Paper's own 1.21.11 release notes and a live server's reported
"Implementing API version" string, so this is the correct artifact for
your exact server version, not an approximation.

## 4. Verifying the compiled jar has no missing classes

Once you have `NexCoreSmartSpawners-1.0.0.jar` (from either option above),
confirm it's complete before installing it:

```
jar tf NexCoreSmartSpawners-1.0.0.jar
```

You should see `plugin.yml`, `config.yml`, and a `.class` file for each of:
`NexCoreSmartSpawners`, `PurchaseService`, `SmartSpawnerShopCommand`,
`config/SpawnerEntry`, `config/SpawnerCatalog`, `gui/SmartSpawnerShopGUI`,
`gui/SmartSpawnerShopHolder`, `integration/MainShopIntegration`,
`listeners/MainShopInjectListener`, `listeners/SpawnerShopListener`,
`util/TextFormatter`.

You will **not** see any SmartSpawner, Vault, or Bukkit/Paper classes in
that list, and that's correct, not a bug: all three are `compileOnly`
dependencies, meaning they're only used to check the code compiles
correctly against their real APIs — they are never bundled into this jar,
because your server already provides all three as separately installed
plugins/server jar. If `jar tf` shows any of those bundled in, the build
configuration was changed from `compileOnly` to something else (like
`implementation`) and should be reverted.

## 5. Where the JAR goes

Copy `NexCoreSmartSpawners-1.0.0.jar` into your server's `plugins/` folder,
alongside `Donut Shop.jar`, `SmartSpawner-1.5.8.jar`, and your Vault jar.

## 6. Which plugins must be installed

- **SmartSpawner 1.5.8** — hard dependency (`depend` in `plugin.yml`). The
  plugin refuses to enable without it.
- **Vault** (+ any economy plugin, e.g. EssentialsX Economy) — hard
  dependency. The plugin refuses to enable without it.
- **DonutShop (CredosShop)** — soft dependency. Without it, the plugin
  still enables and `/smartspawnershop` still works, but no button is
  injected anywhere since there's no main menu to inject into.

## 7. How to configure prices

Edit `plugins/NexCoreSmartSpawners/config.yml`, under `smart-spawners:` —
each entry has its own `price:`. Run `/nexcorespawners reload` — actually,
this build doesn't include a reload subcommand; just restart the server
(see section 8) or `/reload` (not generally recommended on production Paper
servers) after editing. Also configurable: the GUI titles, button text,
messages, sounds, the DonutShop plugin name/command it expects, and which
slot the category button uses.

## 8. How to restart the server safely

1. Warn players / set to whitelist-only if needed.
2. `stop` the server from console (not "kill"), so Paper saves worlds
   cleanly.
3. Drop the new jar into `plugins/`.
4. Start the server back up and watch the console log for:
   - `NexCoreSmartSpawners has been enabled with 8 spawner types configured.`
   - No `SmartSpawner was not found` / `Vault ... was not found` severe
     errors.

## 9. How to test each of the 8 spawners

1. As an OP or test account, open `/shop`.
2. Confirm the existing End/Nether/Gear/Food/Shard buttons still work
   exactly as before.
3. Click the new **Smart Spawners** item (slot 16, spawner-block icon).
4. For each of the 8 icons:
   - Note your balance, click it.
   - Confirm the correct amount was deducted (matches `config.yml`).
   - Confirm you received a spawner item, and that placing it down
     actually spawns the right mob (this proves it's a real,
     SmartSpawner-recognized spawner, not a cosmetic reskin).
   - Try clicking with an empty wallet → should refuse and refund nothing.
   - Fill your inventory completely, then try buying → should refuse with
     the inventory-full message and **not** charge you.
5. Click **Back** → should return to the normal DonutShop main menu.
6. Rapidly double-click a spawner icon → should only ever charge/give you
   once (the purchase guard blocks the second click while the first is
   still processing).

## 10. How to remove the addon if something goes wrong

1. `stop` the server.
2. Delete `plugins/NexCoreSmartSpawners-1.0.0.jar`.
3. Optionally delete `plugins/NexCoreSmartSpawners/` (its config folder).
4. Start the server back up. DonutShop/CredosShop is completely untouched
   and will work exactly as it did before this plugin ever existed.
