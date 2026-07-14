# REI Overstack Exploit Module (Meteor Client Addon)

This repository contains the source code for a Meteor Client addon designed to exploit a vulnerability in the **Leaves** Minecraft server fork.

---

# Vulnerability Report: Packet-based Item Overstacking & Duplication via `move_items_new`

## Overview
- **Software Affected**: Leaves Server and shedaniel's Roughly Enough Items (REI) server-side mod.
- **Versions Affected**: 
  - **Leaves Server**: Minecraft 1.21.8 build 67 and later (containing commit `eb3d87b` which finished the `MOVE_ITEMS_NEW_PACKET` implementation).
  - **REI Server Mod**: All versions of the official REI mod (Fabric/Forge/NeoForge) containing `me.shedaniel.rei.impl.common.transfer.InputSlotCrafter` are similarly vulnerable.
  - **Paper/Spigot/Purpur**: Vanilla Paper/Spigot/Purpur are **not** affected natively as they do not implement custom REI protocol packet handling.
- **Severity**: High / Critical (Allows players to overstack any item up to 64, including Totems of Undying, Potions, TNT Minecarts, Ender Pearls, etc., bypassing default stack limits and causing significant economic imbalance).
- **Vulnerability Type**: Lack of Stack Size Validation in Packet Handler.

---

## Description
The Leaves server implements support for client-side recipe movement packet (`roughlyenoughitems:move_items_new` / `move_items`) to allow Roughly Enough Items (REI) to automatically transfer items into crafting grids.

The vulnerability resides in how the server processes the `InventorySlots` payload to populate and handle recipe inputs. Specifically:

1. In the packet handler (e.g. `REIServerProtocol.java` / `NewInputSlotCrafter.java` / `InputSlotCrafter.java` depending on version), the server reads the client-supplied list of slot indices to use as source inputs (`InventorySlots`).
2. When the server transfers items into the target crafting slots, it repeatedly executes `slot.getItemStack().grow(1)` (or equivalent code that increments the stack size of the target/accumulator slot) for every ingredient listed in the packet payload.
3. Crucially, the server **does not check whether the target stack size exceeds the item's maximum stack count** (`getMaxCount()`) during this iterative growth.
4. As a result, listing the *same target/accumulator slot* N times in the inputs list allows growing the stack size to N (up to 64), creating overstacked items.

---

## Impact
Players can exploit this packet to:
- Overstack **Totems of Undying** up to 64 in a single slot. These stacks function correctly when held in the offhand or hotbar, meaning a player can effectively carry 64 totems in a single slot and will be virtually unkillable.
- Overstack **TNT Minecarts** up to 64, which can be placed on rails to instigate massive explosive payloads (e.g. in TNT minecart PVP).
- Overstack **Potions** and **Splash Potions** up to 64.
- Overstack **Ender Pearls** up to 64.

---

## Steps to Reproduce (Proof of Concept)
1. Install this Meteor addon on a client.
2. Stand near a container block (e.g., Chest, Shulker Box, Crafting Table) with multiple unstacked target items (e.g. 64 individual Totems of Undying) in your inventory/container.
3. Activate the `rei-overstack` module.
4. The module automatically interacts with the container, sends the custom `roughlyenoughitems:move_items_new` packet listing the accumulator slot 64 times, and immediately consolidates the 64 unstacked totems into a single slot of 64 totems.
5. The player can then safely store or move the overstacked item stack.

---

## Mitigation & Remediation
To fix this vulnerability, the server must validate that growing the stack does not exceed the item's maximum stack size:

In the server's slot-filling implementation (e.g., inside `NewInputSlotCrafter` / `InputSlotCrafter`):
Before incrementing the target stack size, verify that:
```java
if (targetStack.getCount() >= targetStack.getItem().getMaxCount()) {
    // Abort or reject packet processing to prevent overstacking
    return;
}
```
Alternatively, ensure the final stack size is clamped to `targetStack.getItem().getMaxCount()`, and return any excess items to the player's inventory or drop them on the ground.
