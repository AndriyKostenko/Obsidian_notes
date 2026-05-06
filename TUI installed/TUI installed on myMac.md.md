
**MY TUIs installed:**

**- “lazydocker” - docker images/containers**

**- “lazygit” - Git tree**

**- “gh dash” - GitHub repos PR / Issues**

**- “dive <your-image-tag>” - docker image info**

**- “ctop” - managing the docker memory/%**

**- “k9s” - Kubernetes CLI**

**- “k6” - for load testing of the app**

**- “btop” - all load/processes**

**- “yazi” - terminal**

**- “lazysql” - for database management**

**- "RTK" - rtk compresses command outputs before they reach the context window. Better reasoning. Longer sessions. Lower costs Better code, longer sessions, lower costs ( "rtk gain")**

**- "Caveman" - A Claude Code skill/plugin and Codex plugin that makes agent talk like caveman — cutting **~75% of output tokens
**Manual install per agent:**

**Manual install per agent:**

| Agent | Command |
|---|---|
| **Claude Code** | `claude plugin marketplace add JuliusBrussee/caveman && claude plugin install caveman@caveman` |
| **Gemini CLI** | `gemini extensions install https://github.com/JuliusBrussee/caveman` |
| **Cursor / Windsurf / Cline / Copilot** | `npx skills add JuliusBrussee/caveman -a <cursor\|windsurf\|cline\|github-copilot>` |
| **Codex / opencode / Roo / Amp / Goose / Kiro / Augment / Aider Desk / Continue / Kilo / Junie / Trae / Warp / Tabnine / Mistral / Qwen / Devin / Droid / ForgeCode / Bob / Crush / iFlow / OpenHands / Qoder / Rovo Dev / Replit / Antigravity** | `npx skills add JuliusBrussee/caveman -a <profile>` (see `install.sh --list` for the full slug list) |
| **Anything else (40+ agents)** | `npx skills add JuliusBrussee/caveman` (auto-detect) |
**