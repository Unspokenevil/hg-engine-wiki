## Troubleshooting

<details>
<summary>My wild Pokémon with forms show up in a weird corrupt Dex and the game crashes? (see attached image)</summary>
<br>

![image](https://user-images.githubusercontent.com/16446370/210486548-d9bbe1a2-33a1-4b7d-b9b0-e9ca1d313c2b.png)

Ensure you are using the encounter macros properly in [`armips/data/encounters.s`](https://github.com/BluRosie/hg-engine/blob/main/armips/data/encounters.s) as per [[Wild Pokémon Table Documentation|Wild-Pokémon-Table-Documentation]].

This specific error was caused by specifying `pokemon SPECIES_RATTATA_ALOLAN` instead of `monwithform SPECIES_RATTATA, 1`.

</details>

<details>
<summary>Build fails when it can't find "msc" from an active repository before 11 Sept. 2023</summary>
<br>

After trying to deal with all of .NET's interoperability issues with WSL, I finally chose to switch on that date to just using `mono` instead.  Existing repositories just need to run:
```
sudo apt install mono-devel
make clean_tools --jobs
make build_tools --jobs
```
before running `make --jobs` again.  This will fix the hanging issue as well for `pngtobtx0` and `swav2swar`.
</details>

<details>
<summary>Build fails when it "doesn't have permissions" for any reason</summary>
<br>

Try running it again with `sudo` in front:  `sudo make build_tools --jobs`
</details>

<details>
<summary>Build fails on otherpoke.narc with a message saying that LF line endings aren't supported</summary>
<br>

Run:
```
git rm -rf --cached .
git reset --hard HEAD
```
The affected pal files in `rawdata/otherpoke/arceus-fairy-normal.pal` and `rawdata/otherpoke/arceus-fairy-normal.pal` should now use CR LF line endings.  Alternatively, you can manually convert them yourselves with Notepad++.

The correct file will show "Windows (CR LF)" at the bottom right of the editor in Notepad++.
</details>

<details>
<summary>WSL fully crashes out when building armips</summary>
<br>

Remove the `--jobs` from the command, running just `make build_tools`.  It will take longer, but will take less memory, and thus won't crash WSL.
</details>

If all of this fails/your problem isn't present, please join [the Kingdom of DS Hacking Discord server](https://discord.gg/zAtqJDW2jC) and wait the 11-minute probationary period before asking your question in `#hg-feature-expansions`.
