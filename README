# swTrophyVault

SpeakWest hub **Trophy Vault** for Paper: one Citizens NPC, chest GUI, per-player trophy collection, hub checkout/return rules.

Unit lesson plugins (Star Academy and others) integrate through a small Java API — they register trophies and call `earn` on completion. Vault owns storage and hub UX.

## Docs

- **[API for unit developers](./swtrophyvault-api-for-unit-devs.md)** — soft-depend, `registerTrophy`, `earn`, events  
- **[Product TZ (RU)](./speakwest-trophy-vault-tz.md)** — full behaviour, MVP slice, deposit vs return copy  

## Server ops (short)

1. Drop `swTrophyVault-*.jar` into `plugins/`.  
2. Soft-depend / install Citizens; place keeper NPC in hub.  
3. `/swtrophy set keeper` while standing on the NPC.  
4. Edit `plugins/swTrophyVault/trophies.yml` if needed.  
5. Players: open NPC → Deposit / Take / Return (EN UI).

## For unit authors

```text
softdepend: [swTrophyVault]
→ SwTrophyVaultAPI.registerTrophy(...)
→ SwTrophyVaultAPI.earn(player, "your_trophy_id")
```

See the API doc for full examples.

## License / branding

SpeakWest. UI strings EN only.
