# Akumu (Source Dump)

![Status](https://img.shields.io/badge/Status-Decompiled-success)
![Architecture](https://img.shields.io/badge/Target-GameMaker_VM-lightgrey)

This repository is a technical case study in retrieving source code from a compiled GameMaker executable. It contains the Game Maker Language (GML) scripts, object event code, room creation code, GameMaker VM disassembly and asset metadata extracted from a build of the game **Akumu** (object names such as `oFinalDemoMonologue` indicate the demo build).

## Scope and legal notice

- Everything under `src/`, `graphics/`, `sounds/`, `strings/` and the `data.win` container is derived from the game and remains the property of its developer. No license is granted for it; it is published here for study of the decompilation process and of the resulting code only.
- The write-up below (pipeline notes and code observations) is the repository author's own commentary.
- If you are the rights holder and want material removed, open an issue and it will be taken down.

## Technical Overview

The primary objective was to perform static analysis on a compiled PE32 Windows executable from a strict **Arch Linux** environment.

The decompilation pipeline involved:

1.  **Cross-Platform Execution:** Leveraging **Wine and Proton** to run the Windows-native analysis tools.
2.  **Archive Unpacking:** Using **QuickBMS** with custom script definitions to parse the IFF-chunk structure (`FORM` container) of `data.win`.
3.  **Bytecode Translation:** Deserializing the GameMaker Virtual Machine (GMVM) instruction stream into human-readable GML. The raw disassembly is kept next to the decompiled GML under `src/assembly/`.

## Code "Smells"

Reverse engineering reveals not just the *how*, but the *why*—and sometimes, the "why" is terrifying. Below are specific examples of technical debt found within the binary.

### 1. The "Wall of Ifs"
In `oBasementFloor2`, room management is handled via a hardcoded O(n) check that lists every single room manually. If the developer adds one room and forgets to update this list, the object self-destructs.

```gml
// Found in: src/objects/oBasementFloor2_Step_0.gml
if (room == rBasement9 || room == rBasement10 || room == rBasement11 ... ) {
    // Logic
} else {
    instance_destroy(); // The nuclear option
    show_debug_message("destroyed");
}
```

### 2. The "Creepy Key" Serialization
The save system (`SaveGame`) avoids standard structs or arrays in favor of manually mapping variables to INI keys that form... sentences?

```gml
// Found in: src/scripts/SaveGame.gml
ini_write_real("stage1_1", "you", sink);
ini_write_real("stage1_1", "shouldn't", toilet);
ini_write_real("stage1_1", "be", paper);
ini_write_real("stage1_1", "here", all_);
```
*Verdict: Efficient? No. Unsettling? Yes.*

### 3. Debug Artifacts
The developer left a functional debug backdoor in the production build. *wowie.*
```gml
// Found in: src/objects/oGlobal_Step_0.gml
if (keyboard_check(vk_subtract)) {
    room_goto(rDebug); // Instantly teleports the player
}
```

### 4. The "Nervous Furniture"
Static objects (Tables, Bookshelves, Beds) contain code in their `Step` event—which runs 60 times per second—solely to update their depth sorting.

```gml
// Found in: src/objects/oTable2_Step_0.gml
// This table is stationary. It will never move.
// Yet, it recalculates its Z-ordering every single frame.
if (normalDepth)
    depth = -bbox_bottom;
```

### 5. Callback Purgatory
Instead of using a Timeline or a proper State Machine for cutscenes, the code relies on a fragile nesting of anonymous functions inside custom `Delay()` scripts.

```gml
// Found in: src/objects/oFinalDemoMonologue_Step_0.gml
Delay(200, function() {
    havingDialogue++;
    OpenTextBox(...);
    Delay(2000, function() {
        oTextBox.pages = -4;
        OpenTextBox(...);
        Delay(2000, function() { // We have to go deeper
            // Hope the frame rate didn't dip,
            // otherwise this sync is ruined forever.
        });
    });
});
```

### 6. Variable Laundering
In `SaveGame`, global variables are not saved directly. Instead, they are assigned to local variables named... numbers?

```gml
// Found in: src/scripts/SaveGame.gml
var zero = global.dialogue;
var one = global.basementpuzzlecomplete;
var two = global.dialogue2;

// Why save "global.dialogue"?
// Let's save "zero" into key "0" instead.
ini_write_real("stage1", "0", zero);
```

## Repository structure

| Path | Contents | Count |
|---|---|---|
| `data.win` | The game's original GameMaker data container (IFF `FORM`, about 30 MB) | 1 |
| `src/objects/` | Decompiled object event code, one file per event (`<object>_<event>_<n>.gml`) | 404 |
| `src/scripts/` | Decompiled global scripts (`addItem`, `SaveGame`, `LoadGame`, `writeText`, …) | 63 |
| `src/rooms/` | Room creation code (`<room>_<n>_Create.gml`) | 189 |
| `src/assembly/` | GameMaker VM disassembly, one `.asm` per code entry | 656 |
| `graphics/` | Extracted textures, sprites, tilesets, room renders, font glyph tables (`.csv`) and compiled HLSL shader blobs (`.bin`) | 1,733 PNG + 4 CSV + 14 BIN |
| `sounds/` | Extracted audio (`audiogroup_default`) | 84 WAV, 12 OGG |
| `strings/` | The string table as plain text (`dump.txt`) and JSON (`json_dump.json`) | 2 |

Browse the directories directly on GitHub; the file names follow GameMaker's own naming (object, event type, event number), so `oPlayer_Step_0.gml` is the Step event of `oPlayer`.

## Tools

*   **Wine / Proton:** compatibility layers used to run the Windows-only analysis tools on Linux.
*   **QuickBMS:** generic file extractor used for the initial IFF chunk separation.

## Code & Asset Forensics

This is not a critique of the game's assets or code, but rather a constructive analysis of the game's strengths and weaknesses. The game's unique elements are well-designed and executed, but the reliance on other people's work can be seen as a weakness. The game's physics and tweening systems are also a weakness, as they are entirely outsourced.
