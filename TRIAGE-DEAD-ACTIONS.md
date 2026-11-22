# Unreachable reducer actions

4 of 233 reducer cases are never dispatched.

A `case` with no dispatcher is a feature with an engine and no door. Most
appear in app.js exactly once - as the case label - and are referenced only
from tests, which means **the suite certifies them as working**.

Unreachable is not the same as missing: some are superseded by a path that
does the same job.

## Three lessons this table has already cost

**Check for a sibling path before choosing a bucket.** GRAPPLE_ATTEMPT and
SHOVE_ATTEMPT were bucketed "unfinished - engine, no door". The Actions
drawer reached both through CONTEST_RESOLVE. Building the recommended door
would have made a third path to grappling.

**Before deleting a superseded case, diff it against the survivor CLAUSE BY
CLAUSE, not test by test.** The deleted grapple and shove paths had four
advantages over the live one. Retargeting their tests found two, because
only two had tests: the live path also applied conditions as a TOGGLE (so a
second application removed them) and ignored a refused economy spend (so a
creature that had used its action could grapple for free). Both shipped, and
both were found by a reader, not by the port. A green suite after a
retarget means the clauses that had tests survived. It says nothing about
the ones that did not.

**The clauses that go missing are the ones that TELL THE PLAYER.** All five
superseded spell cases deleted in v25.5 had correct arithmetic on both
sides; what only the dead case had was the sentence in the log. A heal that
printed no amount, a Sleep that printed neither its rolled pool nor its
leftover, a drain that moved the caster's hit point bar in silence. No test
had caught any of them, because a test that checks the hit points does not
notice that nobody was told. When diffing a case, read its `log(` lines
first: they are the clauses the survivor is most likely to be missing, and
the ones a player notices first.

Regenerate with `node tools/list-dead-actions.js`.

| action | in app.js | referenced by tests | bucket |
|---|---|---|---|
| `EXHAUSTION_SET` | 2 | exhaustion.test.js | **shared** - falls through with EXHAUSTION_ADJUST, which is dispatched - the block is live |
| `SPELL_DELAYED_BLAST_DETONATE` | 1 | remaining-spells.test.js | **superseded-except** - paired with the unreachable GROW case |
| `SPELL_MAGIC_MISSILE` | 1 | remaining-spells.test.js | **superseded-except** - darts land; only an UNEVEN split needs a picker |
| `STAND_UP` | 1 | followups.test.js, scoped-actions.test.js | **superseded-except** - the movement bar stands and charges half; only the success log differs |

## Live actions no test dispatches

76 of 229 reachable actions are never
dispatched by any test. They may be covered through a helper - which is how a
reducer can silently drop a field while the suite stays green.

- `ACTION_COMMIT`
- `AREA_RESOLVE_CLEAR`
- `BLOCK_ZONE_UPSERT`
- `BOOK_READ_RESULT`
- `CHAT_ADD`
- `CHECK_REQUEST_ADD`
- `CHECK_REQUEST_REMOVE`
- `CLAIM_FAMILIAR`
- `CLAIM_MIGRATE`
- `CLAIM_PC`
- `DEATH_SAVE_ELIGIBILITY_SET`
- `DICE_LOG_CLEAR`
- `DM_KICK_PEER`
- `DM_UNCLAIM_FAMILIAR`
- `DM_UNCLAIM_PC`
- `DRAWING_DELETE`
- `DRAWING_UPSERT`
- `ECONOMY_TOGGLE`
- `ENTITY_REORDER`
- `ENTITY_UPSERT`
- `FORCED_VIEW`
- `FORCED_VIEW_PEER_CLEAR_ALL`
- `FORCED_VIEW_PEER_SET`
- `GRANT_PC_CONTROL`
- `GROUP_CHECK_END`
- `GROUP_CHECK_RESULT`
- `GROUP_CHECK_SET_ADV`
- `HAZARD_CLEAR_MAP`
- `HAZARD_DELETE`
- `HAZARD_PENDING_CLEAR`
- `HAZARD_PENDING_REMOVE`
- `HAZARD_UPSERT`
- `INIT_ADD`
- `MAP_IMAGE_RECEIVED`
- `MAP_SCALE_SET`
- `MAP_SWITCH`
- `MAP_VIEWPORT`
- `MOVEMENT_RESET`
- `PRESET_DELETE`
- `PRESET_SAVE`
- `RANGE_HIGHLIGHT_CLEAR`
- `RANGE_HIGHLIGHT_SET`
- `REMINDER_DELETE`
- `REMINDER_UPSERT`
- `REQUEST_ADD`
- `REVOKE_PC_CONTROL`
- `SET_LOCK_OFF_TURN`
- `SET_PLAYER_NAME`
- `SET_PLAYER_THEME`
- `SET_SICKNESS`
- `SOUND_DEREGISTER`
- `SOUND_EVENT`
- `SOUND_REGISTER`
- `SPELL_EFFECT_ADD`
- `SPELL_REQUEST_ADD`
- `SPELL_REQUEST_REMOVE`
- `SPELL_SLOT_RESTORE`
- `SPELL_TEMPLATE_CLEAR`
- `SPELL_TEMPLATE_SET`
- `STUNNING_STRIKE_LAND`
- `SUMMONABLE_TOGGLE`
- `TEMP_EFFECT_ADD`
- `TOKEN_GROUP_DELETE`
- `TOKEN_GROUP_SET_VISIBLE`
- `TOKEN_GROUP_UPDATE`
- `TOKEN_MOVE_EPHEMERAL`
- `TOKEN_PRESET_DELETE`
- `TOKEN_PRESET_UPSERT`
- `TOKEN_REMOVE`
- `TOKEN_REVEAL_ALL_ON_MAP`
- `TOKEN_SCALE`
- `TOKEN_VISIBILITY`
- `UI_CANCEL_TARGETING`
- `UI_SET_SHEET_TAB`
- `UNCLAIM_FAMILIAR`
- `UNCLAIM_PC`


## Progress

- shared: 1
- superseded-except: 3
