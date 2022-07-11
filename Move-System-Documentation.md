# Move System Documentation
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
These scripts are largely where the code that actually does effects are run.  So a move will run through its ``battle_move_seq``, which queues up a corresponding ``battle_eff_seq``, which finally queues up a ``battle_sub_seq`` that executes the effect.

## ``move_anim`` - a010
Every move has its own animation, and they are performed by this using the .SPA particle files from a029.  There is no simple editor for this as of yet despite documentation existing, so we currently just steal them from the Gen 5 games and see if they work in heart Gold well enough.

## ``move_sub_anim`` - a061


# Examples


# Battle Script Command Reference


# Animation Script Command Reference
