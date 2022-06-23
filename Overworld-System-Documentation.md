## Overworld System Documentation

This repo does very little to actually modify the overworld system, only extending the existing one and shuffling things around.

Astute editors of overworlds in the NDS games will be quick to point out that the overworld gfx in the game have a pretty large difference between the number that they appear as in the a081 narc and the number that they are assigned in DSPRE.  This assignment is done by what I choose to call a tag system, each tag corresponding to an entry containing the tag, the graphics it corresponds to, and what I call the callback parameters.  Each of these is a u16, giving 6 bytes for each tag in a table that I have labeled as ``gOWTagToFileNum`` (in ``armips/data/monoverworlds.s``).

The format of each tag is as follows:

```c
struct OVERWORLD_TAG
{
    u16 tag;
    u16 gfx; // index in a081
    u16 callback_params;
};
```

Original overworld tags only went from 428 to 993.  This encompasses all Pokémon with gender differences and forms included--hence why there are 566 of these tags in vanilla HeartGold.

There are then a set of npc Pokémon tags from 993-1049.  For whatever reason, these are all separate from the existing tags that are in the table already.

Great care was taken to rearrange the overworlds to allow space for all the new forms on existing Pokémon.  Alolan, Galarian, Hisuian, and even Cap Pikachu forms all offset every entry afterwards.

There is a separate number that each species has to map it to its entry in a141, which I call the follower ID.  The a141 file chooses other parameters that each Pokémon has, such as ability to enter buildings and how fast it bounces.  In vanilla, this is offset exactly 0x1AC from the tag for all Pokémon--this is how the game gets the tag for each overworld.

Furthermore, there is a table in the vanilla game that maps each species to its follower ID.  I have it in the repo as ``sSpeciesToOWGfx`` (in ``src/pokemon.c``).  Gender differences (if applicable) add 1 to the base follower ID to get the female follower ID, with the male follower ID being the base.  Forms add the form id to the base follower ID to get the form follower ID.

0x1AC is then added to the follower ID to get the tag.  This is then used to retrieve the overworld gfx (the ``gfx`` from ``gOWTagToFileNum``) and how the game should handle it (the ``callback_params`` from ``gOWTagToFileNum``).

Because of the npc Pokémon tags that are present, the solution to this is to split the species down a line whether they are before the npc Pokémon or after.  Because of all of the form offsets, this split now occurs at Lickilicky, and can be seen in the table:

```
.halfword  991,  806, OVERWORLD_SIZE_SMALL // SPECIES_WEAVILE
.halfword  992,  807, OVERWORLD_SIZE_SMALL // SPECIES_MAGNEZONE
.halfword  993,  808, OVERWORLD_SIZE_SMALL // SPECIES_LICKILICKY
// npc mon entries
.halfword 1050,  809, OVERWORLD_SIZE_SMALL // SPECIES_RHYPERIOR
.halfword 1051,  810, OVERWORLD_SIZE_SMALL // SPECIES_TANGROWTH
.halfword 1052,  811, OVERWORLD_SIZE_SMALL // SPECIES_ELECTIVIRE
```

The code that covers this is in ``armips/asm/overworlds.s`` as ``get_mon_ow_tag``.  This split now covers that Rhyperior, with a base follower ID of 566, now needs to map to the tag 1050 (as well as every follower ID above 566).  With Pikachu having both forms and a gender difference, the code handles Pikachu as an edge case where forms actually add ``(form id + 1)`` to the base follower ID.

The main advantage of the tag system is that it allows for us to divorce the tags from the gfx order in a081, allowing us to add more gfx in whatever order we please.  This primarily is useful for adding new forms wherever instead of immediately after the previously Pokémon as has been done already--there is no need to rename the existing Pokémon in ``data/graphics/overworlds/`` to fit the "new" system, and we can add forms wherever we please.

## Editing entries

While there isn't necessarily any reason currently to edit entries, should the need arise, there is quite a bit that comes into play.

Each species is helpfully mapped to its entries, whether in the form of a comment after the entry or preceeding it as its index when in the C array.

For overworlds, everything that you need to edit is in ``src/pokemon.c`` or ``armips/data/monoverworlds.s``.  Apart from adding new Pokémon, ``src/pokemon.c`` may not even need to be touched.

### ``sSpeciesToOWGfx``

In ``src/pokemon.c``, the file starts out with this table that maps each species to its base follower ID (and thus the corresponding file in a141).  This is offset by ``0x1AC`` to get the base species tag if before the Lickilicky species split or ``0x1E4`` if occuring after the species split.

An entry just looks like the species name followed by a number (the whitespace is not important):

```c
[SPECIES_FLAREON            ] =  181,
```

### ``gOWTagToFileNum``

Now we get to ``armips/data/monoverworlds.s``, where the file starts out with ``gOWTagToFileNum``.  This by necessity has to have it for _every_ overworld and not just the Pokémon overworlds, so it is not recommended that you edit anything before line 364 with the comment ``// pokémon follower specific overworlds start here``.

This table then has an entry for each Pokémon and its forms, with the tag, gfx id in a081, and callback enumerated, in that order:

```
.halfword  709,  529, OVERWORLD_SIZE_SMALL // SPECIES_WOBBUFFET
.halfword  710,  530, OVERWORLD_SIZE_SMALL // female
```

Most Pokémon overworlds are 32x32.  Certain species have 64x64 overworlds, denoted here by ``OVERWORLD_SIZE_LARGE``:

```
.halfword  716,  536, OVERWORLD_SIZE_LARGE // SPECIES_STEELIX
.halfword  717,  537, OVERWORLD_SIZE_LARGE // female
```

### ``gDimorphismTable``

This is the table that determines whether or not the Pokémon overworlds have gender differences and thus have a female "form" that adds 1 to the base follower ID.

Each species gets a byte that's either 0 or 1, 0 denoting no gender differences and 1 denoting that there is a gender difference there that appears in the overworld.  An example few entries:

```
/* SPECIES_GLIGAR          */ .byte 0
/* SPECIES_STEELIX         */ .byte 1
/* SPECIES_SNUBBULL        */ .byte 0
```

Gligar, while it has gender differences in its battle sprite, does not have one that is visible in the overworld.  It gets a 0 here.  Steelix has a gender difference that is visible in the overworld.  It gets a 1 here.  Snubbull does not have a gender difference.  It gets a 0 here.

### ``narc a141``

Finally, the armips macro ``overworlddata`` creates the file for each overworld that denotes whether or not it can enter buildings as well as bounce speed.

Each follower ID gets an entry here, which denotes the properties of that follower.  An example entry of a Pokémon that can enter buildings:

```
overworlddata  306, OVERWORLD_CAN_ENTER, OVERWORLD_BOUNCE_FAST // SPECIES_CORSOLA
overworlddata  307, OVERWORLD_CAN_ENTER, OVERWORLD_BOUNCE_FAST // galar
```

An example entry of a Pokémon that can not enter:

```
overworlddata  334,  OVERWORLD_NO_ENTRY, OVERWORLD_BOUNCE_SLOW // SPECIES_LUGIA
overworlddata  335,  OVERWORLD_NO_ENTRY, OVERWORLD_BOUNCE_SLOW // SPECIES_HO_OH
overworlddata  336, OVERWORLD_CAN_ENTER, OVERWORLD_BOUNCE_SLOW // SPECIES_CELEBI
```

Example of Pokémon with forms:

```
overworlddata  438, OVERWORLD_CAN_ENTER,  OVERWORLD_BOUNCE_MED // SPECIES_CASTFORM
overworlddata  439, OVERWORLD_CAN_ENTER,  OVERWORLD_BOUNCE_MED // sunny
overworlddata  440, OVERWORLD_CAN_ENTER,  OVERWORLD_BOUNCE_MED // rainy
overworlddata  441, OVERWORLD_CAN_ENTER,  OVERWORLD_BOUNCE_MED // snowy
```

All of these come together to form a cohesive albeit complicated system for overworlds that hg-engine uses fully.
