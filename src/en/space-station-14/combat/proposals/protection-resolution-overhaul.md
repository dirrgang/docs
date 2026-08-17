# Combat/Protection-Resolution Overhaul

| Designers | Coders   | Implemented | GitHub Links |
| --------- | -------- | ----------- | ------------ |
| dirrgang  | dirrgang | :x: No      | TBD          |

## Overview

This is a proposal for an overhaul of the current combat damage and armor calculation model inspired by RimWorld's Combat Extended mod. Attacks and armor interact through shared mechanical properties before the resulting injury is passed to the existing damage and medical systems.

The goal is to make interactions between weapons, ammunition and armor more predictable and intuitive, while reducing resistance bypasses, content-specific exceptions and balance dependencies on selected damage types.

## Background

The initial push to look into this project came from a downstream server, but the problem can be traced back to the 'root'. I was confused by the unintuitive behavior regarding how different types of ammo interacted with armor, and how a certain amount of meta-knowledge was required in order to properly counter certain types of weapons and armor. For instance, AP ammo basically trumped any kind of other ammo, even HP, due to the "ignore resistances" flag within.

At the moment, damage types fulfill several different roles simultaneously:

Security Vest

```text
Blunt: 0.70
Slash: 0.70
Piercing: 0.70
Heat: 0.80
```

Bulletproof Vest

```text
Blunt: 0.9
Slash: 0.9
Piercing: 0.4
Heat: 0.9
```

Riot Armor

```text
Blunt: 0.4
Slash: 0.4
Piercing: 0.7
Heat: 0.9
```

The idea is easy: The bulletproof vest is supposed to be much better against projectiles than the riot suit, and the riot suit better against blunt damage. This makes the downstream damage vocabulary double as an armor-interaction vocabulary.

Armor effectiveness is expressed as a matrix of modifiers against damage types, meaning that changing the damage composition of an attack can also change its effective armor penetration even when that was not the intended gameplay change. Piercing, for instance, carries at least two meanings: The attack creates piercing damage, and the attack actually penetrates the armor. But these are not equivalent.

Two projectiles can create similar damage after successfully penetrating armor, yet have substantially different armor-piercing behavior. Inversely, armor can stop the penetration of a hit without eliminating the entire mechanical effect of the hit. A spear, a hollow point, and an armor-piercing bullet can all create some type of Piercing damage, but don't necessarily interact with armor the same way.

It seems there is currently no satisfactory way to model this behavior consistently. This seems to lead to content-specific interactions instead of consistent, general rules.

There have been concrete cases where the selection of damage types basically implemented some form of armor penetration system. PR #20836 was merged because incendiary ammo used damage types that existing armor had no resistances against. The ammo basically ignored any type of armor and surpassed even AP ammo. The chosen solution was to change the damage type of the ammo. A similar problem occurred in PR #39204, again with incendiary ammo. The combination of damage types basically made it AP ammo, and again, the chosen solution was to modify the damage types.

This seems to be a consistent failure mode: The question "Can this attack/projectile penetrate this armor?" is often indirectly answered by the question "What damage types does this attack have?". This seems to result in unforeseen byproducts of interactions between ammo types, weapons, species and armor due to the interactions between damage types, modifiers and exceptions.

Prior PRs attempted to address parts of this problem.

PR #18506 attempted to migrate resistance penetration from a Boolean to a numerical value. AP ammo was intended to only partially ignore resistance values instead of wholesale. At the same time it attempted to implement a form of penetration power. The PR was not merged. Unlike #18506, this proposal does not treat armor penetration primarily as a percentage bypass of the existing resistance system. Its central change is to separate mechanical protection interaction from downstream injury types entirely. Entity penetration is also explicitly outside the scope of this proposal.

PR #30710 attempted to implement another missing semantic: Armor should partially convert reduced Piercing damage to Blunt damage. It basically boiled down to:

```text
100 Piercing
-> 40 after resistance
-> 24 Blunt + 16 Piercing
```

The motivation was explicitly that armor that stops an incoming projectile may prevent piercing damage, but should/could still result in Blunt damage from the impact. The PR was closed, but I'm not sure why.

This doesn't mean that penetration simulation has to be the right solution to the problem, but it seems like we're consistently experiencing issues appropriately modeling the desired behavior.

I've been a big fan of RimWorld and its Combat Extended mod, and of how it seems to model the behavior of different weapon and armor types based on very few, simple attributes. The idea was therefore to take that general approach and see whether a minimum viable implementation of the concept could also work in SS14.

This proposal is not intended as a port of Combat Extended. CE is primarily used as an existing reference for mechanics and failure cases that have already seen extensive play.

## Features to be added

### Mechanical protection separate from injury resolution

For mechanical attacks, the game would first determine how the attack interacts with the target's protection. Only the result of that interaction is then converted into damage types used by medical, species and other systems.

The proposed model consists of two interaction dimensions:

* **Breach** describes the capability to defeat a protective layer sufficiently to carry a wound inward.
* **Impact** describes the capability to transmit mechanical loading through protection without requiring Breach and ultimately cause blunt trauma.

These are intentionally abstract and intended for easy comparison. A given amount of Breach should have the same meaning regardless of whether it comes from a firearm, spear or another source.

Protection provides corresponding resistance values:

* **Breach Resistance**
* **Impact Resistance**

This allows armor to specialize without requiring protection to be expressed through downstream injury types. Projectile-oriented armor could have high Breach Resistance, while riot armor may favor Impact Resistance.

### Sequential Layers

Protection is calculated through ordered layers. Each layer reduces the remaining capability of the attack before passing it on to the next layer.

A layer can therefore weaken an attack without completely stopping it, and several individual layers can collectively stop it.

Protection resolves by physical depth, from outside to inside:

```text
external/interposed protection
-> outer equipment
-> inner equipment
-> innate physical protection
-> downstream body injury
```

Existing SS14 equipment semantics should be reused where they already encode physical depth. For example, `OUTERCLOTHING` resolves before `INNERCLOTHING`.

Raw ECS/component enumeration order is not protection ordering. Existing inventory ordering may be used as a deterministic tie-break where no actual physical ordering relationship exists.

### Partial penetration and stopped attacks

Penetration is not binary.

An attack carries a mechanical state:

```text
Breach
Latent Impact
Active Impact
Terminal Damage
```

Latent Impact is mechanical loading still coupled to the penetrating part of the attack. Active Impact is loading that has already become exposed to Impact Resistance and can ultimately create blunt trauma.

When a layer absorbs part of an attack's Breach, the same fraction of the Breach-coupled terminal wound is lost, while the corresponding fraction of latent Impact becomes active at that layer.

For a current Breach value `B` and a layer with Breach Resistance `R_B`:

$$
\Delta B = \min(B, R_B)
$$

The surviving fraction is:

$$
s = \frac{B-\Delta B}{B}
$$

while (B>0).

The surviving Breach-coupled state becomes:

$$
B \leftarrow Bs
$$

$$
I_{\mathrm{latent}} \leftarrow I_{\mathrm{latent}}s
$$

$$
D_B \leftarrow D_Bs
$$

The absorbed fraction of latent Impact becomes active:

$$
\Delta I = I_{\mathrm{latent,old}}(1-s)
$$

The same layer then reduces all active Impact:

$$
I_{\mathrm{active}}
\leftarrow
\max(0, I_{\mathrm{active}}+\Delta I-R_I)
$$

where (R_I) is the layer's Impact Resistance.

This results in three desirable outcomes without implementing three separate mechanics:

1. Weak protection reduces but does not eliminate a penetrating wound.
2. Adequate protection can stop the penetrating wound while still allowing blunt trauma.
3. Adequate protection against both Breach and Impact can greatly or completely mitigate the hit.

The usual labels are therefore derived from the residual Breach rather than selecting different resolver branches:

```text
B_final = B_initial     -> effectively unresisted Breach
0 < B_final < B_initial -> partially resisted Breach
B_final = 0             -> Breach stopped
```

Pure blunt attacks use the same resolver and simply start with active Impact and no Breach. They therefore do not require a separate armor system.

### Damage remains downstream

The resolver does not decide whether the resulting injury is `Piercing`, `Slash`, `Blunt`, or something else.

A bullet and a spear may both use Breach but still produce different terminal injuries. Conversely, a club can use the same Impact interaction as the blunt trauma created when armor stops a bullet.

After all protection layers have been resolved:

* the remaining terminal mechanical damage is passed downstream;
* surviving Active Impact is converted to ordinary Blunt damage;
* existing medical, species and damage systems continue handling the result.

Physical species protection such as shells, scales or similar structures can participate as ordinary protection layers. Biological responses to injury remain downstream.

### Balancing

Existing weapons, ammunition and armor will require migration towards shared Breach and Impact scales.

The exact scale still needs to be calibrated against representative SS14 content. However, the interaction semantics themselves do not depend on particular final values.

The intended authoring relationship is straightforward:

```text
higher Breach
-> better at carrying a wound through protection

higher Impact
-> greater blunt-loading potential

higher Breach Resistance
-> better at stopping breach-capable attacks

higher Impact Resistance
-> better at suppressing blunt trauma
```

For Breach specifically, equality has a simple meaning:

$$
R_B = B
$$

is sufficient to stop the remaining Breach interaction at that layer.

This should make intended behavior easier to reason about than adjusting combinations of damage types and resistance coefficients.

For instance, ordinary AP ammunition can be modeled through higher Breach combined with lower terminal wound damage instead of categorical resistance bypass.

## Game Design Rationale

The goal is not to make SS14 a milsim.

This primarily addresses the [Intuitive and Inter-Connected Simulation](https://docs.spacestation14.com/en/space-station-14/core-design.html#intuitive-and-inter-connected-simulation) design principle. Armor and weapon behavior should be intuitive and understandable through their given properties. Meta-knowledge of `DamageSpecifier` compositions and resistance bypasses should not influence player choice and behavior.

The separation of protection from injury also makes combat more interconnected. Firearms, melee weapons, natural attacks, armor and species attributes can use the same mechanical 'vocabulary' where appropriate without changing behavior in downstream systems like medical.

The changes should allow players to make intuitive choices based on in-game context instead of having to rely on metagaming.

The current design is also intentionally deterministic. There is no RNG angle-based deflection or similar hidden roll in the core resolver.

Finally, the design should make it substantially easier to add new weapons and armor, as their behavior becomes more predictable and is no longer primarily determined by particular weapon-armor-damage-type combinations.

## Roundflow & Player interaction

This doesn't add a new round event or gameplay loop.

It's only relevant whenever mechanical attacks interact with any type of protection, and can therefore happen in any round in which there is combat. So, basically every round.

The most direct interaction between players and this system is during equipment selection. Players can expect weapons and armor to behave according to their mechanical properties and thus make context-dependent choices rather than universally selecting the current metagame-du-jour.

This is of particular importance to Security and antagonists.

Later extensions of the system could also improve non-lethal weapon and armor behavior, but are not required for the initial implementation.

## Administrative & Server Rule Impact (if applicable)

No intended impact.

It might influence some SOPs depending on the resulting balance changes.

# Technical Considerations

## Resolution pipeline

The implementation should use a shared mechanical-protection resolver that can be called by different attack sources rather than being projectile-specific.

Conceptually:

```text
attack source
-> MechanicalAttackState
-> ordered protection stack
-> pure protection resolver
-> resolved mechanical DamageSpecifier
-> existing DamageableSystem / medical pipeline
```

Projectile, melee, natural and future attack sources therefore share the protection semantics without having to share their entire attack implementation.

The resolver itself only operates on the attack state and an already ordered set of protection layers. It does not query ECS state, inventory slots, damage types or attack sources.

## Runtime representation

Conceptually, the relevant runtime state looks roughly like this:

```text
MechanicalAttackState
    Breach
    LatentImpact
    ActiveImpact
    TerminalDamage
```

A protection layer correspondingly contains:

```text
MechanicalProtection
    BreachResistance
    ImpactResistance
```

The exact C# types and names are implementation details.

The important part is that Breach and Impact are not encoded as new downstream damage types or entries in `DamageModifierSet`.

A dedicated mechanical-protection component would likely provide the two resistance values for armor entities and innate physical protection.

## Protection stack construction

Applicable protection layers must be explicitly collected before resolution and ordered by physical depth.

For ordinary equipment the minimum required relationship is:

```text
OUTERCLOTHING
-> INNERCLOTHING
-> innate protection
```

External or explicitly interposed protection resolves before worn equipment.

The existing inventory model should be reused where it already contains useful ordering information, but raw inventory-template order or ECS event enumeration must not accidentally become the protection semantics.

If two protection sources have no meaningful physical relationship, existing deterministic ordering can be retained as a tie-break instead of adding a new universal `ProtectionDepth` stat.

## Integration with the existing damage pipeline

After mechanical protection has been resolved, the resulting terminal wound and generated blunt trauma should still be passed into the existing `DamageableSystem`.

However, migrated mechanical armor must not then modify that damage a second time through the legacy armor path.

The downstream application therefore needs an explicit way to indicate that mechanical protection has already been resolved.

Using the existing broad `ignoreResistances` behavior for this would not be sufficient, because legitimate downstream modifiers may represent species, biological or other injury responses that should still apply.

The intended distinction is therefore:

```text
mechanical protection
-> handled before DamageableSystem

biological / medical / other downstream response
-> remains in the existing damage pipeline
```

`DamageModifierSet` therefore does not have to disappear entirely. Non-mechanical resistances and downstream biological damage can remain separate.

## Source adapters

Different attack systems construct the same mechanical attack state.

A projectile might provide:

```text
Breach
Latent Impact
Terminal Piercing Damage
```

A knife could provide:

```text
Breach
little or no Impact
Terminal Slash Damage
```

An axe could provide:

```text
Breach
Active Impact
Terminal Slash Damage
```

A club could provide:

```text
no Breach
Active Impact
no Breach-coupled terminal wound
```

The resolver does not need to know which of these created the state.

Existing source-specific modifiers may continue modifying terminal damage where appropriate. They should not implicitly change Breach or Impact merely because the resulting `DamageSpecifier` changed. Any mechanic intended to affect protection interaction should modify the mechanical attack profile explicitly.

Projectile entity penetration also remains separate. Existing projectile continuation or penetration-threshold mechanics must not be treated as synonymous with armor Breach.

## Mixed mechanical and non-mechanical payloads

Only mechanical injury coupled to successful Breach belongs in the terminal mechanical damage carried through the resolver.

For example, an incendiary projectile might logically contain:

```text
mechanical terminal wound
+ heat damage
+ ignition or another effect
```

Only the mechanical wound should be scaled by Breach survival.

Otherwise armor penetration could accidentally determine unrelated heat, poison, electrical or reagent effects.

Unrelated payloads should therefore remain in their existing source/effect pipeline outside the mechanical protection resolver.

## Incremental migration

Migration does not need to be a big-bang conversion.

The new path should be opt-in per attack source:

```text
source has mechanical attack profile
-> use new mechanical protection resolution

source has no mechanical attack profile
-> use legacy damage/resistance path
```

A target does not need to provide protection values for the new resolver to work. An unarmored target simply contributes no protection layers.

During migration, armor prototypes may temporarily contain both:

* new mechanical protection values for migrated attacks;
* existing `DamageModifierSet` values for legacy attacks.

This is a migration mechanism rather than the intended final design.

## Networking and prediction

Protection resolution should remain server-authoritative at the actual hit/damage boundary. The transient Breach/Impact state does not need to become replicated ECS state. The resolver can still live in shared code where useful for tests, inspection tooling or later prediction work, but V1 should not network per-layer residual state just because the data exists. If client feedback later needs to distinguish a stopped, partial or effectively unresisted hit, only the required presentation result needs to be communicated.

## Complexity and testing

The resolver is deterministic and operates only on one attack state and a small number of ordered protection layers. Its computational complexity therefore scales linearly with the number of applicable layers and should be negligible compared to the surrounding combat systems. Keeping the resolver pure also makes the actual interaction model independently testable.

No new UI components should be required.

Item descriptions and examine text could/should be updated to expose the new properties in an understandable form once the final content scale has been established.
