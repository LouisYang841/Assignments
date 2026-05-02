## REQ1 Basic Items and Ship

REQ1 Setup the basic senario of workers and ship. Every worker spawns with an inventory limited to 50 units of weight and a Flask allow 5 consumptions. The ship is composed of wall `#`, floor `_`, and locked doors `=` . Three unique items can be found on the ship -- AccessCard (unlock doors), FirstAidKit (heal with cooldown), and SterilisationBox (sterlize).

![[A1_REQ1_UML.drawio.svg]]

---
### 1.1 Inventory Weight Limit

**Requirement:** The inventory of workers have a weight limit of 50.

**Solution:** Creates `WeightLimitedInventory` which extends `Inventory`, override `add()` to return `false` when overweighted. Items without `WEIGHT` will tirgger `IllegalArgumentException` -- all Items that added to the `WeightLimitedInventory` must declare a weight.

| Pros                                                                                                                                                                                     | Cons                          |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| utilized engine mechanise: `PickUpAction` automatically handles the situation where `add()` returning `false` and display the Error message; Rules are encapsulated within its own class | additional subclasses created |
**OOP principle:** `WeightLimitedInventory` is a specific kind of `Inventory` that can be handled as  an `Inventory` , which adheres **Liskov Substituion Principle**

---
### 1.2 Flask and FirstAidKit -- Consumable Interface (Decision)

**Requirements:**
- Flask: weights 3, can be consumed 5 times, heal 1 HP each. keep in inventory when depleted.
- FirstAidKit: weights 25, +1 max HP and full heal. 20 turns of cooldown; cooldown only counts when carried.

**Alternative A:** Separate `Action` classes :`ConsumeFlaskAction`, `ConsumeFirstAidKitAction`,  each with their own effect logics.

**Alternative B:** Create `Consumable` interface that has method `consumedBy()` and `canBeConsumed()`. Both Flask and FirstAidKit implements it. The same kind of `ConsumeAction` handles all `Consumables` , call `consumedBy()` without refering exact class type. 

|       | Pros                                                                                                 | Cons                                                  |
| ----- | ---------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| **A** | Actions are individual; Easy to understand                                                           | Action classes will multiply fast                     |
| **B** | new Item→ Implements `Consumable`, can use same `ConsumeAction`; One `ConsumeAction` for whole type. | Requires an Interface, abstraction lowers readability |

**Justification:** Chose B. Two consumables in REQ1 already, and we have even more in REQ2 (Apple, Cookies, Puddles). It is inefficient to write a corresponding action for every Item. `ConsumeAction` achieves Polymorphism through `Consumable.consumedBy()` . New consumables requires only implementation instad changing code of the Action which adheres to **Open-Closed Principle**. Durability of Flask and cooldown of FirstAidKit is internal detail maintained by classes their own, which adheres to the **Single Responsibility Principle**.

---
### 1.3 Door & SterilisationBox -- Ability Tag (Decision)

**Requirements:**
- There are locked doors on the ship (`=`); AccessCard can open any doors
- SterilisationBox can sterlize polluted items (poison → heal).

**Alternative A:** Check specific class type. Door checks `inventory.contains(AccessCard.class)`; Consumables check `inventory.contains(SterilisationBox.class)`.

**Alternative B:** Items enable Ability tags. AccessCard enables `OPEN_DOOR`; SterilisationBox enables `STERILISE`. Door checks `actor.hasAbility(OPEN_DOOR)`; Consumables check `actor.hasAbility(STERILISE)`.

|       | Pros                                                                                                     | Cons                                                                                                        |
| ----- | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **A** | Straight foward without additional mechanism                                                             | New door opening tools → must change all checks                                                             |
| **B** | Item with same ability are abstracted into same tag, new item with the ability only needs to add the tag | Must handle the tags correctly during operations. Bugs in tags tends to be silent and harder for debugging. |

**Justification:** Choose B. In accordance to **Dependency Inversion Principle**, Door should not rely on specific kind of key to unlock it. Same applies to SterilisationBox: Consumables that can be Sterlized checks `hasAbility(STERILISE)` and does not care the specific item providing such ability. Adding new items with similar or same ability does not require change in current class code, this adheres **Open-Closed Principle.**

---
### 1.4 Flask consumption behavior

**Requirements:** 
- Flask stays in the backpack after depletion

**Approach:** Unlike `Apple` and `Cookies` the `consumeBy()` in flask never calls inventory.remove(this).

**OOP principle:** Flask maintain's it's own behavior during consumption. We dont want `ContractedWorker` or `ConsumeAction` to handle something beyond themselves. Keeping this logic inside adheres to the **Single Responsibility Principle**.

---
### 1.5 FirstAidKit cooldown 

**Requirements:** 
- Requirement: Cooldown only counts down when carried

**Approach:** FirstAidKit overrides `tick(Location,Actor)`, this method is only called when the `Item` is inside an `Actor`'s inventory.

**Why:** This utilized provided engine mechanism.

---

## REQ2 DoT effect, Environment Interaction and more Items

REQ2 introduces new Items, DoT mechanisms and environment Interactions. Workers can find harzardos consumables such as Apple and Cookies that can be sterilized with SterilizationBox. Two simple items: FloppyDisk and CRT Monitor with only weight feature.
A Lantern that leaks oil and cause fire on grounds. The puddle can now be drinked directly.

![[A1_REQ2_UML.drawio.svg]]

---
### 2.1 DamageOverTimeStatus

**Requirement:** 
- Apple: poison player 1 dmg/turn for 5 turns when unsterilized
- Puddles: poison player 1 dmg/turn for 3 turns when unsterilized
- Fire: apply a burn effect 1 dmg/turn for 5 turns when and `Actor` stand on it.

**Approach**: Create a `DamageOverTimeStatus` and `DamageOverTimeType` enum as constructor arguments.

**Reason:** The logic inside every DoT effect are all "x damage per x turn" form. Instead of handling them separately, It is clearly more efficient to abstract them into a general status differentiated with an enum. This is an example of **Polymorphism**.

### 2.2 Puddle -- Utilize Consumable Interface (Decision)

**Requirement:** Puddle can be drinked directly form ground and cause effects, poison player 1 dmg/turn for 3 turns when unsterilized and heal 1 when sterilized.

**Alternative A:** Add logic in `Puddles` or `ContractedWorker` to drink on puddle kind of ground.

**Alternative B:** Utilized and Implement the Consumable interface earlier to handle the drinking of puddle through `ConsumeAction` and maintain the logic in `ConsumedBy()`

|       | Pros                                                                                                                      | Cons                                                   |
| ----- | ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| **A** | Item and Puddle is handled separately and avoid interference                                                              | Redundancy in logics since the logic is being similar. |
| **B** | A single `ConsumeAction` handles all similar kind of action, no matter what exactly it is. Decrease code logic redundancy | Every class must implement `consume()`                 |
**Justification**: Chose B. `Consumable` interface has already been estabilished in REQ1. This modification is not happening in any of previous code, which adheres to **Open-Closed Principle**. `ConsumeAction` handles all `Consumable` things, no matter what exactly it is, this is a kind of **Polymorphism** and gives better extensibility.

### 2.3 Apple & Cookies -- Sterilization Ability check

**Requirement: 
- Apple: weights 1, poison player 1 dmg/turn for 5 turns when unsterilized, heal 3 hp when sterilized. Removed after consumption.
- Cookies: weights 2, with five pieces inside, each one -1 max hp when unsterilized, heal 1 when sterilized. Removed after all consumed.

**Approach:** Both Items Implements `Consumale` and check in `consumedBy()` see if the actor consuming it have the ability of Sterilization, choose different path of outcome and use `DamageOverTimeStatus` for DoT effects. The removal logic is achieved same as `Flask` in REQ1 where it depends how `Item.remove()` is handled inside their `consumedBy()`.

**Reason and OOP principle**:
- `hasAbility()` utlized engine freatures for abstraction.
- The multiple use logic handled within cookie itself, this adheres to the **Single Responsibility Principle**
- Both item does not directly interaction with `SterilizationBox`, only the abstract tags, future items with sterilization ability only needs to opt in through the abstraction layer, which adheres to **Dependency Inversion Principle**
### 2.4 Lantern

**Requirement:** Weights 7, with 10 units of oil. 5% chance of leaking 1 unit of oil and creating a `Fire` on the ground each turn.

**Approach:** The `Lantern` overrides `GameEntity.tick(Location,Actor)`