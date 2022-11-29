## Dex Flag Expansion Documentation

I originally did this completely in assembly.  This makes it a little cumbersome to fix if for whatever reason things are broken.

Currently failsafes are in place to make sure this doesn't happen.  While the actual size of the new dex structure is 0x440 (compared to an old 0x340), this has been expanded up to 0x4C0 to ensure that nothing is getting overwritten accidentally elsewhere.  There seems to be a bitfield that keeps track of which language each mon has been obtained in, and this implementation does not bother to expand that, instead opting to expand the structure so it does not overflow into other areas.  If you run into errors related to the Daycare mons being complete gibberish, then this is the likely culprit.

Currently, the game supports up to double the original amount of dex flags (up to 1024). This is very much a hard limiter, and many structures rely on Pokémon not having indices of more than 1024.  The dex structure as we care about it looked something like this:

```c
struct DexSaveData
{
    u32 magic; // 0xBEEFCAFE
    u8 CaughtMons[0x40]; // 1 bit per mon, 0x40 * 8 = 0x200 = 512
    u8 SeenMons[0x40];
    u8 SeenMaleMon[0x40];
    u8 SeenFemaleMon[0x40];
    u8 otherthings[0x23C]; // the dex has other structures that result in an original total size of 0x340
}
```

With 512 bits at our command, we can have up to 512 Pokémon originally.  This is obviously an issue: with over 900 Pokémon implemented in this expansion, we need to reserve more space for this.

As previously mentioned, many other structures happen to be limited to 1024 Pokémon.  One that instantly comes to mind is the Trainer Mon Data structure, which takes the 6 most significant bits of a u16 and uses those directly as the form for the Pokémon that is added to the trainer party.  I do not plan on supporting more than 1024 Pokémon in any edition of hg-engine that I work on.

That being said, the structure is reorganized as such:

```c
struct DexSaveData
{
    u32 magic; // 0xBEEFCAFE
    u8 CaughtMons[0x80]; // 1 bit per mon, 0x40 * 8 = 0x200 = 512
    u8 SeenMons[0x80];
    u8 otherthings[0x23C]; // the dex has other structures that result in an original total size of 0x340
    u8 SeenMaleMon[0x80];
    u8 SeenFemaleMon[0x80]; // new size: 0x440
}
```

This allows for up to 1024 Pokémon to be seen.  In order to prevent the language bytes from overwriting the seen gender flags, The structure has expanded on this further:

```c
struct DexSaveData
{
    u32 magic; // 0xBEEFCAFE
    u8 CaughtMons[0x80]; // 1 bit per mon, 0x40 * 8 = 0x200 = 512
    u8 SeenMons[0x80];
    u8 otherthings[0x23C]; // the dex has other structures that result in an original total size of 0x340
    u8 spaceforlanguageoverflow[0x80]
    u8 SeenMaleMon[0x80];
    u8 SeenFemaleMon[0x80]; // new size: 0x4C0
}
```

This is currently the only save structure that is expanded.  However, this completely breaks old saves and PKHeX compatibility, as saves are dynamically constructed and read based on functions that get the sizes of fields in the save partition.  Methods used to give Pokémon are not affected similarly, however, as well as Action Replay codes that don't touch the PokéDex (i.e. giving yourself a number of items still works).