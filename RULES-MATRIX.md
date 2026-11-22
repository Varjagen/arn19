# 2014 D&D 5e Rules Traceability Matrix

**Audit date:** v9.07 · **Suite:** 68 test files, 1499 tests, 0 failures ·
**Gates:** build freshness, undefined identifiers, spell schema, 17/17 integration - all pass.

Every row records: what is **implemented**, the **source module** (function or
table in `app.js`), the **automated tests**, what remains **manual**, the
**known limitation**, and how a **GM override** works.

Throughout, "GM override" has a consistent meaning: the GM may set the resulting
field directly on the entity, and an explicit value always beats a derived one.
That is a deliberate design rule, not a per-feature accident.

---

## 1. Ability checks

| | |
| --- | --- |
| **Implemented** | `d20 + ability modifier + proficiency / expertise / half proficiency + item bonus`. Advantage and disadvantage combine and cancel. Sight- and hearing-dependent checks auto-fail for Blinded / Deafened. Reliable Talent floors the die at 10 for proficient checks. |
| **Source** | `resolveCheckWithConditions` (L1868), `skillCheckMod`, `abilityCheckMod`, `resolveD20` (L19238) |
| **Tests** | `d20-service`, `proficiencies`, `backgrounds`, `levelup` (Reliable Talent), `conditions-matrix` (auto-fail) |
| **Manual** | Whether a given check *requires* sight or hearing - the caller passes `requiresSight` / `requiresHearing`. There is no per-skill sense table. Anything the rules do not decide, a player asks for with **Ask the GM** (top bar) since v26.61: the GM answers Yes or No on the request card and the answer is posted to the chat (CONTRACT 11ld). A player asks to make a check with **Request check** in the top bar, which had no reachable button before v26.61. |
| **Limitation** | Group checks and passive-vs-active contests are supported, but "working together" (help granting advantage) is a generic modifier, not a modelled action. |
| **GM override** | `entity.skillProfs[skill]` sets any level directly; `skillBonuses[skill]` on a stat block replaces the whole computation. |

## 2. Saving throws

| | |
| --- | --- |
| **Implemented** | Proficiency, magic-item bonuses (Ring of Protection `+1` to all), condition-driven auto-failure (STR/DEX while Paralyzed, Petrified, Stunned, Unconscious), condition disadvantage (Restrained on DEX, exhaustion 3 on all), advantage (Danger Sense on DEX, Magic Resistance vs spells), and Legendary Resistance converting a failure to a success. |
| **Source** | `resolveSaveWithConditions` (L1842), `savingThrowMod`, `useLegendaryResistance` |
| **Tests** | `conditions-matrix`, `exhaustion`, `armor`, `magic-items`, `monsters`, `cross-system`, `save-auto-fail`, `save-paths-agree` |
| **Manual** | Whether an effect is *magical* - the caller passes `opts.magical`, which gates Magic Resistance. |
| **Limitation** | **Aura of Protection** (Paladin 6) applies since v26.5: the paladin's Charisma modifier, minimum +1, to its own saves and those of allies within 10 ft (30 ft from 18th level), measured from the tokens at the moment of the save, only while the paladin is conscious, and not stacking with a second paladin's (CONTRACT 11iz). **Aura of Courage** (Paladin 10) since v26.15: allies in the same range cannot be frightened, and an ally already frightened has the condition suppressed while it stays in range (CONTRACT 11jj). Allies are decided by `sameSide`, so a monster ally needs the same `faction`. A character sheet's printed save totals do not include it, because it depends on where the creature stands. **Every save goes through `resolveSaveWithConditions`** since v26.1, including the two paths that roll most of them in play. The attack cinematic sends only the d20 faces and any situational mode the GM chose; the host recomputes the total and verdict in `ATTACK_UPDATE`, so a Ring of Protection, Bless and Bane, Magic Resistance, condition modes and automatic failure all apply, and nothing a client claims about the modifier, the total or success survives. A lingering zone's tick calls the same resolver. Before v26.1 the cinematic ignored the ring, Bless and Magic Resistance, the zone tick ignored the ring and Magic Resistance, and a player could claim a +20 save modifier through the host's handler. A save counts as against magic when a spell asks for it, repeat saves to end a spell's effect included. **The GM's mode buttons on the cinematic are situational**: they add advantage or disadvantage on top of the rules, which are shown beside them; to overrule a rule, the GM removes the condition or modifier behind it. **Legendary Resistance** converts a failed save in the cinematic, automatic failures included, from a button the GM is offered on the failure (v26.0). Since v26.4 every other save a creature with uses left fails - a zone's tick, a repeat save, a concentration check, an environmental hazard, the quick pick-a-target cast - waits for the GM's ruling before any of the failure is applied, and the hazard screen offers the same button on its rolled saves (CONTRACT 11it, 11iu, 11iv, 11iy). A save against a Stunning Strike waits on the same ruling since v26.47 (CONTRACT 11kp). |
| **GM override** | `saveBonuses[ability]` replaces the derived total outright. |

## 3. Attacks

| | |
| --- | --- |
| **Implemented** | Distance between creatures is measured edge to edge, by each creature's footprint - a table decision confirmed in v26.72: two Medium creatures with a 5 ft gap reach each other with a 5 ft weapon or a touch spell, a 10 ft gap does not (CONTRACT 11lo). Prepare/commit transaction with ability selection, proficiency, range and reach, cover (from walls and from intervening creatures, through `coverForTargets` - which the target picker, the foe roster and area spells also use since v25.8: before it those three measured an empty map and reported no cover anywhere, so a line-of-effect spell targeted through stone and a Fireball reached round corners; CONTRACT 11is), ranged-in-melee disadvantage, condition advantage from both sides, Prone's range-dependent rule, auto-crit within 5 ft of a Paralyzed/Unconscious target, Sneak Attack eligibility, and Multiattack. |
| **Source** | `prepareWeaponAttack` (L4946), `commitAction`, `conditionRollContext`, `sneakAttackFor` |
| **Tests** | `weapon-attacks`, `targeting-cover`, `cover`, `cover-blocks-effects`, `conditions-matrix`, `levelup`, `monsters`, `cross-system` |
| **Manual** | Improvised weapons and called shots. Underwater attack restrictions are computed (`underwaterAttackRule`) but the attack path does not consult them. |
| **Correction (v26.17)** | **Weapon damage dice were never rolled on the attack flow**: every hit dealt only its modifier (a longsword hit for 2), crits doubled nothing, and Sneak Attack was never added. Fixed in `buildWeaponAttackCinematic` (CONTRACT 11jl). |
| **Correction (v26.42)** | **A weapon from the sheet attacked with the wrong numbers**: the attack path read only `damage`/`kind`/`range`, which Armoury, hand-made and built-in creature weapons do not carry, so a +5 1d8 + 3 longsword hit at +3 for 1d4 + 3 bludgeoning, a bow could not shoot past 5 ft, and a second damage part was dropped. A printed attack is now resolved as written (`weaponAttackView`), with Rage, Archery, Dueling and Great Weapon Fighting on top; Armoury weapons keep their properties (CONTRACT 11kk). Since v26.48 an Armoury weapon's numbers follow the character's proficiency and ability scores until edited by hand (CONTRACT 11kq); a hand-made printed attack's numbers are as typed; since v26.44 the attack flow offers the two-handed and thrown grips, checked by the host (CONTRACT 11km). |
| **Limitation** | Two-weapon fighting and the ready action are modelled in the economy but not as distinct attack transactions. |
| **GM override** | `attacksPerAction` on the entity beats both Multiattack and Extra Attack; `advMode` can be forced on any request. |

## 4. Damage and healing

| | |
| --- | --- |
| **Implemented** | One pipeline for every source. Resistance halves (rounding down), vulnerability doubles, immunity negates, and resistance plus vulnerability cancel. Massive damage causes instant death. Healing cannot exceed the maximum and cannot revive the dead, **adds the caster's spellcasting modifier once per cast**, and **does not reach undead or constructs** unless the spell says it does. Critical hits double dice, not modifiers. |
| **Source** | `applyHpTransaction` (L5881), `autoMitigationForType`, `damageModsOf`, `applyEnvironmentalDamage` (L4807), `healingSpellAmount` and `healingExcluded` (via `buildSpellTargetAttack` and `rollSpellOutcome`) |
| **Tests** | `hp-pipeline`, `hp-invariants`, `mitigation`, `healing-spells`, `quick-heal`, `environment`, `cross-system` |
| **Manual** | Damage type for improvised sources - the caller supplies it. |
| **Limitation** | **`applyHpTransaction` expects an already-mitigated amount.** The attack path resolves resistance upstream; `applyEnvironmentalDamage` resolves it itself; since v26.19 lingering zones and ongoing ticks go through `mitigateDamage` (before it, a fire-resistant creature took a Wall of Fire in full, CONTRACT 11jn). Passing a raw number directly to `ENTITY_HP_ADJUST` still ignores resistance, which is what a GM's manual adjustment wants. |
| **GM override** | `mitigation` may be forced per transaction to `'normal'`, `'resist'`, `'immune'` or `'weak'`. |

> **Row correction (v25.2).** The healing clauses above are new to the row
> because they were new to the game. `healingSpellAmount` and `healingExcluded`
> were called only from a reducer case nothing dispatched, and a real cast took a
> third inline path that added no modifier and checked no creature type - so
> **Cure Wounds from a +4 cleric healed 1d8 rather than 1d8+4**, and undead were
> healed normally (CONTRACT 11im).
>
> **Row correction (v26.3).** v25.2 fixed the full cast path and missed a
> fourth: the quick pick-a-target cast (`rollSpellOutcome`, used by the DM's
> sheet and radial casts and by the host's handler for a player's
> `cast_spell`). It still rolled bare dice with no caster and no creature-type
> check: Cure Wounds from a WIS 18 cleric healed 1..8, and an Undead was healed.
> It now takes the caster and calls the same two functions (CONTRACT 11ix).

## 5. Temporary hit points

| | |
| --- | --- |
| **Implemented** | Absorbed before hit points; they do not stack - a new grant only replaces an existing pool if it is larger. Not affected by healing. |
| **Source** | `grantTempHp` (L6016), `applyHpTransaction` |
| **Tests** | `temp-hp`, `cross-system` |
| **Manual** | None. |
| **Limitation** | Temporary hit points have no duration; they persist until spent or cleared. |
| **GM override** | `grantTempHp(entity, n, 'replace' \| 'keep')` forces either outcome. |

## 6. Death and dying

| | |
| --- | --- |
| **Implemented** | Dropping to 0 knocks a creature unconscious; death saves at three successes / three failures; a natural 20 restores 1 hit point; a natural 1 counts double; damage while at 0 causes an automatic failure and a critical hit causes two; stabilising; massive damage instant death; exhaustion 6. |
| **Source** | `applyHpTransaction`, `MEDICINE_STABILIZE` (dispatched from the token context menu), `usesDeathSavesFor` (L4326 region), `setExhaustionLevel` |
| **Tests** | `death-saves`, `stabilize-nonlethal`, `hp-invariants`, `exhaustion` |
| **Manual** | Who counts as a "named NPC" for death-save eligibility. Stabilising: select the medic, right-click the dying creature. |
| **Limitation** | None known. A dying creature rolls its death save automatically at the start of its turn (`INIT_ADVANCE`, since v8.52) and the roll is logged; the sheet's button rolls one outside the turn order. (This row said otherwise until v26.73.) |

> **Row correction (v24.8).** This row listed stabilising under **Implemented**
> and cited `MEDICINE_STABILIZE` from the day it was written, and nothing
> dispatched that case, so **no creature could be stabilised by anyone**. The
> same overclaim as row 7's concentration entry: a row citing code that cannot
> be reached. The door is built and the claim is now true. The spend also went
> through the shared action transaction, which means it is skipped out of
> combat, like every other action (CONTRACT 11ii).
| **GM override** | `usesDeathSaves: true \| false` forces eligibility either way. |

## 7. Concentration

| | |
| --- | --- |
| **Implemented** | One concentration effect at a time; a new cast ends the old; a CON save on damage at DC 10 or half the damage, whichever is higher; incapacitation and death end it. |
| **Source** | `endConcentration`, the `concentratingOn` block in `ENTITY_HP_ADJUST` (reads `result.concentrationCheck` off the transaction), `resolveSaveWithConditions`, `conditionIsIncapacitating` |
| **Tests** | `concentration-damage`, `concentration-library`, `control-spells`, `exhaustion` |
| **Manual** | Concentration on non-spell effects. |
| **Limitation** | None known. The save is AUTO-ROLLED through `resolveSaveWithConditions` at DC max(10, half the damage), a failure calls `endConcentration` and logs the roll, and each damage packet is its own `ENTITY_HP_ADJUST` so each triggers its own check. (This cell previously said the save was prompted, describing a version that no longer exists.) |
| **GM override** | `ENTITY_PATCH` clears `concentratingOn` directly. |

## 8. Conditions

| | |
| --- | --- |
| **Implemented** | All **14** core conditions as data in `CONDITION_RULES`, plus the terminal states Dead and Broken. Every clause - advantage, auto-failed saves, speed, action economy, crits, visibility, dropped items, falling prone, resistance, condition immunity - is derived, never hardcoded. Implications resolve transitively (Unconscious → Incapacitated + Prone). |
| **Source** | `CONDITION_RULES` (L446), `conditionProfile` (L1448), `conditionRollContext` |
| **Tests** | `conditions-matrix` (109 tests, one per clause, with a completeness check that fails the build if a table key is untested) |
| **Manual** | Nothing *inflicts* conditions automatically outside spells and the hazard path. |
| **Limitation** | `conditionAutoEnds` covers only Grappled's two documented end conditions. |
| **GM override** | `ENTITY_TOGGLE_CONDITION`; `conditionImmunities` blocks application entirely. |

## 9. Movement

| | |
| --- | --- |
| **Implemented** | Speed by mode, difficult terrain, crawling while prone, Dash, squeezing, condition speed overrides (Grappled/Restrained to 0), exhaustion halving and zeroing, heavy-armour and encumbrance flat penalties, class speed bonuses, opportunity attacks, forced movement and teleportation exempt from Frightened's approach restriction. Painted difficult and flood zones cost double since v26.74 (CONTRACT 11lq), and Grease, Entangle and Web leave a difficult zone where they are cast since v26.75 (CONTRACT 11lr). A creature restrained by Web or Entangle can spend its action on the Strength check to break free since v26.76 (CONTRACT 11ls). |
| **Source** | `moveBudget` (L1164), `prepareMove` (L6295), `conditionSpeedProfile` |
| **Tests** | `movement`, `movement-penalties`, `movement-transaction`, `positioning`, `jumping`, `conditions-matrix`, `environment`, `cross-system` |
| **Manual** | Underwater movement cost (`underwaterMovementFor` is computed but not consulted by `prepareMove`). |
| **Limitation** | Ordering is fixed: flat penalties apply to base speed, then multipliers. This is correct for the cases tested but is a convention, not a rule the books state. |
| **GM override** | Forced and teleport movement kinds bypass willing-movement restrictions; `isDM` bypasses budget checks. |

## 9b. Teleportation

| | |
| --- | --- |
| **Implemented** | A teleport is a movement with its own kind: no movement budget charged, no clamping to the creature's speed, no opportunity attack provoked, exempt from Frightened's approach restriction, and the arrival fires any scheduled area the creature lands in. A rider on a controlled mount is carried. A spell that teleports declares it in a `teleport` block (`rangeFt`, `self`, `requiresSight`, `requiresUnoccupied`) and is routed to the shared resolver by `customResolution: "teleport"`. The destination is validated against the spell's own reach, the caster's line of sight and line of effect, and the required empty space, in `prepareSpellCast` - so an illegal destination refuses the cast and spends no slot and no action. Range is measured from the CASTER even when the spell moves somebody else. |
| **Source** | `prepareMove` `kind: 'teleport'`, `teleportDestinationLegal`, `teleportMoverId`, `spellReachFt`, `CUSTOM_RESOLVERS.teleport`, `areaOriginLegal` |
| **Tests** | `teleport-spells`, `positioning`, `movement-transaction`, `opportunity-attack`, `effect-scheduler`, `scheduler-integration` |
| **Manual** | Which square to appear in is always the player's choice; nothing picks one automatically. A teleport that would cross planes, or one whose destination the GM rules unseen for a reason no wall models, is a GM call. |
| **Limitation** | **Misty Step is the only teleport in the library.** Thunder Step, Dimension Door and Shadow Step are not modelled, so they carry no `teleport` block and casting them teleports nobody; Shadow Step exists only as a subclass feature's description text. The resolver is spell-agnostic and needs only the data block, so adding one is a `game-data.js` change. A teleport of more than one creature (Dimension Door's willing passenger) has no model: `teleportMoverId` returns a single mover. Vertical destinations are ignored - a teleport does not change `elevation`. |
| **GM override** | A DM cast passes `isDM`, which skips ownership but NOT the destination rules, since those are the spell's text rather than a permission. To place a creature anywhere regardless, the DM drags the token, which goes through `MOVE_TOKEN` as a reposition. |

## 10. Resting

| | |
| --- | --- |
| **Implemented** | Short rest restores short-rest feature resources, Pact Magic slots and hit dice spending; long rest restores hit points, half the hit dice, all slots, all feature uses, magic-item charges, and reduces exhaustion by one **given food and drink**. |
| **Source** | `SHORT_REST` / `LONG_REST` reducers, `restoreFeatureUses`, `rechargeItems`, `exhaustionAfterLongRest` |
| **Tests** | `long-rest`, `rest-hitdice`, `hit-dice`, `exhaustion`, `timed-effects`, `progression` |
| **Manual** | Interrupting a rest. |
| **Limitation** | A creature at 0 hit points cannot benefit from a long rest, which is enforced - but partial rests are not modelled. Since v26.46 the one-per-24-hours limit is measured in game hours, and a rest nobody can take moves no time (CONTRACT 11ko). |
| **GM override** | `foodAndDrink: false` on the action suppresses exhaustion recovery. |

## 11. Action economy

| | |
| --- | --- |
| **Implemented** | Action, bonus action, reaction and free interaction per turn, tracked per creature; Extra Attack; Action Surge banks an extra action on the fighter's own turn, drawn by the next action, attack or action spell and lost at the turn's end (v26.21, CONTRACT 11jp); item activation spending the declared cost; reactions blocked while incapacitated or surprised. From the player's action drawer, Dash and Disengage take the action before the move (v26.7, CONTRACT 11jb), Cunning Action pays for Dash, Disengage or Hide with the bonus action, Patient Defense (Dodge) and Step of the Wind (Dash or Disengage, jump doubled) do it for 1 ki (v26.21, CONTRACT 11jp), the object interaction spends the free interaction rather than the action, and Stand up stands for half the speed when that much movement is left (v26.8, CONTRACT 11jc). Dodge imposes disadvantage on attacks the dodger can see and grants advantage on its DEX saves until the start of its next turn, and is lost while incapacitated or at speed 0 (v26.10, CONTRACT 11je). |
| **Source** | `prepareAction` / `commitAction`, `normalizeEconomy`, `ACTION_KINDS`, `conditionEconomyProfile`, `workflowForOption`, `standUpCheck` |
| **Tests** | `action-economy`, `action-service`, `live-combat`, `timed-effects`, `cross-system`, `dash-disengage-cunning`, `minor-actions`, `help-action`, `dodge-action`, `hiding`, `ready-lifecycle` |
| **Manual** | Readied actions fire by GM decision, from the token menu; one not fired lapses at the start of the readier's next turn (v26.14, CONTRACT 11ji). |
| **Limitation** | Legendary actions are tracked separately from the normal economy and reset on `INIT_ADVANCE`, not on a strict "end of another creature's turn" trigger. **Hide** (since v26.11) rolls Stealth, gives a hidden attacker advantage and attacks on it disadvantage, keeps it off players' screens, and ends when it attacks, casts aloud, or a hostile can see it clearly or meets its Stealth with passive Perception. Since v26.12 it is per observer: each hostile creature spots it separately, a player is sent it only once their own creature has spotted it, and it is re-checked when light, walls or the creatures present change, not only on moves. **Search** with Perception rolls against the Stealth of each hidden creature the searcher can see and spots those it meets (v26.13). Hearing is not modelled (CONTRACT 11jf, 11jg, 11jh). |
| **GM override** | `ECONOMY_TOGGLE` sets any slot; `force: true` bypasses the check. |

## 12. Weapons

| | |
| --- | --- |
| **Implemented** | Proficiency by category, finesse, versatile, thrown, two-handed, loading, reach, ammunition, ranged range bands with long-range disadvantage, and magic bonuses. |
| **Source** | `weaponAttackAbility`, `weaponIsRanged`, `prepareWeaponAttack` |
| **Tests** | `weapon-attacks`, `range-buffer`, `armor` (proficiency penalties) |
| **Manual** | Improvised weapons; the underwater weapon list is computed but not enforced. |
| **Limitation** | Weapon mastery properties are not a 2014 concept and are absent by design. |
| **GM override** | A weapon's fields are freely editable, including an explicit `attackBonus`. |

## 13. Armour and AC

| | |
| --- | --- |
| **Implemented** | **7 mutually exclusive** base formulas - explicit/stat-block, worn, natural, Mage Armor, Barbarian and Monk Unarmored Defense, unarmoured. Exactly one applies; shields and magic items stack on top. Light/medium/heavy Dexterity rules, including heavy armour contributing exactly 0 (not a negative). Non-proficiency penalties without an equip gate. Heavy-armour Strength requirement with feature exemptions. |
| **Source** | `AC_FORMULAS` (L2291), `resolveArmorClass` (L2410), `armorProficiencyState`, `heavyArmorStrengthCheck` |
| **Tests** | `armor` (67 tests), `magic-items`, `progression`, `cross-system` |
| **Manual** | None. |
| **Limitation** | None known for the registry. Draconic Resilience is 13 + DEX with no armour (a shield adds) and +1 hit point maximum per sorcerer level since v26.73 (CONTRACT 11lp); before it the formula had no entry and fell through to unarmoured. |
| **GM override** | `acOverride` wins outright; `acFormula` forces a specific eligible formula. |

## 14. Spell slots

| | |
| --- | --- |
| **Implemented** | Full, half and third-caster tables; multiclass caster levels combine and read the full-caster table; a single-class caster uses its own table; subclass-granted casting (Eldritch Knight, Arcane Trickster) counts. |
| **Source** | `spellSlotsMaxFor` (L17941), `casterClassesOf`, `subclassCasterKind` |
| **Tests** | `caster-progression`, `slot-enforcement`, `multiclass` |
| **Manual** | None. |
| **Limitation** | Spells known and cantrips known are not tabulated per class per level; the sheet tracks them freely. |
| **GM override** | `spellSlotsMax` per level overrides the derived table entirely. |

## 15. Pact Magic

| | |
| --- | --- |
| **Implemented** | Kept entirely separate from standard slots; Warlock levels contribute nothing to the multiclass caster level; recharges on a short rest; Mystic Arcanum tracked as long-rest resources. |
| **Source** | `pactSlots` on the entity, `spellSlotsMaxFor` (Warlock exclusion), `SHORT_REST` |
| **Tests** | `pact-magic`, `multiclass` |
| **Manual** | Choosing which Arcanum spell is bound. |
| **Limitation** | Agonizing Blast, Repelling Blast, Armor of Shadows, Devil's Sight and Thirsting Blade apply (CONTRACT 11li, 11lj); Eldritch Sight and Mask of Many Faces are the table's. (This row said invocations had no effect until v26.73.) |
| **GM override** | `pactSlots` is directly editable. |

## 16. Spellcasting restrictions

| | |
| --- | --- |
| **Implemented** | Verbal, somatic and material components; costly and consumed materials; ritual eligibility; preparation and known-spell lists; class spell-list eligibility; slot availability; **armour you are not proficient with prevents casting entirely**; one-leveled-spell-per-turn with a bonus action. Every one of these is enforced by `prepareSpellCast` AND reported by the player's cast panel, which asks the same functions - `spellRequirementsCheck` for the requirements, `canCastByTime` for the casting time and the action economy, `castSlotSources` for the slot. Until v25.7 the panel asked the first of those with the wrong signature and so reported no requirement failure ever, which also made the cast intent handler accept casts the adjudicator refused (CONTRACT 11ir). |
| **Source** | `spellRequirementsCheck`, `canCastByTime`, `castReview` (the panel's and the intent handler's shared review), `armorBlocksSpellcasting`, `castingTimeEconomyPatch` |
| **Tests** | `spell-requirements`, `casting-time`, `spell-cast-transaction`, `armor`, `cast-review-agreement` |
| **Manual** | Whether a creature is silenced or bound for component purposes. |
| **Limitation** | Ritual casting does not advance a clock by 10 minutes. **A monster stat block cannot cast a library spell from `spellbook`**: a Monster has no class, so `spellListEligible` refuses any spell that names one, and a stat block carries no component pouch, so `materialCheck` refuses any spell with a material component. That has always been true of `prepareSpellCast`; before v25.7 the panel did not say so, and the cast failed silently at the reducer. Monster spellcasting has its own correct path (`innateSpellsOf` / `MONSTER_ABILITY`, given a door in v24.9), so the `spellbook` field on a monster is a shape the model does not use. |
| **GM override** | Any individual check can be bypassed with `isDM` in `prepareSpellCast`. The player's review has no override by design: it is the player's own panel, and the DM's cast path goes through `prepareSpellCast` directly. |

## 16b. Spell target limits

| | |
| --- | --- |
| **Implemented** | A spell resolves against at most its own `targets`, scaled by slot level (Hold Person 1 at 2nd, 3 at 4th; Magic Missile 3 at 1st, 5 at 3rd). The cap applies to targets that still exist, so a creature that left the map does not consume a slot of it, and anything trimmed is reported in the log. |
| **Source** | `resolveTargetedSpellCast`, `scaleSpell` (the `targets` field) |
| **Tests** | `combat-resolution`, `healing-spells` |
| **Manual** | Which targets to drop when too many are aimed at: the first legal ones are kept. |
| **Limitation** | Before v25.3 there was **no cap at all** on the live path - one slot resolved against as many creatures as were aimed at, so a single-target Hold Person paralysed three. The only enforcement in the codebase was in a reducer case nothing dispatched (CONTRACT 11in). |
| **GM override** | A spell's `targets` is editable per cast. |

## 17. Spell scaling

| | |
| --- | --- |
| **Implemented** | Cantrips scale on **total character level** (5/11/17); leveled spells scale by slot level for dice, targets, duration and area; the cast level propagates through the whole transaction. |
| **Source** | `scaleSpell` (L18793), `cantripDiceMultiplier`, `castLevelFor` |
| **Tests** | `spell-scaling`, `cast-level-propagation`, `multiclass` (cantrips on total level) |
| **Manual** | None. |
| **Limitation** | Scaling is expressed per spell in the library; a spell with an unusual progression needs an explicit entry. |
| **GM override** | The cast level is chosen per cast. |

## 17b. Roll modifiers from spells (Bless, Bane, Faerie Fire, Guiding Bolt)

| | |
| --- | --- |
| **Implemented** | A spell's declarative `modifiers` block becomes real roll-modifier records: advantage, disadvantage, a flat bonus, or dice, each carrying its source, its target, its expiry and its concentration linkage. They are collected on live attacks and saving throws, rolled fresh, and attributed by source in the roll breakdown. The GM can see each one in Active effects and end it. Records arrive when the outcome is known: **immediately** for an unconditional spell (Bless), **on a failed save** (Bane, Faerie Fire), **on a hit** (Guiding Bolt). |
| **Source** | `spellModifierSpecs`, `SPELL_APPLY_MODIFIERS` (dispatched from `resolveTargetedSpellCast` for direct targets and from the cinematic's apply step for gated ones), `collectRollModifiers`, `applyRollModifiers`, `MODIFIER_REMOVE`, `rollModifierLabel` |
| **Tests** | `roll-modifiers`, `fake-timers` (the gated half, driven through the cinematic) |
| **Manual** | Whether an opt-in modifier (Guidance, Help with a check) is spent on a given check is the player's choice, offered rather than applied. Once taken it is spent: the player's roll reports it and the host removes it (`ROLL_MODIFIERS_SPEND`) - before v26.9 nothing did, so one Guidance applied to every check after it, and a modifier's advantage or disadvantage never reached a player's check at all (CONTRACT 11jd). The **Help** action uses the same records since v26.9: advantage on the ally's first attack against an enemy within 5 ft of the helper, or an offered advantage on one ability check, until the start of the helper's next turn. |
| **Limitation** | All four library modifier spells now work. Outcome-gated ones are instantiated at the apply step, so **a modifier only exists once the GM applies the result** - abandoning a cinematic leaves none, which is correct but means a half-resolved cast has no partial effect. Before v24.5 the engine had no producer at all and all four did nothing (CONTRACT 11if); v24.6 added the save- and hit-gated half (CONTRACT 11ig). |
| **GM override** | End any modifier from the target's Active effects list; dropping concentration ends every record the spell owns. |

## 18. Corrected spells

| | |
| --- | --- |
| **Implemented** | **56** library spells validated against a schema (Fog Cloud, Darkness, Grease and Entangle added in v26.75, CONTRACT 11lr). Corrections carried out in this project include **Moonbeam** (cylinder not sphere; CON save for half; triggers on entry *or* start of turn - see below), Mage Armor (a formula, not a bonus), and the five named magic items. |
| **Source** | `STANDARD_SPELLS` in `game-data.js`, `tools/validate-spells.js` |
| **Tests** | `spell-schema`, `named-spells`, `control-spells`, `remaining-spells`, `lingering-zones`, `healing-spells`, `sleep-spell` |
| **Manual** | Spells outside the library are free text. The GM chooses which targets to drop when more are aimed at than the spell allows: the first legal ones are kept and the rest are reported. **Sleep's one-minute duration is not enforced** - see the limitation below. |
| **Limitation** | 56 of the SRD's ~319 spells are modelled. **Sleep never expires on its own.** A slept creature stays `Unconscious` until it takes damage (`ENTITY_HP_ADJUST` clears the `magicalSleep` marker) or someone shakes it awake (`WAKE_CREATURE`, in the token menu since v25.0 - the v24.1 version of this row named the action before anything offered it, which is the overclaim the new check now catches). A superseded reducer case, deleted in v25.5, wrote an `expiresAtRound` that no reader ever consumed, so the duration was never enforced by either path; v24.1 declined to port the appearance of a duration without its enforcement and pinned the gap with tests instead, and v25.5 declined again when the case was deleted, for the same reason (CONTRACT 11ib, 11ip). |
| **GM override** | Any spell's fields are editable per cast. The DM ends Sleep early by right-clicking the sleeper and choosing **Shake awake** (`WAKE_CREATURE`), which costs a selected waker's action or is a GM ruling if none is selected. |

> **Audit correction.** `tests/lingering-zones.test.js` previously asserted
> *"Moonbeam: damages only at the end of a turn (not entry, not start)"*. That is
> **not** the 2014 rule - Moonbeam triggers when a creature first enters the area
> on a turn **or starts its turn there** (PHB p.261) - and it directly
> contradicted `tests/control-spells.test.js`, which asserted the correct
> behaviour. The incorrect test has been rewritten to describe the generic
> end-of-turn zone shape it actually exercised, the spell's description text in
> `game-data.js` has been corrected (it also claimed an end-of-turn trigger), and
> a new test asserts the correct rule so the contradiction cannot return.

## 19. Character progression

| | |
| --- | --- |
| **Implemented** | **All 12 classes** have level tables: features, subclass level and label, ASI levels, resources with recharge, and per-feature choices. Features that replace others do not stack. Proficiency bonus from total level. Feats with prerequisites. Backgrounds (**20**) granting skills, tools, languages, equipment and features, with duplicate detection and replacement. Expertise and half proficiency. |
| **Source** | `CLASS_PROGRESSION` (L3577 region), `resolveProgression`, `BACKGROUND_DEFS`, `resolveProficiencies`, `FEATS` |
| **Tests** | `progression` (82), `backgrounds` (62), `levelup` (31) |
| **Manual** | Spells known/prepared per class level. |
| **Limitation** | The eight classes added later cover everything **except spellcasting progression**. Eldritch Blast fires 1 to 4 beams of 1d10, each its own attack, and a warlock learns invocations at 2, 5, 7, 9, 12, 15 and 18, since v26.66; Agonizing Blast and Repelling Blast apply (CONTRACT 11li); Armor of Shadows, Devil's Sight and Thirsting Blade since v26.67 (CONTRACT 11lj); Eldritch Sight and Mask of Many Faces are the table's. A druid takes a beast's shape with Wild Shape, within its CR and speed limits (Circle of the Moon: as a bonus action and to higher CR), keeping its mind and features; it ends by a bonus action, at 0 beast hit points with the excess carried over, on falling unconscious, or after half the druid level in hours, and blocks spellcasting until Beast Spells, since v26.43 (CONTRACT 11kl); a Moon druid heals in beast form with a spell slot since v26.51 (CONTRACT 11kt). Lay on Hands heals a touched creature for any amount from its pool, or cures a poison for 5, and Bardic Inspiration's uses equal the Charisma modifier, since v26.20 (CONTRACT 11jo). A map with no light set follows the time of day and "Always dark" for the rules too (dusk dim, night darkness) since v26.37 (CONTRACT 11kf). Short and long rests advance the world clock (1 h / 8 h), and timed spell modifiers end on it, since v26.33 (CONTRACT 11kb). Minute-long feature effects also end by world time outside combat since v26.32 (CONTRACT 11ka). A paladin's Channel Divinity gives Sacred Weapon (Devotion) or Vow of Enmity (Vengeance) for a minute since v26.31 (CONTRACT 11jz). Turn the Unholy (Devotion) and Abjure Enemy (Vengeance) since v26.49, each Channel Divinity option its own drawer entry (CONTRACT 11kr). The Life Domain's Preserve Life and the War Domain's Guided Strike since v26.50 (CONTRACT 11ks). War God's Blessing (War Domain 6), a nearby cleric's reaction on another creature's attack roll, since v26.57 (CONTRACT 11kz). A cleric's Channel Divinity turns (or, from 5th level, destroys) undead within 30 ft since v26.30 (CONTRACT 11jy). A sorcerer converts slots and sorcery points both ways with Flexible Casting since v26.29 (CONTRACT 11jx). Metamagic since v26.52: Quickened, Subtle, Distant and Twinned are applied by the cast itself (CONTRACT 11ku), Heightened and Careful through the spell's save since v26.53, for area spells too (CONTRACT 11kv); Extended doubles the spell's timers and Empowered rerolls low damage dice on the attack screen since v26.59 (CONTRACT 11lb). The DM casts with Metamagic from the target and aim banners since v26.60, and an area cast reaches its resolver as Metamagic changed it (CONTRACT 11lc). Indomitable rerolls a failed save on the attack screen since v26.28 (CONTRACT 11jw). Arcane Recovery and Natural Recovery restore chosen slots within the rule since v26.27 (CONTRACT 11jv). Using a class feature spends the action or bonus action it costs, and druids have Wild Shape (the table had lost it), since v26.26 (CONTRACT 11ju). Stunning Strike (Monk 5), Deflect Missiles (Monk 3) and Uncanny Dodge (Rogue 5) are offered on the attack as it resolves since v26.25, and a stun lasts until the end of the monk's next turn (CONTRACT 11jt). A missile deflected to 0 is caught and can be thrown back for 1 ki since v26.58 (CONTRACT 11la). Bardic Inspiration gives an ally a die it may add to a check, attack roll or save after seeing the d20, since v26.24 (CONTRACT 11js). Rage applies since v26.19: advantage on Strength checks and saves, Rage Damage, resistance to bludgeoning, piercing and slashing, no spellcasting, and it ends after a minute, on falling unconscious, or after a turn without attacking or taking damage (CONTRACT 11jn). Fighting Styles apply since v26.18 (Archery, Defense, Dueling, Great Weapon Fighting, Two-Weapon Fighting; Protection is called by the table) (CONTRACT 11jm). Divine Smite is chosen on a paladin's melee hit and spends a slot for its radiant dice, and Improved Divine Smite adds 1d8 radiant to every melee hit, since v26.23 (CONTRACT 11jr). Ki is spent on Patient Defense and Step of the Wind since v26.21 (CONTRACT 11jp). Flurry of Blows, the Martial Arts bonus strike and the two-weapon off-hand attack are bonus actions in the drawer and the attack flow since v26.22, and a monk strikes with Dexterity when it is better; since v26.45 the Martial Arts die replaces a monk weapon's die when larger (CONTRACT 11jq, 11kn). Cunning Action works since v26.7: Dash, Disengage or Hide as a bonus action from the drawer (CONTRACT 11jb). Evasion applies to every Dexterity save that halves damage since v26.6: the attack cinematic (and so every area spell), a lingering zone's tick, the hazard screen and the quick cast (CONTRACT 11ja). Aura of Protection applies since v26.5 (CONTRACT 11iz). |
| **GM override** | `extraAttacks`, `unarmoredDefense`, `halfProficiency` and `skillProfs` can all be set directly. |

## 20. Multiclassing

| | |
| --- | --- |
| **Implemented** | Independent class levels; total character level; prerequisites checked for the class being entered **and every class already held**; the reduced multiclass proficiency package; hit dice pooled by die size preserving spent dice; combined spell slots; Pact Magic separate; per-class spellcasting ability and preparation limits; ASIs per class; **Extra Attack does not stack**. |
| **Source** | `classEntriesOf`, `canMulticlassInto`, `resolveMulticlassProgression` (L3906), `hitDicePoolsForClasses` |
| **Tests** | `multiclass` (90) |
| **Manual** | None. |
| **Limitation** | Migration derives the `classes` array from the existing class and level, so no statistic changes - verified by test. |
| **GM override** | `addClassLevel(..., { force: true })` bypasses prerequisites. |

## 21. Monsters

| | |
| --- | --- |
| **Implemented** | Printed saving-throw and skill totals replacing derived ones; damage vulnerabilities, resistances, immunities; condition immunities; Magic Resistance; recharge abilities **rolled at the start of the turn with the roll reported in full**; Multiattack; legendary actions with per-option costs; legendary resistances; lair actions on an initiative count; regional effects; reactions; death-save eligibility; size, space and reach; innate and standard spellcasting; trait-carried modifiers. Fully editable in `StatblockEditor`. |
| **Source** | `monsterProfile`, `rollRechargeAbilities`, `spendLegendaryAction`, `useLegendaryResistance`, `restoreDailyMonsterUses`, `lairActionsDue`, `MONSTER_ABILITY` (dispatched from the token context menu, and for Legendary Resistance from the attack cinematic), `LEGENDARY_RESISTANCE_DECIDE` and `HAZARD_EVENT_LEGENDARY` for every other save |
| **Tests** | `monsters`, `legendary-resistance` |
| **Manual** | Legendary actions, recharge abilities and innate spells are *offered*, not auto-taken: right-click the monster. Lair actions fire on their initiative count. **Legendary Resistance is offered on any failed save, to the GM only** - on the attack cinematic, on the hazard screen, and for every other save in a prompt that holds the failure until the GM rules (`LegendaryResistancePrompt`). Whether to spend it is the GM's choice for the monster, so it is never spent automatically. |
| **Limitation** | **The printed-save-bonus test is not discriminating** - removing the short-circuit in `savingThrowMod` does not turn the suite red, though the behaviour is correct and verified by hand. Do not treat that one mechanic as regression-protected. **A held failure lands late.** While the GM decides, the rest of the turn carries on; the failure's damage and condition arrive when the ruling is made, not at the moment of the roll. A concentration check applies its damage at once and holds only the concentration. Per-day uses (Legendary Resistance and innate spells) come back **at dawn**, which needs the clock to be advanced (the Environment panel's travel tab), and on a long rest for a creature that takes one; the Long Rest button is offered only to player characters and familiars, so for a monster the dawn is the route (CONTRACT 11iu). |
| **GM override** | Every field is a control in the stat block editor. |

> **Row correction (v24.9).** The Manual line read "Legendary and lair actions
> are *offered*, not auto-taken", which read as though they were offered.
> Lair actions were; legendary actions, recharge abilities and innate spells
> were **not offered anywhere** - `MONSTER_ABILITY` had no dispatcher, so a
> monster could not use its own abilities. The turn boundary already refilled
> them, so the budget refreshed every round and nothing could draw on it, and
> the stat block editor let the GM define legendary actions they could never
> take. Three of the four modes now have a control (CONTRACT 11ij).

## 22. Vision

| | |
| --- | --- |
| **Implemented** | Three light levels and two obscurity states; normal vision, darkvision (darkness→dim, dim→bright, **colourless**), blindsight, tremorsense, truesight; invisibility; hiding. **Vision, line of sight, line of effect and knowledge of location are four separate functions.** Passive Perception with proficiency, expertise, half proficiency, feats, items and the **±5 advantage swing**. The fog renders bright, dim and darkvision distinctly, and desaturates darkvision areas. Since v26.68 fog zones and magical-darkness zones block sight (magical darkness blocks darkvision too; Devil's Sight, truesight and blindsight see through it), and every weapon attack takes disadvantage when the attacker cannot see its target and advantage when the target cannot see the attacker, for reasons of light, fog, darkness or walls (CONTRACT 11lk); spell attacks read the same rules, with conditions, Bless and Help, cover and ranged-in-melee disadvantage, since v26.69 (CONTRACT 11ll). A creature in fog or magical darkness that none of a player's creatures can see is not sent to that player since v26.70 (CONTRACT 11lm). The host checks a spell's chosen targets (range, sight, line of effect) and a charmed creature cannot target its charmer with an attack, a grapple, a shove or a harmful spell, since v26.71 (CONTRACT 11ln). Fog Cloud and Darkness leave their zone where they are cast, and it ends with the spell (by rounds, by the world clock, or when concentration ends) since v26.75 (CONTRACT 11lr). |
| **Source** | `lightLevelAt` (L4403), `canSeeCreature` (L4451), `knowsLocationOf`, `passivePerceptionOf` (L4544), `computePlayerVisionSources` |
| **Tests** | `lighting` (85), `security-filter`, `cross-system` |
| **Manual** | Whether a specific Perception check depends on sight. |
| **Limitation** | The renderer composites each **viewer's** sources; standalone `state.lights[mapId]` entries not attached to a token affect the rules but are not drawn. |
| **GM override** | `passivePerception` on a stat block replaces the base (but still takes the ±5 swing). |

**Security note.** `filterStateForPlayer` is the boundary that decides what each
client receives. It now honours the sense model, including sending an invisible
creature's token only to a viewer with truesight or blindsight reaching it.
Before v9.05 that token was transmitted to any client whose ordinary vision
radius covered it. Treat changes here as security-relevant. Since v26.77 the
snapshot is an allowlist: every top-level state key needs a rule in
`PLAYER_SNAPSHOT_RULES`, and a key without one is not sent (CONTRACT 11lt).
A new state field must be given a rule there before players receive it.
Since v26.78 a system log line that names a creature the party does not
know of is whispered to the GM, or, when it also names a known creature,
reaches players with that name replaced by "something unseen"; the attack on
screen is redacted the same way (CONTRACT 11lu).
Since v26.79 the GM's notes on a creature reach no player, its owner included,
and no player can write them; `playerDescription` is the shared field
(CONTRACT 11lv).
Since v26.81 a connection that has not been admitted (no `hello` yet, or held
for GM approval) can send only `hello` and is sent only the handshake's
answers; the host's action, intent and cancel handlers refuse it as well
(CONTRACT 11lx).

## 23. Environmental rules

| | |
| --- | --- |
| **Implemented** | Carrying capacity (STR × 15 × size) and push/drag/lift; variant encumbrance behind a campaign switch, reaching `moveBudget`; falling (1d6/10 ft, capped 20d6, lands prone); holding breath and suffocation timing; underwater movement and weapon rules; extreme heat and cold; forced march; food and water; travel pace. **Every hazard resolves through the universal d20 resolver and every point of damage through the central HP pipeline.** All **8** subsystems independently switchable. |
| **Source** | `CAMPAIGN_RULES_DEFAULT`, `resolveHazardSave` (L4795), `applyEnvironmentalDamage` (L4807), `encumbranceOf`, `advanceSuffocation` |
| **Tests** | `environment` (87), `cross-system` |
| **Manual** | **Triggering.** Nothing watches the clock to demand an hourly heat save; the GM applies hazards from the Environment tab. |
| **Limitation** | Since v26.74 a painted difficult zone, and a flood zone for a creature without a swimming speed, cost an extra foot per foot of a move that touches them, and a creature standing in a flood zone attacks under the underwater weapon rules (CONTRACT 11lq). The extra cost applies to the whole move that touches the zone; immersion does not yet grant fire resistance. |
| **GM override** | Every subsystem has an on/off switch; hazards accept an injected `d20`. |

## 24. Magic items

| | |
| --- | --- |
| **Implemented** | Equipped and attuned state as a single gate; attunement requirements and the **limit of three** (configurable); charges with dawn/dusk/short/long recharge; consumables; activation costing the declared action; six effect kinds - set-scores, AC, saves, advantage, disadvantage-on-others, vision. **5** named items with **11** entries including variants. |
| **Source** | `MAGIC_ITEMS` (L2593), `magicItemActive`, `canAttune`, `resolveArmorClass`, `magicItemSaveBonus` |
| **Tests** | `magic-items` (62), `item-effects`, `timed-effects`, `cross-system` |
| **Manual** | Identifying items; cursed items. |
| **Limitation** | Item durations record `expiresAtHour` but only expire when the GM advances the clock. |
| **GM override** | `ITEM_ATTUNE` respects the limit; `attunementLimit` on the entity changes it. |

---

## Uncovered critical paths found by this audit

The per-subsystem suites are thorough in isolation; the gaps were all at the
**interactions**. `tests/cross-system.test.js` (26 tests) was added to cover:

- exhaustion against long rest, Aid, hazard saves, layered speed modifiers and the proficiency bonus
- Petrified resistance applied to falling damage; Petrified suppressing Poisoned
- temporary hit points absorbing before hit points and refusing a lower grant
- non-proficient armour hitting STR/DEX saves but **not** a CON hazard save
- multiclass proficiency bonus reaching skills, saves and passive scores alike
- Multiattack overriding the class calculation
- a set-score item raising carrying capacity
- Pact Magic never leaking into standard slots
- Goggles of Night reaching `canSeeCreature`; Blinded + truesight; darkvision leaving Perception at disadvantage
- a live attack composing condition, armour and item modifiers simultaneously

## Confidence

Every mechanical claim above is backed by a passing test, and most subsystems
additionally have **negative controls** recorded in `tests/README.md` - an
injected regression that must turn the suite red. Five times during this project
a negative control revealed something a green suite had hidden; three of those
were weak tests rather than weak code.

The one place where confidence is **lower than the green suite suggests** is the
printed-save-bonus mechanic in §21, which is documented there and in
`tests/README.md` rather than left implied.
