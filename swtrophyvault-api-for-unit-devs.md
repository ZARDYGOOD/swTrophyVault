# swTrophyVault API — for unit plugin developers (EN)

**Plugin:** `swTrophyVault` · **Version:** 0.1.0 · **Package:** `academy.speakwest.trophyvault.api`

Vault owns persistence, hub checkout rules, and the keeper GUI. Unit plugins only register trophies and call `earn` when a student finishes.

---

## 1. Soft-depend

In your unit `plugin.yml`:

```yaml
softdepend: [swTrophyVault]
```

Do **not** hard-depend unless your unit cannot run without Vault.

---

## 2. On enable — register

```java
import academy.speakwest.trophyvault.api.SwTrophyVaultAPI;
import academy.speakwest.trophyvault.api.TrophySpec;
import org.bukkit.Bukkit;
import org.bukkit.Material;

SwTrophyVaultAPI api = Bukkit.getServicesManager().load(SwTrophyVaultAPI.class);
if (api == null) {
    getLogger().warning("swTrophyVault not present — trophies will not sync.");
    return;
}

api.registerTrophy(TrophySpec.builder("star_academy_u1")
        .displayName("Star Trophy")
        .unitLabel("Star Academy · Unit 1")
        .sourcePlugin("StarAcademy")
        .icon(Material.NETHER_STAR)
        .lore("Star Academy", "Unit 1")
        .cardBlurb("You finished Star Academy Unit 1.")
        .build());
```

**Duplicate `id`:** first registration wins. `trophies.yml` loads before runtime register on Vault enable, so YAML wins over a late `registerTrophy` for the same id.

---

## 3. On unit complete — earn

```java
import academy.speakwest.trophyvault.api.EarnResult;

EarnResult result = api.earn(player, "star_academy_u1");
// EARNED | ALREADY_OWNED | UNKNOWN_TROPHY | INVALID_PLAYER
```

Optionally still give a display item in the unit world. Prefer stamping PDC later so Vault can recognize checkout items; until then Vault can soft-match by material + display name + lore (see `trophies.yml` `match-*` fields).

---

## 4. Queries

```java
boolean owned = api.has(player, "star_academy_u1");
boolean out = api.isCheckedOut(player, "star_academy_u1");
```

---

## 5. Events (optional)

| Event | When |
| --- | --- |
| `TrophyRegisterEvent` | After a successful runtime register |
| `TrophyEarnEvent` | After the player's **first** earn |
| `TrophyCheckoutEvent` | After Take from Vault |
| `TrophyReturnEvent` | After Return or first Deposit (`firstDeposit()`) |

All live under `academy.speakwest.trophyvault.api.event`.

---

## 6. Star Academy soft-hook (follow-up)

**Not in this JAR.** When ready, patch StarAcademy so trophy give calls:

`api.earn(player, "star_academy_u1")`

and stamps vault PDC (or keep deposit-only soft-match on name `Star Trophy` + lore `Star Academy` until that patch lands).

---

## 7. Permissions (player-facing)

| Node | Default | Use |
| --- | --- | --- |
| `swtrophy.use` | true | Open vault / return |
| `swtrophy.teacher` | op | `/swtrophy peek` |
| `swtrophy.admin` | op | set keeper, reload, grant |
| `swtrophy.bypass` | op | Leave hub with trophy |

---

*Contract is semver-stable inside `…trophyvault.api`. Thin `swTrophyVault-api` publish jar comes later when external authors need it.*
