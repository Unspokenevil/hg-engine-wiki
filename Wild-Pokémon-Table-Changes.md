## Wild Pokémon Table Changes

The wild Pokémon encounter table didn't receive many overhauls, just adding form support in the form of turning the numerous ``species`` fields into ``(form << 10 | species)``.  Here is the structure of the encounter tables, which is unchanged from vanilla:

```c
typedef struct
{
    u8 minLevel;
    u8 maxLevel;
    u16 species;
} __attribute__((packed)) StandardEncounter;

typedef	struct
{
    u8 walkingEncounterRate;
    u8 surfEncounterRate;
    u8 rockSmashEncounterRate;
    u8 oldRodRate;

    u8 goodRodRate;
    u8 superRodRate;
    u16 padding_x6; // get word-aligned for the next array

    u8 walkingLevels[12];
    u16 morningMons[12];
    u16 dayMons[12];
    u16 nightMons[12];

    u16 hoennMons[2];
    u16 sinnohMons[2];

    StandardEncounter surfEncounters[5];
    StandardEncounter rockSmashEncounters[2];
    StandardEncounter oldRodEncounters[5];
    StandardEncounter goodRodEncounters[5];
    StandardEncounter superRodEncounters[5];

    u16 swarmMons[4];    
} __attribute__((packed)) EncounterData;
```

Every ``u16`` field in this structure (including the ``StandardEncounter`` substructure) represents a species of Pokémon.  These have all been changed to be interpreted as ``(form << 10 | species)``.  This allows for up to 1024 indices of Pokémon, and mirrors what was implemented in vanilla for trainer Pokémon species.

This is currently not supported to be edited by any existing tool, nor is it planned to be.  I may dump it into this repo and build it similar to the trainers or even expand on these structures by making the unused ``padding_x6`` field into a set of flags that the game can use to expand on the structure for more customizability concerning wild Pokémon.