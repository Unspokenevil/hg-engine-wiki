# Documentation on the Form System
This page serves to document the additional forme changes and to document how the form system works.

Firstly, the form system in HGSS is very all over the place.  Each individual system (icons, learnsets, personal data, etc.) has some sort of appending to the original narc that occurs that contains the form data.  Most functions then have a massive if statement table to determine what the actual index is.  Icons specifically have 50 extra slots after the original 493.

As a result of this, expanding this system starts at 544.  Expanded Pokemon indices go from 544 (Victini) to 955 (Enamorus).  New form data, in all narc files, then resumes at 956.  Forms for Gens 3 and 4 still use their sprite and icon data from the vanilla game for the most part (as that is still handled by the if statements), but new forms are handled by the table ``PokeFormDataTbl`` in ``src/pokemon.c``.  This is an array of ``FormData`` structures and specifies the base species, the form number, whether or not the form should be reverted after battle, and the new species that the form data will come from.

The ``FormData`` structure:
```c
struct FormData
{
    u16 species;
    
    u16 form_no:15;
    u16 need_rev:1;
    
    u16 file;
};
```
This structure is then iterated through to find which file to read from for Pokemon with forms.

Adding new forms is then exactly just like adding new species, just adding the new data to all the folders necessary.  We use ``SPECIES_*`` constants just about everywhere we can--the new form data is no exception, with each form data getting a constant, i.e. ``SPECIES_MELOETTA_PIROUETTE`` or ``SPECIES_KYUREM_BLACK``.

## Supported Forms
<details> <!-- megas -->
<summary>- Mega Evolutions</summary>
<br>

    Venusaur
    Charizard X
    Charizard Y
    Blastoise
    Beedrill
    Pidgeot
    Alakazam
    Slowbro
    Gengar
    Kangaskhan
    Pinsir
    Gyarados
    Aerodactyl
    Mewtwo X
    Mewtwo Y
    Ampharos
    Steelix
    Scizor
    Heracross
    Houndoom
    Tyranitar
    Sceptile
    Blaziken
    Swampert
    Gardevoir
    Sableye
    Mawile
    Aggron
    Medicham
    Manectric
    Sharpedo
    Camerupt
    Altaria
    Banette
    Absol
    Glalie
    Salamence
    Metagross
    Latias
    Latios
    Rayquaza (evolves via using Dragon's Ascent)
    Lopunny
    Garchomp
    Lucario
    Abomasnow
    Gallade
    Audino
    Diancie
</details>
<details> <!-- primal reversions -->
<summary>- Primal Reversions</summary>
<br>

    Kyogre
    Groudon
</details>
<details> <!-- alolans -->
<summary>- Alolan Forms</summary>
<br>

    Rattata
    Raticate
    Raichu
    Sandshrew
    Sandslash
    Vulpix
    Ninetales
    Diglett
    Dugtrio
    Meowth
    Persian
    Geodude
    Graveler
    Golem
    Grimer
    Muk
    Exeggutor
    Marowak
</details>
<details> <!-- galarians -->
<summary>- Galarian Forms</summary>
<br>

    Meowth
    Ponyta
    Rapidash
    Slowpoke
    Slowbro
    Farfetch'd
    Weezing
    Mr. Mime
    Articuno
    Zapdos
    Moltres
    Slowking
    Corsola
    Zigzagoon
    Linoone
    Darumaka
    Darmanitan (both Zen Modes included)
    Yamask
    Stunfisk
</details>
<details> <!-- cosmetics -->
<summary>- Cosmetic Forms</summary>
<br>
<details>
<summary>    Pikachu</summary>

        Cosplay
        Rock Star
        Belle
        Pop Star
        Ph.D
        Libre
        Original Cap
        Hoenn Cap
        Sinnoh Cap
        Unova Cap
        Kalos Cap
        Alola Cap
        Partner Cap
        World Cap
</details>
<details>
<summary>    Basculin</summary>

        Red Stripe is form 0
        Blue Stripe
        White Stripe
</details>
<details>
<summary>    Deerling</summary>

        Spring is form 0
        Summer
        Autumn
        Winter
</details>
<details>
<summary>    Sawsbuck</summary>

        Spring is form 0
        Summer
        Autumn
        Winter
</details>
    Tornadus Therian
    <br>
    Thundurus Therian
    <br>
    Landorus Therian
    <br>
<details>
<summary>    Kyurem</summary>

        White
        Black
</details>
    Keldeo Resolute
    <br>
<details>
<summary>    Genesect</summary>

        Douse
        Shock
        Burn
        Chill
</details>
    Greninja Battle Bond
    <br>
<details>
<summary>    Vivillon</summary>

        Meadow is form 0
        Polar
        Tundra
        Continental
        Garden
        Elegant
        Meadow
        Modern
        Marine
        Archipelago
        High Plains
        Sandstorm
        River
        Monsoon
        Savanna
        Sun
        Ocean
        Jungle
        Fancy
        Poké Ball
</details>
<details>
<summary>    Flabébé</summary>

        Red Flower is form 0
        Yellow Flower
        Orange Flower
        Blue Flower
        White Flower
</details>
<details>
<summary>    Floette</summary>

        Red Flower is form 0
        Yellow Flower
        Orange Flower
        Blue Flower
        White Flower
        Eternal Flower
</details>
<details>
<summary>    Florges</summary>

        Red Flower is form 0
        Yellow Flower
        Orange Flower
        Blue Flower
        White Flower
</details>
<details>
<summary>    Furfrou</summary>

        Natural is form 0
        Heart
        Star
        Diamond
        Debutante
        Matron
        Dandy
        La Reine
        Kabuki
        Pharaoh
</details>
<details>
<summary>    Pumpkaboo</summary>

        Medium is form 0
        Small
        Large
        Super
</details>
<details>
<summary>    Gourgeist</summary>

        Medium is form 0
        Small
        Large
        Super
</details>
    Hoopa Unbound
    <br>
<details>
<summary>    Oricorio</summary>

        Baile is form 0
        Pom Pom
        Pa'u
        Sensu
</details>
    Rockruff Own Tempo
    <br>
<details>
<summary>    Lycanroc</summary>

        Day is form 0
        Midnight
        Dusk
</details>
    Magearna Original
    <br>
    Toxtricity Low Key
    <br>
    Sinistea Antique
    <br>
    Polteageist Antique
    <br>
<details>
<summary>    Alcremie</summary>

        Strawberry Sweet is form 0
        Berry Sweet
        Love Sweet
        Star Sweet
        Clover Sweet
        Flower Sweet
        Ribbon Sweet
</details>
    Urshifu Rapid Strike
    Zarude Dada
<details>
<summary>    Calyrex</summary>

        Ice Rider
        Shadow Rider
</details>
</details>
<details> <!-- battle forms -->
<summary>- Battle Forms</summary>
<br>
<details>
<summary>    Castform</summary>

        Normal is form 0
        Sunny
        Rainy
        Snowy
</details>
    Cherrim Sunshine
    <br>
    Shellos East Sea
    <br>
    Gastrodon East Sea
    <br>
    Dialga Origin
    <br>
    Palkia Origin
    <br>
    Meloetta Pirouette
    <br>
    Greninja Ash
    <br>
    Aegislash Blade
    <br>
    Xerneas Active
    <br>
<details>
<summary>    Zygarde</summary>

        50% is form 0
        10%
        10% Power Construct
        50% Power Construct
        10% Complete
        50% Complete
</details>
    Wishiwashi School
    <br>
<details>
<summary>    Minior</summary>

        Red is form 0
        Orange
        Yellow
        Green
        Blue
        Indigo
        Violet
        Core Red
        Core Orange
        Core Yellow
        Core Green
        Core Blue
        Core Indigo
        Core Violet
</details>
    Mimikyu Busted
    <br>
<details>
<summary>    Necrozma</summary>

        Base is form 0
        Dusk Mane
        Dawn Wings
        Ultra Dusk Mane
        Ultra Dawn Wings
</details>
<details>
<summary>    Cramorant</summary>

        Base is form 0
        Gulping
        Gorging
</details>
    Eiscue NoIce Face
    <br>
    Morpeko Hangry
    <br>
    Zacian Crowned
    <br>
    Zamazenta Crowned
    <br>
    Eternatus Eternamax
    <br>
    Enamorus Therian
    <br>


</details>
<details> <!-- hisuians -->
<summary>- Hisuian Forms</summary>
<br>

    Growlithe
    Arcanine
    Voltorb
    Electrode
    Typhlosion
    Qwilfish
    Sneasel
    Samurott
    Lilligant
    Zorua
    Zoroark
    Braviary
    Sliggoo
    Goodra
    Avalugg
    Decidueye
</details>

## Additional Form-Changing Methods
Along with new forms, various new methods to change specific Pokemon forms have also been implemented.  These are discussed below.

### Mega Evolution
Every mega evolution is actually just a form of the base Pokemon.  When holding the specific mega stone for the species, the Pokemon will mega evolve.

### Darmanitan's Zen Mode
Darmanitan with their Hidden Ability bit set get the ability Zen Mode.  This allows for Darmanitan to swap between Zen Mode and Normal Mode as it pleases.  Galarian Darmanitan will become Galarian Zen Mode Darmanitan as well.

### Meloetta Pirouette
A Meloetta using Relic Song changes into its Pirouette Forme.  When switching out or fainting, the Meloetta changes back to its normal forme.

### Genesect Formes
A Genesect given any one of its Drive items will have that drive inserted on its head.

### Therian Formes
Tornadus, Thundurus, Landorus, and Enamorus--when a Reveal Mirror is used on them--will transform to and from their Therian formes and their Incarnate formes.

### Kyurem Black & White
Kyurem that have the DNA Splicers used on them set a chain of events off:
- First, it checks for the forme of the Kyurem.
- If the forme is 0, then it looks for a Reshiram or a Zekrom in the party starting from the front going to the back.
    - If it finds one, then it stores the Reshiram or Zekrom in the save as it is.  A flag (separate from the scripting flags) also in the save is set that tells us that a Reshiram/Zekrom is stored.
    - The Reshiram or the Zekrom is deleted from the party.
    - The Kyurem then transforms into the respective forme (White or Black depending on if Reshiram or Zekrom was chosen).  
    - Finally, Scary Face and Glaciate are replaced with Fusion Flare/Bolt and Ice Burn/Freeze Shock, respectively (if they are present).
- If the forme is not 0 and a Reshiram/Zekrom is stored in the save properly, then it looks for a free party spot starting from the front going to the back.
    - If there is a free spot, then the Reshiram or Zekrom is added to the first available free spot.
    - The Reshiram or the Zekrom is deleted from the save and the flag is cleared.
    - The Kyurem then reverts to forme 0.
    - Fusion Flare/Bolt is replaced with Scary Face, and Ice Burn/Freeze Shock is replaced with Glaciate.

### Deerling and Sawsbuck - TODO
Deerling and Sawsbuck in the PC do not change form.  They only change form in the party.  They change form if the code ever accesses the player's party directly from the save (which it will basically instantly).

Each season lasts a month.  Spring is in January, May, and September.  Summer is in February, June, and October.  Autumn is in March, July, and November.  Winter is then in April, August, and December.  This mimics the Gen 5 season system.

### Keldeo Resolute - TODO
While there is currently no way to transform Keldeo Ordinary to Resolute, Keldeo Resolute, upon forgetting Sacred Sword, will transform back into Keldeo Ordinary.
