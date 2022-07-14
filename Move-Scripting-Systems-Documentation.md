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
```
critcalc
calculates the critical multiplier (set to 1 in the case that there isn't one)

damagecalc
the basic damage calculator.

endscript
ends the script and hands exection back to the overall battle engine
```
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

checkitemeffect checker, battler, effect, address
conditional flow command that is based on item effect
- when "checker" is true, checkitemeffect jumps to "address" if battler doesn't have the item with effect "effect"
  - if false, checkitemeffect jumps to "address" if the battler does have the item with effect "effect"
- battler is the battler to check against
- effect is the held item effect to compare to
- address is the address to jump to

getitempower battler, variable
grabs the item power field from the item data narc and puts it in variable
- battler is the battler to grab the item power from
- variable is the variable to store the item power in

changevar2 operator, destvar, srcvar
changevar except the constant is now a variable
- operator is the same as changevar
- destvar is a variable that may or may not hold a value already that will be changed
- srcvar is a variable that operator uses to complete its operation
```
DSPRE introduces a separation between what it calls "scripts" and "functions."  Those do not exist here at all.  Conditional execution is handled much similarly as in raw assembly:  there are a number of conditional branches that go to other areas, and identicality to the original script files was a goal of this system to prevent complications.

Now to break down what exactly is happening in the script.

The ``if`` at the beginning checks to make sure that the field effect variable doesn't already have the rain bits set.  If the rain bits are already set, then the script jumps to ``_0094`` where ``VAR_10``, which stores move results among other things, has its "move failed" bit set.  If the move does not fail, the script continues at the ``preparemessage`` immediately following.  The ``preparemessage`` takes its ``id`` as 0x31F (799), which according to ``data/text/197.txt`` is the line ``It started to rain!``  As there are no string buffers in this, there is nothing to buffer, so it takes a ``tag`` of ``0x0`` and just prepares that message to print.  The ``changevar`` that clears the mask ``0x80FF`` from the field effect variable is ensuring that all other weathers that are active at the time of using Rain Dance are dissipated.  The script then sets the rain bit, sets the turns to 5, and queues up a subscript.  The Damp Rock's item effect is checked for, and if the attacker doesn't have a Damp Rock (as the ``checker`` is ``0x1``), then the script ends and jumps to ``_0090``.  Otherwise, the ``VAR_WEATHER_TURNS`` has the item power field from the ``getitempower`` script command added to it.  VAR_09 is typically used as a (very) temporary variable--random number calculations are stored there on top of other calculation interim steps that are necessary, both in battle script usage as well as the battle engine.

The script queues up the ``0x5D (93)`` status effect in the var and says that the target is the whole field (``0x200000000``).  Looking at ``move_effect_to_subscripts`` again...
```c
u32 move_effect_to_subscripts[] =
{
// ...
    [ 92] = 101,
    [ 93] = 103, // subscript 103 is the queued subscript
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
goes to subscript "num"
- num is the index of the subscript to jump to
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
The queued subscript is solely responsible for printing the message prepared in the ``battle_eff_seq`` script!  How comical.

There are a number of "optimizations" like this that are done for whatever reason.  In dissecting how even simple attacks like Rain Dance work, sometimes you end up down massive trails of redundancy that make things challenging to dissect sometimes.  It is alright to be confused!  But it is especially important that the control flow commands (if, goto, gotosubscript, etc.) are well understood.

# Battle Script Command Reference
<details>
<summary>Battle Script Command Reference Dropdown</summary>
<br>
<details>
<summary>startencounter - 0x00</summary>
<br>
<pre>
startencounter
- initializes battle information
</pre>
</details>
<details>
<summary>pokemonencounter - 0x01</summary>
<br>
<pre>
pokemonencounter
- 
</pre>
</details>
<details>
<summary>pokemonslidein - 0x02</summary>
<br>
<pre>
pokemonslidein
- 
</pre>
</details>
<details>
<summary>pokemonappear - 0x03</summary>
<br>
<pre>
pokemonappear
- 
</pre>
</details>
<details>
<summary>returnpokemon - 0x04</summary>
<br>
<pre>
returnpokemon
- 
</pre>
</details>
<details>
<summary>deletepokemon - 0x05</summary>
<br>
<pre>
deletepokemon
- 
</pre>
</details>
<details>
<summary>starttrainerencounter - 0x06</summary>
<br>
<pre>
starttrainerencounter
- 
</pre>
</details>
<details>
<summary>throwpokeball - 0x07</summary>
<br>
<pre>
throwpokeball
- 
</pre>
</details>
<details>
<summary>preparetrainerslide - 0x08</summary>
<br>
<pre>
preparetrainerslide
- 
</pre>
</details>
<details>
<summary>trainerslidein - 0x09</summary>
<br>
<pre>
trainerslidein
- 
</pre>
</details>
<details>
<summary>backgroundslidein - 0x0A</summary>
<br>
<pre>
backgroundslidein
- 
</pre>
</details>
<details>
<summary>hpgaugeslidein - 0x0B</summary>
<br>
<pre>
hpgaugeslidein
- 
</pre>
</details>
<details>
<summary>hpgaugeslidewait - 0x0C</summary>
<br>
<pre>
hpgaugeslidewait
- 
</pre>
</details>
<details>
<summary>preparehpgaugeslide - 0x0D</summary>
<br>
<pre>
preparehpgaugeslide
- 
</pre>
</details>
<details>
<summary>waitmessage - 0x0E</summary>
<br>
<pre>
waitmessage
- 
</pre>
</details>
<details>
<summary>damagecalc - 0x0F</summary>
<br>
<pre>
damagecalc
- 
</pre>
</details>
<details>
<summary>damagecalc2 - 0x10</summary>
<br>
<pre>
damagecalc2
- 
</pre>
</details>
<details>
<summary>printattackmessage - 0x11</summary>
<br>
<pre>
printattackmessage
- 
</pre>
</details>
<details>
<summary>printmessage - 0x12</summary>
<br>
<pre>
printmessage
- 
</pre>
</details>
<details>
<summary>printmessage2 - 0x13</summary>
<br>
<pre>
printmessage2
- 
</pre>
</details>
<details>
<summary>printpreparedmessage - 0x14</summary>
<br>
<pre>
printpreparedmessage
- 
</pre>
</details>
<details>
<summary>preparemessage - 0x15</summary>
<br>
<pre>
preparemessage
- 
</pre>
</details>
<details>
<summary>printmessagepassbattler - 0x16</summary>
<br>
<pre>
printmessagepassbattler
- 
</pre>
</details>
<details>
<summary>seteffectprimary  - 0x17 (playanimation)</summary>
<br>
<pre>
seteffectprimary
- 
</pre>
</details>
<details>
<summary>seteffectsecondary  - 0x18 (playanimation2)</summary>
<br>
<pre>
seteffectsecondary
- 
</pre>
</details>
<details>
<summary>monflicker - 0x19</summary>
<br>
<pre>
monflicker
- 
</pre>
</details>
<details>
<summary>datahpupdate - 0x1A</summary>
<br>
<pre>
datahpupdate
- 
</pre>
</details>
<details>
<summary>healthbarupdate - 0x1B</summary>
<br>
<pre>
healthbarupdate
- 
</pre>
</details>
<details>
<summary>tryfaintmon - 0x1C</summary>
<br>
<pre>
tryfaintmon
- 
</pre>
</details>
<details>
<summary>dofaintanimation - 0x1D</summary>
<br>
<pre>
dofaintanimation
- 
</pre>
</details>
<details>
<summary>wait - 0x1E</summary>
<br>
<pre>
wait
- 
</pre>
</details>
<details>
<summary>playse - 0x1F</summary>
<br>
<pre>
playse
- 
</pre>
</details>
<details>
<summary>if - 0x20</summary>
<br>
<pre>
if
- 
</pre>
</details>
<details>
<summary>ifmonstat - 0x21</summary>
<br>
<pre>
ifmonstat
- 
</pre>
</details>
<details>
<summary>fadeout - 0x22</summary>
<br>
<pre>
fadeout
- 
</pre>
</details>
<details>
<summary>jumptosubseq - 0x23</summary>
<br>
<pre>
jumptosubseq
- 
</pre>
</details>
<details>
<summary>jumptocurmoveeffectscript - 0x24</summary>
<br>
<pre>
jumptocurmoveeffectscript
- 
</pre>
</details>
<details>
<summary>jumptoeffectscript - 0x25</summary>
<br>
<pre>
jumptoeffectscript
- 
</pre>
</details>
<details>
<summary>critcalc - 0x26</summary>
<br>
<pre>
critcalc
- 
</pre>
</details>
<details>
<summary>shouldgetexp - 0x27</summary>
<br>
<pre>
shouldgetexp
- 
</pre>
</details>
<details>
<summary>initexpget - 0x28</summary>
<br>
<pre>
initexpget
- 
</pre>
</details>
<details>
<summary>getexp - 0x29</summary>
<br>
<pre>
getexp
- 
</pre>
</details>
<details>
<summary>getexploop - 0x2A</summary>
<br>
<pre>
getexploop
- 
</pre>
</details>
<details>
<summary>showmonlist - 0x2B</summary>
<br>
<pre>
showmonlist
- 
</pre>
</details>
<details>
<summary>waitformonselection - 0x2C</summary>
<br>
<pre>
waitformonselection
- 
</pre>
</details>
<details>
<summary>switchindataupdate - 0x2D</summary>
<br>
<pre>
switchindataupdate
- 
</pre>
</details>
<details>
<summary>jumpifcantswitch - 0x2E</summary>
<br>
<pre>
jumpifcantswitch
- 
</pre>
</details>
<details>
<summary>initcapture - 0x2F</summary>
<br>
<pre>
initcapture
- 
</pre>
</details>
<details>
<summary>capturemon - 0x30</summary>
<br>
<pre>
capturemon
- 
</pre>
</details>
<details>
<summary>setmultihit - 0x31</summary>
<br>
<pre>
setmultihit
- 
</pre>
</details>
<details>
<summary>changevar - 0x32</summary>
<br>
<pre>
changevar
- 
</pre>
</details>
<details>
<summary>statbuffchange - 0x33</summary>
<br>
<pre>
statbuffchange
- 
</pre>
</details>
<details>
<summary>changevartomonvalue - 0x34</summary>
<br>
<pre>
changevartomonvalue
- 
</pre>
</details>
<details>
<summary>clearstatus2 - 0x35</summary>
<br>
<pre>
clearstatus2
- 
</pre>
</details>
<details>
<summary>togglevanish - 0x36</summary>
<br>
<pre>
togglevanish
- 
</pre>
</details>
<details>
<summary>abilitycheck - 0x37</summary>
<br>
<pre>
abilitycheck
- 
</pre>
</details>
<details>
<summary>random - 0x38</summary>
<br>
<pre>
random
- 
</pre>
</details>
<details>
<summary>changevar2 - 0x39</summary>
<br>
<pre>
changevar2
- 
</pre>
</details>
<details>
<summary>changevartomonvalue2 - 0x3A</summary>
<br>
<pre>
changevartomonvalue2
- 
</pre>
</details>
<details>
<summary>goto - 0x3B</summary>
<br>
<pre>
goto
- 
</pre>
</details>
<details>
<summary>gotosubscript - 0x3C</summary>
<br>
<pre>
gotosubscript
- 
</pre>
</details>
<details>
<summary>gotosubscript2 - 0x3D</summary>
<br>
<pre>
gotosubscript2
- 
</pre>
</details>
<details>
<summary>checkifchatot - 0x3E</summary>
<br>
<pre>
checkifchatot
- 
</pre>
</details>
<details>
<summary>sethaze - 0x3F</summary>
<br>
<pre>
sethaze
- 
</pre>
</details>
<details>
<summary>setsomeflag - 0x40</summary>
<br>
<pre>
setsomeflag
- 
</pre>
</details>
<details>
<summary>clearsomeflag - 0x41</summary>
<br>
<pre>
clearsomeflag
- 
</pre>
</details>
<details>
<summary>setstatusicon - 0x42</summary>
<br>
<pre>
setstatusicon
- 
</pre>
</details>
<details>
<summary>trainermessage - 0x43</summary>
<br>
<pre>
trainermessage
- 
</pre>
</details>
<details>
<summary>calcmoney - 0x44</summary>
<br>
<pre>
calcmoney
- 
</pre>
</details>
<details>
<summary>setstatus2effect - 0x45</summary>
<br>
<pre>
setstatus2effect
- 
</pre>
</details>
<details>
<summary>setstatus2effect2 - 0x46</summary>
<br>
<pre>
setstatus2effect2
- 
</pre>
</details>
<details>
<summary>setstatus2effect3 - 0x47</summary>
<br>
<pre>
setstatus2effect3
- 
</pre>
</details>
<details>
<summary>returnmessage - 0x48</summary>
<br>
<pre>
returnmessage
- 
</pre>
</details>
<details>
<summary>sentoutmessage - 0x49</summary>
<br>
<pre>
sentoutmessage
- 
</pre>
</details>
<details>
<summary>encountermessage - 0x4A</summary>
<br>
<pre>
encountermessage
- 
</pre>
</details>
<details>
<summary>encountermessage2 - 0x4B</summary>
<br>
<pre>
encountermessage2
- 
</pre>
</details>
<details>
<summary>trainermessage2 - 0x4C</summary>
<br>
<pre>
trainermessage2
- 
</pre>
</details>
<details>
<summary>tryconversion - 0x4D</summary>
<br>
<pre>
tryconversion
- 
</pre>
</details>
<details>
<summary>if2 - 0x4E</summary>
<br>
<pre>
if2
- 
</pre>
</details>
<details>
<summary>ifmonstat2 - 0x4F</summary>
<br>
<pre>
ifmonstat2
- 
</pre>
</details>
<details>
<summary>dopayday - 0x50</summary>
<br>
<pre>
dopayday
- 
</pre>
</details>
<details>
<summary>setlightscreen - 0x51</summary>
<br>
<pre>
setlightscreen
- 
</pre>
</details>
<details>
<summary>setreflect - 0x52</summary>
<br>
<pre>
setreflect
- 
</pre>
</details>
<details>
<summary>setmist - 0x53</summary>
<br>
<pre>
setmist
- 
</pre>
</details>
<details>
<summary>tryonehitko - 0x54</summary>
<br>
<pre>
tryonehitko
- 
</pre>
</details>
<details>
<summary>damagediv - 0x55</summary>
<br>
<pre>
damagediv
- 
</pre>
</details>
<details>
<summary>damagediv2 - 0x56</summary>
<br>
<pre>
damagediv2
- 
</pre>
</details>
<details>
<summary>trymimic - 0x57</summary>
<br>
<pre>
trymimic
- 
</pre>
</details>
<details>
<summary>metronome - 0x58</summary>
<br>
<pre>
metronome
- 
</pre>
</details>
<details>
<summary>trydisable - 0x59</summary>
<br>
<pre>
trydisable
- 
</pre>
</details>
<details>
<summary>counter - 0x5A</summary>
<br>
<pre>
counter
- 
</pre>
</details>
<details>
<summary>mirrorcoat - 0x5B</summary>
<br>
<pre>
mirrorcoat
- 
</pre>
</details>
<details>
<summary>tryencore - 0x5C</summary>
<br>
<pre>
tryencore
- 
</pre>
</details>
<details>
<summary>tryconversion2 - 0x5D</summary>
<br>
<pre>
tryconversion2
- 
</pre>
</details>
<details>
<summary>trysketch - 0x5E</summary>
<br>
<pre>
trysketch
- 
</pre>
</details>
<details>
<summary>trysleeptalk - 0x5F</summary>
<br>
<pre>
trysleeptalk
- 
</pre>
</details>
<details>
<summary>flaildamagecalc - 0x60</summary>
<br>
<pre>
flaildamagecalc
- 
</pre>
</details>
<details>
<summary>tryspite - 0x61</summary>
<br>
<pre>
tryspite
- 
</pre>
</details>
<details>
<summary>healbell - 0x62</summary>
<br>
<pre>
healbell
- 
</pre>
</details>
<details>
<summary>trythief - 0x63</summary>
<br>
<pre>
trythief
- 
</pre>
</details>
<details>
<summary>willprotectwork - 0x64</summary>
<br>
<pre>
willprotectwork
- 
</pre>
</details>
<details>
<summary>trysubstitute - 0x65</summary>
<br>
<pre>
trysubstitute
- 
</pre>
</details>
<details>
<summary>trywhirlwind - 0x66</summary>
<br>
<pre>
trywhirlwind
- 
</pre>
</details>
<details>
<summary>transform - 0x67</summary>
<br>
<pre>
transform
- 
</pre>
</details>
<details>
<summary>tryspikes - 0x68</summary>
<br>
<pre>
tryspikes
- 
</pre>
</details>
<details>
<summary>checkspikes - 0x69</summary>
<br>
<pre>
checkspikes
- 
</pre>
</details>
<details>
<summary>tryperishsong - 0x6A</summary>
<br>
<pre>
tryperishsong
- 
</pre>
</details>
<details>
<summary>orderbattlersbyspeed - 0x6B</summary>
<br>
<pre>
orderbattlersbyspeed
- 
</pre>
</details>
<details>
<summary>exitloopatvalue - 0x6C</summary>
<br>
<pre>
exitloopatvalue
- 
</pre>
</details>
<details>
<summary>weatherdamagecalc - 0x6D</summary>
<br>
<pre>
weatherdamagecalc
- 
</pre>
</details>
<details>
<summary>rolloutdamagecalc - 0x6E</summary>
<br>
<pre>
rolloutdamagecalc
- 
</pre>
</details>
<details>
<summary>furycutterdamagecalc - 0x6F</summary>
<br>
<pre>
furycutterdamagecalc
- 
</pre>
</details>
<details>
<summary>tryattract - 0x70</summary>
<br>
<pre>
tryattract
- 
</pre>
</details>
<details>
<summary>trysafeguard - 0x71</summary>
<br>
<pre>
trysafeguard
- 
</pre>
</details>
<details>
<summary>trypresent - 0x72</summary>
<br>
<pre>
trypresent
- 
</pre>
</details>
<details>
<summary>magnitudedamagecalc - 0x73</summary>
<br>
<pre>
magnitudedamagecalc
- 
</pre>
</details>
<details>
<summary>tryswitchinmon - 0x74</summary>
<br>
<pre>
tryswitchinmon
- 
</pre>
</details>
<details>
<summary>dorapidspineffect - 0x75</summary>
<br>
<pre>
dorapidspineffect
- 
</pre>
</details>
<details>
<summary>changehprecoverybasedonweather - 0x76</summary>
<br>
<pre>
changehprecoverybasedonweather
- 
</pre>
</details>
<details>
<summary>hiddenpowerdamagecalc - 0x77</summary>
<br>
<pre>
hiddenpowerdamagecalc
- 
</pre>
</details>
<details>
<summary>dopsychup - 0x78</summary>
<br>
<pre>
dopsychup
- 
</pre>
</details>
<details>
<summary>tryfuturesight - 0x79</summary>
<br>
<pre>
tryfuturesight
- 
</pre>
</details>
<details>
<summary>checkhitrate - 0x7A</summary>
<br>
<pre>
checkhitrate
- 
</pre>
</details>
<details>
<summary>tryteleport - 0x7B</summary>
<br>
<pre>
tryteleport
- 
</pre>
</details>
<details>
<summary>beatupdamagecalc - 0x7C</summary>
<br>
<pre>
beatupdamagecalc
- 
</pre>
</details>
<details>
<summary>dofollowme - 0x7D</summary>
<br>
<pre>
dofollowme
- 
</pre>
</details>
<details>
<summary>tryhelpinghand - 0x7E</summary>
<br>
<pre>
tryhelpinghand
- 
</pre>
</details>
<details>
<summary>trytrick - 0x7F</summary>
<br>
<pre>
trytrick
- 
</pre>
</details>
<details>
<summary>trywish - 0x80</summary>
<br>
<pre>
trywish
- 
</pre>
</details>
<details>
<summary>tryassist - 0x81</summary>
<br>
<pre>
tryassist
- 
</pre>
</details>
<details>
<summary>trymagiccoat - 0x82</summary>
<br>
<pre>
trymagiccoat
- 
</pre>
</details>
<details>
<summary>trymagiccoat2 - 0x83</summary>
<br>
<pre>
trymagiccoat2
- 
</pre>
</details>
<details>
<summary>dorevenge - 0x84</summary>
<br>
<pre>
dorevenge
- 
</pre>
</details>
<details>
<summary>trybreakscreens - 0x85</summary>
<br>
<pre>
trybreakscreens
- 
</pre>
</details>
<details>
<summary>tryyawn - 0x86</summary>
<br>
<pre>
tryyawn
- 
</pre>
</details>
<details>
<summary>tryknockitemoff - 0x87</summary>
<br>
<pre>
tryknockitemoff
- 
</pre>
</details>
<details>
<summary>eruptiondamagecalc - 0x88</summary>
<br>
<pre>
eruptiondamagecalc
- 
</pre>
</details>
<details>
<summary>tryimprison - 0x89</summary>
<br>
<pre>
tryimprison
- 
</pre>
</details>
<details>
<summary>trygrudge - 0x8A</summary>
<br>
<pre>
trygrudge
- 
</pre>
</details>
<details>
<summary>trysnatch - 0x8B</summary>
<br>
<pre>
trysnatch
- 
</pre>
</details>
<details>
<summary>lowkickdamagecalc - 0x8C</summary>
<br>
<pre>
lowkickdamagecalc
- 
</pre>
</details>
<details>
<summary>weatherballdamagecalc - 0x8D</summary>
<br>
<pre>
weatherballdamagecalc
- 
</pre>
</details>
<details>
<summary>trypursuit - 0x8E</summary>
<br>
<pre>
trypursuit
- 
</pre>
</details>
<details>
<summary>typecheck - 0x8F</summary>
<br>
<pre>
typecheck
- 
</pre>
</details>
<details>
<summary>checkoneturnflag - 0x90</summary>
<br>
<pre>
checkoneturnflag
- 
</pre>
</details>
<details>
<summary>setoneturnflag - 0x91</summary>
<br>
<pre>
setoneturnflag
- 
</pre>
</details>
<details>
<summary>gyroballdamagecalc - 0x92</summary>
<br>
<pre>
gyroballdamagecalc
- 
</pre>
</details>
<details>
<summary>metalburstdamagecalc - 0x93</summary>
<br>
<pre>
metalburstdamagecalc
- 
</pre>
</details>
<details>
<summary>paybackdamagecalc - 0x94</summary>
<br>
<pre>
paybackdamagecalc
- 
</pre>
</details>
<details>
<summary>trumpcarddamagecalc - 0x95</summary>
<br>
<pre>
trumpcarddamagecalc
- 
</pre>
</details>
<details>
<summary>wringoutdamagecalc - 0x96</summary>
<br>
<pre>
wringoutdamagecalc
- 
</pre>
</details>
<details>
<summary>trymefirst - 0x97</summary>
<br>
<pre>
trymefirst
- 
</pre>
</details>
<details>
<summary>trycopycat - 0x98</summary>
<br>
<pre>
trycopycat
- 
</pre>
</details>
<details>
<summary>punishmentdamagecalc - 0x99</summary>
<br>
<pre>
punishmentdamagecalc
- 
</pre>
</details>
<details>
<summary>trysuckerpunch - 0x9A</summary>
<br>
<pre>
trysuckerpunch
- 
</pre>
</details>
<details>
<summary>checkbattlercondition - 0x9B</summary>
<br>
<pre>
checkbattlercondition
- 
</pre>
</details>
<details>
<summary>tryfeint - 0x9C</summary>
<br>
<pre>
tryfeint
- 
</pre>
</details>
<details>
<summary>trypsychoshift - 0x9D</summary>
<br>
<pre>
trypsychoshift
- 
</pre>
</details>
<details>
<summary>trylastresort - 0x9E</summary>
<br>
<pre>
trylastresort
- 
</pre>
</details>
<details>
<summary>trytoxicspikes - 0x9F</summary>
<br>
<pre>
trytoxicspikes
- 
</pre>
</details>
<details>
<summary>checktoxicspikes - 0xA0</summary>
<br>
<pre>
checktoxicspikes
- 
</pre>
</details>
<details>
<summary>moldbreakerabilitycheck - 0xA1</summary>
<br>
<pre>
moldbreakerabilitycheck
- 
</pre>
</details>
<details>
<summary>checkbattlersequal - 0xA2</summary>
<br>
<pre>
checkbattlersequal
- 
</pre>
</details>
<details>
<summary>trypickup - 0xA3</summary>
<br>
<pre>
trypickup
- 
</pre>
</details>
<details>
<summary>dotrickroom - 0xA4</summary>
<br>
<pre>
dotrickroom
- 
</pre>
</details>
<details>
<summary>checkmovefinished - 0xA5</summary>
<br>
<pre>
checkmovefinished
- 
</pre>
</details>
<details>
<summary>checkitemeffect - 0xA6</summary>
<br>
<pre>
checkitemeffect
- 
</pre>
</details>
<details>
<summary>getitemeffect - 0xA7</summary>
<br>
<pre>
getitemeffect
- 
</pre>
</details>
<details>
<summary>getitempower - 0xA8</summary>
<br>
<pre>
getitempower
- 
</pre>
</details>
<details>
<summary>trycamouflage - 0xA9</summary>
<br>
<pre>
trycamouflage
- 
</pre>
</details>
<details>
<summary>donaturepower - 0xAA</summary>
<br>
<pre>
donaturepower
- 
</pre>
</details>
<details>
<summary>dosecretpower - 0xAB</summary>
<br>
<pre>
dosecretpower
- 
</pre>
</details>
<details>
<summary>trynaturalgift - 0xAC</summary>
<br>
<pre>
trynaturalgift
- 
</pre>
</details>
<details>
<summary>trypluck - 0xAD</summary>
<br>
<pre>
trypluck
- 
</pre>
</details>
<details>
<summary>tryfling - 0xAE</summary>
<br>
<pre>
tryfling
- 
</pre>
</details>
<details>
<summary>yesnobox - 0xAF</summary>
<br>
<pre>
yesnobox
- 
</pre>
</details>
<details>
<summary>yesnowait - 0xB0</summary>
<br>
<pre>
yesnowait
- 
</pre>
</details>
<details>
<summary>monlist - 0xB1</summary>
<br>
<pre>
monlist
- 
</pre>
</details>
<details>
<summary>monlistwait - 0xB2</summary>
<br>
<pre>
monlistwait
- 
</pre>
</details>
<details>
<summary>setbattleresult - 0xB3</summary>
<br>
<pre>
setbattleresult
- 
</pre>
</details>
<details>
<summary>checkstealthrock - 0xB4</summary>
<br>
<pre>
checkstealthrock
- 
</pre>
</details>
<details>
<summary>checkeffectactivation - 0xB5</summary>
<br>
<pre>
checkeffectactivation
- 
</pre>
</details>
<details>
<summary>checkchatteractivation - 0xB6</summary>
<br>
<pre>
checkchatteractivation
- 
</pre>
</details>
<details>
<summary>getmoveparameter - 0xB7</summary>
<br>
<pre>
getmoveparameter
- 
</pre>
</details>
<details>
<summary>mosaic - 0xB8</summary>
<br>
<pre>
mosaic
- 
</pre>
</details>
<details>
<summary>changeform - 0xB9</summary>
<br>
<pre>
changeform
- 
</pre>
</details>
<details>
<summary>changebackground - 0xBA</summary>
<br>
<pre>
changebackground
- 
</pre>
</details>
<details>
<summary>recoverstatus - 0xBB</summary>
<br>
<pre>
recoverstatus
- 
</pre>
</details>
<details>
<summary>tryescape - 0xBC</summary>
<br>
<pre>
tryescape
- 
</pre>
</details>
<details>
<summary>initstartballguage - 0xBD</summary>
<br>
<pre>
initstartballguage
- 
</pre>
</details>
<details>
<summary>deletestartballguage - 0xBE</summary>
<br>
<pre>
deletestartballguage
- 
</pre>
</details>
<details>
<summary>initballguage - 0xBF</summary>
<br>
<pre>
initballguage
- 
</pre>
</details>
<details>
<summary>deleteballguage - 0xC0</summary>
<br>
<pre>
deleteballguage
- 
</pre>
</details>
<details>
<summary>loadballgfx - 0xC1</summary>
<br>
<pre>
loadballgfx
- 
</pre>
</details>
<details>
<summary>deleteballgfx - 0xC2</summary>
<br>
<pre>
deleteballgfx
- 
</pre>
</details>
<details>
<summary>incrementgamestat - 0xC3</summary>
<br>
<pre>
incrementgamestat
- 
</pre>
</details>
<details>
<summary>cmd_C4 - 0xC4</summary>
<br>
<pre>
cmd_C4
- 
</pre>
</details>
<details>
<summary>checkifcurrentmovehits - 0xC5</summary>
<br>
<pre>
checkifcurrentmovehits
- 
</pre>
</details>
<details>
<summary>cmd_C6 - 0xC6</summary>
<br>
<pre>
cmd_C6
- 
</pre>
</details>
<details>
<summary>cmd_C7 - 0xC7</summary>
<br>
<pre>
cmd_C7
- 
</pre>
</details>
<details>
<summary>checkwipeout - 0xC8</summary>
<br>
<pre>
checkwipeout
- 
</pre>
</details>
<details>
<summary>tryacupressure - 0xC9</summary>
<br>
<pre>
tryacupressure
- 
</pre>
</details>
<details>
<summary>removeitem - 0xCA</summary>
<br>
<pre>
removeitem
- 
</pre>
</details>
<details>
<summary>tryrecycle - 0xCB</summary>
<br>
<pre>
tryrecycle
- 
</pre>
</details>
<details>
<summary>itemeffectcheckonhit - 0xCC</summary>
<br>
<pre>
itemeffectcheckonhit
- 
</pre>
</details>
<details>
<summary>battleresultmessage - 0xCD</summary>
<br>
<pre>
battleresultmessage
- 
</pre>
</details>
<details>
<summary>runawaymessage - 0xCE</summary>
<br>
<pre>
runawaymessage
- 
</pre>
</details>
<details>
<summary>giveupmessage - 0xCF</summary>
<br>
<pre>
giveupmessage
- 
</pre>
</details>
<details>
<summary>cmd_D0_checkhpsomething - 0xD0</summary>
<br>
<pre>
cmd_D0_checkhpsomething
- 
</pre>
</details>
<details>
<summary>trynaturalcure - 0xD1</summary>
<br>
<pre>
trynaturalcure
- 
</pre>
</details>
<details>
<summary>checknostatus - 0xD2</summary>
<br>
<pre>
checknostatus
- 
</pre>
</details>
<details>
<summary>checkcloudnine - 0xD3</summary>
<br>
<pre>
checkcloudnine
- 
</pre>
</details>
<details>
<summary>cmd_D4 - 0xD4</summary>
<br>
<pre>
cmd_D4
- 
</pre>
</details>
<details>
<summary>checkwhenitemmakesmovehit - 0xD5</summary>
<br>
<pre>
checkwhenitemmakesmovehit
- 
</pre>
</details>
<details>
<summary>cmd_D6 - 0xD6</summary>
<br>
<pre>
cmd_D6
- 
</pre>
</details>
<details>
<summary>playmovesoundeffect - 0xD7</summary>
<br>
<pre>
playmovesoundeffect
- 
</pre>
</details>
<details>
<summary>playsong - 0xD8</summary>
<br>
<pre>
playsong
- 
</pre>
</details>
<details>
<summary>checkifsafariencounterdone - 0xD9</summary>
<br>
<pre>
checkifsafariencounterdone
- 
</pre>
</details>
<details>
<summary>waitwithoutbuttonpress - 0xDA</summary>
<br>
<pre>
waitwithoutbuttonpress
- 
</pre>
</details>
<details>
<summary>checkmovetypematches - 0xDB</summary>
<br>
<pre>
checkmovetypematches
- 
</pre>
</details>
<details>
<summary>getdatafrompersonalnarc - 0xDC</summary>
<br>
<pre>
getdatafrompersonalnarc
- 
</pre>
</details>
<details>
<summary>refreshmondata - 0xDD</summary>
<br>
<pre>
refreshmondata
- 
</pre>
</details>
<details>
<summary>cmd_DE - 0xDE</summary>
<br>
<pre>
cmd_DE
- 
</pre>
</details>
<details>
<summary>cmd_DF - 0xDF</summary>
<br>
<pre>
cmd_DF
- 
</pre>
</details>
<details>
<summary>endscript - 0xE0</summary>
<br>
<pre>
endscript
- 
</pre>
</details>
</details>


# Animation Script Command Reference
