# Move Scripting Systems Documentation
Moves in Heart Gold have 2 of their own bytecode scripting languages that correspond with certain pieces of code that would have to be repeated constantly across moves.  This is similar in function to the various scripting systems from Gen 3 as well.

Like Gen 3, there are move battle scripts and move animation scripts.  Also like Gen 3, the battle scripting language is also used for various other things--handling ability stat raises, handling leftovers' HP increase, setting the weather from the overworld, etc.  Most of these are in combination with assembly code that is spread throughout the battle system--the code checks for the Drizzle ability, then queues up a battle script that sets rain.  The code checks for the Frisk ability, then queues up a battle script that handles the message being displayed.  The code checks for Mold Breaker on switch in, then queues a battle script to actually display the message that the Pokémon is breaking the mold.

Within these battle scripts, there are 3 separate narc files that each are different in function, in order they will be covered:
- ``battle_move_seq``
- ``battle_eff_seq``
- ``battle_sub_seq``

Move animation scripts are often called within battle scripts to show something happening, be it the Pokémon sliding around or small particles coming up to show that the Pokémon is honing its claws.  In addition to this, every single move has its own animation that is played when the move is used.  There are two classes of these, and admittedly I'm not too sure when the latter is used, but they are dumped in this repo for completeness and future-proofing:
- ``move_anim``
- ``move_sub_anim``

A lot of what this documentation will deal with is the initial output of the script dumper from well over a year ago.  Many script command names are in progress.  Many fields hadn't (still haven't!) been identified properly as vars/battlers/etc. and are thus dumped as raw numbers.  ``printmessage`` in particular is all of common, fully known, and dumped such that all of its parameters are initially raw numbers.  It is not all that high a priority to revamp the initial dump such that it's perfectly readable, instead preferring to add new entries/revamp old ones as they become relevant.  The initial dump will be shown, then the "prettyified" version will be commented next to it--just a better-readable version that corresponds to the same thing/interpretation of the value.

## ``battle_move_seq`` - a000
Every single move has a script in this narc.  Very few moves actually have substance in their script in here, instead jumping to the current ``battle_eff_seq`` script corresponding to the move that was selected.  Adding new moves requires that a new script be added to this narc, else no script will be dumped when the game looks for a move to run and the game crashes.  The ``battle_eff_seq`` file index that is chosen is the same as the ``battleeffect`` field in the moves structure.

## ``battle_eff_seq`` - a030
The first two bytes of the file with the move's index in the move data narc correspond with the script that is run from the ``battle_eff_seq`` narc.  These often primarily just do setup and queue up a ``battle_sub_seq``, interestingly enough.  A notable exception is Judgment's ``battle_eff_seq`` script, which determines the type that Judgment is going to be when used.  Similarly, 

## ``battle_sub_seq`` - a001
These scripts are largely where the code that actually does effects are run.  So a move will run through its ``battle_move_seq``, which queues up a corresponding ``battle_eff_seq``, most of which finally queue up a ``battle_sub_seq`` that executes the effect.

## ``move_anim`` - a010
Every move has its own animation in this file, and they are performed by these scripts using the .SPA particle files from a029.  There is no simple editor for the .SPA files as of yet despite documentation existing, so we currently just steal them from the Gen 5 games and see if they work in Heart Gold well enough.  Most ``move_anim`` scripts just load a particle file (the .SPA handles most of the movement of said particle), place particles from the file, and play a sound.

## ``move_sub_anim`` - a061
I believe this is used intermittently for when an animation needs to happen and there isn't a move slot that has the animation already defined.  I haven't looked too much into this as of yet, but things like Thief's animation for stealing an item (as opposed to the attack animation) and Pokémon using an item to heal are probably in this narc.

# Battle Script Examples
I personally learn primarily by example.  A lot of what the beginning stages of battle script editing were in Gens 2 and 3 was mashing hexadecimal together and seeing what all worked--before there were any competent editors (and even for a while after there were, as people preferred pasting assembled hex into the rom directly for some reason).  A similar approach can be taken for this, identifying certain blocks that look like, when isolated, can be ported and placed wherever.

## Example 1 - Tackle
To start out, we can look at Tackle and how it works.  Tackle, with move index 33, uses ``armips/move/battle_move_seq/033.s`` as the script that it immediately executes:
```
a000_033:
    jumptocurmoveeffectscript
```
This is the script that is present for almost all moves.  Because of this, I will omit looking at ``battle_move_seq`` in future examples to save space--what you can take away is basically that the ``battle_eff_seq`` is what is the main script and that a ``battle_move_seq`` script needs to be present for the mvoe lest the game crashes.  Most moves just jump to the current move effect script.  Looking at the ``battleeffect`` field from its move data entry (see ``armips/data/moves.s``):
```
movedata MOVE_TACKLE
    battleeffect 0
    pss SPLIT_PHYSICAL
    basepower 40
...
```
With a ``battleeffect`` field of 0, the script jumps to ``battle_eff_seq`` script 0 (``armips/move/battle_eff_seq/000.s``).

This script is then the basic damage-dealing script with no bells or whistles:
```
a030_000:
    critcalc
    damagecalc
    endscript
```
No ``battle_sub_seq`` is run in this case because the effect completely handles it by itself.
```
critcalc
calculates the critical multiplier (set to 1 in the case that there isn't one)

damagecalc
the basic damage calculator.  called for all damaging moves, but will disable itself if a separate damage calc is detected

endscript
ends the script and hands exection back to the overall battle engine if nothing else is queued
```
From there, we can look at other simple cases:  moves that lower the target's stats.  We can look at two examples of this to determine the differences, and this also introduces another core concept in move scripts with the connection between ``battle_eff_seq`` and ``battle_sub_seq``.

## Example 2 - Tail Whip/Growl
Let's take a look at the battle scripts of Growl and Tail Whip:
```
movedata MOVE_TAIL_WHIP
    battleeffect 19
    pss SPLIT_STATUS
    basepower 0
...

movedata MOVE_GROWL
    battleeffect 18
    pss SPLIT_STATUS
    basepower 0
...
```
``armips/move/battle_eff_seq/019.s`` (Tail Whip):
```
a030_019:
    changevar VAR_OP_SET, VAR_ADD_STATUS1, 0x80000017
    endscript
```
``armips/move/battle_eff_seq/018.s`` (Growl):
```
a030_018:
    changevar VAR_OP_SET, VAR_ADD_STATUS1, 0x80000016
    endscript
```

These ``battle_eff_seq`` scripts are queuing up a ``battle_sub_seq`` script that is executed after the ``endscript`` command seen here.

This script also introduces the concept of battle variables.  Unlike overworld scripting variables, battle variables all have a specific purpose (as they are a way for the script commands to directly access and manipulate fields from the massive battle structure that is used in the code).
These variables are mostly manipulated through the ``changevar`` script command, which has 3 parameters:
```
changevar operator, var, value
perform math operations on "var" using "value"
- operator is the math operation done on the variable
- var is the variable to change
- value is the argument for the operator
```
A list of operators for use with ``changevar`` are in the Battle Script Command Reference at the end.

The variable ``VAR_ADD_STATUS1`` tells the engine code that there is still something to be done to a Pokémon on the field when it is nonzero.  The engine code then moves to interpret ``VAR_ADD_STATUS1`` to map it to another battle script with the move's effect described.

There are two parts to the value that it sets:  the first part in the most significant bits (the 0x80000000).  More on those later.  The rest of it is a way that the game grabs the desired subscript.

We can break down Tail Whip's script as such:
```
a030_019:
    changevar VAR_OP_SET, VAR_ADD_STATUS1, 0x80000000 | 23 // 0x17 = 23, hexadecimal to decimal
    endscript
```
The most significant bits are dedicated to determining which Pokémon on the field the effect should apply to.  0x80000000 signals to the code that the target is the Pokémon that the effect applies to, and 0x40000000 signals that the effect applies to the attacker.  There are a few more values as well, but those are the more important ones at the moment.  These have convenient defines in ``armips/include/battlescriptcmd.s`` that allow us to break this down further:
```
a030_019:
    changevar VAR_OP_SET, VAR_ADD_STATUS1, ADD_STATUS_DEFENDER | 23 // 0x17 = 23, hexadecimal to decimal
    endscript
```
Now we know that this battle script tells the game to apply a certain effect to the defender.

The ``23`` correlates to a ``battle_sub_seq`` script through the table ``move_effect_to_subscripts`` in ``src/moves.c``:
```c
u32 move_effect_to_subscripts[] =
{
// ...
    [ 22] =  12, // attack -1
    [ 23] =  12, // defense -1
    [ 24] =  12, // speed -1
// ...
};
```
Now ``battle_sub_seq`` script 12 (``armips/move/battle_sub_seq/012.s``) is responsible for every single stat adjustment in battles, just fed slightly different parameters.  The modularity makes it difficult to understand, and we may cover it more comprehensively later.  Just know off the bat that it is responsible for stat changes--it will come up later.

The big takeaway currently is that Growl, when treated similarly to Tail Whip:
```
a030_018:
    changevar VAR_OP_SET, VAR_ADD_STATUS1, ADD_STATUS_DEFENDER | 22 // 0x16 = 22 decimal
    endscript
```
That 22 gives the ``attack -1`` entry from ``move_effect_to_subscripts``, right above Tail Whip's ``defense -1``.

## Example 3 - Rain Dance
A sort of complicated ``battle_eff_seq`` script that goes into a simple ``battle_sub_seq`` script, Rain Dance:
```
movedata MOVE_RAIN_DANCE
    battleeffect 136
    pss SPLIT_STATUS
    basepower 0
...
```
``armips/move/battle_eff_seq/136.s``:
```
a030_136:
    if IF_MASK, VAR_FIELD_EFFECT, 0x3, _0094
    preparemessage 0x31F, 0x0, "NaN", "NaN", "NaN", "NaN", "NaN", "NaN"
    changevar VAR_OP_CLEARMASK, VAR_FIELD_EFFECT, 0x80FF
    changevar VAR_OP_SETMASK, VAR_FIELD_EFFECT, 0x1
    changevar VAR_OP_SET, VAR_WEATHER_TURNS, 0x5
    changevar VAR_OP_SET, VAR_ADD_STATUS2, 0x2000005D
    checkitemeffect 0x1, BATTLER_ATTACKER, 0x71, _0090
    getitempower BATTLER_ATTACKER, VAR_09
    changevar2 VAR_OP_ADD, VAR_WEATHER_TURNS, VAR_09
_0090:
    endscript
_0094:
    changevar VAR_OP_SETMASK, VAR_10, 0x40
    endscript
```
Here we are introduced to some control flow things that the script system supports as well as some new commands:
```
if operator, var, value, address
conditional flow command
- operator is the math operation done on the variable, see Battle Script Command Reference
- var is the variable with the value to test
- value is the argument for the operator, always a constant for if
- address is the destination that the script will jump to if the if operator returns true

preparemessage id, tag, battlers 1-6
prepares a message to be used by printpreparedmessage
- id is the message index in the narc a027 file 197
- tag determines the strings that are buffered in which order from the Pokémon on the field
- the battlers determine which battlers on the field to grab information to buffer strings from
  - the tag detemines how many battlers are specified--"NaN" signifies that no battler will be built from that parameter

printpreparedmessage
prints the message prepared by "preparemessage"

checkitemeffect checker, battler, effect, address
conditional flow command that is based on item effect
- when "checker" is true, checkitemeffect jumps to "address" if battler doesn't have the item with effect "effect"
  - if false, checkitemeffect jumps to "address" if the battler does have the item with effect "effect"
- battler is the battler to check against
- effect is the held item effect to compare to
- address is the address to jump to

getitempower battler, variable
grabs the item power field from the item data narc and puts it in variable
- battler is the battler that has the item to grab the item power from
- variable is the variable to store the item power in

changevar2 operator, destvar, srcvar
changevar except the constant is now a variable
- operator is the same as changevar
- destvar is a variable that may or may not hold a value already that will be changed
- srcvar is a variable that operator uses to complete its operation
```
DSPRE introduces a separation between what it calls "scripts" and "functions."  Those do not exist here at all, at least not how DSPRE handles what the game is actually doing.  Conditional execution is handled much similarly as in raw assembly:  there are a number of conditional branches that go to other areas, and identicality to the original script files was a goal of this system to prevent complications.

Now to break down what exactly is happening in the script.

The ``if`` at the beginning checks to make sure that the field effect variable doesn't already have the rain bits set.  If the rain bits are already set, then the script jumps to ``_0094`` where ``VAR_10``, which stores move results among other things, has its "move failed" bit set.  If the move does not fail, the script continues at the ``preparemessage`` immediately following.  The ``preparemessage`` takes its ``id`` as 0x31F (799), which according to ``data/text/197.txt`` is the line ``It started to rain!``  As there are no string buffers in this, there is nothing to buffer, so it takes a ``tag`` of ``0x0`` and just prepares that message to print.  The ``changevar`` that clears the mask ``0x80FF`` from the field effect variable is ensuring that all other weathers that are active at the time of using Rain Dance are dissipated.  The script then sets the rain bit, sets the turns to 5, and queues up a ``battle_sub_seq`` script.  The Damp Rock's item effect is checked for, and if the attacker doesn't have a Damp Rock (as the ``checker`` is ``0x1``), then the script ends and jumps to ``_0090``.  Otherwise, the ``VAR_WEATHER_TURNS`` has the item power field from the ``getitempower`` script command added to it.  VAR_09 is typically used as a (very) temporary variable--random number calculations are stored there on top of other calculation interim steps that are necessary, both in battle script usage as well as the battle engine.

The script queues up the ``0x5D (93)`` status effect in the var and says that the target is the temporary work battler (``0x200000000``).  Looking at ``move_effect_to_subscripts`` again...
```c
u32 move_effect_to_subscripts[] =
{
// ...
    [ 92] = 101,
    [ 93] = 103, // subscript 103 is the queued battle_sub_seq script
    [ 94] = 105,
// ...
};
```
This takes us down a rabbit trail (``armips/move/battle_sub_seq/103.s``):
```
a001_103:
    gotosubscript 57
    endscript
```
New command, but the name is pretty explanatory:
```
gotosubscript num
calls battle_sub_seq script "num".  returns to the caller after an endscript is reached
- num is the index of the ``battle_sub_seq`` script to jump to
```
Which ends at (``armips/move/battle_sub_seq/057.s``):
```
a001_057:
    printpreparedmessage
    waitmessage
    wait 0x1E
    endscript
```
Two new script commands:
```
wait time
pause script execution for "time" frames
- time is the amount of frames to pause for

waitmessage
pause script execution until current message is done printing.  not just done for messages, also used for various states that take up time that script execution needs to pause for (although not animations).
```
This script just prints the prepared message from the ``preparemessage`` command back in ``armips/move/battle_eff_seq/136.s``, waits for 30 frames, and ends the script.

The queued ``battle_sub_seq`` script, despite taking us through another ``battle_sub_seq`` script, is solely responsible for printing the message prepared in the ``battle_eff_seq`` script!  

There are a number of "optimizations" like this that are done for whatever reason.  In dissecting how even simple attacks like Rain Dance work, sometimes you end up down massive trails of redundancy that make things challenging to dissect sometimes.  It is alright to be confused!  But it is especially important that the control flow commands (if, goto, gotosubscript, etc.) are well understood.

## Example 4 - Metal Claw/Charge Beam
What about moves that raise a stat when attacking (or have a chance to)?  For that we can look to Metal Claw and Charge Beam:

```
movedata MOVE_METAL_CLAW
    battleeffect 139
    pss SPLIT_PHYSICAL
    basepower 50
...

movedata MOVE_CHARGE_BEAM
    battleeffect 276
    pss SPLIT_SPECIAL
    basepower 50
...
```
``armips/battle_eff_seq/139.s``:
```
a030_139:
    changevar VAR_OP_SET, VAR_ADD_STATUS2, 0x4000000F // ADD_STATUS_ATTACKER | 15
    critcalc
    damagecalc
    endscript
```
``armips/battle_eff_seq/276.s``:
```
a030_276:
    changevar VAR_OP_SET, VAR_ADD_STATUS2, 0x40000012 // ADD_STATUS_ATTACKER | 18
    critcalc
    damagecalc
    endscript
```
Making a damaging move that also raises a stat is almost exactly like mashing the two effect scripts together!  A damaging attack, with the 3 basic ``critcalc``/``damagecalc``/``endscript`` commands!  Note that since the ``changevar`` queues up a script, it makes sense that the damage still occurs before the stat raise despite it queuing before the other script commands that one would expect to do the damaging--the script it queues actually occurs after the ``endscript`` of this script.

Both this and Rain Dance used ``VAR_ADD_STATUS2`` instead of just ``VAR_ADD_STATUS1``.  While I am not 100% sure why, I believe it is because ``VAR_ADD_STATUS1`` queues up a script to do a primary effect, whereas ``VAR_ADD_STATUS2`` queues up a secondary effect script.  Regardless, we can look again to ``move_effect_to_subscripts`` to see which subscript is queued up:
```c
u32 move_effect_to_subscripts[] =
{
...
    [ 14] =  63,
    [ 15] =  12, // attack +1
    [ 16] =  12, // defense +1
    [ 17] =  12, // speed +1
    [ 18] =  12, // spatk +1
    [ 19] =  12, // spdef +1
...
};
```
As we can see, Metal Claw queues up an attack +1 while Charge Beam queues up a spatk +1.


## Example 5 - Curse/Ancient Power/Superpower
So how do moves that manipulate multiple stats work?  We can look at moves like Curse and then AncientPower for that case:
```
movedata MOVE_CURSE
    battleeffect 109
    pss SPLIT_STATUS
    basepower 0
...

movedata MOVE_ANCIENT_POWER
    battleeffect 140
    pss SPLIT_SPECIAL
    basepower 60
...
```
``armips/move/battle_eff_seq/109.s``:
```
a030_109:
    ifmonstat IF_EQUAL, BATTLER_ATTACKER, MON_DATA_TYPE_1, TYPE_GHOST, _0044
    ifmonstat IF_EQUAL, BATTLER_ATTACKER, MON_DATA_TYPE_2, TYPE_GHOST, _0044
    changevar VAR_OP_SET, VAR_ADD_STATUS1, 0x40000058 // ADD_STATUS_ATTACKER | 88
    endscript
_0044:
    if2 IF_NOTEQUAL, VAR_ATTACKER, 0x10, _0060
    cmd_D4 BATTLER_ATTACKER
_0060:
    changevar VAR_OP_SET, VAR_ADD_STATUS1, 0x20000059 // ADD_STATUS_WORK | 89
    changevar VAR_OP_SET, VAR_MOVE_EFFECT, 0x1
    endscript
```
``armips/move/battle_eff_seq/140.s``:
```
a030_140:
    changevar VAR_OP_SET, VAR_ADD_STATUS2, 0x40000022 // ADD_STATUS_ATTACKER | 34
    critcalc
    damagecalc
    endscript
```
First off, new script commands from the Curse battle script:
```
ifmonstat operator, battler, field, value, address
jump to "address" if the "battler"'s stat designated by "field" is related to "value" as determined by "operator"
- operator is the math operation done on the variable, enumerated in "if"
- battler is the pokémon to get data from
- field is the data to grab from.  see field enumerations in the ifmonstat documentation
- value is the value to check against, used in the operator calculations as they appear in "if"
- address is the location to jump to when the if is true

if2 operator, var1, var2, address
jump to "address" if "var1" is related to "var2" as determined by "operator"
- operator is the same as the "if" operators
- var1 is a var to compare
- var2 is another var to compare against
- address is the location to jump to when the if is true

cmd_D4 battler
not sure what this command does.
- battler is the battler to affect
```
As can be seen, Curse has a number of extra things that come into play because of the Ghost type check!  We will look at AncientPower first.
```c
u32 move_effect_to_subscripts[] =
{
...
    [ 33] =  93,
    [ 34] = 119, // ancient power battle_sub_seq script
    [ 35] = 115,
...
};
```
``armips/battle_sub_seq/119.s``:
```
a001_119:
    changevar VAR_OP_SETMASK, VAR_60, 0x80
    changevar VAR_OP_SET, VAR_34, 0xF // 15
    gotosubscript 12
    changevar VAR_OP_SET, VAR_34, 0x10 // 16
    gotosubscript 12
    changevar VAR_OP_SET, VAR_34, 0x11 // 17
    gotosubscript 12
    changevar VAR_OP_SET, VAR_34, 0x12 // 18
    gotosubscript 12
    changevar VAR_OP_SET, VAR_34, 0x13 // 19
    gotosubscript 12
    changevar VAR_OP_CLEARMASK, VAR_60, 0x2
    changevar VAR_OP_CLEARMASK, VAR_60, 0x80
    endscript
```
AncientPower raises all of the main 6 stats that can be raised when successful.  This is clear here: it jumps to ``battle_sub_seq`` script 12 a total of 5 times each with a different value in ``VAR_34``.  These values hopefully by now look a bit familiar:
```c
u32 move_effect_to_subscripts[] =
{
...
    [ 14] =  63,
    [ 15] =  12, // attack +1
    [ 16] =  12, // defense +1
    [ 17] =  12, // speed +1
    [ 18] =  12, // spatk +1
    [ 19] =  12, // spdef +1
    [ 20] =  12, // accuracy +1
...
};
```
So in order to manipulate multiple stats in a move, one just needs to call subscript 12 over and over with different values in ``VAR_34``.  This is further confirmed by Superpower:
```
movedata MOVE_SUPERPOWER
    battleeffect 182
    pss SPLIT_PHYSICAL
    basepower 120
...
```
``armips/move/battle_sub_seq/182.s``:
```
a030_182:
    changevar VAR_OP_SET, VAR_ADD_STATUS2, 0x60000025 // ADD_STATUS_ATTACKER | ADD_STATUS_WORK | 37
    critcalc
    damagecalc
    endscript
```
``move_effect_to_subscripts`` in ``src/moves.c``:
```c
u32 move_effect_to_subscripts[] =
{
...
    [ 36] = 130,
    [ 37] = 138, // superpower's battle_sub_seq script
    [ 38] = 147,
...
};
```
``armips/move/battle_sub_seq/138.s``:
```
a001_138:
    changevar VAR_OP_SETMASK, VAR_60, 0x80
    changevar VAR_OP_SET, VAR_34, 0x16 // 22
    gotosubscript 12
    changevar VAR_OP_SET, VAR_34, 0x17 // 23
    gotosubscript 12
    changevar VAR_OP_CLEARMASK, VAR_60, 0x2
    changevar VAR_OP_CLEARMASK, VAR_60, 0x80
    endscript
```
``move_effect_to_subscripts`` in ``src/moves.c``:
```c
u32 move_effect_to_subscripts[] =
{
...
    [ 21] =  12, // evasion +1
    [ 22] =  12, // attack -1
    [ 23] =  12, // defense -1
    [ 24] =  12, // speed -1
...
};
```
I am honestly not sure of what the masks that it sets are, but pretty sure one toggles animations (the ``0x80``) instead of having them replay for every stat gain.  The last one probably signals that everything is over and the stat gains are done with.

### Curse as a stat-changing move
Finally, we can look at the rest of Curse (battle script repasted here for convenience, ``armips/move/battle_eff_seq/109.s``):
<details>
<summary>a030_109 - Curse effect script (click to dropdown!)</summary>

```
a030_109:
    ifmonstat IF_EQUAL, BATTLER_ATTACKER, MON_DATA_TYPE_1, TYPE_GHOST, _0044 // if the pokémon is of type ghost, go to _0044
    ifmonstat IF_EQUAL, BATTLER_ATTACKER, MON_DATA_TYPE_2, TYPE_GHOST, _0044
    changevar VAR_OP_SET, VAR_ADD_STATUS1, 0x40000058 // ADD_STATUS_ATTACKER | 88
    endscript
_0044:
    if2 IF_NOTEQUAL, VAR_ATTACKER, VAR_DEFENDER, _0060
    cmd_D4 BATTLER_ATTACKER
_0060:
    changevar VAR_OP_SET, VAR_ADD_STATUS1, 0x20000059 // ADD_STATUS_WORK | 89
    changevar VAR_OP_SET, VAR_MOVE_EFFECT, 0x1
    endscript
```
</details>
I am not sure what setting the ``VAR_MOVE_EFFECT`` to ``0x1`` does here.  Perhaps it is done when showing that the move effect is changing somehow?  I suspect that it has something to do with Curse explicitly, as I can't seem to nail it down.

``armips/move/battle_eff_seq/140.s``:
```
a030_140:
    changevar VAR_OP_SET, VAR_ADD_STATUS2, 0x40000022 // ADD_STATUS_ATTACKER | 34
    critcalc
    damagecalc
    endscript
```
Checking the ``battle_sub_seq`` scripts that Curse queues up from its ``battle_eff_seq`` script:
```c
u32 move_effect_to_subscripts[] =
{
...
    [ 87] =  95,
    [ 88] =  96, // curse not ghost - what we are currently interested in
    [ 89] =  97, // curse ghost
    [ 90] = 126,
...
};
```
``armips/battle_sub_seq/096.s``
```
a001_096:
    changevar VAR_OP_SET, VAR_34, 0x18 // 24
    gotosubscript 12
    changevar VAR_OP_SETMASK, VAR_06, 0x4001
    changevar VAR_OP_SETMASK, VAR_60, 0x80
    changevar VAR_OP_SET, VAR_34, 0xF // 15
    gotosubscript 12
    changevar VAR_OP_SET, VAR_34, 0x10 // 16
    gotosubscript 12
    changevar VAR_OP_CLEARMASK, VAR_60, 0x2
    changevar VAR_OP_CLEARMASK, VAR_60, 0x80
    endscript
```
Looking back at those familiar numbers:
```c
u32 move_effect_to_subscripts[] =
{
...
    [ 15] =  12, // attack +1
    [ 16] =  12, // defense +1
    [ 17] =  12, // speed +1
...
    [ 23] =  12, // defense -1
    [ 24] =  12, // speed -1
...
};
```
We can see that the speed -1 is queued first, followed by the attack +1 and the speed +1.  We can further see that when switching from the negative stat boosts to the positive stat boosts, ``VAR_06`` is masked with ``0x4001``.  Furthermore, the animation plays twice--once for the decrease, and once again for the increase--we can tell because the ``VAR_60`` mask with ``0x80`` thus doesn't happen until after the speed decrease happens.

### Curse as a Ghost type
Now let's look at Curse as a Ghost type (``armips/battle_sub_seq/097.s``):
```
a001_097:
    if IF_MASK, VAR_10, 0x10000, _00D0
    checknostatus BATTLER_DEFENDER, _00D0
    ifmonstat IF_MASK, BATTLER_DEFENDER, MON_DATA_STATUS_2, 0x10000000, _00D0
    gotosubscript 76
    changevartomonvalue VAR_OP_SETMASK, BATTLER_DEFENDER, MON_DATA_STATUS_2, 0x10000000
    changevartomonvalue2 VAR_OP_GET_RESULT, BATTLER_ATTACKER, MON_DATA_MAX_HP, VAR_HP_TEMP
    changevar VAR_OP_MUL, VAR_HP_TEMP, -1
    damagediv VAR_HP_TEMP, 2
    changevar VAR_OP_SETMASK, VAR_06, 0x40
    changevar2 VAR_OP_SET, VAR_BATTLER_SOMETHING, VAR_ATTACKER
    gotosubscript 2
    printmessage 0x1A1, 0x9, 0x1, 0x2, "NaN", "NaN", "NaN", "NaN"
    waitmessage
    wait 0x1E
    endscript
_00D0:
    changevar VAR_OP_SETMASK, VAR_10, 0x40
    endscript
```
A few new script commands here:
```
changevartomonvalue operator, battler, field, value
a more accurate name would be "changemondatabyvalue," will work on fixing this.
changes mon data "field" by "value" as specified by "operator"
- operator is the same as the "changevar" operators
- battler is the battler to grab the data from
- field is the data to grab/set from the battler
- value is the argument for the operator

changevartomonvalue2 operator, battler, field, var
a more accurate name would be "changemondatabyvar," will work on fixing this.
changes mon data "field" by "var"'s value as specified by "operator"
- operator is the same as the "changevar" operators.  if looking to set the var to the mon data field, use VAR_OP_GET_RESULT
- battler is the battler to grab the data from
- field is the data to grab/set from the battler
- var is the variable the engine grabs from to manipulate the mon data field

damagediv var, value
sets damage to be "var" / "value"
- var is the numerator in the division
- value is the denominator in the division
```
If ``VAR_10`` has ``0x10000`` set, the script sets the move failed bit in ``VAR_10`` and ends the script.  Similar for the ``checknostatus`` command, where if the ``BATTLER_DEFENDER`` has some status then the move fails and the script ends.  Finally, if the curse bit is set in the Pokémon's ``MON_DATA_STATUS_2`` field, then the move fails as well.  The move then calls ``battle_sub_seq`` script 76 (``armips/move/battle_sub_seq/076.s``):
```
a001_076:
    printattackmessage
    waitmessage
    seteffectprimary BATTLER_ATTACKER
    waitmessage
    endscript
```
Two new very simple commands from this one:
```
printattackmessage
prints the current attack message from a027 file 003 (data/text/003.txt).

seteffectprimary battler
a more accurate name would be "playanimation," will work on fixing this.
plays the current attack's animation based on "battler"
- battler is the primary battler to base the animation on
```
This is a simple battle script that generically prints the attack message and plays the move animation before ending.

So at ``gotosubscript 76`` from the Curse subscript above, the attack name is printed and the animation played.  Repasting the Curse script:
```
a001_097:
...
    gotosubscript 76
    changevartomonvalue VAR_OP_SETMASK, BATTLER_DEFENDER, MON_DATA_STATUS_2, 0x10000000
    changevartomonvalue2 VAR_OP_GET_RESULT, BATTLER_ATTACKER, MON_DATA_MAX_HP, VAR_HP_TEMP
    changevar VAR_OP_MUL, VAR_HP_TEMP, -1
    damagediv VAR_HP_TEMP, 2
    changevar VAR_OP_SETMASK, VAR_06, 0x40
    changevar2 VAR_OP_SET, VAR_BATTLER_SOMETHING, VAR_ATTACKER
    gotosubscript 2
    printmessage 0x1A1, 0x9, 0x1, 0x2, "NaN", "NaN", "NaN", "NaN"
    waitmessage
    wait 0x1E
    endscript
_00D0:
    changevar VAR_OP_SETMASK, VAR_10, 0x40
    endscript
```
The Curse bit is then set (you can think of this as the ``BATTLER_DEFENDER`` is inflicted with Curse by the ``changevartomonvalue`` command there), and the ``BATTLER_ATTACKER``'s max HP is placed into ``VAR_HP_TEMP``.  ``VAR_HP_TEMP`` is made to be negative (by multiplying by -1), and the damage to be dealt is set to ``VAR_HP_TEMP / 2``--half of the max HP.  The bit in ``VAR_06`` is set to signal to subscript 2 that is coming up that the move sound effect should not be played, that the HP should just be manipulated (as we will see in a moment).  Finally, the ``VAR_BATTLER_SOMETHING`` is set to be ``VAR_ATTACKER``--the temporary work battler is set to be the attacker so that subscript 2, which is written to be generic, will know what to do.  We go to subscript 2 (``armips/move/battle_sub_seq/002.s``):
```
a001_002:
    if IF_MASK, VAR_06, 0x40, _0044
    playmovesoundeffect BATTLER_xFF
    monflicker 0xFF
    waitmessage
    if IF_EQUAL, VAR_69, 0x0, _0044
    gotosubscript 264
_0044:
    changevar VAR_OP_CLEARMASK, VAR_06, 0x40
    healthbarupdate BATTLER_xFF
    waitmessage
    datahpupdate BATTLER_xFF
    tryfaintmon BATTLER_xFF
    if IF_GREATER, VAR_HP_TEMP, 0x0, _0094
    changevar2 VAR_OP_SET, VAR_ASSURANCE_DAMAGE, VAR_HP_TEMP
_0094:
    endscript
```
Documenting more new script commands:
```
playmovesoundeffect battler
plays the move's damaging sound effect
- battler is the basis of the sound pan

monflicker battler
makes "battler" flicker as if hit
- battler is the Pokémon to flicker

healthbarupdate battler
updates "battler"'s hp bar
- battler is the battler that has the hp bar to update

datahpupdate battler
updates the information on the "battler"'s hp bar, specifically the hp data
- battler is the owner of the hp data to update

tryfaintmon battler
tries to faint "battler".  nothing happens if fails, the mon will slide down otherwise
- battler is the mon to faint
```
So we can see that this is another generic subscript made to be used under many circumstances.  It deals damage and then does all the updating of the HP bars as necessary.  I will let you check out subscript 264 if you want, but it is not relevant to the current case, as ``VAR_06`` has its ``0x40`` bit set and the script jumps to ``_0044`` instead of executing all of that.  It is the script responsible for the super-effective weakening berries, and we may be looking at it as a specific use-case later where we want to add something like, say, the Roseli Berry.

Finally, back to the ``printmessage`` command from the Curse script:
```
a001_097:
...
    gotosubscript 2
    printmessage 0x1A1, 0x9, 0x1, 0x2, "NaN", "NaN", "NaN", "NaN"
    waitmessage
    wait 0x1E
    endscript
_00D0:
    changevar VAR_OP_SETMASK, VAR_10, 0x40
    endscript
```
This ``printmessage`` finally gets to a different ``tag`` (that isn't 0)!  Here, we can grab the ``tag`` value from the documentation at the bottom:
```c
#define TAG_TRNAME                      (8)     //trainername

#define TAG_NICK_NICK                   (9)     //nickname      nickname
#define TAG_NICK_MOVE                   (10)    //nickname      move
```
Here we see that there are two fields that it grabs, so it requires 2 of the 6 ``battler`` parameters.  The rest are left as "NaN" as before.  To understand what is going on, we check out line ``0x1A1 (417)`` from ``data/text/197.txt`` (with brackets added for ease of reference):
```
[417] {STRVAR_1 1, 0, 0} cut its own HP\nand laid a curse on {STRVAR_1 1, 1, 0}!
[418] {STRVAR_1 1, 0, 0} cut its own HP\nand laid a curse on the wild\f{STRVAR_1 1, 1, 0}!
[419] {STRVAR_1 1, 0, 0} cut its own HP\nand laid a curse on the foe’s\f{STRVAR_1 1, 1, 0}!
[420] The wild {STRVAR_1 1, 0, 0} cut its own HP\nand laid a curse on {STRVAR_1 1, 1, 0}!
[421] The wild {STRVAR_1 1, 0, 0} cut its own HP\nand laid a curse on the wild\f{STRVAR_1 1, 1, 0}!
[422] The foe’s {STRVAR_1 1, 0, 0} cut its own HP\nand laid a curse on {STRVAR_1 1, 1, 0}!
[423] The foe’s {STRVAR_1 1, 0, 0} cut its own HP\nand laid a curse on the foe’s\f{STRVAR_1 1, 1, 0}!
```
Here we are introduced to a way of doing things that is rather interesting and is very much a massive waste of space:  The only string variables used are for the nicknames directly.  Otherwise, the ``printmessage`` command will just add on for the correct circumstance to achieve a new message that correctly reflects the battle at hand.  Generally speaking, most moves that only buffer 1 battler from the field will have 3 messages, one for the player, the wild mon, and the foe's mon--this is even the case for the ``printattackmessage`` text archive, ``data/text/003.txt``.  In Curse's case, because one battler is using the move on another battler and both are buffered, there are 7 cases, all automatically adjusted for like the above.  It then takes two battlers, and the command looks somewhat better like this:
```
printmessage 417, TAG_NICK_NICK, BATTLER_ATTACKER, BATTLER_DEFENDER, "NaN", "NaN", "NaN", "NaN"
```
Meaning that the first ``{STRVAR_1 1, 0, 0}`` will be replaced with the nickname of the attacker, and the second ``{STRVAR_1 1, 1, 0}`` will be replaced with the nickname of the defender.  The script then ends after waiting for the message to print.

# Synthesizing new move effects
In adding a new move that doesn't have an effect that already exists, we need to add a new ``battle_eff_seq`` script.  In this repo, it's simply a matter of creating a new `.s` file in the folder.  As an example, I will be looking to implement Simple Beam--an effect that isn't currently done in the repo--as move effect 292 that will queue up subscript 330 to do its effect.  

## Adding Simple Beam's Effect
Simple Beam sets the defender's ability to Simple unless if the ability it would overwrite is any of Truant, Multitype, Stance Change, Schooling, Comatose, Shields Down, Disguise, RKS System, Battle Bond, Power Construct, Ice Face, Gulp Missile, or As One.  All of these checks can be done directly from the battle scripts themselves using ``ifmonstat`` and ``changevartomonvalue``.

To start out, we can finally discuss the headers of each script file, which you'd see if you opened any of the files directly from the repo.  These will largely be the same for each scripts, with exceptions depending on if more constants are needed:
```
.nds
.thumb

.include "armips/include/battlescriptcmd.s"
.include "armips/include/abilities.s"
.include "armips/include/itemnums.s"
.include "armips/include/monnums.s"
.include "armips/include/movenums.s"
```
These are all directives that tell armips to either set things up differently or read separate files for defines and other asm if included.  The ``.nds`` directive configures the output to be little-endian and sets the assembler to output ARM 9 code in ARM mode.  The ``.thumb`` directive fixes the ARM mode output to be in thumb mode, which for ARM assembly code is a cut-down space-conserving instruction set.  The ``.include`` directive opens up the file specified by the filepath in the string, assembling said file similar to the ``#include`` directive from C.
```
.create "build/move/battle_eff_seq/0_001", 0
```
The ``.create`` directive tells armips to create the file at the filepath in the string and will start writing to it from the base offset 0.

Finally, each file ends with the ``.close`` directive to tell armips to close out of the file and make no further changes.

So we create a new file in ``armips/move/battle_eff_seq`` called ``292.s`` with the contents at the moment solely:
```
.nds
.thumb

.include "armips/include/battlescriptcmd.s"
.include "armips/include/abilities.s"
.include "armips/include/itemnums.s"
.include "armips/include/monnums.s"
.include "armips/include/movenums.s"

.create "build/move/battle_eff_seq/0_292", 0

simpleBeamScript: // a030_292
```
There is nothing that we can actually do from the ``battle_eff_seq`` that needs to be done for Simple Beam, which just gives the defender the Simple ability (instead of changing move damage or something like that that isn't battler-specific).  As a result, we just go straight to queuing up a ``battle_sub_seq`` and ending the script right away:
```
simpleBeamScript: // a030_292
    changevar VAR_OP_SET, VAR_ADD_STATUS1, ADD_STATUS_DEFENDER | xxx
    endscript
```
So now we know that we queue up a script that affects the defender.  What do we put in for ``xx``, the index of ``move_effect_to_subscripts``?  In this repo, you can actually just add on to the end of ``move_effect_to_subscripts`` in ``src/moves.c``.  So we add on a new entry to the end:
```c
u32 move_effect_to_subscripts[] =
{
...
    [152] = 318, // shell smash
    [153] = 319, // v create
    [154] = 320, // autotomize
    [155] = 330, // simple beam - new entry
};
```
Which makes the ``battle_eff_seq`` script:
```
simpleBeamScript: // a030_292
    changevar VAR_OP_SET, VAR_ADD_STATUS1, ADD_STATUS_DEFENDER | 155 // queue up simple beam subscript
    endscript
```
Now we look to make the Simple Beam subscript, adding a ``battle_sub_seq`` script to the end of the folder as ``330.s``:
```
.nds
.thumb

.include "armips/include/battlescriptcmd.s"
.include "armips/include/abilities.s"
.include "armips/include/itemnums.s"
.include "armips/include/monnums.s"
.include "armips/include/movenums.s"

.create "build/move/battle_sub_seq/1_330", 0

simpleBeamSubScript: // a001_330
```
Now to build the script.  We need to change the defender's ability to Simple.  We've covered a command to do so, ``changevartomonvalue`` (which should be ``changemondatabyvalue``):
```
simpleBeamSubScript: // a001_330
    changevartomonvalue VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_SIMPLE // set the defender's ability to simple
    endscript
```
But now the animation just plays and the move ends with no indication of what happened.  From there, we need to print a message, for which we need to add a new message as well:
```
simpleBeamSubScript: // a001_330
    changevartomonvalue VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_SIMPLE // set the defender's ability to simple
    printmessage 1348, TAG_NICK_ABILITY, BATTLER_DEFENDER, BATTLER_DEFENDER, "NaN", "NaN", "NaN", "NaN"
    endscript
```
Adding the message to the ``197.txt``, just have to add as line 1348 (when the first line is zero, brackets added as reference and not actually inserted):
```
[1348] {STRVAR_1 1, 0, 0}’s ability\nchanged to {STRVAR_1 5, 1, 0}!
[1349] The wild {STRVAR_1 1, 0, 0}’s\nability changed to {STRVAR_1 5, 1, 0}!
[1350] The foe’s {STRVAR_1 1, 0, 0}’s\nability changed to {STRVAR_1 5, 1, 0}!
```
This text entry (specifically the string variables used) is based on Flash Fire's, at entry 656:
```
[656] {STRVAR_1 1, 0, 0}’s {STRVAR_1 5, 1, 0} raised\nthe power of its Fire-type moves!
[657] The wild {STRVAR_1 1, 0, 0}’s\n{STRVAR_1 5, 1, 0} raised the power\fof its Fire-type moves!
[658] The foe’s {STRVAR_1 1, 0, 0}’s\n{STRVAR_1 5, 1, 0} raised the power\fof its Fire-type moves!
```
Now we have Simple Beam, which replaces the defender's ability with Simple, printing a message that will look something like:
```
The foe's Elgyem's
ability changed to Simple!
```
Finally, we have to add that the move fails when the defender's ability is any number of abilities.  A move fails when ``VAR_10`` has its ``0x40`` bit set, as discussed previously:
```
moveFails:
    changevar VAR_OP_SETMASK, VAR_10, 0x40
    endscript
```
So now we add a check for the ability Truant that will jump to ``moveFails`` if the defender has the ability using ``ifmonstat``.
```
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_TRUANT, moveFails
```
Combining it all together:
```
simpleBeamSubScript: // a001_330
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_TRUANT, moveFails // move fails if the defender has truant

    changevartomonvalue VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_SIMPLE // set the defender's ability to simple
    printmessage 1348, TAG_NICK_ABILITY, BATTLER_DEFENDER, BATTLER_DEFENDER, "NaN", "NaN", "NaN", "NaN"
    endscript

moveFails:
    changevar VAR_OP_SETMASK, VAR_10, 0x40
    endscript
```
Now Simple Beam fails when it's used on a defender with Truant.  Adding the rest of the abilities:
```
simpleBeamSubScript: // a001_330
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_TRUANT, moveFails           // move fails if the defender has truant
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_MULTITYPE, moveFails        // move fails if the defender has multitype
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_STANCE_CHANGE, moveFails    // move fails if the defender has stance change
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_SCHOOLING, moveFails        // move fails if the defender has schooling
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_COMATOSE, moveFails         // move fails if the defender has comatose
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_SHIELDS_DOWN, moveFails     // move fails if the defender has shields down
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_DISGUISE, moveFails         // move fails if the defender has disguise
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_RKS_SYSTEM, moveFails       // move fails if the defender has rks system
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_BATTLE_BOND, moveFails      // move fails if the defender has battle bond
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_POWER_CONSTRUCT, moveFails  // move fails if the defender has power construct
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_ICE_FACE, moveFails         // move fails if the defender has ice face
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_GULP_MISSILE, moveFails     // move fails if the defender has gulp missile
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_AS_ONE, moveFails           // move fails if the defender has as one

    changevartomonvalue VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_SIMPLE // set the defender's ability to simple
    printmessage 1348, TAG_NICK_ABILITY, BATTLER_DEFENDER, BATTLER_DEFENDER, "NaN", "NaN", "NaN", "NaN"
    endscript

moveFails:
    changevar VAR_OP_SETMASK, VAR_10, 0x40
    endscript
```
That should be all needed to do the effect for Simple Beam!  For future examples, I will be skipping the header creation for simplicity--just know that it will be there.

To recap:  Add a new ``battle_eff_seq`` script that queues up a ``battle_sub_seq`` script, add a new entry to ``move_effect_to_subscripts`` in ``src/moves.c`` that corresponds to the one you queued up in the ``battle_eff_seq`` script, and then add a new ``battle_sub_seq`` that corresponds to the entry you just added to ``move_effect_to_subscripts`` that then performs all of the effects.
<details>
<summary>Simple Beam created scripts (click to dropdown!)</summary>

Battle effect script:
```
.nds
.thumb

.include "armips/include/battlescriptcmd.s"
.include "armips/include/abilities.s"
.include "armips/include/itemnums.s"
.include "armips/include/monnums.s"
.include "armips/include/movenums.s"

.create "build/move/battle_eff_seq/0_292", 0

simpleBeamScript: // a030_292
    changevar VAR_OP_SET, VAR_ADD_STATUS1, ADD_STATUS_DEFENDER | 155 // queue up simple beam subscript
    endscript
```
Battle sub script:
```
.nds
.thumb

.include "armips/include/battlescriptcmd.s"
.include "armips/include/abilities.s"
.include "armips/include/itemnums.s"
.include "armips/include/monnums.s"
.include "armips/include/movenums.s"

.create "build/move/battle_sub_seq/1_330", 0

simpleBeamSubScript: // a001_330
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_TRUANT, moveFails           // move fails if the defender has truant
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_MULTITYPE, moveFails        // move fails if the defender has multitype
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_STANCE_CHANGE, moveFails    // move fails if the defender has stance change
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_SCHOOLING, moveFails        // move fails if the defender has schooling
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_COMATOSE, moveFails         // move fails if the defender has comatose
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_SHIELDS_DOWN, moveFails     // move fails if the defender has shields down
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_DISGUISE, moveFails         // move fails if the defender has disguise
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_RKS_SYSTEM, moveFails       // move fails if the defender has rks system
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_BATTLE_BOND, moveFails      // move fails if the defender has battle bond
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_POWER_CONSTRUCT, moveFails  // move fails if the defender has power construct
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_ICE_FACE, moveFails         // move fails if the defender has ice face
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_GULP_MISSILE, moveFails     // move fails if the defender has gulp missile
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_AS_ONE, moveFails           // move fails if the defender has as one
    changevartomonvalue VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_ABILITY, ABILITY_SIMPLE // set the defender's ability to simple
    printmessage 1348, TAG_NICK_ABILITY, BATTLER_DEFENDER, BATTLER_DEFENDER, "NaN", "NaN", "NaN", "NaN"
    endscript
moveFails:
    changevar VAR_OP_SETMASK, VAR_10, 0x40
    endscript
```
</details>

## Adding Fairy Type Handling to Judgment
Old moves are also updatable in hg-engine--a few that have been are Rapid Spin and Judgment.  Let's look at what Judgment does to see how we can add Fairy type handling to it, quick look back at ``armips/data/moves.s``:
```
movedata MOVE_JUDGMENT
    battleeffect 268
    pss SPLIT_SPECIAL
    basepower 100
```
So we look at ``battle_eff_seq`` script 268 (``armips/move/battle_eff_seq/268.s``) (the old version [here](https://github.com/BluRosie/hg-engine/blob/7180691503c90a80ff184802d069f299584013d4/armips/move/battle_eff_seq/268.s)):
<details>
<summary>a030_268 - Judgment effect script unlabeled (click to dropdown!)</summary>

```
a030_268:
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x83, _0148
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x86, _0160
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x84, _0178
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x85, _0190
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x89, _01A8
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x88, _01C0
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x8A, _01D8
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x8D, _01F0
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x7E, _0208
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x7F, _0220
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x81, _0238
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x80, _0250
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x87, _0268
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x82, _0280
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x8B, _0298
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x8C, _02B0
    goto _02C0
_0148:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0x1
    goto _02C0
_0160:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0x2
    goto _02C0
_0178:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0x3
    goto _02C0
_0190:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0x4
    goto _02C0
_01A8:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0x5
    goto _02C0
_01C0:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0x6
    goto _02C0
_01D8:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0x7
    goto _02C0
_01F0:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0x8
    goto _02C0
_0208:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0xA
    goto _02C0
_0220:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0xB
    goto _02C0
_0238:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0xC
    goto _02C0
_0250:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0xD
    goto _02C0
_0268:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0xE
    goto _02C0
_0280:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0xF
    goto _02C0
_0298:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0x10
    goto _02C0
_02B0:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, 0x11
_02C0:
    critcalc
    damagecalc
    endscript
```
</details>
While long, it should be clear what it is doing:  with the ``checker`` of ``checkitemeffect`` being ``0x0`` for each command, the commands are all checking to see if the ``BATTLER_ATTACKER`` has an item with that item effect.  A quick detour into the item data shows that each of these move effects belong directly to a plate--you can see another breakdown [here](https://github.com/BluRosie/hgss-filesys-example/blob/fairy-type/asm/fairy.s#L18-L41), where I modify one of the many ASM places that dictate how Arceus gets its type to include the Fairy type.

We can label this a bit better to show what is happening:
<details>
<summary>a030_268 - Judgment effect script better labeled (click to dropdown!)</summary>

```
a030_268:
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x83, _setFighting   // TYPE_FIGHTING
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x86, _setFlying     // TYPE_FLYING
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x84, _setPoison     // TYPE_POISON
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x85, _setGround     // TYPE_GROUND
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x89, _setRock       // TYPE_ROCK
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x88, _setBug        // TYPE_BUG
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x8A, _setGhost      // TYPE_GHOST
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x8D, _setSteel      // TYPE_STEEL
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x7E, _setFire       // TYPE_FIRE
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x7F, _setWater      // TYPE_WATER
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x81, _setGrass      // TYPE_GRASS   
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x80, _setElectric   // TYPE_ELECTRIC
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x87, _setPsychic    // TYPE_PSYCHIC 
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x82, _setIce        // TYPE_ICE     
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x8B, _setDragon     // TYPE_DRAGON  
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x8C, _setDark       // TYPE_DARK    
    goto _return
_setFighting:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_FIGHTING
    goto _return
_setFlying:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_FLYING
    goto _return
_setPoison:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_POISON
    goto _return
_setGround:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_GROUND
    goto _return
_setRock:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_ROCK
    goto _return
_setBug:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_BUG
    goto _return
_setGhost:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_GHOST
    goto _return
_setSteel:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_STEEL
    goto _return
_setFire:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_FIRE
    goto _return
_setWater:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_WATER
    goto _return
_setGrass:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_GRASS
    goto _return
_setElectric:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_ELECTRIC
    goto _return
_setPsychic:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_PSYCHIC
    goto _return
_setIce:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_ICE
    goto _return
_setDragon:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_DRAGON
    goto _return
_setDark:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_DARK
_return:
    critcalc
    damagecalc
    endscript
```
</details>
With all of the type values labeled and the labels themselves having the names, this script looks even clearer.  Adding a new held item effect that turns Judgment into a Fairy type move should just be following the pattern, adding something like this:

```
a030_268:
...
    checkitemeffect 0x0, BATTLER_ATTACKER, XXXX, _setFairy      // TYPE_FAIRY
...
_setFairy:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_FIRE
    goto _return
...
```

And if you're not replacing an old item for the Pixie Plate, then the ``XXXX`` becomes a brand new item effect slot.  The game ends at ``0x92``, so we use ``0x93``:
```
a030_268:
...
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x93, _setFairy      // TYPE_FAIRY
...
_setFairy:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_FAIRY
    goto _return
...
```
This is actually the method used directly in hg-engine for updating Judgment:
<details>
<summary>Fully-Updated Fairy Judgment (click to drop down!)</summary>

Fairy handling has spaces around it for emphasis:
```
.nds
.thumb

.include "armips/include/battlescriptcmd.s"
.include "armips/include/abilities.s"
.include "armips/include/config.s"
.include "armips/include/constants.s"
.include "armips/include/itemnums.s"
.include "armips/include/monnums.s"
.include "armips/include/movenums.s"

.create "build/move/battle_eff_seq/0_268", 0

a030_268:
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x83, _setFighting   // TYPE_FIGHTING
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x86, _setFlying     // TYPE_FLYING  
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x84, _setPoison     // TYPE_POISON  
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x85, _setGround     // TYPE_GROUND  
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x89, _setRock       // TYPE_ROCK    
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x88, _setBug        // TYPE_BUG     
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x8A, _setGhost      // TYPE_GHOST   
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x8D, _setSteel      // TYPE_STEEL  

    checkitemeffect 0x0, BATTLER_ATTACKER, 0x93, _setFairy      // TYPE_FAIRY

    checkitemeffect 0x0, BATTLER_ATTACKER, 0x7E, _setFire       // TYPE_FIRE    
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x7F, _setWater      // TYPE_WATER   
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x81, _setGrass      // TYPE_GRASS   
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x80, _setElectric   // TYPE_ELECTRIC
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x87, _setPsychic    // TYPE_PSYCHIC 
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x82, _setIce        // TYPE_ICE     
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x8B, _setDragon     // TYPE_DRAGON  
    checkitemeffect 0x0, BATTLER_ATTACKER, 0x8C, _setDark       // TYPE_DARK    
    goto _return
_setFighting:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_FIGHTING
    goto _return
_setFlying:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_FLYING
    goto _return
_setPoison:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_POISON
    goto _return
_setGround:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_GROUND
    goto _return
_setRock:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_ROCK
    goto _return
_setBug:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_BUG
    goto _return
_setGhost:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_GHOST
    goto _return
_setSteel:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_STEEL
    goto _return

_setFairy:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_FAIRY
    goto _return

_setFire:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_FIRE
    goto _return
_setWater:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_WATER
    goto _return
_setGrass:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_GRASS
    goto _return
_setElectric:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_ELECTRIC
    goto _return
_setPsychic:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_PSYCHIC
    goto _return
_setIce:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_ICE
    goto _return
_setDragon:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_DRAGON
    goto _return
_setDark:
    changevar VAR_OP_SET, VAR_MOVE_TYPE, TYPE_DARK
_return:
    critcalc
    damagecalc
    endscript

.close
```
</details>

## Adding Fairy Type Reduction Berry
Gen 4 introduced all of the berries that activate when a super-effective move is used against the holder.  The damage they take is then halved as a result, making them very frequent in competitive play as a result, with every turn being important in the overall scheme of battles.  As mentioned previously when covering Curse as a Ghost type, the type-halving berries are covered in ``battle_sub_seq`` script 264, a generic script that actually has that as its primary purpose--check if the damage being done should be halved by the berries or not.

The script dump as it originally appears is as follows, taken from [here](https://github.com/BluRosie/hg-engine/blob/7180691503c90a80ff184802d069f299584013d4/armips/move/battle_sub_seq/264.s):
<details>
<summary>a001_264 - Type Reduction Berries</summary>

```
a001_264:
    if IF_MASK, VAR_06, 0x8800, _041C
    if IF_MASK, VAR_10, 0x20, _041C
    abilitycheck 0x1, BATTLER_ATTACKER, ABILITY_NORMALIZE, _0054
    changevar VAR_OP_SET, VAR_09, 0x0
    goto _0088
_0054:
    if IF_EQUAL, VAR_MOVE_TYPE, 0x0, _0080
    changevar2 VAR_OP_GET_RESULT, VAR_MOVE_TYPE, VAR_09
    goto _0088
_0080:
    getmoveparameter 0x3
_0088:
    getitemeffect BATTLER_xFF, 0x2B
    if IF_EQUAL, VAR_43, 0x23, _0204
    if IF_NOTMASK, VAR_10, 0x2, _041C
    if IF_EQUAL, VAR_43, 0x13, _0220
    if IF_EQUAL, VAR_43, 0x14, _023C
    if IF_EQUAL, VAR_43, 0x15, _0258
    if IF_EQUAL, VAR_43, 0x16, _0274
    if IF_EQUAL, VAR_43, 0x17, _0290
    if IF_EQUAL, VAR_43, 0x18, _02AC
    if IF_EQUAL, VAR_43, 0x19, _02C8
    if IF_EQUAL, VAR_43, 0x1A, _02E4
    if IF_EQUAL, VAR_43, 0x1B, _0300
    if IF_EQUAL, VAR_43, 0x1C, _031C
    if IF_EQUAL, VAR_43, 0x1D, _0338
    if IF_EQUAL, VAR_43, 0x1E, _0354
    if IF_EQUAL, VAR_43, 0x1F, _0370
    if IF_EQUAL, VAR_43, 0x20, _038C
    if IF_EQUAL, VAR_43, 0x21, _03A8
    if IF_EQUAL, VAR_43, 0x22, _03C4
    goto _041C
_0204:
    if IF_EQUAL, VAR_09, 0x0, _03D8
    goto _041C
_0220:
    if IF_EQUAL, VAR_09, 0xA, _03D8
    goto _041C
_023C:
    if IF_EQUAL, VAR_09, 0xB, _03D8
    goto _041C
_0258:
    if IF_EQUAL, VAR_09, 0xD, _03D8
    goto _041C
_0274:
    if IF_EQUAL, VAR_09, 0xC, _03D8
    goto _041C
_0290:
    if IF_EQUAL, VAR_09, 0xF, _03D8
    goto _041C
_02AC:
    if IF_EQUAL, VAR_09, 0x1, _03D8
    goto _041C
_02C8:
    if IF_EQUAL, VAR_09, 0x3, _03D8
    goto _041C
_02E4:
    if IF_EQUAL, VAR_09, 0x4, _03D8
    goto _041C
_0300:
    if IF_EQUAL, VAR_09, 0x2, _03D8
    goto _041C
_031C:
    if IF_EQUAL, VAR_09, 0xE, _03D8
    goto _041C
_0338:
    if IF_EQUAL, VAR_09, 0x6, _03D8
    goto _041C
_0354:
    if IF_EQUAL, VAR_09, 0x5, _03D8
    goto _041C
_0370:
    if IF_EQUAL, VAR_09, 0x7, _03D8
    goto _041C
_038C:
    if IF_EQUAL, VAR_09, 0x10, _03D8
    goto _041C
_03A8:
    if IF_EQUAL, VAR_09, 0x11, _03D8
    goto _041C
_03C4:
    if IF_NOTEQUAL, VAR_09, 0x8, _041C
_03D8:
    setstatus2effect BATTLER_xFF, 0xA
    waitmessage
    damagediv 32, 2
    printmessage 0x46B, 0x18, 0x15, 0x1, "NaN", "NaN", "NaN", "NaN"
    waitmessage
    wait 0x1E
    removeitem BATTLER_xFF
_041C:
    endscript
```
</details>

A few new commands from this:
```
abilitycheck checker, battler, ability, destination
jump to "destination" if "battler" has or doesn't have "ability" based on "checker"--if "checker" is 0, jumps if the "battler" has the "ability", otherwise jump if doesn't have "ability"
- checker determines to check if "battler" has or doesn't have "ability"--if "checker" is 0, jumps if the "battler" has the "ability", otherwise jump if doesn't have "ability"
- battler is the battler whose ability to check
- ability is the ability to check for
- destination is where to jump if the check succeeds

setstatus2effect battler, value
sets status2 "value" bits for "battler" (condition2 in the BattlePokemon structure)
- battler is the battler to grab status2 from
- value comprises the bits to set in status2

getmoveparameter field
grabs parameter "field" from the move data structure and stores in VAR_09
- "field" is the data to grab from the move, enumerations in the documentation

getitemeffect battler, var
grabs the item held effect from "battler" and puts it in "var"
- battler is the battler to grab the item held effect from
- var is the var to store the item held effect in

removeitem battler
removes the item from battler
- battler is the battler whose item to remove
```
Making it easier to read with various defines and label names:
<details>
<summary>a001_264 - Type Reduction Berries (Nicer)</summary>

```
a001_264:
    if IF_MASK, VAR_06, 0x8800, _endScript
    if IF_MASK, VAR_10, 0x20, _endScript
    abilitycheck 0x1, BATTLER_ATTACKER, ABILITY_NORMALIZE, _skipNormalize
    changevar VAR_OP_SET, VAR_09, TYPE_NORMAL
    goto _checkBerries
_skipNormalize:
    if IF_EQUAL, VAR_MOVE_TYPE, 0x0, _grabDefaultType
    changevar2 VAR_OP_GET_RESULT, VAR_MOVE_TYPE, VAR_09
    goto _checkBerries
_grabDefaultType:
    getmoveparameter MOVE_DATA_TYPE // stores the move type from the move data table in VAR_09

_checkBerries:
    getitemeffect BATTLER_xFF, VAR_43
    if IF_EQUAL, VAR_43, 0x23, _checkNormal
    if IF_NOTMASK, VAR_10, 0x2, _endScript // if the move used was not supereffective, end the script
    if IF_EQUAL, VAR_43, 0x13, _checkFire
    if IF_EQUAL, VAR_43, 0x14, _checkWater
    if IF_EQUAL, VAR_43, 0x15, _checkElectric
    if IF_EQUAL, VAR_43, 0x16, _checkGrass
    if IF_EQUAL, VAR_43, 0x17, _checkIce
    if IF_EQUAL, VAR_43, 0x18, _checkFighting
    if IF_EQUAL, VAR_43, 0x19, _checkPoison
    if IF_EQUAL, VAR_43, 0x1A, _checkGround
    if IF_EQUAL, VAR_43, 0x1B, _checkFlying
    if IF_EQUAL, VAR_43, 0x1C, _checkPsychic
    if IF_EQUAL, VAR_43, 0x1D, _checkBug
    if IF_EQUAL, VAR_43, 0x1E, _checkRock
    if IF_EQUAL, VAR_43, 0x1F, _checkGhost
    if IF_EQUAL, VAR_43, 0x20, _checkDragon
    if IF_EQUAL, VAR_43, 0x21, _checkDark
    if IF_EQUAL, VAR_43, 0x22, _checkSteel
    goto _endScript
_checkNormal:
    if IF_EQUAL, VAR_09, TYPE_NORMAL, _halveDamage
    goto _endScript
_checkFire:
    if IF_EQUAL, VAR_09, TYPE_FIRE, _halveDamage
    goto _endScript
_checkWater:
    if IF_EQUAL, VAR_09, TYPE_WATER, _halveDamage
    goto _endScript
_checkElectric:
    if IF_EQUAL, VAR_09, TYPE_ELECTRIC, _halveDamage
    goto _endScript
_checkGrass:
    if IF_EQUAL, VAR_09, TYPE_GRASS, _halveDamage
    goto _endScript
_checkIce:
    if IF_EQUAL, VAR_09, TYPE_ICE, _halveDamage
    goto _endScript
_checkFighting:
    if IF_EQUAL, VAR_09, TYPE_FIGHTING, _halveDamage
    goto _endScript
_checkPoison:
    if IF_EQUAL, VAR_09, TYPE_POISON, _halveDamage
    goto _endScript
_checkGround:
    if IF_EQUAL, VAR_09, TYPE_GROUND, _halveDamage
    goto _endScript
_checkFlying:
    if IF_EQUAL, VAR_09, TYPE_FLYING, _halveDamage
    goto _endScript
_checkPsychic:
    if IF_EQUAL, VAR_09, TYPE_PSYCHIC, _halveDamage
    goto _endScript
_checkBug:
    if IF_EQUAL, VAR_09, TYPE_BUG, _halveDamage
    goto _endScript
_checkRock:
    if IF_EQUAL, VAR_09, TYPE_ROCK, _halveDamage
    goto _endScript
_checkGhost:
    if IF_EQUAL, VAR_09, TYPE_GHOST, _halveDamage
    goto _endScript
_checkDragon:
    if IF_EQUAL, VAR_09, TYPE_DRAGON, _halveDamage
    goto _endScript
_checkDark:
    if IF_EQUAL, VAR_09, TYPE_DARK, _halveDamage
    goto _endScript
_checkSteel:
    if IF_NOTEQUAL, VAR_09, TYPE_STEEL, _endScript

_halveDamage:
    setstatus2effect BATTLER_xFF, 0xA
    waitmessage
    damagediv VAR_HP_TEMP, 2
    printmessage 1131, TAG_ITEM_MOVE, BATTLER_x15, BATTLER_ATTACKER, "NaN", "NaN", "NaN", "NaN"
    waitmessage
    wait 0x1E
    removeitem BATTLER_xFF

_endScript:
    endscript
```
</details>

So a few things that we can gather from this is the difference between how this is programmed and how the Judgment plates are programmed.  Instead of using many different ``checkitemeffect`` commands for each individual berry, this script stores the item effect in a variable and performs many comparisons on that variable.  This demonstrates a sort of flexibility present in the scripting system: you can do any of multiple things to accomplish the same goal.  You can even rewrite this to be the same as the other one (or vice versa!).  The important part is testing everything out before leaving it be and tracking issues as they pop up.

Looking at the script again, the pattern for each type becomes clear again, and thus a way to expand it to include Fairy type with the snippet:
```
a001_264:
...
_checkBerries:
...
    if IF_EQUAL, VAR_43, XXXX, _checkFairy
...
_checkFairy:
    if IF_EQUAL, VAR_09, TYPE_FAIRY, _halveDamage
    goto _endScript
...
```
Once again, we need to replace ``XXXX`` with a held item effect.  The Pixie Plate (as added last section) had held effect ``0x93``, so if we aren't replacing another item to implement the Roseli Berry's effect, then we just add onto that by using ``0x94``.  Otherwise can just use the held item effect of the item that you are replacing.
```
a001_264:
...
_checkBerries:
...
    if IF_EQUAL, VAR_43, 0x94, _checkFairy
...
_checkFairy:
    if IF_EQUAL, VAR_09, TYPE_FAIRY, _halveDamage
    goto _endScript
...
```
And finally, the script in which Fairy is spaced out to make clear that it was added there:

<details>
<summary>a001_264 - Fairy Type Reduction Berry Added</summary>

```
a001_264:
    if IF_MASK, VAR_06, 0x8800, _endScript
    if IF_MASK, VAR_10, 0x20, _endScript
    abilitycheck 0x1, BATTLER_ATTACKER, ABILITY_NORMALIZE, _skipNormalize
    changevar VAR_OP_SET, VAR_09, TYPE_NORMAL
    goto _checkBerries
_skipNormalize:
    if IF_EQUAL, VAR_MOVE_TYPE, 0x0, _grabDefaultType
    changevar2 VAR_OP_GET_RESULT, VAR_MOVE_TYPE, VAR_09
    goto _checkBerries
_grabDefaultType:
    getmoveparameter MOVE_DATA_TYPE // stores the move type from the move data table in VAR_09

_checkBerries:
    getitemeffect BATTLER_xFF, VAR_43
    if IF_EQUAL, VAR_43, 0x23, _checkNormal
    if IF_NOTMASK, VAR_10, 0x2, _endScript // if the move used was not supereffective, end the script
    if IF_EQUAL, VAR_43, 0x13, _checkFire
    if IF_EQUAL, VAR_43, 0x14, _checkWater
    if IF_EQUAL, VAR_43, 0x15, _checkElectric
    if IF_EQUAL, VAR_43, 0x16, _checkGrass
    if IF_EQUAL, VAR_43, 0x17, _checkIce
    if IF_EQUAL, VAR_43, 0x18, _checkFighting
    if IF_EQUAL, VAR_43, 0x19, _checkPoison
    if IF_EQUAL, VAR_43, 0x1A, _checkGround
    if IF_EQUAL, VAR_43, 0x1B, _checkFlying
    if IF_EQUAL, VAR_43, 0x1C, _checkPsychic
    if IF_EQUAL, VAR_43, 0x1D, _checkBug
    if IF_EQUAL, VAR_43, 0x1E, _checkRock
    if IF_EQUAL, VAR_43, 0x1F, _checkGhost
    if IF_EQUAL, VAR_43, 0x20, _checkDragon
    if IF_EQUAL, VAR_43, 0x21, _checkDark

    if IF_EQUAL, VAR_43, 0x94, _checkFairy

    if IF_EQUAL, VAR_43, 0x22, _checkSteel
    goto _endScript
_checkNormal:
    if IF_EQUAL, VAR_09, TYPE_NORMAL, _halveDamage
    goto _endScript
_checkFire:
    if IF_EQUAL, VAR_09, TYPE_FIRE, _halveDamage
    goto _endScript
_checkWater:
    if IF_EQUAL, VAR_09, TYPE_WATER, _halveDamage
    goto _endScript
_checkElectric:
    if IF_EQUAL, VAR_09, TYPE_ELECTRIC, _halveDamage
    goto _endScript
_checkGrass:
    if IF_EQUAL, VAR_09, TYPE_GRASS, _halveDamage
    goto _endScript
_checkIce:
    if IF_EQUAL, VAR_09, TYPE_ICE, _halveDamage
    goto _endScript
_checkFighting:
    if IF_EQUAL, VAR_09, TYPE_FIGHTING, _halveDamage
    goto _endScript
_checkPoison:
    if IF_EQUAL, VAR_09, TYPE_POISON, _halveDamage
    goto _endScript
_checkGround:
    if IF_EQUAL, VAR_09, TYPE_GROUND, _halveDamage
    goto _endScript
_checkFlying:
    if IF_EQUAL, VAR_09, TYPE_FLYING, _halveDamage
    goto _endScript
_checkPsychic:
    if IF_EQUAL, VAR_09, TYPE_PSYCHIC, _halveDamage
    goto _endScript
_checkBug:
    if IF_EQUAL, VAR_09, TYPE_BUG, _halveDamage
    goto _endScript
_checkRock:
    if IF_EQUAL, VAR_09, TYPE_ROCK, _halveDamage
    goto _endScript
_checkGhost:
    if IF_EQUAL, VAR_09, TYPE_GHOST, _halveDamage
    goto _endScript
_checkDragon:
    if IF_EQUAL, VAR_09, TYPE_DRAGON, _halveDamage
    goto _endScript
_checkDark:
    if IF_EQUAL, VAR_09, TYPE_DARK, _halveDamage
    goto _endScript

_checkFairy:
    if IF_EQUAL, VAR_09, TYPE_FAIRY, _halveDamage
    goto _endScript

_checkSteel:
    if IF_NOTEQUAL, VAR_09, TYPE_STEEL, _endScript

_halveDamage:
    setstatus2effect BATTLER_xFF, 0xA
    waitmessage
    damagediv VAR_HP_TEMP, 2
    printmessage 1131, TAG_ITEM_MOVE, BATTLER_x15, BATTLER_ATTACKER, "NaN", "NaN", "NaN", "NaN"
    waitmessage
    wait 0x1E
    removeitem BATTLER_xFF

_endScript:
    endscript
```
</details>

## Making a New Stat-Raising Move
Various new moves have stat-raising capabilities that have yet to exist in Gen 4.  Quiver Dance and Hone Claws both come to mind as moves that have unimplemented combos of stats to change, with Hone Claws raising Attack and Accuracy and Quiver Dance raising all of Special Attack, Special Defense, and Speed.

This section will go about implementing those moves using new ``battle_eff_seq`` and ``battle_sub_seq`` scripts while adding a few new entries to ``move_effect_to_subscripts``.  Everything that is needed to do this has been covered above--now it is a matter of implementing it.  Specifically, hg-engine has already done Quiver Dance, but rehashing how it was done will be important.

We start with a new ``battle_eff_seq`` that queues up a ``battle_sub_seq`` once again.  In hg-engine, Quiver Dance's effect is 283, so we add it as that:
```
a030_283:
    changevar VAR_OP_SET, VAR_ADD_STATUS1, ADD_STATUS_ATTACKER | ADD_STATUS_QUIVER_DANCE
    endscript
```
Here, we see that a ``battle_sub_seq`` script queued up through ``VAR_ADD_STATUS1`` that supposedly is ``ADD_STATUS_QUIVER_DANCE``--we define new ``ADD_STATUS`` constants as they appear at the top of ``armips/include/battlescriptcmd.s``.  This even further clarifies the purpose of the script, so that when we're looking back at what we did and see an ``ADD_STATUS_QUIVER_DANCE`` instead of ``148`` we know better what to do.  Ideally, new scripts don't have to use any sort of numbers at all!  We can define whatever constants whenever we need them.

Adding the new entry to ``move_effect_to_subscripts``:
```c
u32 move_effect_to_subscripts[] =
{
...
    [146] = 312, // guard split
    [147] = 313, // power split
    [148] = 314, // quiver dance - new entry
};
```
Seeing that the last ``battle_sub_seq`` used was 313, we can move to map ``ADD_STATUS_QUIVER_DANCE`` to ``battle_sub_seq`` script 314, a new one.

Finally, we can move to make the ``battle_sub_seq`` script, looking at Curse as a non-ghost type as reference:
```
a001_096: // script for Curse as a non-ghost type
    changevar VAR_OP_SET, VAR_34, 0x18 // 24
    gotosubscript 12
    changevar VAR_OP_SETMASK, VAR_06, 0x4001
    changevar VAR_OP_SETMASK, VAR_60, 0x80
    changevar VAR_OP_SET, VAR_34, 0xF // 15
    gotosubscript 12
    changevar VAR_OP_SET, VAR_34, 0x10 // 16
    gotosubscript 12
    changevar VAR_OP_CLEARMASK, VAR_60, 0x2
    changevar VAR_OP_CLEARMASK, VAR_60, 0x80
    endscript
```
As we did before, we can look at ``move_effect_to_subscripts`` and remember what is going on here:
```c
u32 move_effect_to_subscripts[] =
{
...
    [ 15] =  12, // attack +1 // curse queues this up
    [ 16] =  12, // defense +1 // curse queues this up
    [ 17] =  12, // speed +1
    [ 18] =  12, // spatk +1
    [ 19] =  12, // spdef +1
    [ 20] =  12, // accuracy +1
    [ 21] =  12, // evasion +1
    [ 22] =  12, // attack -1
    [ 23] =  12, // defense -1
    [ 24] =  12, // speed -1 // curse queues this up
    [ 25] =  12, // spatk -1
    [ 26] =  12, // spdef -1
    [ 27] =  12, // accuracy -1
    [ 28] =  12, // evasion -1
...
};
```
We can see that Curse queues up speed -1, attack +1 and defense +1 between the bitmasks that it does the ``VAR_60`` with ``gotosubscript 12`` commands, showing that the Pokémon is having its Speed lowered and Attack and Defense increased when Curse occurs.

So we look to make one that raises Special Attack, Special Defense, and Speed:
```
a001_314:
    changevar VAR_OP_SETMASK, VAR_60, 0x80
    changevar VAR_OP_SET, VAR_34, 18 // spatk +1 (see move_effect_to_subscripts)
    gotosubscript 12
    changevar VAR_OP_SET, VAR_34, 19 // spdef +1
    gotosubscript 12
    changevar VAR_OP_SET, VAR_34, 17 // speed +1
    gotosubscript 12
    changevar VAR_OP_CLEARMASK, VAR_60, 0x2
    changevar VAR_OP_CLEARMASK, VAR_60, 0x80
    endscript
```
And thus the Quiver Dance script is done.  The ``statbuffchange`` command from ``battle_sub_seq`` script 12 handles ensuring that all the stats raised don't overflow or anything, failing if they do.

## Making a Move that Sets Attacker to Random Typing
Here we are looking to add some randomness to our battle script to determine the effect that happens.  For this, we can use the ``random`` battle script command:
```
random range, start
chooses a random number between "start" and "start"+"range" inclusive, storing it in VAR_09
- range is the range of numbers above "start" to choose from
- start is the beginning of the random numbers to choose from
```
The goal is to set the attacker's typing to a random type that isn't the one it already has.  We make a new ``battle_eff_seq`` that queues up a new ``battle_sub_seq`` script applying to the attacker (with ``ADD_STATUS_ATTACKER``) to set the player's type to something in and of itself.  We skip the ``battle_eff_seq`` script and move to the ``battle_sub_seq``, as this ``battle_eff_seq`` is similar to the rest of them.
```
setType:
    changevartomonvalue VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_TYPE_1, TYPE_WATER
    changevartomonvalue VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_TYPE_2, TYPE_WATER
    endscript
```
This script sets the Pokémon's type to be monotype water.  There is no indication that it happens--it just happens, and the animation of the move plays.  We add a message to print as well as a jump to subscript 76 so that the animation and all plays in order:
```
setType:
    changevartomonvalue VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_TYPE_1, TYPE_WATER
    changevartomonvalue VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_TYPE_2, TYPE_WATER
    gotosubscript 76
    printmessage 178, TAG_NICK_TYPE, BATTLER_ATTACKER, BATTLER_WORK, "NaN", "NaN", "NaN", "NaN" // {STRVAR_1 1, 0, 0} transformed\ninto the {STRVAR_1 15, 1, 0} type!
    waitmessage
    wait 0x1E
    endscript
```
We can directly change the type to Water this way and even use the Conversion battle message for it (from ``armips/move/battle_sub_seq/045.s``).  When the type changes in-game, however, the new type fails to buffer correctly.  For this, we turn to a new variable:  ``VAR_22``.  ``VAR_22`` is the temporary message buffer variable, and it determines many indices that the script system uses to buffer words into string variables.  When using ``TAG_NICK_TYPE``, the ``printmessage`` command first buffers the nickname of the Pokémon that is specified by the first ``battler`` parameter.  Then the type name is buffered using the value in ``VAR_22`` directly.

Changing the script to buffer the Water type properly:
```
setType:
    changevartomonvalue VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_TYPE_1, TYPE_WATER
    changevartomonvalue VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_TYPE_2, TYPE_WATER
    changevar VAR_OP_SET, VAR_22, TYPE_WATER
    gotosubscript 76
    printmessage 178, TAG_NICK_TYPE, BATTLER_ATTACKER, BATTLER_WORK, "NaN", "NaN", "NaN", "NaN" // {STRVAR_1 1, 0, 0} transformed\ninto the {STRVAR_1 15, 1, 0} type!
    waitmessage
    wait 0x1E
    endscript
```
Now we need to make sure that the move just fails if the user is already Water-type (but only if both types are water--the move will work on Water/Poison mons, for example):
```
setType:
    ifmonstat IF_NOT_EQUAL, BATTLER_DEFENDER, MON_DATA_TYPE_1, TYPE_WATER, moveWorks
    ifmonstat IF_EQUAL, BATTLER_DEFENDER, MON_DATA_TYPE_2, TYPE_WATER, moveFails

moveWorks:
    changevartomonvalue VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_TYPE_1, TYPE_WATER
    changevartomonvalue VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_TYPE_2, TYPE_WATER
    changevar VAR_OP_SET, VAR_22, TYPE_WATER
    gotosubscript 76
    printmessage 178, TAG_NICK_TYPE, BATTLER_ATTACKER, BATTLER_WORK, "NaN", "NaN", "NaN", "NaN" // {STRVAR_1 1, 0, 0} transformed\ninto the {STRVAR_1 15, 1, 0} type!
    waitmessage
    wait 0x1E
    endscript

moveFails:
    changevar VAR_OP_SETMASK, VAR_10, 0x40
    endscript
```
Here, we add 2 ``ifmonstat`` checks to see if the types are both Water or not.  Note that the second one only ever runs if the first one is successful--i.e. if the script execution falls through to ``moveWorks`` (without the jump from the first ``ifmonstat``) then the first Pokémon's type is Water, but the second is different--which still qualifies for what we want to do with it.

Now we need to generate a random type and adjust all of these checks to be against another variable instead of against a constant.  ``ifmonstat`` becomes ``ifmonstat2``, ``changevartomonvalue`` becomes ``changevartomonvalue2``, ``changevar`` becomes ``changevar2``.  A quick documentation addition:
```
ifmonstat2 operator, battler, field, variable, address
ifmonstat except variable-based
jump to "address" if the "battler"'s stat designated by "field" is related to "variable" as determined by "operator"
- operator is the same as the "if" operators
- battler is the pokémon to get data from
- field is the data to grab from.  enumerations below
- variable is the variable to check against, used in the operator calculations as they appear in "if"
- address is the location to jump to when the if is true
```
```
setRandomType:
    random NUM_OF_TYPES-1, 0 // generate random number from 0 - (NUM_OF_TYPES - 1) inclusive, store it in VAR_09
    if IF_EQUAL, VAR_09, TYPE_MYSTERY, setRandomType // if fairy type hasn't been introduced as type 9, need to check for it here and regenerate a type if necessary

    ifmonstat2 IF_NOT_EQUAL, BATTLER_DEFENDER, MON_DATA_TYPE_1, VAR_09, moveWorks
    ifmonstat2 IF_EQUAL, BATTLER_DEFENDER, MON_DATA_TYPE_2, VAR_09, moveFails

moveWorks:
    changevartomonvalue2 VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_TYPE_1, VAR_09
    changevartomonvalue2 VAR_OP_SET, BATTLER_DEFENDER, MON_DATA_TYPE_2, VAR_09
    changevar2 VAR_OP_SET, VAR_22, VAR_09
    gotosubscript 76
    printmessage 178, TAG_NICK_TYPE, BATTLER_ATTACKER, BATTLER_WORK, "NaN", "NaN", "NaN", "NaN" // {STRVAR_1 1, 0, 0} transformed\ninto the {STRVAR_1 15, 1, 0} type!
    waitmessage
    wait 0x1E
    endscript

moveFails:
    changevar VAR_OP_SETMASK, VAR_10, 0x40
    endscript
```


# Battle Script Command Reference
<details>
<summary>Battle Script Command Reference Dropdown</summary>

<details>
<summary>startencounter - 0x00</summary>

```
startencounter
initializes battle information.  not much is known on this command, and the name is speculative
```
</details>
<details>
<summary>pokemonencounter - 0x01</summary>

```
pokemonencounter battler
initalizes wild information.  not much is known on this command, and the name is speculative
- battler is the pokémon that is being initialized, i.e. is called twice for doubles
```
</details>
<details>
<summary>pokemonslidein - 0x02</summary>

```
pokemonslidein battler
queues up the animation for the pokémon sliding in as in a wild battle.
- battler is the pokémon sliding in
```
</details>
<details>
<summary>pokemonappear - 0x03</summary>

```
pokemonappear battler
sends out the pokémon from the poké ball it is in, solely initializing the sprite.  not much is known on this command, and the name is speculative
- battler is the pokémon being sent out
```
</details>
<details>
<summary>returnpokemon - 0x04</summary>

```
returnpokemon battler
returns the pokémon to the trainer in its ball.  not much is known on this command, and the name is speculative
- battler is the pokémon returning
```
</details>
<details>
<summary>deletepokemon - 0x05</summary>

```
deletepokemon battler
deletes the sprite of battler.  not much is known on this command, and the name is speculative
- battler is the pokémon whose sprite is being deleted
```
</details>
<details>
<summary>starttrainerencounter - 0x06</summary>

```
starttrainerencounter battler
initalizes the trainer party.  not much is known on this command, and the name is speculative
- battler is the trainer being initialized
```
</details>
<details>
<summary>throwpokeball - 0x07</summary>

```
throwpokeball battler, type
throws a poké ball at the enemy.
- battler is always "BATTLER_PLAYER" or "BATTLER_OPPONENT" depending on where the ball is being thrown from
- type is the animation to be played

type enumerations:
#define THROWPOKEBALL_TYPE_SEND_OUT_MON 0
#define THROWPOKEBALL_TYPE_THROW_BALL_AT_WILD 1
#define THROWPOKEBALL_TYPE_THROW_MUD 2
#define THROWPOKEBALL_TYPE_THROW_BAIT 3
#define THROWPOKEBALL_TYPE_RETURN_MON_TO_BALL 4
```
</details>
<details>
<summary>preparetrainerslide - 0x08</summary>

```
preparetrainerslide battler
load in the gfx for the trainer.  not much is known on this command, and the name is speculative
- battler is the trainer position about to slide in
```
</details>
<details>
<summary>trainerslidein - 0x09</summary>

```
trainerslidein battler, pos
not much is known on this command, and the name is speculative
- battler is the trainer sliding in
- pos is where the trainer is sliding to
```
</details>
<details>
<summary>backgroundslidein - 0x0A</summary>

```
backgroundslidein
not much is known on this command, and the name is speculative
```
</details>
<details>
<summary>hpgaugeslidein - 0x0B</summary>

```
hpgaugeslidein battler
not much is known on this command, and the name is speculative
- battler is the hp gauge sliding in
```
</details>
<details>
<summary>hpgaugeslidewait - 0x0C</summary>

```
hpgaugeslidewait battler
not much is known on this command, and the name is speculative
- battler is the hp gauge sliding in to wait for
```
</details>
<details>
<summary>preparehpgaugeslide - 0x0D</summary>

```
preparehpgaugeslide
not much is known on this command, and the name is speculative
- battler is the hp gauge to initialize to slide in
```
</details>
<details>
<summary>waitmessage - 0x0E</summary>

```
waitmessage
pause script execution until current message is done printing.  not just done for messages, also used for various states that take up time that script execution needs to pause for (although not animations).
```
</details>
<details>
<summary>damagecalc - 0x0F</summary>

```
damagecalc
the basic damage calculator.  called for all damaging moves, but will disable itself if a separate damage calc is detected
```
</details>
<details>
<summary>damagecalc2 - 0x10</summary>

```
damagecalc2
not much is known on this command, and the name is speculative
```
</details>
<details>
<summary>printattackmessage - 0x11</summary>

```
printattackmessage
prints the current attack message from a027 file 003 (data/text/003.txt).
```
</details>
<details>
<summary>printmessage - 0x12</summary>

```
printmessage id, tag, (varargs) battlers 1-6
prints a message
- id is the message index in the narc a027 file 197 (data/text/197.txt)
- tag determines the strings that are buffered in which order from the Pokémon on the field
- the battlers determine which battlers on the field to grab information to buffer strings from
  - the tag detemines how many battlers are specified--"NaN" signifies that no battler will be built from that parameter
    - this is because of a limitation with armips where i can't overload a macro name to have different types

message tags:
#define TAG_NONE                        (0)     //nothing

#define TAG_NONE_DIR                    (1)     //nothing (but judgment type?)
#define TAG_NICK                        (2)     //nickname
#define TAG_MOVE                        (3)     //move
#define TAG_STAT                        (4)     //stat
#define TAG_ITEM                        (5)     //helditem
#define TAG_NUM                         (6)     //number
#define TAG_NUMS                        (7)     //number(right aligned)
#define TAG_TRNAME                      (8)     //trainername

#define TAG_NICK_NICK                   (9)     //nickname      nickname
#define TAG_NICK_MOVE                   (10)    //nickname      move
#define TAG_NICK_ABILITY                (11)    //nickname      ability
#define TAG_NICK_STAT                   (12)    //nickname      stat
#define TAG_NICK_TYPE                   (13)    //nickname      type
#define TAG_NICK_POKE                   (14)    //nickname      pokemon
#define TAG_NICK_ITEM                   (15)    //nickname      helditem
#define TAG_NICK_UNK                    (16)    //nickname      ?
#define TAG_NICK_NUM                    (17)    //nickname      number
#define TAG_NICK_TRNAME                 (18)    //nickname      trainername
#define TAG_NICK_BOX                    (19)    //nickname      boxname
#define TAG_MOVE_DIR                    (20)    //move          (but judgment type?)
#define TAG_MOVE_NICK                   (21)    //move          nickname
#define TAG_MOVE_MOVE                   (22)    //move          move
#define TAG_ABILITY_NICK                (23)    //ability       nickname
#define TAG_ITEM_MOVE                   (24)    //helditem      move
#define TAG_NUM_NUM                     (25)    //number        number
#define TAG_TRNAME_TRNAME               (26)    //trainername   trainername
#define TAG_TRNAME_NICK                 (27)    //trainername   nickname
#define TAG_TRNAME_ITEM                 (28)    //trainername   helditem
#define TAG_TRNAME_NUM                  (29)    //trainername   number
#define TAG_TRTITLE_TRNAME              (30)    //trainertitle  trainername

#define TAG_NICK_NICK_MOVE              (31)    //nickname      nickname        move
#define TAG_NICK_NICK_ABILITY           (32)    //nickname      nickname        ability
#define TAG_NICK_NICK_ITEM              (33)    //nickname      nickname        helditem
#define TAG_NICK_MOVE_MOVE              (34)    //nickname      move            move
#define TAG_NICK_MOVE_NUM               (35)    //nickname      move            number
#define TAG_NICK_ABILITY_NICK           (36)    //nickname      ability         nickname
#define TAG_NICK_ABILITY_MOVE           (37)    //nickname      ability         move
#define TAG_NICK_ABILITY_ITEM           (38)    //nickname      ability         helditem
#define TAG_NICK_ABILITY_STAT           (39)    //nickname      ability         stat
#define TAG_NICK_ABILITY_TYPE           (40)    //nickname      ability         type
#define TAG_NICK_ABILITY_COND           (41)    //nickname      ability         condition
#define TAG_NICK_ABILITY_NUM            (42)    //nickname      ability         number
#define TAG_NICK_ITEM_NICK              (43)    //nickname      helditem        nickname
#define TAG_NICK_ITEM_MOVE              (44)    //nickname      helditem        move
#define TAG_NICK_ITEM_STAT              (45)    //nickname      helditem        stat
#define TAG_NICK_ITEM_COND              (46)    //nickname      helditem        condition
#define TAG_NICK_BOX_BOX                (47)    //nickname      box             box
#define TAG_ITEM_NICK_TASTE             (48)    //helditem      nickname        taste
#define TAG_TRNAME_NICK_NICK            (49)    //trainername   nickname        nickname
#define TAG_TRTYPE_TRNAME_NICK          (50)    //trainertitle  trainername     nickname
#define TAG_TRTYPE_TRNAME_ITEM          (51)    //trainertitle  trainername     helditem

#define TAG_NICK_ABILITY_NICK_MOVE      (52)    //nickname      ability         nickname        move
#define TAG_NICK_ABILITY_NICK_ABILITY   (53)    //nickname      ability         nickname        ability
#define TAG_NICK_ABILITY_NICK_STAT      (54)    //nickname      ability         nickname        stat
#define TAG_NICK_ITEM_NICK_ITEM         (55)    //nickname      helditem        nickname        helditem
#define TAG_TRNAME_NICK_TRNAME_NICK     (56)    //trainername   nickname        trainername     nickname
#define TAG_TRTYPE_TRNAME_NICK_NICK     (57)    //trainertitle  trainername     nickname        nickname
#define TAG_TRTYPE_TRNAME_NICK_TRNAME   (58)    //trainertitle  trainername     nickname        trainername
#define TAG_TRTYPE_TRNAME_TRTYPE_TRNAME (59)    //trainertitle  trainername     trainertitle    trainername

#define TAG_TRTYPE_TRNAME_NICK_TRTYPE_TRNAME_NICK (60)    //trainertitle  trainername     nickname        trainertitle  trainername     nickname

```
</details>
<details>
<summary>printmessage2 - 0x13</summary>

```
printmessage2 id, tag, (varargs) battlers 1-6
prints a message, can't seem to tell the difference between this and printmessage
- id is the message index in the narc a027 file 197 (data/text/197.txt)
- tag determines the strings that are buffered in which order from the Pokémon on the field.  tags are enumerated in "printmessage"
- the battlers determine which battlers on the field to grab information to buffer strings from
  - the tag detemines how many battlers are specified--"NaN" signifies that no battler will be built from that parameter
    - this is because of a limitation with armips where i can't overload a macro name to have different types
```
</details>
<details>
<summary>printpreparedmessage - 0x14</summary>

```
printpreparedmessage
prints the message prepared by "preparemessage"
```
</details>
<details>
<summary>preparemessage - 0x15</summary>

```
preparemessage id, tag, (varargs) battlers 1-6
prepares a message to be used by printpreparedmessage
- id is the message index in the narc a027 file 197
- tag determines the strings that are buffered in which order from the Pokémon on the field.  tags are enumerated in "printmessage"
- the battlers determine which battlers on the field to grab information to buffer strings from
  - the tag detemines how many battlers are specified--"NaN" signifies that no battler will be built from that parameter

see printmessage for tag id documentation
```
</details>
<details>
<summary>printmessagepassbattler - 0x16</summary>

```
printmessagepassbattler battler, id, tag, (varargs) battlers 1-6
this is currently used just once (armips/move/battle_sub_seq/050.s).  name is speculative, is used to swap between side statuses?
prints "Your team’s {STRVAR_1 6, 0, 0}\nwore off!" or "The foe’s {STRVAR_1 6, 0, 0}\nwore off!" under "TAG_MOVE"
- battler appears to be the battler that belongs to the side that switches the message id.  if on the opponent's side, will add one to the message id.
- id is the message index in the narc a027 file 197
- tag determines the strings that are buffered in which order from the Pokémon on the field.  tags are enumerated in "printmessage"
- the battlers determine which battlers on the field to grab information to buffer strings from
  - the tag detemines how many battlers are specified--"NaN" signifies that no battler will be built from that parameter
```
</details>
<details>
<summary>seteffectprimary  - 0x17 (suggested name: playanimation)</summary>

```
seteffectprimary battler
a more accurate name would be "playanimation," will work on fixing this.
plays the current attack's animation based on "battler"
- battler is the primary battler to base the animation on
```
</details>
<details>
<summary>seteffectsecondary  - 0x18 (suggested name: playanimation2)</summary>

```
seteffectsecondary battler, attacker, defender
a more accurate name would be "playanimation2," will work on fixing this.
plays the current attack's animation based on "battler".  notably, transform only works under this for some reason.
- battler is the primary battler to base the animation on
- attacker is the battler used as the attacker
- defender is the battler used as the defender
```
</details>
<details>
<summary>monflicker - 0x19</summary>

```
monflicker battler
makes "battler" flicker as if hit
- battler is the Pokémon to flicker
```
</details>
<details>
<summary>datahpupdate - 0x1A</summary>

```
datahpupdate battler
updates the information on the "battler"'s hp bar, specifically the hp data
- battler is the owner of the hp data to update
```
</details>
<details>
<summary>healthbarupdate - 0x1B</summary>

```
healthbarupdate battler
updates "battler"'s hp bar
- battler is the battler that has the hp bar to update
```
</details>
<details>
<summary>tryfaintmon - 0x1C</summary>

```
tryfaintmon battler
tries to faint "battler".  nothing happens if fails, the mon will slide down otherwise
- battler is the mon to faint
```
</details>
<details>
<summary>dofaintanimation - 0x1D</summary>

```
dofaintanimation
i believe it plays the animation that drags down BATTLER_FAINTED as written to by "tryfaintmon"
```
</details>
<details>
<summary>wait - 0x1E</summary>

```
wait time
pause script execution for "time" frames
- time is the amount of frames to pause for
```
</details>
<details>
<summary>playse - 0x1F</summary>

```
playse battler, id
play sound effect "id" with pan based on "battler"
- battler is the battler that the sound effect will be biased towards depending on which side of the field it is on
- id is the sound effect id to play
```
</details>
<details>
<summary>if - 0x20</summary>

```
if operator, var, value, address
conditional flow command
- operator is the math operation done on the variable, enumerated below
- var is the variable with the value to test
- value is the argument for the operator, always a constant for if
- address is the destination that the script will jump to if the if operator returns true

if conditional operators:
#define IF_EQUAL    0 // var == value
#define IF_NOTEQUAL 1 // var != value
#define IF_GREATER  2 // var > value
#define IF_LESSTHAN 3 // var < value
#define IF_MASK     4 // var | value // i think
#define IF_NOTMASK  5 // !(var | value) // i think
#define IF_AND      6 // (var & value) // i think
```
</details>
<details>
<summary>ifmonstat - 0x21</summary>

```
ifmonstat operator, battler, field, value, address
jump to "address" if the "battler"'s stat designated by "field" is related to "value" as determined by "operator"
- operator is the math operation done on the variable, enumerated in "if"
- battler is the pokémon to get data from
- field is the data to grab from.  enumerations below
- value is the value to check against, used in the operator calculations as they appear in "if"
- address is the location to jump to when the if is true

ifmonstat fields:
#define MON_DATA_SPECIES (0)
#define MON_DATA_ATTACK (1)
#define MON_DATA_DEFENSE (2)
#define MON_DATA_SPEED (3)
#define MON_DATA_SPATK (4)
#define MON_DATA_SPDEF (5)
#define MON_DATA_MOVE_1 (6)
#define MON_DATA_MOVE_2 (7)
#define MON_DATA_MOVE_3 (8)
#define MON_DATA_MOVE_4 (9)
#define MON_DATA_10 (10)
#define MON_DATA_11 (11)
#define MON_DATA_12 (12)
#define MON_DATA_13 (13)
#define MON_DATA_14 (14)
#define MON_DATA_15 (15)
#define MON_DATA_EGG_FLAG (16)
#define MON_DATA_NICKNAME_FLAG (17)
#define MON_DATA_18 (18)
#define MON_DATA_STAT_STAGE_ATTACK (19)
#define MON_DATA_STAT_STAGE_DEFENSE (20)
#define MON_DATA_STAT_STAGE_SPEED (21)
#define MON_DATA_STAT_STAGE_SPATK (22)
#define MON_DATA_STAT_STAGE_SPDEF (23)
#define MON_DATA_STAT_STAGE_ACCURACY (24)
#define MON_DATA_STAT_STAGE_EVASION (25)
#define MON_DATA_ABILITY (26)
#define MON_DATA_TYPE_1 (27)
#define MON_DATA_TYPE_2 (28)
#define MON_DATA_GENDER (29)
#define MON_DATA_30 (30)
#define MON_DATA_PP_1 (31)
#define MON_DATA_PP_2 (32)
#define MON_DATA_PP_3 (33)
#define MON_DATA_PP_4 (34)
#define MON_DATA_PP_BONUS_1 (35)
#define MON_DATA_PP_BONUS_2 (36)
#define MON_DATA_PP_BONUS_3 (37)
#define MON_DATA_PP_BONUS_4 (38)
#define MON_DATA_PP_MAX_1 (39)
#define MON_DATA_PP_MAX_2 (40)
#define MON_DATA_PP_MAX_3 (41)
#define MON_DATA_PP_MAX_4 (42)
#define MON_DATA_LEVEL (43)
#define MON_DATA_FRIENDSHIP (44)
#define MON_DATA_NICKNAME (45)
#define MON_DATA_46 (46)
#define MON_DATA_HP (47)
#define MON_DATA_MAX_HP (48)
#define MON_DATA_49 (49)
#define MON_DATA_EXP (50)
#define MON_DATA_PID (51)
#define MON_DATA_STATUS_1 (52)
#define MON_DATA_STATUS_2 (53)
#define MON_DATA_54 (54)
#define MON_DATA_ITEM (55)
#define MON_DATA_56 (56)
#define MON_DATA_57 (57)
#define MON_DATA_58 (58)
#define MON_DATA_MOVE_STATE (59)
#define MON_DATA_MOVE_STATE_2 (60)
#define MON_DATA_DISABLE_COUNTER (61)
#define MON_DATA_ENCORE_COUNTER (62)
#define MON_DATA_CHARGE_COUNTER (63)
#define MON_DATA_TAUNT_COUNTER (64)
#define MON_DATA_65 (65)
#define MON_DATA_PERISH_SONG_COUNTER (66)
#define MON_DATA_ROLLOUT_COUNTER (67)
#define MON_DATA_FURY_CUTTER_COUNTER (68)
#define MON_DATA_STOCKPILE_COUNT (69)
#define MON_DATA_STOCKPILE_DEF_COUNT (70)
#define MON_DATA_STOCKPILE_SPDEF_COUNT (71)
#define MON_DATA_72 (72)
#define MON_DATA_73 (73)
#define MON_DATA_LOCKON_TARGET (74)
#define MON_DATA_MIMIC_BIT (75)
#define MON_DATA_BIND_TARGET (76)
#define MON_DATA_MEAN_LOOK_TARGET (77)
#define MON_DATA_LAST_RESORT_COUNTER (78)
#define MON_DATA_79 (79)
#define MON_DATA_HEAL_BLOCK_COUNTER (80)
#define MON_DATA_81 (81)
#define MON_DATA_82 (82)
#define MON_DATA_METRONOME_VAR (83)
#define MON_DATA_84 (84)
#define MON_DATA_85 (85)
#define MON_DATA_86 (86)
#define MON_DATA_87 (87)
#define MON_DATA_FAKE_OUT_COUNTER (88)
#define MON_DATA_SLOW_START_COUNTER (89)
#define MON_DATA_SUBSTITUTE_HP (90)
#define MON_DATA_91 (91)
#define MON_DATA_DISABLED_MOVE (92)
#define MON_DATA_ENCORED_MOVE (93)
#define MON_DATA_94 (94)
#define MON_DATA_HP_RECOVERED_BY_ITEM (95)
#define MON_DATA_SLOW_START_ACTIVE (96)
#define MON_DATA_SLOW_START_INACTIVE (97)
#define MON_DATA_FORM (98)
#define MON_DATA_VARIABLE (100)
```
</details>
<details>
<summary>fadeout - 0x22</summary>

```
fadeout
fades the battle in preparation for the battle ending and whatever else to take over
```
</details>
<details>
<summary>jumptosubseq - 0x23</summary>

```
jumptosubseq id
jumps to the battle_sub_seq script "id" without return (whereas gotosubscript and gotosubscript2 will return upon completion)
- id is the battle_sub_seq script to jump to
```
</details>
<details>
<summary>jumptocurmoveeffectscript - 0x24</summary>

```
jumptocurmoveeffectscript
jumps to the current move's battle_eff_seq script.  most of the battle_move_seq scripts are just this command.
```
</details>
<details>
<summary>jumptoeffectscript - 0x25</summary>

```
jumptoeffectscript id
jumps to battle_eff_seq "id"
- id is the battle_eff_seq script to jump to
```
</details>
<details>
<summary>critcalc - 0x26</summary>

```
critcalc
calculates the critical multiplier (set to 1 in the case that there isn't one)
```
</details>
<details>
<summary>shouldgetexp - 0x27</summary>

```
shouldgetexp address
used in battle_sub_seq script 276 to determine if any pokemon should get experience, jumps to "address" otherwise
- address is the address to jump to if nobody gets experience
```
</details>
<details>
<summary>initexpget - 0x28</summary>

```
initexpget
- 
```
</details>
<details>
<summary>getexp - 0x29</summary>

```
getexp
- 
```
</details>
<details>
<summary>getexploop - 0x2A</summary>

```
getexploop
- 
```
</details>
<details>
<summary>showmonlist - 0x2B</summary>

```
showmonlist
- 
```
</details>
<details>
<summary>waitformonselection - 0x2C</summary>

```
waitformonselection
- 
```
</details>
<details>
<summary>switchindataupdate - 0x2D</summary>

```
switchindataupdate
- 
```
</details>
<details>
<summary>jumpifcantswitch - 0x2E</summary>

```
jumpifcantswitch
- 
```
</details>
<details>
<summary>initcapture - 0x2F</summary>

```
initcapture
- 
```
</details>
<details>
<summary>capturemon - 0x30</summary>

```
capturemon
- 
```
</details>
<details>
<summary>setmultihit - 0x31</summary>

```
setmultihit
- 
```
</details>
<details>
<summary>changevar - 0x32</summary>

```
changevar operator, var, value
perform math operations on "var" using a constant "value"
- operator is the math operation done on the variable
- var is the variable to change
- value is the argument for the operator

changevar operators:
#define VAR_OP_SET         ( 7) // var = value;
#define VAR_OP_ADD         ( 8) // var += value;
#define VAR_OP_SUB         ( 9) // var -= value;
#define VAR_OP_SETMASK     (10) // var |= value;
#define VAR_OP_CLEARMASK   (11) // var &= ~(value);
#define VAR_OP_MUL         (12) // var *= value;
#define VAR_OP_DIV         (13) // var /= value;
#define VAR_OP_LSH         (14) // var <<= value;
#define VAR_OP_RSH         (15) // var >>= value;
#define VAR_OP_TO_BIT      (16) // var = 1 << value;
#define VAR_OP_GET_RESULT  (17) // var = result of script command?  used with changemondatabyvar to put mondata in var // needs research
#define VAR_OP_SUB_TO_ZERO (18) // while (var > value) { var -= value; }
#define VAR_OP_XOR         (19) // var ^= value;
#define VAR_OP_AND         (20) // var &= value;
```
</details>
<details>
<summary>statbuffchange - 0x33</summary>

```
statbuffchange
- 
```
</details>
<details>
<summary>changevartomonvalue - 0x34 (suggested name:  changemondatabyvalue)</summary>

```
changevartomonvalue operator, battler, field, value
a more accurate name would be "changemondatabyvalue," will work on fixing this.
changes mon data "field" by "value" as specified by "operator"
- operator is the same as the "changevar" operators
- battler is the battler to grab the data from
- field is the data to grab/set from the battler, same as "ifmonstat"
- value is the argument for the operator
```
</details>
<details>
<summary>clearstatus2 - 0x35</summary>

```
clearstatus2
- 
```
</details>
<details>
<summary>togglevanish - 0x36</summary>

```
togglevanish
- 
```
</details>
<details>
<summary>abilitycheck - 0x37</summary>

```
abilitycheck checker, battler, ability, destination
jump to "destination" if "battler" has or doesn't have "ability" based on "checker"--if "checker" is 0, jumps if the "battler" has the "ability", otherwise jump if doesn't have "ability"
- checker determines to check if "battler" has or doesn't have "ability"--if "checker" is 0, jumps if the "battler" has the "ability", otherwise jump if doesn't have "ability"
- battler is the battler whose ability to check
- ability is the ability to check for
- destination is where to jump if the check succeeds
```
</details>
<details>
<summary>random - 0x38</summary>

```
random range, start
chooses a random number between "start" and "start"+"range" inclusive, storing it in VAR_09
- range is the range of numbers above "start" to choose from
- start is the beginning of the random numbers to choose from
```
</details>
<details>
<summary>changevar2 - 0x39</summary>

```
changevar2 operator, destvar, srcvar
changevar except the constant is now a battle variable
- operator is the same as changevar
- destvar is a variable that may or may not hold a value already that will be changed
- srcvar is a variable that operator uses to complete its operation

see changevar for operator enumerations
```
</details>
<details>
<summary>changevartomonvalue2 - 0x3A (suggested name:  changemondatabyvar)</summary>

```
changevartomonvalue2 operator, battler, field, var
a more accurate name would be "changemondatabyvar," will work on fixing this.
changes mon data "field" by "var"'s value as specified by "operator"
- operator is the same as the "changevar" operators.  if looking to set the var to the mon data field, use VAR_OP_GET_RESULT
- battler is the battler to grab the data from
- field is the data to grab/set from the battler
- var is the variable the engine grabs from to manipulate the mon data field
```
</details>
<details>
<summary>goto - 0x3B</summary>

```
goto
- 
```
</details>
<details>
<summary>gotosubscript - 0x3C</summary>

```
gotosubscript num
calls battle_sub_seq script "num".  returns to the caller after an endscript is reached
- num is the index of the battle_sub_seq script to jump to
```
</details>
<details>
<summary>gotosubscript2 - 0x3D</summary>

```
gotosubscript2
- 
```
</details>
<details>
<summary>checkifchatot - 0x3E</summary>

```
checkifchatot
- 
```
</details>
<details>
<summary>sethaze - 0x3F</summary>

```
sethaze
- 
```
</details>
<details>
<summary>setsomeflag - 0x40</summary>

```
setsomeflag
- 
```
</details>
<details>
<summary>clearsomeflag - 0x41</summary>

```
clearsomeflag
- 
```
</details>
<details>
<summary>setstatusicon - 0x42</summary>

```
setstatusicon
- 
```
</details>
<details>
<summary>trainermessage - 0x43</summary>

```
trainermessage
- 
```
</details>
<details>
<summary>calcmoney - 0x44</summary>

```
calcmoney
- 
```
</details>
<details>
<summary>setstatus2effect - 0x45</summary>

```
setstatus2effect battler, value
sets status2 "value" bits for "battler" (condition2 in the BattlePokemon structure)
- battler is the battler to grab status2 from
- value comprises the bits to set in status2
```
</details>
<details>
<summary>setstatus2effect2 - 0x46</summary>

```
setstatus2effect2
- 
```
</details>
<details>
<summary>setstatus2effect3 - 0x47</summary>

```
setstatus2effect3
- 
```
</details>
<details>
<summary>returnmessage - 0x48</summary>

```
returnmessage
- 
```
</details>
<details>
<summary>sentoutmessage - 0x49</summary>

```
sentoutmessage
- 
```
</details>
<details>
<summary>encountermessage - 0x4A</summary>

```
encountermessage
- 
```
</details>
<details>
<summary>encountermessage2 - 0x4B</summary>

```
encountermessage2
- 
```
</details>
<details>
<summary>trainermessage2 - 0x4C</summary>

```
trainermessage2
- 
```
</details>
<details>
<summary>tryconversion - 0x4D</summary>

```
tryconversion
- 
```
</details>
<details>
<summary>if2 - 0x4E</summary>

```
if2 operator, var1, var2, address
jump to "address" if "var1" is related to "var2" as determined by "operator"
- operator is the same as the "if" operators
- var1 is a var to compare
- var2 is another var to compare against (i.e. IF_LESSTHAN is true if var1 < var2)
- address is the location to jump to when the if is true
```
</details>
<details>
<summary>ifmonstat2 - 0x4F</summary>

```
ifmonstat2 operator, battler, field, variable, address
ifmonstat except variable-based
jump to "address" if the "battler"'s stat designated by "field" is related to "variable" as determined by "operator"
- operator is the same as the "if" operators
- battler is the pokémon to get data from
- field is the data to grab from.  enumerations below
- variable is the variable to check against, used in the operator calculations as they appear in "if"
- address is the location to jump to when the if is true
```
</details>
<details>
<summary>dopayday - 0x50</summary>

```
dopayday
- 
```
</details>
<details>
<summary>setlightscreen - 0x51</summary>

```
setlightscreen
- 
```
</details>
<details>
<summary>setreflect - 0x52</summary>

```
setreflect
- 
```
</details>
<details>
<summary>setmist - 0x53</summary>

```
setmist
- 
```
</details>
<details>
<summary>tryonehitko - 0x54</summary>

```
tryonehitko
- 
```
</details>
<details>
<summary>damagediv - 0x55</summary>

```
damagediv var, value
sets damage to be "var" / "value"
- var is the numerator in the division
- value is the denominator in the division
```
</details>
<details>
<summary>damagediv2 - 0x56</summary>

```
damagediv2
- 
```
</details>
<details>
<summary>trymimic - 0x57</summary>

```
trymimic
- 
```
</details>
<details>
<summary>metronome - 0x58</summary>

```
metronome
- 
```
</details>
<details>
<summary>trydisable - 0x59</summary>

```
trydisable
- 
```
</details>
<details>
<summary>counter - 0x5A</summary>

```
counter
- 
```
</details>
<details>
<summary>mirrorcoat - 0x5B</summary>

```
mirrorcoat
- 
```
</details>
<details>
<summary>tryencore - 0x5C</summary>

```
tryencore
- 
```
</details>
<details>
<summary>tryconversion2 - 0x5D</summary>

```
tryconversion2
- 
```
</details>
<details>
<summary>trysketch - 0x5E</summary>

```
trysketch
- 
```
</details>
<details>
<summary>trysleeptalk - 0x5F</summary>

```
trysleeptalk
- 
```
</details>
<details>
<summary>flaildamagecalc - 0x60</summary>

```
flaildamagecalc
- 
```
</details>
<details>
<summary>tryspite - 0x61</summary>

```
tryspite
- 
```
</details>
<details>
<summary>healbell - 0x62</summary>

```
healbell
- 
```
</details>
<details>
<summary>trythief - 0x63</summary>

```
trythief
- 
```
</details>
<details>
<summary>willprotectwork - 0x64</summary>

```
willprotectwork
- 
```
</details>
<details>
<summary>trysubstitute - 0x65</summary>

```
trysubstitute
- 
```
</details>
<details>
<summary>trywhirlwind - 0x66</summary>

```
trywhirlwind
- 
```
</details>
<details>
<summary>transform - 0x67</summary>

```
transform
- 
```
</details>
<details>
<summary>tryspikes - 0x68</summary>

```
tryspikes
- 
```
</details>
<details>
<summary>checkspikes - 0x69</summary>

```
checkspikes
- 
```
</details>
<details>
<summary>tryperishsong - 0x6A</summary>

```
tryperishsong
- 
```
</details>
<details>
<summary>orderbattlersbyspeed - 0x6B</summary>

```
orderbattlersbyspeed
- 
```
</details>
<details>
<summary>exitloopatvalue - 0x6C</summary>

```
exitloopatvalue
- 
```
</details>
<details>
<summary>weatherdamagecalc - 0x6D</summary>

```
weatherdamagecalc
- 
```
</details>
<details>
<summary>rolloutdamagecalc - 0x6E</summary>

```
rolloutdamagecalc
- 
```
</details>
<details>
<summary>furycutterdamagecalc - 0x6F</summary>

```
furycutterdamagecalc
- 
```
</details>
<details>
<summary>tryattract - 0x70</summary>

```
tryattract
- 
```
</details>
<details>
<summary>trysafeguard - 0x71</summary>

```
trysafeguard
- 
```
</details>
<details>
<summary>trypresent - 0x72</summary>

```
trypresent
- 
```
</details>
<details>
<summary>magnitudedamagecalc - 0x73</summary>

```
magnitudedamagecalc
- 
```
</details>
<details>
<summary>tryswitchinmon - 0x74</summary>

```
tryswitchinmon
- 
```
</details>
<details>
<summary>dorapidspineffect - 0x75</summary>

```
dorapidspineffect
- 
```
</details>
<details>
<summary>changehprecoverybasedonweather - 0x76</summary>

```
changehprecoverybasedonweather
- 
```
</details>
<details>
<summary>hiddenpowerdamagecalc - 0x77</summary>

```
hiddenpowerdamagecalc
- 
```
</details>
<details>
<summary>dopsychup - 0x78</summary>

```
dopsychup
- 
```
</details>
<details>
<summary>tryfuturesight - 0x79</summary>

```
tryfuturesight
- 
```
</details>
<details>
<summary>checkhitrate - 0x7A</summary>

```
checkhitrate
- 
```
</details>
<details>
<summary>tryteleport - 0x7B</summary>

```
tryteleport
- 
```
</details>
<details>
<summary>beatupdamagecalc - 0x7C</summary>

```
beatupdamagecalc
- 
```
</details>
<details>
<summary>dofollowme - 0x7D</summary>

```
dofollowme
- 
```
</details>
<details>
<summary>tryhelpinghand - 0x7E</summary>

```
tryhelpinghand
- 
```
</details>
<details>
<summary>trytrick - 0x7F</summary>

```
trytrick
- 
```
</details>
<details>
<summary>trywish - 0x80</summary>

```
trywish
- 
```
</details>
<details>
<summary>tryassist - 0x81</summary>

```
tryassist
- 
```
</details>
<details>
<summary>trymagiccoat - 0x82</summary>

```
trymagiccoat
- 
```
</details>
<details>
<summary>trymagiccoat2 - 0x83</summary>

```
trymagiccoat2
- 
```
</details>
<details>
<summary>dorevenge - 0x84</summary>

```
dorevenge
- 
```
</details>
<details>
<summary>trybreakscreens - 0x85</summary>

```
trybreakscreens
- 
```
</details>
<details>
<summary>tryyawn - 0x86</summary>

```
tryyawn
- 
```
</details>
<details>
<summary>tryknockitemoff - 0x87</summary>

```
tryknockitemoff
- 
```
</details>
<details>
<summary>eruptiondamagecalc - 0x88</summary>

```
eruptiondamagecalc
- 
```
</details>
<details>
<summary>tryimprison - 0x89</summary>

```
tryimprison
- 
```
</details>
<details>
<summary>trygrudge - 0x8A</summary>

```
trygrudge
- 
```
</details>
<details>
<summary>trysnatch - 0x8B</summary>

```
trysnatch
- 
```
</details>
<details>
<summary>lowkickdamagecalc - 0x8C</summary>

```
lowkickdamagecalc
- 
```
</details>
<details>
<summary>weatherballdamagecalc - 0x8D</summary>

```
weatherballdamagecalc
- 
```
</details>
<details>
<summary>trypursuit - 0x8E</summary>

```
trypursuit
- 
```
</details>
<details>
<summary>typecheck - 0x8F</summary>

```
typecheck
- 
```
</details>
<details>
<summary>checkoneturnflag - 0x90</summary>

```
checkoneturnflag
- 
```
</details>
<details>
<summary>setoneturnflag - 0x91</summary>

```
setoneturnflag
- 
```
</details>
<details>
<summary>gyroballdamagecalc - 0x92</summary>

```
gyroballdamagecalc
- 
```
</details>
<details>
<summary>metalburstdamagecalc - 0x93</summary>

```
metalburstdamagecalc
- 
```
</details>
<details>
<summary>paybackdamagecalc - 0x94</summary>

```
paybackdamagecalc
- 
```
</details>
<details>
<summary>trumpcarddamagecalc - 0x95</summary>

```
trumpcarddamagecalc
- 
```
</details>
<details>
<summary>wringoutdamagecalc - 0x96</summary>

```
wringoutdamagecalc
- 
```
</details>
<details>
<summary>trymefirst - 0x97</summary>

```
trymefirst
- 
```
</details>
<details>
<summary>trycopycat - 0x98</summary>

```
trycopycat
- 
```
</details>
<details>
<summary>punishmentdamagecalc - 0x99</summary>

```
punishmentdamagecalc
- 
```
</details>
<details>
<summary>trysuckerpunch - 0x9A</summary>

```
trysuckerpunch
- 
```
</details>
<details>
<summary>checkbattlercondition - 0x9B</summary>

```
checkbattlercondition
- 
```
</details>
<details>
<summary>tryfeint - 0x9C</summary>

```
tryfeint
- 
```
</details>
<details>
<summary>trypsychoshift - 0x9D</summary>

```
trypsychoshift
- 
```
</details>
<details>
<summary>trylastresort - 0x9E</summary>

```
trylastresort
- 
```
</details>
<details>
<summary>trytoxicspikes - 0x9F</summary>

```
trytoxicspikes
- 
```
</details>
<details>
<summary>checktoxicspikes - 0xA0</summary>

```
checktoxicspikes
- 
```
</details>
<details>
<summary>moldbreakerabilitycheck - 0xA1</summary>

```
moldbreakerabilitycheck
- 
```
</details>
<details>
<summary>checkbattlersequal - 0xA2</summary>

```
checkbattlersequal
- 
```
</details>
<details>
<summary>trypickup - 0xA3</summary>

```
trypickup
- 
```
</details>
<details>
<summary>dotrickroom - 0xA4</summary>

```
dotrickroom
- 
```
</details>
<details>
<summary>checkmovefinished - 0xA5</summary>

```
checkmovefinished
- 
```
</details>
<details>
<summary>checkitemeffect - 0xA6</summary>

```
checkitemeffect checker, battler, effect, address
conditional flow command that is based on item effect
- when "checker" is true, checkitemeffect jumps to "address" if battler doesn't have the item with effect "effect"
  - if false, checkitemeffect jumps to "address" if the battler does have the item with effect "effect"
- battler is the battler to check against
- effect is the held item effect to compare to
- address is the address to jump to
```
</details>
<details>
<summary>getitemeffect - 0xA7</summary>

```
getitemeffect battler, var
grabs the item held effect from "battler" and puts it in "var"
- battler is the battler to grab the item held effect from
- var is the var to store the item held effect in
```
</details>
<details>
<summary>getitempower - 0xA8</summary>

```
getitempower battler, variable
grabs the item power field from the item data narc and puts it in variable
- battler is the battler that has the item to grab the item power from
- variable is the variable to store the item power in
```
</details>
<details>
<summary>trycamouflage - 0xA9</summary>

```
trycamouflage
- 
```
</details>
<details>
<summary>donaturepower - 0xAA</summary>

```
donaturepower
- 
```
</details>
<details>
<summary>dosecretpower - 0xAB</summary>

```
dosecretpower
- 
```
</details>
<details>
<summary>trynaturalgift - 0xAC</summary>

```
trynaturalgift
- 
```
</details>
<details>
<summary>trypluck - 0xAD</summary>

```
trypluck
- 
```
</details>
<details>
<summary>tryfling - 0xAE</summary>

```
tryfling
- 
```
</details>
<details>
<summary>yesnobox - 0xAF</summary>

```
yesnobox
- 
```
</details>
<details>
<summary>yesnowait - 0xB0</summary>

```
yesnowait
- 
```
</details>
<details>
<summary>monlist - 0xB1</summary>

```
monlist
- 
```
</details>
<details>
<summary>monlistwait - 0xB2</summary>

```
monlistwait
- 
```
</details>
<details>
<summary>setbattleresult - 0xB3</summary>

```
setbattleresult
- 
```
</details>
<details>
<summary>checkstealthrock - 0xB4</summary>

```
checkstealthrock
- 
```
</details>
<details>
<summary>checkeffectactivation - 0xB5</summary>

```
checkeffectactivation
- 
```
</details>
<details>
<summary>checkchatteractivation - 0xB6</summary>

```
checkchatteractivation
- 
```
</details>
<details>
<summary>getmoveparameter - 0xB7</summary>

```
getmoveparameter field
grabs parameter "field" from the move data structure and stores in VAR_09
- "field" is the data to grab from the move, enumerations below
```
```c
getmoveparameter fields:

#define MOVE_DATA_BATTLE_EFFECT 0
#define MOVE_DATA_SPLIT 1
#define MOVE_DATA_BASE_POWER 2
#define MOVE_DATA_TYPE 3
#define MOVE_DATA_ACCURACY 4
#define MOVE_DATA_PP 5
#define MOVE_DATA_EFFECT_CHANCE 6
#define MOVE_DATA_TARGET 7
#define MOVE_DATA_PRIORITY 8
#define MOVE_DATA_FLAGS 9
#define MOVE_DATA_APPEAL 10
#define MOVE_DATA_CONTEST_TYPE 11
```
</details>
<details>
<summary>mosaic - 0xB8</summary>

```
mosaic
- 
```
</details>
<details>
<summary>changeform - 0xB9</summary>

```
changeform
- 
```
</details>
<details>
<summary>changebackground - 0xBA</summary>

```
changebackground
- 
```
</details>
<details>
<summary>recoverstatus - 0xBB</summary>

```
recoverstatus
- 
```
</details>
<details>
<summary>tryescape - 0xBC</summary>

```
tryescape
- 
```
</details>
<details>
<summary>initstartballguage - 0xBD</summary>

```
initstartballguage
- 
```
</details>
<details>
<summary>deletestartballguage - 0xBE</summary>

```
deletestartballguage
- 
```
</details>
<details>
<summary>initballguage - 0xBF</summary>

```
initballguage
- 
```
</details>
<details>
<summary>deleteballguage - 0xC0</summary>

```
deleteballguage
- 
```
</details>
<details>
<summary>loadballgfx - 0xC1</summary>

```
loadballgfx
- 
```
</details>
<details>
<summary>deleteballgfx - 0xC2</summary>

```
deleteballgfx
- 
```
</details>
<details>
<summary>incrementgamestat - 0xC3</summary>

```
incrementgamestat
- 
```
</details>
<details>
<summary>cmd_C4 - 0xC4</summary>

```
cmd_C4
- 
```
</details>
<details>
<summary>checkifcurrentmovehits - 0xC5</summary>

```
checkifcurrentmovehits
- 
```
</details>
<details>
<summary>cmd_C6 - 0xC6</summary>

```
cmd_C6
- 
```
</details>
<details>
<summary>cmd_C7 - 0xC7</summary>

```
cmd_C7
- 
```
</details>
<details>
<summary>checkwipeout - 0xC8</summary>

```
checkwipeout
- 
```
</details>
<details>
<summary>tryacupressure - 0xC9</summary>

```
tryacupressure
- 
```
</details>
<details>
<summary>removeitem - 0xCA</summary>

```
removeitem
- 
```
</details>
<details>
<summary>tryrecycle - 0xCB</summary>

```
tryrecycle
- 
```
</details>
<details>
<summary>itemeffectcheckonhit - 0xCC</summary>

```
itemeffectcheckonhit
- 
```
</details>
<details>
<summary>battleresultmessage - 0xCD</summary>

```
battleresultmessage
- 
```
</details>
<details>
<summary>runawaymessage - 0xCE</summary>

```
runawaymessage
- 
```
</details>
<details>
<summary>giveupmessage - 0xCF</summary>

```
giveupmessage
- 
```
</details>
<details>
<summary>cmd_D0_checkhpsomething - 0xD0</summary>

```
cmd_D0_checkhpsomething
- 
```
</details>
<details>
<summary>trynaturalcure - 0xD1</summary>

```
trynaturalcure
- 
```
</details>
<details>
<summary>checknostatus - 0xD2</summary>

```
checknostatus
- 
```
</details>
<details>
<summary>checkcloudnine - 0xD3</summary>

```
checkcloudnine
- 
```
</details>
<details>
<summary>cmd_D4 - 0xD4</summary>

```
cmd_D4 battler
not sure what this command does.
- battler is the battler to affect
```
</details>
<details>
<summary>checkwhenitemmakesmovehit - 0xD5</summary>

```
checkwhenitemmakesmovehit
- 
```
</details>
<details>
<summary>cmd_D6 - 0xD6</summary>

```
cmd_D6
- 
```
</details>
<details>
<summary>playmovesoundeffect - 0xD7</summary>

```
playmovesoundeffect battler
plays the move's damaging sound effect
- battler is the basis of the sound pan
```
</details>
<details>
<summary>playsong - 0xD8</summary>

```
playsong
- 
```
</details>
<details>
<summary>checkifsafariencounterdone - 0xD9</summary>

```
checkifsafariencounterdone
- 
```
</details>
<details>
<summary>waitwithoutbuttonpress - 0xDA</summary>

```
waitwithoutbuttonpress
- 
```
</details>
<details>
<summary>checkmovetypematches - 0xDB</summary>

```
checkmovetypematches
- 
```
</details>
<details>
<summary>getdatafrompersonalnarc - 0xDC</summary>

```
getdatafrompersonalnarc
- 
```
</details>
<details>
<summary>refreshmondata - 0xDD</summary>

```
refreshmondata
- 
```
</details>
<details>
<summary>cmd_DE - 0xDE</summary>

```
cmd_DE
- 
```
</details>
<details>
<summary>cmd_DF - 0xDF</summary>

```
cmd_DF
- 
```
</details>
<details>
<summary>endscript - 0xE0</summary>

```
endscript
ends the script and hands exection back to the overall battle engine if nothing else is queued
```
</details>
</details>


# Animation Script Command Reference
