# Combat/Protection-Resolution Overhaul

| Designers | Coders | Implemented | GitHub Links |
|---|---|---|---|
| dirrgang | dirrgang | :x: No | TBD |

## Overview

This is a proposal for an overhaul of the current combat damage and armor calculation model inspired by Rimworld's Combat Extended mod. Attacks and aremor interact through shared properties before the resulting injury is passed to the existing damage and medical systems.

The goal is to make interactions between weapons, ammunition and armor more predictable and intuitive, while reducing resistance bypasses, content-specific exceptiosn and balance depndencies on selected damage types.

## Background

The initial push to look into this project came from a downstream server, but problem can be traced back to the 'root'. I was confused by the unintuitive behavior regarding how different types of ammo interacted with armor, and how a certain amount of meta-knowledge was required in order to properly counter certain types of weapons and armor. For instance, AP ammo basically trumped any kind of other ammo, even HP, due to the "ignore resistances" flag within. 

At the moment, damage types fulfill several different roles simultaneously:

Security Vest
```
Blunt: 0.70
Slash: 0.70
Piercing: 0.70
Heat: 0.80
```

Bulletproof Vest
```
Blunt: 0.9
Slash: 0.9
Piercing: 0.4
Heat: 0.9
```
Riot Armor
```
Blunt: 0.4
Slash: 0.4
Piercing: 0.7
Heat: 0.9
```

The idea is easy: The bulletproof vest is supposed to be much better against projectiles than the riot suit, and the riot suit better against blunt damage. This makes the downstream damage vocabulary double as an armor-interaction vocabulary. Armor effectiveness is expressed as a matrix of modifiers against damage types, meaning that changing the damage composition of an attack can also change its effective armor penetration even when that was not the intended gameplay change. Piercing for instance, carries at least two meanings: The attack creates piercing damage, and the attack actually penetrates the armor. But these are not equivalent.

Two projectiles can create similar damage after successful penetration of the armor, yet have substantially different armor-piercing behavior. Inversely, armor can avoid a successful penetration of a hit, without eliminating the entire impact of the hit. A spear, a hollow point, and an armor-piercing bullet can all create a type of Piercing damage, but don't necessarily interact with armor the same way.

It seems there is currently no satisfactory way to model this behavior consistently. This seems to lead to content-specific interactions instead of consistent, general rules. 

There have been concrete cases where the selection of damage types basically implemented some form of armor penetration system. PR #20836 has been merged, because incendiary ammo used damage types, that existing armor had no resistances against. The ammo basically ignored any type of armor and surpassed even AP ammo. The chosen solution was to change the damage type of the ammo. A similar problem was in PR #39204", again with incendiary ammo. The combination of damage types basically made it AP ammo, and again, the chosen solution was to modify the damage types.

This seems to be a consistent failure mode: The question "Can this attack/projectile penetrate this armor?" is often indirectly answered by the question "what damage types does this attack have?". This seems to result in unforeseen byproducts of interactions between ammo types, weapons, species and armor, due to the interactions between damage types, modifiers and exceptions.

Prior PRs attempted to address this problem. PR #18506 attempted to migrate resistance penetration from a Boolean to a numerical value. AP ammo was intended to only partially ignore resistance values, instead of wholesale. At the same time it attempted to implement a form of penetration power. The PR was not merged. Unlike #18506, this proposal does not treat armor penetration primarily as a percentage bypass of the existing resistance system. Its central change is to separate mechanical protection interaction from downstream injury types entirely. Entity penetration is also explicitly outside the scope of this proposal.

PR #30710 attempted to implement a missing semantic: Armor should partially convert reduced Piercing damage to Blunt damage. It basically boiled down to: 100 Piercing -> 40 Piercing after Resistance -> 24 Blunt + 16 Piercing. The motivation was explicitly that armor that stops an incoming projectile may prevent piercing damage, but should/could still result in Blunt damage from the impact. The PR was closed, but I'm not sure why.

This doesn't mean that penetration simulation has to be the right solution the problem, but it seems like we're consistently experiencing issues to appropriately model the desired behavior.

I've been a big fan of Rimworld and its Combat Extended mod, and how it seems to realistically model the behavior of different weapon and armor types based on very few, simple attributes. The idea was therefore to take this idea and try and see if the minimum viable implementation of the concept could also work in SS14. 

## Features to be added

### Mechanical protection seperate from injury resolution

For mechanical attacks, the game would first determine how the attack interacts with the target's protection. Only the result of that interaction is then converted into damage types used by medical, species and other system.

The currently proposed model consists of two attributes:
- **Breach** describes the capability to defeat a protective layer sufficiently to carry a wound inward.
- **Impact** describes the cpability to transmit mechanical loading through protection without requiring Breach and ultimately cause blunt trauma.

These are intentionally abstract, intended for easier comparison. A given amount of "Breach Power" should have the same meaning regardless whether it comes from a firearm, spear or other source.

Protection provides corresponding resistance values. This allows different armor to specialize for different types of damage protection. Projectile-oriented armor could have a high Breach resistance, while riot armor may favour Impact resistance.

### Sequential Layers

Protection is calculated through ordered layers. Each layer reduces the remaining capability of the attack before passing it on to the next layer. A layer could therefore weaken an attack without completely stopping it, and several individual layers could collectively stop it. The exact ordering would be deterministic and explicit.

### Partial penetration and stopped attacks

Penetration is not binary. When a layer absorbs part of an attack's Breach capability, the remaining portion continues inward. When the protection absorbs penetrating capability, the corresponding portion fo the attack's coupled Impact becomes active at that layer, and is reduced by that layer's Impact Resistance.

This would result in three desireable results without having to implement three seperate mechanics:

1. Weak protection reduces but does not eliminate a penetrating wound,
2. Adequate protection can stop the penetrating wound while still allowing blunt trauma,
3. Adequate protection against both penetration and impact can greatly or even completely mitigate the hit.

Pure blunt attacks use the same resolver and thus do not require a seperate armor system.

### Damage remains downstream
This does not meddle with the resulting types of injuries. Medical and species behavior remains unchanged. It does, however, allow for species attributes (thick scales or whatever) to work consistently.

### Scope

The current design covers only mechanical protection. It doesn't require projectile continuation, simulated projectile energy, caliber, or other real-world attributes, random armor RNG, body-part hit locations or coverage, armor durability or degredation, or a distinction between soft and hard armor.

Parts of these could be added later, if they are considered a gameplay plus.

### Balancing

Existing weapons, ammo and armor will require a migration towards the new shared scales. However, it should be possible to model the desired behavior much more easily than before.

## Game Design Rationale

The goal is not to make SS14 a milsim. This primarily addresses the [Intuitive and Inter-Connected Simulation](https://docs.spacestation14.com/en/space-station-14/core-design.html#intuitive-and-inter-connected-simulation) design principle. Armor and weapon behavior should be intuitive and understandable by their given properties. Meta-knowledge of DamageSpecifier compositions and resistance bypasses should not influence player choice and behavior.

The seperation of protection from injury also makes combat more interconnected. Firearms, melee weapons, natural attacks, armor and species attributes can use the same 'vocabulary' where appropriate while not influencing behavior in downstream systems like medical. The changes should allow players to make intuitive choices based on in-game context instead of having to rely on metagaming.

The current design is also intended to be deterministic - no RNG angle-based deflection or something like that.

Finally, the design should make it vastly easier to add new weapons and armor, as their behavior becomes more predicable and not based on particular weapon-armor-combinatorics.

## Roundflow & Player interaction

This doesn't add a new round event or gameplay loop. It's only relevant whenver mechanical attacks interact with any type of protection, and can thus happen any round in which there is combat. (So.. every round).

The most direct interaction between players and this system is during equipment selection. Players can expect weapons and armor to behave as they should, and thus allow contenxt-dependent selection, rather than universal choice of the current metagame-du-jour. This is of particular importance to sec and antags. Later extensions of the system could also improve non-lethal weapons and armor behavior.

## Administrative & Server Rule Impact (if applicable)

No intended impact. It might influence some SOPs.

# Technical Considerations

The implementation uses a shared mechanical-protection resolver that can be called by different attack sources, rather than being projectile-specific. Attacks need to provide a small profile containing their breach capability, Impact behavior and the resulting DamageSpecifier that remains after the resolution. Mechanical protection sources correspondingly provide Breach and Impact resistance.

Applicable protection layers must be collected in a deterministic order.

After the protection has been resolved, the resulting terminal wound and any generated blunt trauma are simply passed through the existing damage pipeline. Existing medical damage types thus remain unchanged.

This also means that the existing DamageModifiersSet does not have to disappear entirely. Non-mechanical resistances and downstream biological damage can remain seperate, migrated equipment simply musn't apply the legacy armor modifiers after the resolver has already handled the hit.

The resolver itself is deterministic and oeprates only on the attack state and a small number of protection layers. Therefore, the computational cost should scale linearly with the number of protective layers and should be largely negligable.

This would need no new UI components, some item descriptions could/should be udpated, however.

Migration could be implemented incrementally. A simple selector that applies either the legacy or updated resolver based on whether both interacting objects have the required attributes should be sufficient. First 'real' target would be a vertical slices of a regular projectile, melee weapon and armor type.
