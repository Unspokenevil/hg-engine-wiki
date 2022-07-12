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

# Examples
I personally learn primarily by example.  A lot of what the beginning stages of battle script editing were in Gens 2 and 3 was mashing hexadecimal together and seeing what all worked--before there were any competent editors (and even for a while after there were, as people preferred pasting assembled hex into the rom directly for some reason).  A similar approach can be taken for this, identifying certain blocks that look like, when isolated, can be ported and placed wherever.

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

``critcalc`` calculates the critical multiplier (if any).  This script command takes all things into affect, like Razor Claw and Sniper.  If the move will be a critical hit, it makes sure to queue up a script stating that it is such.

``damagecalc`` is the basic damage calculator.  It takes into account all of the abilities, held effects, etc.  Damage rolls are done later upon actually dealing the damage--the damage calculator returns the highest roll 100% damage.

``endscript`` just ends the script and hands exection back to the overall battle engine.

From there, we can look at other simple cases:  moves that lower the target's stats.  We can look at two examples of this to determine the differences, and this also introduces another core concept in move scripts with the connection between ``battle_eff_seq`` and ``battle_sub_seq``.

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
changevar, operator, var, value
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

The ``23`` correlates to a subscript through the table ``move_effect_to_subscripts`` in ``src/moves.c``:
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
Now subscript 12 (``armips/move/battle_sub_seq/012.s``) is responsible for every single stat adjustment in battles, just fed slightly different parameters.  The modularity makes it difficult to understand, and we will cover it more comprehensively later.

The big takeaway currently is that Growl, when treated similarly to Tail Whip:
```
a030_018:
    changevar VAR_OP_SET, VAR_ADD_STATUS1, ADD_STATUS_DEFENDER | 22 // 0x16 = 22 decimal
    endscript
```
That 22 gives the ``attack -1`` entry from ``move_effect_to_subscripts``, right above Tail Whip's ``defense -1``.

# Battle Script Command Reference


# Animation Script Command Reference
