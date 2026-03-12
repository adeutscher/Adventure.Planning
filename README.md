# "Adventure" Game Framework Research Project Planning

This repository is an ongoing scratchpad for items that I'm currently grappling with in my game design research project.

For items that are more complete, see the [Adventure.Report](https://github.com/adeutscher/Adventure.Report) repository.

## Past Problems

An archive of solved problems can be found on [this page](./past-problems.md).

## NPC Stats

This section is a post-mortem of my original attempt at NPCs in a prototype project.

### The Setup

In the initial version, NPCs were defined like so:

* NPCs attributes were defined in instances of a template class and assigned to a spawn pool.
    * This part worked just fine and isn't what this section is about, but I felt that it was necessary to set the stage.
* NPCs were assigned a faction ID
    * For the moment, a NPC considered a nearby unit to be hostile if the nearby unit did not have a matching faction ID.
* NPCs were assigned 5 combat stats:
    * Health (technically, this one was hard-coded and not part of the template class)
    * Attack damage min
    * Attack damage max
    * Attack power stat
    * Attack speed
* Though I've yet to mix player characters with NPCs in one project, they are both fundamentally UnitEntity instances and have the same stat fields.
* Units were given an attack damage calculation formula copied from World of Warcraft: `WeaponDamage + (AttackPower / 14) * WeaponSpeed`, with 14 being a constant for attack power stat scaling chosen by their team
* NPCs were set to spawn on a flat plane and to fight the units not from their faction when they detected each other or were attacked.
* On death, a unit would despawn and respawn shortly afterward to repeat the cycle
* Defined two types of units with different stats: Cubes and Capsules
    * The intent was for the single Capsule on the field to be the strong one who could handle multiple weaker Cubes if the cubes ganged up on it.

### Observation

With this setup, it came as shock when a single Cube unit absolutely destroyed the Capsule unit. I thought I had given the Capsule unit stats to outgun the Cube unit, but the proof was in the pudding. A single Cube could defeat a Capsule 4-5 times before dying (since this was the first version where NPCs could take damage from each other, this also meant that NPCs had no need to regenerate health yet WoW-style between fights)

### The Problem

Even with just 5 statistics, this became a thing that would be tricky to debug if I had to balance even just these two units. I think that the problem lies in directly setting unit stats in the template.

In hindsight, keeping a 1:1 template:stat-value setup would create a pile of maintenance nightmares like these:

* Adding a new type of unit stat and having to assign a new value to EVERY template
* Forgetting to add a stat (e.g. base weapon damage, weapon speed)
* Having to re-scale an enemy or a group of enemies to be less tough would require focused attention on the different formulas
    * Requiring detailed formula knowledge goes against the same philosophy of team accessibility that drives other features of the overall project like the APIs and the World Builder portal.
* Having to re-balance a particular stat and then re-tune every single NPC.
* Simply upping the level of an NPC would still require detailed re-tuning to match that level
* All this doesn't even consider enemies that could have a variable level to add spice to the environment. For example, imagine a Wolf that could have a level between level 3 and 5. One would expect the level 5 wolf to be a harder-hitting enemy than the level 3.

Even if I had all day to spend just on NPC stats, this system requires too much specialized knowledge and just doesn't feel scaleable in general.

### The Next Version

In a future commit of my prototype, I'm considering something along the lines of the following system:

* The stats of an NPC shall primarily hinge on their level.
* The NPC template table(s) shall have abstract *modifiers* that tweak the stats rather than directly defining the stats themselves. Examples:
    * A Dire Rat is the most generic enemy you that you can find. Its modifiers all default to 0. However:
        * A level 1 Dire Rat can still hurt the player (setting aside that a low-level monster should not be hitting for much of anything, this is just an example)
        * A level 2 Dire Rat shall hit for more damage and have more HP than a level 1 Dire Rat
        * You could, in theory, rebalance the Dire Rat to be a max-level enemy by changing only one value
    * A Wolf is a melee-focused enemy:
        * It has an attack power modifier rating of 1. This shall make its attack power somewhat different from a Dire Rat.
    * A Warlock is a somewhat squishy caster enemy with more powerful magic than average. It has:
        * A melee modifier of -1.5 to blunt its melee attacks
        * A spell modifier of 1 to boost its magic attacks.
        * An armour modifier of -0.25 to affect its armour and make it squishy
    * Lets say we have an 'elite' enemy flag down the road. If we assume that it's more than just a cosmetic marker under the hood in WoW, then this would boost the value of some stats.
* In addition to the specific dials like the above modifiers, each NPC could have a 'challenge rating' dial that globally tweaks difficulty rating within a level without necessarily having to get to deep in the weeds.

I plan to implement this in a testable way by further abstracting out the duties of the class responsible for populating NPC properties into one or two layers of formula application.

```csharp
unit.Strength = npcStatCalculationService.CalculateStrength(chosenLevel, npcTemplate)
```

The initial version of this modifier system certainly doesn't need perfect formulas, but it should be structured off the jump to be easily planned, modified, and discretely unit tested (for example, the formula for a unit's Strength shouldn't care about the formula for the Strength modifier).

## Character Animations

These are some early thoughts on handling unit animation. Actual implementation is still a ways off as my focus for now is on mechanics over visuals, but this page makes for a slightly less obscure place for these notes than in the comments section for the mostly-unrelated Unit Auto-Attack task.

Unity's FishNet module has a NetworkAnimator. The NetworkAnimator can work in a client-authoratative or server-authoratative mode. As I detail below, I don't think it's a perfect fit.

### Requirements

If we (once again) imitate the WoW model, then we would have the following needs:

* The client's player character should continue running uninterrupted, even if the connection is having a bad time. This suggests that a client-authoratative setup would work for player units.
* Players and units can perform emotes to make themselves perform an animation that appears to themselves and observers. This would also be compatible with a client-authoratative setup.
* Players can be forced by the server to perform animations that appear for the player and all observers. For example, an enemy ability could force a knock-down animation. This is where my first-glance of the client-authoratative model starts to break down. If the client is the only authority that can make an animation happen in a client-authoratative model, does that mean that the emote command has to round-trip it's way from the server to the specific client?
    * Combat mechanics could also be said to fall under this umbrella. Under a client-authoratative model where animations are controlled by a Client, is a weapon swing supposed to go from server->attackerClient->server->observerClient?
* Units also have the occasional idle animation. Is this something that's supposed to be handled by the server for every NPC?

### Other Items

Other discussion points:

* How much overhead is involved in the maintenance of a NetworkAnimator? And how well does that scale?

### WoW OpCodes

We cannot use a WoW private servers' code for GPL reasons, but I figure that  we can still use the research that supports them. WoW's networking setup is built on a series of operation codes. It's part of what drew me to FishNet's Broadcast system. From the [wowdev.wiki page on OpCodes](https://wowdev.wiki/Opcodes), even just the titles of the following OpCodes suggests that this is something that the WoW team considered as well:

* `MSG_MOVE_START_FORWARD`
* `MSG_MOVE_START_BACKWARD`
* `MSG_MOVE_STOP`
* `MSG_MOVE_JUMP`
* `MSG_MOVE_START_STRAFE_LEFT`
* `MSG_MOVE_START_STRAFE_RIGHT`
* `MSG_MOVE_START_TURN_LEFT`
* `MSG_MOVE_START_TURN_RIGHT`
* `MSG_MOVE_SET_RUN_MODE`
* `MSG_MOVE_SET_WALK_MODE`
* `MSG_MOVE_FALL_LAND`
* `MSG_MOVE_START_SWIM`
* `SMSG_EMOTE`
* `SMSG_PLAY_DANCE`
    * Off-topic: How complex was WoW's dancing system planned to be if OpCodes like `SMSG_LEARNED_DANCE_MOVES` exist? It sounds like it goes way beyond the `/dance` emote.

The list goes on, but I think I've made my point. Based on these OpCodes, I'm fairly confident that a WoW server does not give a hoot about maintaining animations beyond keeping track of the on/off state of things like running.

As a side-note, this could suggest an explaination for why WoW characters might appear to run in place before stopping on a disconnect: If a player's animation is client-authoratative via these OpCodes, then a disconnected client would lost the chance to say "hey, I've stopped running".

### Proposal

* A series of animation states and flags on the unit shall be shared with the Client via FishNet SyncVars.
* All further animation details are a Client-only concern.

#### Requirements

Off the cuff, we would need some changes from our current setup:

* Our character controller would need to be modified to send broadcasts to the server that are equivalent to many of these WoW OpCodes.
    * The broadcasts shall set SyncVar values for that particular unit, which FishNet shall then set on observers.
* Clients need to add to spawning UnitEntities some sort of "UnitAnimationController" which acts on these.
* For player-controlled unit animations to keep happening despite connection funtimes, the "UnitAnimationController" should have access to some local values that take priority if it's managing the unit being controlled by that particular client.
* On top of all this back-end stuff, we actually need to get the animations.

Other:

* This will definitely need a new prototype project specifically for animations.
* Extending NPCs to have animations could make the NPC template impossible to share after a certain point. I might be over the line as it is with the UI test, and animations feel like another tier over the UI elements. Best not to test our luck further.

### Timeline

All this being said, confirming what I started off this section by saying that this is all very much not a Today problem. Mechanics come first, and I'm fine with the game being World of CapsuleCraft for the time being.

This project doesn't have specific dates, but there are some features that I'd want to implement at a minimum before thinking further about animation:

* NPC stat prototyping
* Integrating the NPC prototype into the main project
* Player unit death/respawning
* Player unit auto-attacks
* Item use (basic spellcasting groundwork)
* Spell casting
    * Instant effect
    * Delayed effect (missiles)
    * Delayed effect representations (render missiles)
* Auras Groundwork (Buffs/Debuffs)
* Auras UI
* Chat/Combat Log
* Confirm character/camera controller
    * I like the character controller asset that I purchased, but I think like a different asset's camera controller more. I need to spend some time making a decision and commit to whether I'm sticking with the setup as-is, going for a blend, or none of the above. I should only invest time into tinkering with the controller for animation broadcasts when I'm more certain that I'm not going to swap out the entire implementation down the road.

In short, there's a lot to do!

That being said, I'm still tempted to at least making some rough flags enum for animation and methods along the lines of "SetRunningEnabled". But that would be a distraction and might need to be rewritten if major developments happen during the implementation of any of the above features. Better to try to do it all in one go.