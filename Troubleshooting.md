## Troubleshooting

<details>
<summary>Build fails when it "doesn't have permissions" for any reason</summary>
<br>

Try running it again with `sudo` in front:  `sudo make build_tools --jobs`
</details>

<details>
<summary>Build fails when it can't find "csc.exe"</summary>
<br>

Make sure you added your .NET framework file path to the PATH system environment variable.  Just to make sure, add it to both the specific user's PATH and the System's PATH in both of the panes of the window that pops up.  Make sure to close and reopen the WSL terminal.
</details>