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
* In addition to the specific dials like the above modifiers, each NPC could have a 'challenge rating' dial that globally tweaks difficulty rating without necessarily having to get to deep in the weeds

I plan to implement this in a testable way by further abstracting out the duties of the class responsible for populating NPC properties into one or two layers of formula application.

```csharp
unit.Strength = npcStatCalculationService.CalculateStrength(chosenLevel, npcTemplate)
```

The initial version of this modifier system certainly doesn't need perfect formulas, but it should be structured off the jump to be easily planned, modified, and discretely unit tested (for example, the formula for a unit's Strength shouldn't care about the formula for the Strength modifier).