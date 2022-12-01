hg-engine is an engine overhaul for English Pokémon HeartGold with a focus on bringing battles up to par with recent mainline Pokémon games and their mechanics.  

## Table of Contents

- [Vision/Goals](#visiongoals)
- [Current Status](#current-status)
- [Wiki Contents](#wiki-contents)
- [FAQ](#faq)

### Vision/Goals
The primary focus of this is a sort of step-by-step process:

1) Make sure everything (moves, abilities, items, Pokémon, mechanics) is implemented in *some* capacity
2) Make sure everything is implemented *accurately* using in-game tests, Smogon, and the [rh-hideout Emerald battle expansion](https://github.com/rh-hideout/pokeemerald-expansion)
3) Revamp the battle AI to properly use new moves and make a much more difficult trainer battle experience

There is no timeline for this.  I have been working with a small team on this since the beginning of 2022, largely spearheading it myself.  This is a massive undertaking that is only done in our free time in addition to other projects and real life.  We have jobs, friends, family, everything.  There is no expectation that anything be done on any given timeline, and this has been a driving philosophy behind this project so far.  This is to prevent burnout on the project and eventual satisfaction with the work being done.

### Current Status
A rough list of what all comes in this repository and the status:
- Mega Evolution*
- Hidden Abilities* (detailed in [[Hidden Ability Documentation|Hidden Ability Documentation]])
- Level-influence experience system* (as in gens 5, 7, and 8, detailed in [[Experience Overhaul Documentation|Experience Overhaul Documentation]])
- [Brand new Abilities](https://github.com/users/BluRosie/projects/1)
- Brand new forms (current support detailed in [[Form System Documentation|Form System Documentation]]
- Brand new Moves (work in progress, currently [a third of the way through Gen 5](https://github.com/BluRosie/hg-engine/issues/17))
- All Pokémon through gen 8 have data defined.  All of the data is built from files in this repository.  For editing, see [[Editing Pokémon Data|Editing Pokémon Data]].  We are working on getting sprites done for all of the Pokémon in the style of HeartGold.  Reach out to me if you would like to help!
- Completely customizable trainers, including specifying IV's, EV's, shininess, and even specifying specific stat values (see [[Trainer Pokémon Structure Documentation|Trainer Pokémon Structure Documentation]])
- Wild Pokémon can have their form specified (see [[Wild Pokémon Table Changes|Wild Pokémon Table Changes]]).
- 60 FPS battles*
- Always set battles*
- Fairy type*

<sup><sub><b>*</b> - configurable at compile time, can be completely disabled, see [CONFIG.md](https://github.com/BluRosie/hg-engine/blob/main/CONFIG.md)</sub></sup>

### Wiki Contents

- Tutorials
  - [[Editing Pokémon Data|Editing Pokémon Data]]
  - [[Move Scripting Systems Documentation|Move Scripting Systems Documentation]] is a big tutorial with documentation at the end
- Documentation
  - [[Dex Flag Expansion|Dex Flag Expansion Documentation]]
  - [[Experience Overhaul|Experience Overhaul Documentation]]
  - [[Form System|Form System Documentation]]
  - [[Hidden Ability System|Hidden Ability Documentation]]
  - [[Move Data Structure|Move Data Structure Documentation]]
  - [[Move Scripting Systems Documentation|Move Scripting Systems Documentation]]
  - [[Overworld System Documentation|Overworld System Documentation]]
  - [[Trainer Pokémon Structure Documentation|Trainer Pokémon Structure Documentation]]
  - [[Wild Pokémon Table Changes|Wild Pokémon Table Changes]]


### FAQ
<details>
<summary>How do I use this?</summary>
<br>

Read the [README from the repository](https://github.com/BluRosie/hg-engine/blob/main/README.md).  This goes through how to install it from a fresh install of Windows, Debian Linux, and MacOS.  If you are capable of building the [CFRU](https://github.com/Skeli789/Complete-Fire-Red-Upgrade), you are capable of building this.
</details>

<details>
<summary>I encountered a glitch!  How can it be fixed?</summary>
<br>

Please create an issue on the [Github Issues page](https://github.com/BluRosie/hg-engine/issues/).  This should detail reproduction steps and, if you're the best bug reporter ever, include a save file or a save state so that we can easily reproduce it.
</details>

<details>
<summary>What is all configurable in the repository?</summary>
<br>

See [CONFIG.md](https://github.com/BluRosie/hg-engine/blob/main/CONFIG.md).
</details>

<details>
<summary>What can I use this for?</summary>
<br>

You are free to use this for your own hacks as you please with slight restriction.

The only restriction that I have is similar to the CFRU's only restriction:

ROM Hacking is a hobby.  Without my express permission, please do not use this repository or the code within it as part of a project that results in any sort of financial gain that isn't *completely avoidable* by the people that want to play the game in its most recent form.

That is to say, if you want to have Ko-Fi donations for your work or whatever, then feel free to do so.  However, *a reward for donation can not be a version of the game that is not publicly accessible without the donation having occurred*.  I think it is reasonable to ask for truly optional donations that are for your work and not the code from this repository.  A publicly accessible patch should be available for the ROM Hack in question that does not require the donation to unlock anything.

This includes optional donations, streamer-tailored hacks, whatever.  There is a drastic increase lately in streamers essentially paying modding figures to make mods for content and it has debilitated XY, Breath of the Wild, BDSP, and Super Mario Odyssey to be limited to a small team of workers that would rather be exclusive with their findings to ensure that they have a continuous stream of revenue from streamers looking to make content.
</details>

<details>
<summary>Where can I get help for this?</summary>
<br>

Join the [Kingdom of DS Hacking Discord server](https://discord.gg/zAtqJDW2jC).  Read the rules there and make sure you're comfortable in the 11-minute anti-spam probationary period that every new join goes through in that server.  Once you get access, head over to `#hg-feature-expansion` and ask there.  *Please do not reask your question*.  It will be addressed if possible.
</details>
