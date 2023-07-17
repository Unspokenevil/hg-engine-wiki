# Move Animation Scripting System Documentation
Moves in Heart Gold have 2 of their own bytecode scripting languages that correspond with certain pieces of code that would have to be repeated constantly across moves.  This is similar in function to the various scripting systems from Gen 3 as well.

Move animations have their own, just like battle scripts, overworld scripts, 

Move animation scripts are often called within battle scripts to show something happening, be it the Pokémon sliding around or small particles coming up to show that the Pokémon is honing its claws.  In addition to this, every single move has its own animation that is played when the move is used.  There are two classes of these, and admittedly I'm not too sure when the latter is used, but they are dumped in this repo for completeness and future-proofing:
- ``move_anim``
- ``move_sub_anim``

## ``move_anim`` - a010
Every move has its own animation in this file, and they are performed by these scripts using the .SPA particle files from a029.  We currently just steal them from the Gen 5 games and see if they work in Heart Gold well enough.  Most ``move_anim`` scripts just load a particle file (the .SPA handles most of the movement of said particle), place particles from the file, and play a sound.

## ``move_sub_anim`` - a061
I believe this is used intermittently for when an animation needs to happen and there isn't a move slot that has the animation already defined.  I haven't looked too much into this as of yet, but things like Thief's animation for stealing an item (as opposed to the attack animation) and Pokémon using an item to heal are probably in this narc.

This documentation is inherently aiming to be less thorough than the similar [[Move Scripting Systems Documentation|Move Scripting Systems Documentation]].  Less is known about move animation scripts, but the bulk of the animation is put on the shoulders of the particle (SPA) files.  These will be shortly covered at the end of the tutorial.  Moves often just place a SPA file, maybe blend the Pokémon with a color, shake it or something similar, and play a sound.

The average move script will look something like this (see [Quiver Dance's animation](https://github.com/BluRosie/hg-engine/blob/main/armips/move/move_anim/486.s))
```
a010_486:
    loadparticlefromspa 0, 499
    waitparticle

    callfunction 34, 6, 2, 0, 1, red | green << 5 | blue << 10, 10, 15, "NaN", "NaN", "NaN", "NaN"
    callfunction 36, 5, 3, 0, 1, 10, 264, "NaN", "NaN", "NaN", "NaN", "NaN" // shake target mon
    addparticle 0, 2, 3
    addparticle 0, 1, 3
    addparticle 0, 0, 3
    playsepan 1911, -117
    wait 2
    playsepan 1911, -117
    wait 2
    playsepan 1911, -117
    wait 2
    playsepan 1911, -117
    wait 2
    playsepan 1911, -117
    wait 2
    playsepan 1911, -117
    wait 2
    playsepan 1911, -117
    waitparticle

    unloadparticle 0
    waitstate
    end
```

