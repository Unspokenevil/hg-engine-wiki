## Editing Pokémon Data

The repo builds all information from all Pokémon from files that are already within the repo.  Most of these are text files with the intent of being all of portable, easy to edit, and trackable via git.

Files discussed are all in ``armips/data``.  These are discussed in alphabetical order as follows, or listed as unknown if they are unknown and not covered:
- ``pokedex/sortlists/*.s``
- ``pokedex/000.s`` - unknown
- ``pokedex/002.s`` - unknown
- ``pokedex/areadata.s``
- ``pokedex/femalemonscaledivisor.s`` and ``pokedex/malemonscaledivisor.s``
- ``pokedex/femalemonypos.s`` and ``pokedex/malemonypos.s``
- ``pokedex/femaletrainerscaledivisor.s`` and ``pokedex/maletrainerscaledivisor.s``
- ``pokedex/femaletrainerypos.s`` and ``pokedex/maletrainerypos.s``
- ``pokedex/pokedexdata.s``
- ``pokedex/weight.s``
- ``trainers/trainers.s`` is discussed in Trainer Pokémon Structure Documentation.
- ``babymons.s``
- ``baseexp.s``
- ``eggmoves.s``
- ``evodata.s``
- ``heighttable.s``
- ``hiddenabilities.s`` is discussed in Hidden Ability Documentation.
- ``iconpalettetable.s``
- ``levelupdata.s``
- ``mondata.s``
- ``monoverworlds.s`` is discussed in Overworld System Documentation.
- ``moves.s`` is discussed in Move Data Structure Documentation.
- ``regionaldex.s``
- ``spriteoffsets.s``
- ``tmlearnset.s``
- ``tutordata.s``

Finally, this repo also builds all (most) of the graphics files.  Due to differences in how the sprites are stored, the sprites are formatted differently, so it is important to pay attention to exactly how sprites are formatted to properly use and add entries.

The graphics currently supported are:

- ``data/graphics/icongfx``
- ``data/graphics/overworlds``
- ``data/graphics/sprites``

For sprite manipulation, I personally recommend aseprite.  This tool does not, however, always report the indexed status accurately between 4bpp and 8bpp.  For this purpose, irfanview has always worked well for me personally.

### ``pokedex/sortlists/*.s``

The Dex has options to search for Pokémon within certain criteria, including body form, type, weight, height, and alphabetical by name.

These lists are enumerated in narc a214.  Every one corresponds to a separate category, of which the file name describes the contents of the file.

Every file contains all of the species that belong to that category with a halfword per species that belongs to that file.

``044-061.s`` are all empty.  Not sure at all what they were in the past, if anything.

``bodytype*.s`` each describe all of the species that belong to the body type.  These can be searched on Bulbapedia if need be.

``heaviest.s`` describes all species in order from heaviest to lightest.  When 2 species weigh the same, they are ordered by national dex number.

``lightest.s`` describes all species in order from lightest to heaviest.  When 2 species weigh the same, they are ordered by national dex number.

``name*to*.s`` appear to be unused, but describe the species in alphabetical order that starts with the letters in the filename inclusive.  For example, ``nameJtoL.s`` has all of the species that have names that start with the letter J, K, and L.

``name*.s`` each describe the species in alphabetical order that start with the letter in the filename.  For example, ``nameG.s`` has all of the species that have names that start with the letter G.

``nationalnum.s`` describes all species by national dex number.

``regionalnum.s`` describes the species that appear in the regional dex by the regional dex order.

``smallest.s`` describes all species in order from smallest to tallest.  When 2 species are the same height, they are ordered by national dex number.

``tallest.s`` describes all species in order from tallest to smallest.  When 2 species are the same height, they are ordered by national dex number.

``type*.s`` each describe the species that are at least part that type in national dex number.  For example, ``typeice.s`` has all of the species that are at least part ice type.

### ``pokedex/areadata.s``

``areadata.s`` describes where on the region map a Pokémon can appear.  These are separated into routes and cities vs. special areas.  

Special areas are distinct from routes and cities in that they are the landmarks (i.e. Sprout Tower, Mt. Silver, Mt. Moon, etc.)

Each Pokémon has 8 files, one for each of...
- special areas morning
- special areas day
- special areas night
- routes and cities morning
- routes and cities day
- routes and cities night
- special areas special (flashes red quickly in between yellow flashes)
- routes and cities special

This is clearly separate areas for each time of day as well.  The "special" time of day typically refers to Headbutt trees.

The narc has all of these sequentially for each Pokémon in this fashion:
- special areas morning 0
- special areas morning bulbasaur
- special areas morning ivysaur
- special areas morning venusaur
- ...
- special areas morning arceus
- special areas day bulbasaur
- special areas day ivysaur
- ...
- special areas day arceus
- ...

And so on consecutively in the order the 8 files are described above.  This leads to the files describing Bulbasaur being 3, 498, 993, 1488, 1983, 2478, 2973, and 3468.

As just explained, the narc a133 is structured very annoyingly for this application.

The macros take care of it and allow for each species to be grouped together for ease of editing.

Each area has a 4-byte word constant assigned to it that will flag where the dex should flash to show that the species shows up there.  These are enumerated at line 120 of ``armips/include/constants.s``.

Thus each Pokémon has its area data defined like that.  Each file is a list of 4-byte words that correspond to an area that will flash when hovering over the Pokémon.

An example in Larvitar, who appears in Mt. Silver all throughout the day and nowhere else:

```
specialareas SPECIES_LARVITAR, DEX_MORNING
    .word DEX_MT_SILVER_CAVE
    dexendareadata


specialareas SPECIES_LARVITAR, DEX_DAY
    .word DEX_MT_SILVER_CAVE
    dexendareadata


specialareas SPECIES_LARVITAR, DEX_NIGHT
    .word DEX_MT_SILVER_CAVE
    dexendareadata


routesandcities SPECIES_LARVITAR, DEX_MORNING
    dexendareadata


routesandcities SPECIES_LARVITAR, DEX_DAY
    dexendareadata


routesandcities SPECIES_LARVITAR, DEX_NIGHT
    dexendareadata


specialareas SPECIES_LARVITAR, DEX_SPECIAL
    dexendareadata


routesandcities SPECIES_LARVITAR, DEX_SPECIAL
    dexendareadata
```

Or Houndour, who only appears in Route 7 of Kanto at night:

```
specialareas SPECIES_HOUNDOUR, DEX_MORNING
    dexendareadata


specialareas SPECIES_HOUNDOUR, DEX_DAY
    dexendareadata


specialareas SPECIES_HOUNDOUR, DEX_NIGHT
    dexendareadata


routesandcities SPECIES_HOUNDOUR, DEX_MORNING
    dexendareadata


routesandcities SPECIES_HOUNDOUR, DEX_DAY
    dexendareadata


routesandcities SPECIES_HOUNDOUR, DEX_NIGHT
    .word DEX_ROUTE_7
    dexendareadata


specialareas SPECIES_HOUNDOUR, DEX_SPECIAL
    dexendareadata


routesandcities SPECIES_HOUNDOUR, DEX_SPECIAL
    dexendareadata
```

Entries can get rather complicated, as does Hoothoot's, who looks to appear just about everywhere via special encounter.

### ``pokedex/femalemonscaledivisor.s`` and ``pokedex/malemonscaledivisor.s``

In the Dex, there is a tab dedicated to showing off how the player sizes up to the Pokémon species currently being viewed and the weight comparison.  This tab has a number of things that go into it.

The Dex individually scales the Pokémon silhouette and the player silhouette differently depending on whether or not the player is female.  Pokémon are often scaled identically between the two player genders, while the player sprite is scaled down a little more for the female player.  This is likely because the female sprite is noticeably taller, so the denominator as defined in ``femaletrainerscaledivisor`` is bigger.

The sprites are then individually shfited down pixels to have a consistent ground level between them that roughly coincides with the bottom of the Dex box allocated for them.

For obvious reasons, this is asinine to keep up, especially as most players will never see the two perspectives, let alone side by side.  Additions Gen 5 and up just use the same for both instances.

That being said...

The ``fe/malemonscaledivisor.s`` are files containing 2-byte halfword numbers for each Pokémon that are applied as the denominator to the size calculation that determines how the Pokémon sprite is scaled relative to the base 80x80 sprite when the player is female (femalemonscaledivisor) or male (malemonscaledivisor).

The numerator in this equation is ``0x100``, so the divisor is defined to decide on scale.  A divisor less than ``0x100`` will increase the size.

### ``pokedex/femalemonypos.s`` and ``pokedex/malemonypos.s``

See the start of the description in the section above for an explanation of what goes into sprite manipulation in the Dex.

The ``monypos`` files have a 2-byte halfword for each Pokémon.

``pokedex/femalemonypos.s`` defines how many pixels the Pokémon sprite is shifted down when the player is female.  The sprite is scaled then shifted, so this happens second.

``pokedex/malemonypos.s`` defines how many pixels the Pokémon sprite is shifted down when the player is male.  The sprite is scaled then shifted, so this happens second.

The amount shifted by can be negative.

### ``pokedex/femaletrainerscaledivisor.s`` and ``pokedex/maletrainerscaledivisor.s``

See the start of the description in the section above for an explanation of what goes into sprite manipulation in the Dex.

The ``trainerscaledivisor`` files have a 2-byte halfword for each Pokémon.

``pokedex/femaletrainerscaledivisor.s`` defines the denominator in the scaling equation applied to the trainer sprite when the player is female.  This is with a numerator of 0x100.  Denominators less than 0x100 increase the size of the sprite.

``pokedex/maletrainerscaledivisor.s`` defines the denominator in the scaling equation applied to the trainer sprite when the player is male.  This is with a numerator of 0x100.  Denominators less than 0x100 increase the size of the sprite.

### ``pokedex/femaletrainerypos.s`` and ``pokedex/maletrainerypos.s``

See the start of the description in the section above for an explanation of what goes into sprite manipulation in the Dex.

The ``trainerypos`` files have a 2-byte halfword for each Pokémon.

``pokedex/femaletrainerypos.s`` defines how many pixels the trainer sprite is shifted down when the player is female.  The sprite is scaled then shifted, so this happens second.

``pokedex/maletrainerypos.s`` defines how many pixels the trainer sprite is shifted down when the player is male.  The sprite is scaled then shifted, so this happens second.

The amount shifted by can be negative.

### ``pokedex/pokedexdata.s``

This file includes every dex file to prevent having to establish via Makefile that every file needs to be built.  It shouldn't be edited unless you know _exactly_ what you are doing.

### ``pokedex/weight.s``

This file has a 4-byte word per Pokémon that defines how many hectograms (0.1 kg) it weighs.  For example, Bulbasaur is 6.9 kg, so the word stored for Bulbasaur is 69.

This is actually used not just for weight comparison, but also for weight damage calculation for moves like Low Kick.

### ``babymons.s``

This file has a 2-byte halfword per Pokémon that defines which species hatch from eggs that this Pokémon has.  Incense baby Pokémon are handled as exceptions in the code (i.e. Pikachu list Pichu as their entries in this file).

An example entry for an evolutionary line:

```
babymon SPECIES_TRAPINCH, SPECIES_TRAPINCH
babymon SPECIES_VIBRAVA, SPECIES_TRAPINCH
babymon SPECIES_FLYGON, SPECIES_TRAPINCH
```

### ``baseexp.s``

Starting in generation 5, Pokémon have base experience yields of over 255.  This is the file that lists all of them for every Pokémon, with a 2-byte halfword entry per Pokémon that defines the base experience as used in the calculator.  This is part of the ``Experience Overhaul Documentation`` as well.

### ``eggmoves.s``

Each first-in-its-line species has an entry here that describes the moves that it has.  It is technically a big 2-byte halfword array with each species being read as a move, so to speak--the species are offset by a constant 40,000 to denote that a new species is being read.

This is made nicely readable in this repository.

Note that any species that can come out of an egg will have moves defined:

```
eggmoveentry SPECIES_MARILL
    eggmove MOVE_LIGHT_SCREEN
    eggmove MOVE_PRESENT
    eggmove MOVE_AMNESIA
    eggmove MOVE_FUTURE_SIGHT
    eggmove MOVE_BELLY_DRUM
    eggmove MOVE_PERISH_SONG
    eggmove MOVE_SUPERSONIC
    eggmove MOVE_SUBSTITUTE
    eggmove MOVE_AQUA_JET
    eggmove MOVE_SUPERPOWER
    eggmove MOVE_REFRESH
    eggmove MOVE_BODY_SLAM
```

Here, Marill gets an entry despite the baby Azurill existing.  Likewise, Pikachu, Wobbuffet, and others of their kind get entries.

Pending further research, each Pokémon is limited to 16 egg moves.

### ``evodata.s``

Evolutions have a file per Pokémon describing method of evolution, parameter for that method, and the species that it evolves into.  Each of these is a 2-byte halfword, totaling 6 bytes per evolution.

Vanilla HGSS has 7 evolutions defined for each Pokémon.  This repo has expanded that to 9 to support Eevee and whatever happens in the upcoming generations.

Each entry needs to be padded out to the limit to ensure that no invalid data is read.

This makes an example look like this:

```
evodata SPECIES_GLOOM
    evolution EVO_USE_ITEM, ITEM_LEAF_STONE, SPECIES_VILEPLUME
    evolution EVO_USE_ITEM, ITEM_SUN_STONE, SPECIES_BELLOSSOM
    evolution EVO_NONE, 0, SPECIES_NONE
    evolution EVO_NONE, 0, SPECIES_NONE
    evolution EVO_NONE, 0, SPECIES_NONE
    evolution EVO_NONE, 0, SPECIES_NONE
    evolution EVO_NONE, 0, SPECIES_NONE
    evolution EVO_NONE, 0, SPECIES_NONE
    evolution EVO_NONE, 0, SPECIES_NONE
    terminateevodata
```

The currently supported evolution methods:

```
EVO_NONE                      equ 0
EVO_HAPPINESS                 equ 1
EVO_HAPPINESS_DAY             equ 2
EVO_HAPPINESS_NIGHT           equ 3
EVO_LEVEL_UP                  equ 4
EVO_TRADE                     equ 5
EVO_TRADE_ITEM                equ 6
EVO_USE_ITEM                  equ 7
EVO_LEVEL_MORE_ATTACK         equ 8
EVO_LEVEL_ATK_DEF_EQUAL       equ 9
EVO_LEVEL_MORE_DEFENSE        equ 10
EVO_LEVEL_PID_LOW             equ 11
EVO_LEVEL_PID_HIGH            equ 12
EVO_LEVEL_GEN_NEW_MON_1       equ 13
EVO_LEVEL_GEN_NEW_MON_2       equ 14
EVO_MAX_BEAUTY                equ 15
EVO_USE_ITEM_MALE             equ 16
EVO_USE_ITEM_FEMALE           equ 17
EVO_USE_ITEM_DAY              equ 18
EVO_USE_ITEM_NIGHT            equ 19
EVO_KNOWS_MOVE                equ 20
EVO_MON_IN_PARTY              equ 21
EVO_LEVEL_MALE                equ 22
EVO_LEVEL_FEMALE              equ 23
EVO_LEVEL_ELECTRIC_FIELD      equ 24
EVO_LEVEL_MOSSY_STONE         equ 25
EVO_LEVEL_ICY_STONE           equ 26
```

Pending further research, this is what we are sticking with.  Placeholder evolutions exist for Pokémon that will need extra programming with completely new methods to determine how to evolve them.

### ``heighttable.s``

This file describes the in-battle shift downwards in pixels that happens to the sprite when displayed.  Each Pokémon gets 4 files in the narc, each a byte long and sequentially positioned.

Each file, in order, is the one that shifts the female back, male back, female front, and male front in that order.

An example of what this looks like in the file:

```
heightentry SPECIES_NIDORAN_M, "null", 11, "null", 20
```

Here, the female NidoranM sprite doesn't exist, so the file for each shift to the female sprite doesn't exist.  The male back sprite is shifted down 11 pixels, and the male front sprite is shifted down 20 pixels.

Species that are Gender Unknown by default only have a male sprite and will have null entries here.  Often this is too cumbersome to keep up, so many of the new Pokémon just have sprites for them defined regardless.

Pokeditor supports visualizing this very well and is recommended for getting values for this.

### ``iconpalettetable.s``

In this table located in the synthetic overlay, each Pokémon gets a byte that is 0, 1, or 2 that determines which of the 3 palettes are used for Pokémon icons that are displayed.

Forms are originally coded to load an icon from after the end of the normal species defines.  This is actually the primary decision to make new species start at 544 instead of 494--old form icons take all the way up until 543.  As a result, new form icons function similarly.

Exceptions involving gender are coded on a case-by-case basis, and are currently just Frillish, Jellicent, Meowstic, and Pyroar.

### ``levelupdata.s``

Each Pokémon gets a file in the narc detailing its learnset.  By default, the game supports learnsets of length 24 moves and move indices up to 511.  hg-engine has expanded this to learnsets of 30 moves with indices up to 65,535 each.

An example learnset:

```
levelup SPECIES_SWADLOON
    learnset MOVE_GRASS_WHISTLE, 1
    learnset MOVE_TACKLE, 1
    learnset MOVE_STRING_SHOT, 1
    learnset MOVE_BUG_BITE, 1
    learnset MOVE_RAZOR_LEAF, 1
    learnset MOVE_PROTECT, 20
    terminatelearnset
```

Moves that are learned on evolution are not yet supported.  However, when they are, it will be coded as a move learned at level 0 and located before all the other moves, similar to how Gen 7+ handles it.

### ``mondata.s``

Each Pokémon gets a file in the personal narc detailing a number of things, particularly base stats, TM learnsets, gender ratio (number of Pokémon that will be female out of 254), ev yields, etc.

Hidden abilities are not included here to maintain compatibility with existing tools.

TM learnsets appear here as constants that are defined in a separate file, ``tmlearnset.s``.  This is to avoid cluttering this file.

``basestats`` and ``evyields`` are listed in the order HP, Attack, Defense, Speed, Sp. Attack, and Sp. Defense, as they are internally.

Example ``mondata`` entry:

```
mondata SPECIES_GLIGAR
    basestats 65, 75, 105, 85, 35, 65
    types TYPE_GROUND, TYPE_FLYING
    catchrate 60
    baseexp 108
    evyields 0, 0, 1, 0, 0, 0
    items ITEM_NONE, ITEM_NONE
    genderratio 127
    eggcycles 20
    basefriendship 70
    growthrate GROWTH_MEDIUM_SLOW
    egggroups EGG_GROUP_BUG, EGG_GROUP_BUG
    abilities ABILITY_HYPER_CUTTER, ABILITY_SAND_VEIL
    runchance 0
    colorflip BODY_COLOR_PURPLE, 0
    tmdata SPECIES_GLIGAR_TM_DATA_0, SPECIES_GLIGAR_TM_DATA_1, SPECIES_GLIGAR_TM_DATA_2, SPECIES_GLIGAR_TM_DATA_3
```

### ``regionaldex.s``

Each Pokémon gets a halfword that tells the game the regional dex number of the Pokémon, 0 if the Pokémon is not in the regional dex.  While ``pokedex/sortlists/regionalnum.s`` tells whether or not the Pokémon appears in the regional dex, the number that is loaded from here deterines what number is displayed next the the Pokémon in the Dex as well as in the summary screen.

### ``spriteoffsets.s``

Each Pokémon gets 0x59 bytes sequentially that describes primarily the animation that it receives.  Inside of this, however, are also global sprite descriptions that aren't fully known yet but have entirely to do with the animations that play for each species when viewed in the summary screen or when coming out of a Ball.

New mons are prefilled with Bulbasaur's data as placeholder.

The byte at offset 1 of each species' entry is the front animation.  The byte at offset 0x23 is the back animation.  These can be any sort of numbers and (pending further research) don't seem to match up with anything as far as we know.  More knowledge will involve looking at the code, but we also have perfectly functional examples as it is, so this is not a high priority.

The byte at offset 0x2C describes the back animation.

There are then 3 bytes at offset 0x56 that each describe the y offset (moving down being positive), the shadow's x offset (moving right being positive), and the size of the shadow (0-3 for no shadow to largest shadow).

This appears in ``sprietoffsets.s`` as:

```
dataentry SPECIES_SIMIPOUR  ,  2, 5, -7,  -1,  SHADOW_SIZE_MEDIUM
```

Where the format is:

```
dataentry species, monfrontanim, monbackanim, monoffy, shadowoffx, shadowsize
```

### ``tmlearnset.s``

This file is included by ``mondata.s`` to define the TM's that each species can learn.

The format is 4 4-byte words for a total of 128 bits.  TM's 1-32 go in the first word, TM's 33-64 go in the second word, TM's 65-92 and HM's 1-4 in the third word, and HM's 5-8 in the final word.

There are constants defined for each TM that look like TM### where each TM has its own number.  The ability to change which move is taught by which TM is currently in the works, but manually updating this list is an exercise left to the user.

Each new constant needs to be `or`'d with each other to ensure that all of the data is properly put in the array.

Expanding this to include more TM's is also in the works.  There are currently placeholder constants for every base form through Enamorus.

### ``tutordata.s``

Each species gets 2 4-byte words to define the Tutor moves that each species can learn.  This goes for a total of 64 bits, of which 52 are used--all from the first word and 20 from the second.

Each move can only be handled within its own word--``TUTOR_DIVE`` when placed into the second word will correspond to ``TUTOR_TRICK`` and will teach that instead.  Care needs to be exercised to ensure that everything is configured properly here.

A particularly involved tutor move setup is Lugia's example:

```
tutordata SPECIES_LUGIA, \
                  TUTOR_DIVE | TUTOR_MUD_SLAP | TUTOR_ICY_WIND | \
                  TUTOR_IRON_HEAD | TUTOR_AQUA_TAIL | TUTOR_OMINOUS_WIND | TUTOR_SNORE | TUTOR_AIR_CUTTER | \
                  TUTOR_ANCIENT_POWER | TUTOR_SIGNAL_BEAM | TUTOR_ZEN_HEADBUTT | \
                  TUTOR_EARTH_POWER | TUTOR_TWISTER  | 0, \
                  TUTOR_TRICK | TUTOR_SWIFT | \
                  TUTOR_TAILWIND | \
                  TUTOR_SKY_ATTACK | TUTOR_HEADBUTT | \
                  0
```

Here, everything before ``TUTOR_TRICK`` is in the first word (while everything after and including is in the second word).  To armips, the ``\\`` character shows that the same line is continued on the next line--this is to ensure that nothing is graphically cluttered when dumping and is not necessary.

Each new constant needs to be `or`'d with all the other ``TUTOR_*`` constants in its word, all of it coming together to describe the moves that the Pokémon can learn.

Changing tutor moves is actually currently documented inside of ``documentation/ov1_23AE0_movetutordata.s``.  This just needs to be adapted to the repo format for use.  There are not currently plans to expand this.

An example of the first few tutor move entries:

```
tutormove MOVE_DIVE, 40, TUTOR_TOP_LEFT
tutormove MOVE_MUD_SLAP, 32, TUTOR_TOP_RIGHT
tutormove MOVE_FURY_CUTTER, 32, TUTOR_TOP_LEFT
```

With the format being:

```
tutormove move, bp, tutorid
```

Note that the Headbutt tutor has its own ID and actually has an entry here where it costs 0 BP to teach.  I am unaware if changing this will automatically reflect it.

The ``TOP_LEFT``, ``TOP_RIGHT``, and ``BOTTOM_RIGHT`` designations are based on the tutors' locations in the house and are arbitrary in that sense.  There is likely a script command parameter differentiating all of them, so this may be really easy to expand if there aren't any limiters and the code is all based around this table.

### ``data/graphics/icongfx``

The name format for this is a 4-letter number containing the species.  The image must be indexed to one of the 3 already-existing icon palettes and must be 4bpp in that each pixel is 4 bits, and building this is handled by ``nitrogfx``, the tool from the decomps.  Ideally, all of the sprites are built using this tool eventually, as it is very flexible.

### ``data/graphics/overworlds``

The name format for this is a 4-letter number containing the entry id as it appears in narc a081.  This should correspond with the middle u16 from the overworld table (see Overworld System Documentation).

The image handler from this is very versatile and indexes the image on its own.  However, a limit of 16 colors (including the background) must be observed--the tool will error out otherwise.

### ``data/graphics/sprites``

Each species gets a folder with its 4-digit ID on it.  Inside of this folder, there are 2 more folders, male and female.  Finally, each of these folders has a ``front.png`` and a ``back.png`` with a ``.key`` file for each.  The shiny palette is derived entirely from the male back sprite in the instance it exist, otherwise the female back sprite is used, while the normal palette is derived from the male front sprite (the female front sprite if the male front sprite doesn't exist).

Be careful in creating shiny backsprite palettes to ensure that all colors are handled properly.  Genderless Pokémon only have male sprites.  Solely male or female Pokémon only have the sprites that correspond with their gender.

Similar to the icons, ``nitrogfx`` is used for these.  All the sprites that are currently in the repository have their 0-index color set to be transparent, but this is not necessary at all.

The top left 2 pixels in the first row also can not be any sprite data--they must be transparent.  This is due to how the encryption key is stored in the file, and it being messed up when other data is present there.  The ``.key`` files contain this encryption key, and is just a 4-byte file containing the key.  This can safely be copied from any other sprite and function just fine.

## Needs Further Research

Pending further research, there are a few things that are currently not editable by this repo.  These are all TODO items.

Known instances of these are:
- Pokéathlon stats
- dex forms that appear when viewing the forms tab (default is gender)
- where the dex screen flashes when a species appears there (can edit where a species appears within the constraints of the areas already defined)
- adding new evolution methods
- adding new TM's
- editing the moves taught by TM's
- footprints
