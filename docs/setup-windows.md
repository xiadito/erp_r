# Windows Setup Guide (Stage 0)

Step-by-step installation of **every tool** the Restaurant ERP project needs, from the basics through deployment, on **Windows 11** (Windows 10 differences are called out where relevant). Written for a developer who is new to the .NET and Angular ecosystems: each tool says what it is, why the project needs it, how to install it, how to verify it, and what usually goes wrong.

> **Last reviewed: October 19, 2026.** Tools move fast. Where a version or package ID could not be confirmed against the official page at writing time, the text says **(verify)**; run the suggested `winget show` / `winget search` command or open the linked official page before relying on it.

**Contents**

0. [How to use this guide](#0-how-to-use-this-guide)
1. [Prerequisites](#1-prerequisites)
2. [Git and GitHub CLI](#2-git-and-github-cli)
3. [Visual Studio Code and extensions](#3-visual-studio-code-and-extensions)
4. [.NET 10 SDK, dev certificate, and dotnet-ef](#4-net-10-sdk-dev-certificate-and-dotnet-ef)
5. [Node.js (via fnm) and Angular CLI 22](#5-nodejs-via-fnm-and-angular-cli-22)
6. [SQLite tooling](#6-sqlite-tooling)
7. [API testing tools](#7-api-testing-tools)
8. [Deployment tooling (installed now, used later)](#8-deployment-tooling-installed-now-used-later)
9. [Project directory and repository setup](#9-project-directory-and-repository-setup)
10. [Verify everything](#10-verify-everything)
11. [Done when (Stage 0)](#11-done-when-stage-0)
12. [Troubleshooting quick reference](#12-troubleshooting-quick-reference)
13. [Decisions made and unverified items](#13-decisions-made-and-unverified-items)

---

## 0. How to use this guide

- Work **top to bottom**. Later sections assume earlier ones.
- Run commands in **PowerShell 7** (`pwsh`) inside **Windows Terminal** (installed in section 1). Until PowerShell 7 is installed, the built-in Windows PowerShell 5.1 works for section 1.
- Commands marked **(admin)** need an elevated terminal: right-click Windows Terminal, **Run as administrator**. Everything else runs as your normal user.
- **Close and reopen the terminal** after every install that changes `PATH` (almost all of them). "Command not found" right after an install is almost always this.
- Package installs use **winget**. Each section also links the official installer as an alternative.

### What you are installing and when it is used

| Tool | Purpose in this project | Needed from |
| --- | --- | --- |
| Windows Terminal, PowerShell 7 | Modern shell and terminal for every command | Now |
| Git, GitHub CLI | Version control, GitHub auth, later CI/CD | Now |
| VS Code (+ extensions) | Main editor for C# and Angular | Now |
| .NET 10 SDK, `dotnet-ef` | Build/run the API, EF Core migrations | Stage 0-1 |
| Node.js (fnm), Angular CLI 22 | Build/run the Angular front end | Stage 5 |
| DB Browser for SQLite, `sqlite3` | Inspect the SQLite database | Stage 1 |
| Swagger UI (in project), REST Client / Bruno | Test the API | Stage 2 |
| WSL 2, Docker Desktop, Azure CLI, GitHub Actions | Containers, deployment, CI/CD | **After v1 is validated** |

> **Deployment tooling is installed up front but only USED after v1 is validated.** The project rule (see "Do NOT do now" in `CLAUDE.md`) is: no Docker, deploy, or CI/CD until a full month has been closed in the system without going back to the spreadsheet. Installing now just saves setup time later.

---

## 1. Prerequisites

### 1.1 Windows version and Windows Update

**What/why:** current tooling (winget, WSL 2, Docker Desktop, Visual Studio 2026) assumes a supported Windows build.

1. Use **Windows 11** (23H2 or newer recommended).
2. Open **Settings > Windows Update**, click **Check for updates**, install everything, and reboot.
3. Check your build:
   ```powershell
   winver
   ```

> **Windows 10:** most tools still install, but Windows 10 left mainstream support on October 14, 2025, and newer tools increasingly require Windows 11 (verify each tool's requirements). If you must stay on Windows 10, use version 22H2. `winget` and `wsl --install` work on 22H2; Windows Terminal and PowerShell 7 must be installed manually via winget (they are not preinstalled).

Docs: [Windows Update](https://support.microsoft.com/windows/windows-update-faq-8a903416-6f45-0718-f5c7-375e92dddeb2).

### 1.2 winget (App Installer)

**What/why:** Windows' official package manager. One command installs and updates each tool, which keeps this guide short and repeatable.

1. Windows 11 ships with it. Check:
   ```powershell
   winget --version
   ```
   Expected: a version number such as `v1.x`.
2. If the command is not found, install or update **App Installer** from the Microsoft Store (search "App Installer"), then reopen the terminal.
3. Accept the source agreements on first use (`winget search git`, answer `Y`).

Useful commands: `winget search <name>`, `winget show --id <Id>` (details and exact versions), `winget upgrade` (list updates), `winget upgrade --all`.

Docs: [Use the winget tool](https://learn.microsoft.com/windows/package-manager/winget/).

**Pitfalls:** winget does not need admin for most packages, but some installers raise a UAC prompt; approve it. If a package is "not found", the ID is wrong: check with `winget search`.

### 1.3 Windows Terminal

**What/why:** the modern tabbed terminal that hosts PowerShell, Git Bash, and WSL.

```powershell
winget install --id Microsoft.WindowsTerminal -e
```

It is preinstalled on Windows 11 22H2+. Open it from the Start menu. Verify:

```powershell
wt --version
```

Docs: [Install Windows Terminal](https://learn.microsoft.com/windows/terminal/install).

### 1.4 PowerShell 7

**What/why:** the cross-platform PowerShell (`pwsh`). The guide's examples use `&&`, `;`, and other syntax that Windows PowerShell 5.1 (the blue built-in one) does not fully support. Both can coexist.

```powershell
winget install --id Microsoft.PowerShell -e
```

Close the terminal, open a new tab, and verify:

```powershell
pwsh --version
$PSVersionTable.PSVersion
```

Expected: `PowerShell 7.x`.

**Make it the default:** Windows Terminal > **Settings** > **Startup** > **Default profile** > **PowerShell** (the one with the black icon, not "Windows PowerShell").

Docs: [Installing PowerShell on Windows](https://learn.microsoft.com/powershell/scripting/install/installing-powershell-on-windows).

**Pitfalls:** "PowerShell" and "Windows PowerShell" are different entries in the profile list. Check `$PSVersionTable.PSVersion.Major` is 7 or higher.

### 1.5 Execution policy (needed for npm and Angular scripts)

**What/why:** PowerShell blocks `.ps1` scripts by default on some systems. npm and the Angular CLI install `npm.ps1` / `ng.ps1` shims, and fnm's profile line is also a script. If blocked you see "running scripts is disabled on this system".

```powershell
Get-ExecutionPolicy -List
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

`RemoteSigned` allows local scripts and requires a signature only for scripts downloaded from the internet. `-Scope CurrentUser` changes only your account and needs no admin.

Docs: [About execution policies](https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_execution_policies).

### 1.6 Enable long file paths (recommended)

**What/why:** `node_modules` creates deeply nested paths. Windows historically limits paths to 260 characters.

**(admin)**
```powershell
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
  -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
```

Reboot afterward. Git needs its own switch, set in section 2.

Docs: [Maximum path length limitation](https://learn.microsoft.com/windows/win32/fileio/maximum-file-path-limitation).

### 1.7 Quality-of-life setting: show file extensions

File Explorer > **View > Show > File name extensions**. It prevents accidents like `appsettings.json.txt`.

---

## 2. Git and GitHub CLI

### 2.1 Git for Windows

**What/why:** version control. Every stage ends with a commit. Git for Windows also bundles **Git Credential Manager**, which stores your GitHub login securely.

```powershell
winget install --id Git.Git -e
```

Reopen the terminal, then verify:

```powershell
git --version
```

Expected: `git version 2.x.x.windows.x`.

Official installer (alternative): https://git-scm.com/download/win

#### Initial global configuration

```powershell
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global core.autocrlf true
git config --global core.longpaths true
git config --global credential.helper manager
git config --global pull.rebase false
```

| Setting | Why |
| --- | --- |
| `user.name`, `user.email` | Authorship on every commit. Use the email tied to your GitHub account (or GitHub's `noreply` address if you hide your email). |
| `init.defaultBranch main` | New repositories start on `main`, matching GitHub. |
| `core.autocrlf true` | Windows tools use CRLF line endings, repositories store LF. With `true`, Git converts LF to CRLF on checkout and back to LF on commit. This is the Git for Windows default recommendation. A repo-level `.gitattributes` (section 9) makes the rule explicit for everyone. |
| `core.longpaths true` | Lets Git handle paths over 260 characters (pairs with section 1.6). |
| `credential.helper manager` | Uses Git Credential Manager for HTTPS credentials (a browser sign-in once, then it remembers). |
| `pull.rebase false` | Explicit merge strategy for `git pull`, avoids a hint message. |

Review everything with `git config --global --list`.

Docs: [git-config](https://git-scm.com/docs/git-config), [gitattributes](https://git-scm.com/docs/gitattributes), [Git Credential Manager](https://github.com/git-ecosystem/git-credential-manager).

**Pitfalls:** warnings such as `LF will be replaced by CRLF` are normal with `autocrlf=true`; they are not errors. If `git` is not found after install, open a **new** terminal.

### 2.2 GitHub CLI (`gh`)

**What/why:** GitHub from the terminal: clone, authenticate, create PRs, and later inspect GitHub Actions runs. It is also the easiest way to authenticate Git to GitHub.

```powershell
winget install --id GitHub.cli -e
```

Reopen the terminal, then:

```powershell
gh --version
gh auth login
```

In the prompts choose: **GitHub.com**, protocol **HTTPS**, "Authenticate Git with your GitHub credentials?" **Yes**, then **Login with a web browser** and enter the one-time code. Verify:

```powershell
gh auth status
```

Expected: `Logged in to github.com account <your-user>`.

Docs: [GitHub CLI](https://cli.github.com/), [`gh auth login`](https://cli.github.com/manual/gh_auth_login).

**Pitfalls:** if `git push` later asks for a password, run `gh auth setup-git` to wire Git to the `gh` credentials.

---

## 3. Visual Studio Code and extensions

### 3.1 VS Code

**What/why:** the primary editor for both the C# API and the Angular app.

```powershell
winget install --id Microsoft.VisualStudioCode -e
```

Reopen the terminal and verify the `code` command is on `PATH`:

```powershell
code --version
```

Expected: three lines, the first like `1.1xx.x`.

Official installer (alternative): https://code.visualstudio.com/Download. If you use it, keep **Add to PATH** checked.

Docs: [VS Code on Windows](https://code.visualstudio.com/docs/setup/windows).

### 3.2 Recommended extensions

Extensions are installed from the command line with `code --install-extension <publisher.id>` (IDs are case-insensitive).

```powershell
$extensions = @(
  'ms-dotnettools.csdevkit',            # C# Dev Kit (Solution Explorer, test explorer); pulls in the C# extension
  'ms-dotnettools.csharp',              # C# language support (IntelliSense, debugging)
  'angular.ng-template',                # Angular Language Service
  'editorconfig.editorconfig',          # Applies .editorconfig rules
  'esbenp.prettier-vscode',             # Formatter commonly used in Angular projects
  'qwtel.sqlite-viewer',                # View SQLite files inside VS Code
  'humao.rest-client',                  # Send requests from .http files
  'ms-azuretools.vscode-docker',        # Docker (used after v1)
  'github.vscode-github-actions',       # GitHub Actions workflows (used after v1)
  'github.vscode-pull-request-github',  # Pull requests and issues
  'ms-vscode-remote.remote-wsl',        # Work inside WSL (used with Docker)
  'ms-vscode.powershell',               # PowerShell language support
  'ms-azuretools.vscode-azureappservice'# Azure App Service (used after v1)
)
$extensions | ForEach-Object { code --install-extension $_ }
code --list-extensions
```

| Extension | Why |
| --- | --- |
| C# Dev Kit + C# | IntelliSense, debugging, project/solution management for .NET. |
| Angular Language Service | Type-checks and autocompletes Angular templates. |
| EditorConfig | Makes VS Code obey the repo's `.editorconfig`. |
| Prettier | Consistent formatting for TypeScript/HTML/CSS/JSON. |
| SQLite Viewer | Quick look at `erp.db` without leaving the editor. |
| REST Client | Runs the `RestaurantErp.Api.http` file generated by the template. |
| Docker, GitHub Actions, WSL, Azure App Service | Used only after v1 is validated. |

> **Licensing note:** C# Dev Kit is free for individuals, students, and open-source use under its own license terms; check https://code.visualstudio.com/docs/csharp/get-started if you use it commercially.

Docs: [Extension Marketplace: command line](https://code.visualstudio.com/docs/configure/extensions/extension-marketplace), [C# in VS Code](https://code.visualstudio.com/docs/csharp/get-started), [Angular Language Service](https://angular.dev/tools/language-service).

**Pitfalls:** if an ID fails with "extension not found", search for it in the Extensions view (it may have been renamed) and use the ID shown on its page. Restart VS Code after installing C# Dev Kit.

### 3.3 Optional alternatives (not required)

- **Visual Studio 2026 Community** (free for individuals): `winget install --id Microsoft.VisualStudio.Community -e`; select the **ASP.NET and web development** workload. https://visualstudio.microsoft.com/
- **JetBrains Rider** (free for non-commercial use, verify current terms): `winget install --id JetBrains.Rider -e`. https://www.jetbrains.com/rider/

This guide and the project are written around VS Code; stay with it unless you have a reason.

---

## 4. .NET 10 SDK, dev certificate, and dotnet-ef

### 4.1 .NET 10 SDK

**What/why:** the **SDK** is the toolbox that creates and builds projects (`dotnet new`, `dotnet build`, `dotnet run`). The **runtime** executes the compiled program. The SDK installer includes the .NET runtime and the **ASP.NET Core runtime**, so one install covers both for development. .NET 10 is the current **LTS** release (long-term support).

```powershell
winget install --id Microsoft.DotNet.SDK.10 -e
```

Official installer (alternative): https://dotnet.microsoft.com/download/dotnet/10.0

Reopen the terminal and verify:

```powershell
dotnet --version
dotnet --list-sdks
dotnet --list-runtimes
dotnet --info
```

Expected: `dotnet --version` prints `10.0.xxx` (the latest patch I could confirm was SDK `10.0.112` with runtime `10.0.12` in September 2026; **verify** the current patch on the download page). `--list-runtimes` should include `Microsoft.AspNetCore.App 10.0.x` and `Microsoft.NETCore.App 10.0.x`.

Docs: [Install .NET on Windows](https://learn.microsoft.com/dotnet/core/install/windows), [.NET support policy](https://dotnet.microsoft.com/platform/support/policy/dotnet-core).

**Optional but good practice:** pin the SDK used by the repo so every machine builds the same way:

```powershell
dotnet new globaljson --sdk-version 10.0.112 --roll-forward latestFeature
```

(Replace `10.0.112` with your installed version. Run it in the repo root once the repo exists.) Docs: [global.json](https://learn.microsoft.com/dotnet/core/tools/global-json).

**Optional:** opt out of CLI telemetry for your user:

```powershell
[Environment]::SetEnvironmentVariable('DOTNET_CLI_TELEMETRY_OPTOUT', '1', 'User')
```

**Pitfalls:**
- Multiple SDKs installed: `dotnet --version` shows the one `global.json` or the latest selects. Use `dotnet --list-sdks` to see all.
- 32-bit vs 64-bit: install the x64 SDK on 64-bit Windows (winget picks the right one); on ARM64 Windows winget installs the ARM64 build.

### 4.2 HTTPS development certificate

**What/why:** ASP.NET Core serves `https://localhost` in development with a self-signed certificate. Trusting it removes browser warnings.

```powershell
dotnet dev-certs https --trust
```

Windows shows a security dialog asking to install the certificate: click **Yes**. Check:

```powershell
dotnet dev-certs https --check --trust
```

Expected: `A valid HTTPS certificate with a trusted root was found.`

Docs: [dotnet dev-certs](https://learn.microsoft.com/dotnet/core/tools/dotnet-dev-certs), [Trust the ASP.NET Core HTTPS development certificate](https://learn.microsoft.com/aspnet/core/security/enforcing-ssl#trust-the-aspnet-core-https-development-certificate-on-windows-and-macos).

**Pitfalls:** if the browser still warns, close **all** browser windows and reopen. Reset with `dotnet dev-certs https --clean` then `--trust` again. (The project starts with HTTP only; this prepares you for deployment and for the template's HTTPS profile.)

### 4.3 `dotnet-ef` (EF Core command-line tools)

**What/why:** a **global .NET tool** that creates and applies EF Core migrations (`dotnet ef migrations add`, `dotnet ef database update`). **Its major version must match the EF Core packages in the project (`10.x`)**; a mismatch is the number-one cause of odd migration errors.

```powershell
dotnet tool install --global dotnet-ef --version 10.*
dotnet ef --version
```

`--version 10.*` installs the newest `10.x` release (this is the form Microsoft's EF Core tools page uses for a major version). Expected output of `dotnet ef --version`: the "Entity Framework Core .NET Command-line Tools" banner followed by `10.0.x`.

Update later with:

```powershell
dotnet tool update --global dotnet-ef --version 10.*
dotnet tool list --global
```

Docs: [EF Core tools reference (.NET CLI)](https://learn.microsoft.com/ef/core/cli/dotnet), [.NET tools](https://learn.microsoft.com/dotnet/core/tools/global-tools).

#### PATH check for global tools

The .NET installer adds `%USERPROFILE%\.dotnet\tools` to your user `PATH` automatically. Verify (after opening a new terminal):

```powershell
$env:PATH -split ';' | Select-String 'dotnet\\tools'
Test-Path "$env:USERPROFILE\.dotnet\tools\dotnet-ef.exe"
```

If the first command prints nothing, add it for your user and reopen the terminal:

```powershell
$toolsPath = "$env:USERPROFILE\.dotnet\tools"
$userPath  = [Environment]::GetEnvironmentVariable('Path', 'User')
if (($userPath -split ';') -notcontains $toolsPath) {
  [Environment]::SetEnvironmentVariable('Path', "$userPath;$toolsPath", 'User')
}
```

**Pitfalls:** `dotnet ef` printing "Could not execute because the specified command or file was not found" means PATH (above) or no new terminal. The tool version and the `Microsoft.EntityFrameworkCore.*` package versions must both be `10.x`.

---

## 5. Node.js (via fnm) and Angular CLI 22

### 5.1 Which Node.js version

Angular 22 declares supported Node.js versions on its official versions page (https://angular.dev/reference/versions). At the time of writing, the Angular 22.0.x row lists **Node.js `^22.22.0 || ^24.13.1 || ^26.0.0`** with **TypeScript `>=6.0.0 <6.1.0`**. Node 20 is no longer supported by Angular 22. **Verify the table yourself before installing** (it changes with patch releases).

**Choice: Node.js 24 LTS.** It is inside Angular 22's range, is an LTS line (supported through April 2028), and is the version the project guide chose. Node 26 is also in range; it was the "Current" line as of this writing and is expected to become LTS in late October 2026 (verify at https://nodejs.org/en/about/previous-releases). Because you will use a version manager, switching later is one command.

### 5.2 Node version manager: fnm

**What/why:** a version manager lets you install several Node versions side by side and switch per project. That matters because Angular majors pin Node ranges, and future projects will need different versions.

**Choice: [fnm](https://github.com/Schniz/fnm) (Fast Node Manager).** Reasons:
- It is a single native binary, fast, and installs with winget.
- It does not need admin rights or symlinks, unlike nvm-windows.
- It reads a `.node-version` file, so the repo can declare its Node version.
- The Node.js download page lists fnm as the Windows version-manager option (verify at https://nodejs.org/en/download).

Alternative: [nvm-windows](https://github.com/coreybutler/nvm-windows) (a separate project from the Linux/macOS `nvm`). Do not install both.

#### Install fnm

```powershell
winget install --id Schniz.fnm -e
```

Reopen the terminal, then enable it in your PowerShell profile so every new terminal (including VS Code's integrated terminal) uses it:

```powershell
# Create the profile file if it does not exist, then open it
if (-not (Test-Path $PROFILE)) { New-Item -Path $PROFILE -ItemType File -Force }
notepad $PROFILE
```

Add this line at the **end** of the profile, save, and close:

```powershell
fnm env --use-on-cd --shell powershell | Out-String | Invoke-Expression
```

Reopen the terminal and install Node:

```powershell
fnm install 24
fnm default 24
fnm use 24
node --version
npm --version
fnm list
```

Expected: `node --version` prints `v24.x.x` (**24.13.1 or newer** to satisfy Angular 22); `npm --version` prints `11.x`.

Docs: [fnm README](https://github.com/Schniz/fnm), [Node.js downloads](https://nodejs.org/en/download), [Node.js releases](https://nodejs.org/en/about/previous-releases).

> **Alternative without a version manager:** `winget install --id OpenJS.NodeJS.LTS -e` installs the current LTS directly (verify it is within Angular's table). Simpler, but you lose easy switching.

**Pitfalls:**
- `fnm: command not found` after install: open a new terminal.
- `node` not found after fnm install: the profile line is missing, or the execution policy blocks the profile (section 1.5). Run `$PROFILE` to see its path and `Get-ExecutionPolicy -List`.
- VS Code terminal uses a different `node`: restart VS Code after editing the profile.

### 5.3 Angular CLI 22

**What/why:** the `ng` command creates, serves, builds, and tests Angular apps (Stage 5). Installing it globally means `ng new web` works without `npx` each time.

```powershell
npm install -g @angular/cli@22
ng version
```

Expected: an Angular ASCII banner, then lines such as `Angular CLI: 22.x.x`, `Node: 24.x.x`, `Package Manager: npm 11.x.x`, and `OS: win32 x64`. Outside an Angular project, the "Angular:" package list may show `Error: This command is not available when running the Angular CLI outside a workspace`; that is expected.

Optional: disable analytics prompts with `ng analytics disable`.

Docs: [Installation](https://angular.dev/installation), [Version compatibility](https://angular.dev/reference/versions), [Angular CLI](https://angular.dev/tools/cli).

**Pitfalls:**
- `ng : File ...\ng.ps1 cannot be loaded because running scripts is disabled` is the execution policy (section 1.5).
- `ng` not found after the global install: global npm packages live in `%APPDATA%\npm` (or in fnm's per-version folder). With fnm, a global install belongs to the active Node version; switching versions hides it until you reinstall.
- Node version error from `ng new`: your Node is outside the table; run `node --version` and compare.

---

## 6. SQLite tooling

### 6.1 DB Browser for SQLite

**What/why:** a graphical viewer/editor for `erp.db`. You will use it to see the `Transactions` table, run SQL, and compare it to the DDL in `data-model.md`.

```powershell
winget install --id DBBrowserForSQLite.DBBrowserForSQLite -e
```

Launch it from the Start menu (**DB Browser for SQLite**) and open `api\erp.db` once it exists (Stage 1). Official installer (alternative): https://sqlitebrowser.org/dl/

Verification: the app starts; the package listed by `winget list DBBrowserForSQLite` shows a `3.x` version (latest I could confirm: `3.13.1`; verify).

**Pitfalls:** do not edit data in the browser while the API is running (file locks, and WAL side files). Close the database in the tool (**Close Database**) before running the app again if you see "database is locked".

### 6.2 `sqlite3` command-line shell (optional)

**What/why:** a scriptable shell for quick checks such as `.schema Transactions` and `EXPLAIN QUERY PLAN`.

```powershell
winget install --id SQLite.SQLite -e
```

This winget package ID is listed in a SQLite forum post about winget; **verify with `winget search sqlite`** because the package may be portable or renamed. If it is unavailable, download the "sqlite-tools" bundle for Windows from https://www.sqlite.org/download.html and put the folder on `PATH`.

Verify:

```powershell
sqlite3 --version
```

Expected: `3.x.x <date> ...`.

Docs: [SQLite command-line shell](https://www.sqlite.org/cli.html).

---

## 7. API testing tools

### 7.1 Swagger UI (comes with the project)

Nothing to install. In Stage 2 the project adds `Swashbuckle.AspNetCore.SwaggerUI`, and the API serves an interactive page at `/swagger` in development. Docs: [OpenAPI in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/openapi/overview).

### 7.2 VS Code REST Client (installed in section 3)

The `dotnet new webapi` template generates a `RestaurantErp.Api.http` file. With the **REST Client** extension, click **Send Request** above any request in that file. Requests live in the repo and are reviewable in Git.

### 7.3 Optional: Bruno or Postman

A GUI API client with saved collections. Pick **one**, only if you want it:

```powershell
winget install --id Bruno.Bruno -e          # Bruno: open source, collections stored as plain files
# or
winget install --id Postman.Postman -e      # Postman
```

Both IDs are commonly used but **verify** with `winget search bruno` / `winget search postman` before installing. Downloads: https://www.usebruno.com/downloads and https://www.postman.com/downloads/.

---

## 8. Deployment tooling (installed now, used later)

> **Install now, use later.** Everything in this section is for after v1 is validated (a full month closed in the system). Do not add Dockerfiles, workflows, or Azure resources to the repository before that.

### 8.1 WSL 2

**What/why:** the Windows Subsystem for Linux runs a real Linux kernel inside Windows. Docker Desktop uses it as its engine, and it is where Linux-based deployment behavior can be tested.

**(admin)**
```powershell
wsl --install
```

This enables the required Windows features and installs the default Ubuntu distribution. **Reboot when asked**, then Ubuntu finishes setup on first launch (it asks for a Linux username and password). Verify:

```powershell
wsl --status
wsl --list --verbose
wsl --update
```

Expected: default version `2`, and the distribution shown with `VERSION 2`.

Requirements: hardware virtualization must be enabled in BIOS/UEFI. Check in **Task Manager > Performance > CPU > Virtualization: Enabled**.

Docs: [Install WSL](https://learn.microsoft.com/windows/wsl/install).

**Pitfalls:** "WSL 2 requires an update to its kernel component" means run `wsl --update`. If virtualization is disabled you must enable it in BIOS. On Windows 10, `wsl --install` needs version 2004+ (build 19041+); 22H2 is fine.

### 8.2 Docker Desktop

**What/why:** builds and runs containers. After v1 you may package the API in a container for portable deployment.

```powershell
winget install --id Docker.DockerDesktop -e
```

Reboot or sign out/in if the installer asks (it may add you to the `docker-users` group). Start **Docker Desktop**, accept the terms, and keep **Use WSL 2 based engine** enabled in **Settings > General**. Verify:

```powershell
docker --version
docker version
docker run --rm hello-world
```

Expected: the last command prints `Hello from Docker!`.

**Licensing note (verify current terms at https://docs.docker.com/subscription/desktop-license/):** Docker Desktop is free for **personal use, education, non-commercial open-source projects, and small businesses (fewer than 250 employees AND less than $10 million in annual revenue)**. Larger organizations need a paid subscription. A personal portfolio project is within the free tier. If you ever work on this inside a larger company, check the terms or use Docker Engine inside WSL instead.

Docs: [Install Docker Desktop on Windows](https://docs.docker.com/desktop/setup/install/windows-install/).

**Pitfalls:** Docker Desktop must be running for `docker` commands to work. "Docker Desktop requires WSL 2" means finish section 8.1 first. Docker uses a lot of RAM by default; limit it in `%USERPROFILE%\.wslconfig` if needed.

### 8.3 Azure CLI (`az`): chosen deploy target

**What/why:** the command-line tool for Microsoft Azure. For a .NET portfolio application, **Azure App Service** is the most direct deployment target: it runs ASP.NET Core natively, supports deployment from GitHub Actions, and has a free tier suitable for demos (verify current pricing and limits). The Angular app can later be served by the same API (same origin, so no CORS) or hosted separately; that is a decision for after v1.

```powershell
winget install --id Microsoft.AzureCLI -e
```

Close and reopen the terminal (Microsoft states this is required), then:

```powershell
az version
az login
az account show
```

Expected: `az version` prints a JSON object with `"azure-cli": "2.x.x"`; `az login` opens a browser sign-in. You need a (free) Azure account for `az login`: https://azure.microsoft.com/free/

Docs: [Install Azure CLI on Windows](https://learn.microsoft.com/cli/azure/install-azure-cli-windows), [Deploy an ASP.NET Core app to App Service](https://learn.microsoft.com/azure/app-service/quickstart-dotnetcore).

**Pitfalls:** SQLite is a file; on a hosted service you must think about where the file lives and whether it survives restarts and redeploys. Revisit the database choice (PostgreSQL / Azure SQL) when you design the deployment. Do not create Azure resources now.

### 8.4 GitHub Actions (CI/CD)

**What/why:** GitHub's built-in automation. Later it will run `dotnet build`/tests and deploy on push. **There is nothing to install**: workflows are YAML files under `.github/workflows/`, executed by GitHub. You already have the tools to work with it:

- `gh` (section 2.2): `gh run list`, `gh run watch`, `gh workflow list`
- The **GitHub Actions** VS Code extension (section 3.2)

Docs: [GitHub Actions documentation](https://docs.github.com/actions), [Building and testing .NET](https://docs.github.com/actions/use-cases-and-examples/building-and-testing/building-and-testing-net).

---

## 9. Project directory and repository setup

### 9.1 Choose the folder

Use a short path on a local drive, for example **`C:\dev\erp_r`**.

Why:
- **Avoid OneDrive-synced folders** (`Documents`, `Desktop`, `Pictures`). OneDrive syncs `node_modules`, `bin`, and `obj` (tens of thousands of small files), locks files during builds, and can corrupt a SQLite database that is being written.
- **Avoid long paths.** `node_modules` nests deeply; a short root keeps you under the 260-character limit even if long paths are not enabled (section 1.6).
- Keep it free of spaces and non-ASCII characters; many Node and .NET tools handle them poorly.

```powershell
New-Item -ItemType Directory -Path C:\dev -Force | Out-Null
Set-Location C:\dev
```

### 9.2 Clone the repository

The repository is `xiadito/erp_r` on GitHub. Pick one:

```powershell
gh repo clone xiadito/erp_r
# or
git clone https://github.com/xiadito/erp_r.git
```

Then:

```powershell
Set-Location C:\dev\erp_r
git status
git remote -v
code .
```

Expected: `On branch main`, a clean working tree, and `origin` pointing at GitHub.

### 9.3 Repository-level files

Create these three files at the repository root. They are small and you write them yourself (learning mode); below is what each is for and how to generate it.

**`.gitignore`: what Git must never track**

```powershell
dotnet new gitignore
Add-Content -Path .gitignore -Value '', '# SQLite database files (real financial data)', '*.db', '*.db-shm', '*.db-wal', '*.db-journal'
```

`dotnet new gitignore` writes the official .NET template (ignores `bin/`, `obj/`, `.vs/`, user files). Angular's `ng new` creates its own `web/.gitignore` later (covers `node_modules/`, `dist/`, `.angular/`). The database lines matter most: the database holds real financial data and must never be committed. `Migrations/` **is** committed.

Docs: [`dotnet new` gitignore template](https://learn.microsoft.com/dotnet/core/tools/dotnet-new-sdk-templates#gitignore), [gitignore](https://git-scm.com/docs/gitignore).

**`.gitattributes`: line-ending and diff rules for the repository**

Windows uses CRLF, Linux (and GitHub, CI, containers) use LF. This file makes Git's behavior identical on every machine regardless of personal config. A minimal example:

```gitattributes
# Normalize line endings: store LF in the repository
* text=auto

# Windows-only script types keep CRLF in the working tree
*.bat text eol=crlf
*.cmd text eol=crlf

# Never touch binary files
*.db binary
*.png binary
*.jpg binary
```

Docs: [gitattributes](https://git-scm.com/docs/gitattributes), [GitHub: configuring line endings](https://docs.github.com/get-started/git-basics/configuring-git-to-handle-line-endings).

**`.editorconfig`: editor and style rules**

Tells every editor (VS Code with the EditorConfig extension, Visual Studio, Rider) and the .NET compiler how to format and which style rules to apply (indentation, final newline, C# naming and `var` preferences).

```powershell
dotnet new editorconfig
```

This generates the official .NET `.editorconfig` template. Review it, then add the basics that apply to every file type:

```ini
root = true

[*]
charset = utf-8
indent_style = space
indent_size = 4
insert_final_newline = true
trim_trailing_whitespace = true

[*.{ts,html,css,json,yml,yaml}]
indent_size = 2
```

Docs: [EditorConfig for .NET](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/code-style-rule-options), [editorconfig.org](https://editorconfig.org/).

**Optional:** add a `.node-version` file at `web\` (contents: `24`) later so fnm selects the right Node when you enter that folder.

After creating the files, **do not commit yet** unless you are ready: `git status` should show them as untracked. When you do commit, use a Conventional Commit such as `chore: add gitignore, gitattributes, and editorconfig`.

---

## 10. Verify everything

Run this in PowerShell 7 from any folder. It prints each tool's version; a failing line means that tool is missing or not on `PATH` (open a new terminal first).

```powershell
function Show-Version($label, [scriptblock]$cmd) {
  try {
    $out = & $cmd 2>&1 | Select-Object -First 1
    Write-Host ("{0,-16} {1}" -f $label, $out)
  } catch {
    Write-Host ("{0,-16} NOT FOUND" -f $label) -ForegroundColor Red
  }
}

Write-Host "=== Base ===" -ForegroundColor Cyan
Show-Version 'Windows'      { (Get-CimInstance Win32_OperatingSystem).Caption }
Show-Version 'winget'       { winget --version }
Show-Version 'PowerShell'   { $PSVersionTable.PSVersion.ToString() }
Show-Version 'Git'          { git --version }
Show-Version 'GitHub CLI'   { gh --version }
Show-Version 'VS Code'      { code --version }

Write-Host "=== .NET ===" -ForegroundColor Cyan
Show-Version '.NET SDK'     { dotnet --version }
Show-Version 'dotnet-ef'    { (dotnet ef --version) | Select-Object -Last 1 }
Show-Version 'Dev cert'     { dotnet dev-certs https --check --trust }

Write-Host "=== Node / Angular ===" -ForegroundColor Cyan
Show-Version 'fnm'          { fnm --version }
Show-Version 'Node'         { node --version }
Show-Version 'npm'          { npm --version }
Show-Version 'Angular CLI'  { (ng version | Select-String 'Angular CLI').Line.Trim() }

Write-Host "=== SQLite ===" -ForegroundColor Cyan
Show-Version 'sqlite3'      { sqlite3 --version }
Show-Version 'DB Browser'   { (winget list --id DBBrowserForSQLite.DBBrowserForSQLite -e | Select-Object -Last 1) }

Write-Host "=== Deployment tooling (used after v1) ===" -ForegroundColor Cyan
Show-Version 'WSL'          { wsl --version }
Show-Version 'Docker'       { docker --version }
Show-Version 'Azure CLI'    { (az version --output tsv --query '"azure-cli"') }

Write-Host "=== Auth / config ===" -ForegroundColor Cyan
Show-Version 'git user'     { git config --global user.name }
Show-Version 'git email'    { git config --global user.email }
Show-Version 'autocrlf'     { git config --global core.autocrlf }
Show-Version 'gh auth'      { gh auth status 2>&1 | Select-String 'Logged in' }
Write-Host "Extensions:" ; code --list-extensions
```

### Expected output

| Check | Expected |
| --- | --- |
| Windows | Windows 11 (build 22631 or newer recommended) |
| winget | `v1.x` |
| PowerShell | `7.x.x` |
| Git | `git version 2.x.x.windows.x` |
| GitHub CLI | `gh version 2.x.x` |
| VS Code | `1.1xx.x` (first line) |
| .NET SDK | `10.0.xxx` |
| dotnet-ef | `10.0.x` |
| Dev cert | `A valid HTTPS certificate with a trusted root was found.` |
| fnm | `fnm 1.x.x` |
| Node | `v24.x.x` (24.13.1 or newer; must satisfy Angular 22's range) |
| npm | `11.x.x` |
| Angular CLI | `Angular CLI: 22.x.x` |
| sqlite3 (optional) | `3.x.x` |
| DB Browser for SQLite | `3.13.x` or newer |
| WSL | `WSL version: 2.x.x` |
| Docker | `Docker version 2x.x.x` (any current version; Docker Desktop must be running for `docker version`) |
| Azure CLI | `2.x.x` |
| git user / email | your name and email |
| autocrlf | `true` |
| gh auth | `Logged in to github.com account <you>` |
| Extensions | the list from section 3.2 |

---

## 11. Done when (Stage 0)

Stage 0 is complete when **all** of the following are true on your Windows machine:

1. `dotnet --version` shows `10.x` **and** `dotnet ef --version` responds with `10.x`.
2. `node --version` shows a version inside Angular 22's supported range (24.13.1 or newer for Node 24) **and** `ng version` shows `Angular CLI: 22.x`.
3. `git --version`, `gh auth status` (logged in), and `code --version` all succeed.
4. The repository is cloned at `C:\dev\erp_r` (outside OneDrive), opens in VS Code, and `git status` works.
5. The root `.gitignore` (with the four SQLite ignore lines), `.gitattributes`, and `.editorconfig` exist.
6. The "Verify everything" script in section 10 prints a version for every tool without `NOT FOUND` (the deployment tools may be skipped if you chose to postpone them, but they are recommended now).

Then start Stage 1 (see [`data-model.md`](data-model.md) section 11).

---

## 12. Troubleshooting quick reference

| Symptom | Likely cause and fix |
| --- | --- |
| `X is not recognized as a cmdlet, function, ...` right after install | `PATH` not refreshed. Close **all** terminals (and VS Code) and open a new one. |
| `dotnet ef` not found | `%USERPROFILE%\.dotnet\tools` not on `PATH` (section 4.3), or the tool was not installed. |
| Strange errors in `dotnet ef migrations add` | EF Core package version differs from `dotnet-ef` (both must be `10.x`), or the project does not build (`dotnet build`). |
| `running scripts is disabled on this system` | Execution policy (section 1.5). |
| `node`/`ng` not found in VS Code terminal only | Restart VS Code after editing `$PROFILE` (section 5.2). |
| `ng new` fails with a Node version error | Node is outside Angular 22's table. `fnm install 24` and `fnm default 24`. |
| `LF will be replaced by CRLF` | Informational with `core.autocrlf=true`. Ignore. |
| `Filename too long` | Enable long paths (sections 1.6 and 2.1). |
| `database is locked` | Close DB Browser for SQLite / stop the running API. |
| `docker` cannot connect to the daemon | Start Docker Desktop and wait for it to say "running"; complete WSL setup (section 8.1). |
| Browser warns about `https://localhost` | `dotnet dev-certs https --trust`, then restart the browser. |

---

## 13. Decisions made and unverified items

### Decisions

| Topic | Decision | Reason |
| --- | --- | --- |
| Shell | PowerShell 7 in Windows Terminal | Cross-platform syntax (`&&`), consistent with all project docs. |
| Node.js | **24 LTS**, installed with **fnm** | In Angular 22's supported range; long support window; fnm is lightweight, needs no admin or symlinks, and reads `.node-version`. nvm-windows is the documented alternative. |
| Angular CLI | Global (`@angular/cli@22`) | The owner asked for a global CLI; `npx` also works. |
| Line endings | `core.autocrlf=true` + repo `.gitattributes` (`* text=auto`) | Git for Windows default; the repo file makes behavior consistent for every contributor and CI. |
| Deploy target | **Azure App Service** (via Azure CLI and GitHub Actions) | Native ASP.NET Core hosting, GitHub Actions integration, and a free tier for a portfolio demo. Not used until v1 is validated. |
| Containers | WSL 2 + Docker Desktop | Standard Windows container setup; free for personal use. |
| Editor | VS Code | Covers both C# and Angular; Visual Studio 2026 Community and Rider are optional alternatives. |

### Could not be fully verified at writing time (October 19, 2026)

The documentation sites (learn.microsoft.com, angular.dev, nodejs.org) could not be fetched directly from the writing environment; facts below came from web search results and prior knowledge. Check each before relying on it.

- Exact current **.NET 10 SDK patch** (latest confirmed: 10.0.112 / runtime 10.0.12, from September 2026).
- **Angular 22 supported Node.js range** (`^22.22.0 || ^24.13.1 || ^26.0.0`, TypeScript `>=6.0.0 <6.1.0`) came from the `next.angular.dev` copy of the versions page; confirm on https://angular.dev/reference/versions. The exact `@angular/cli` 22.x patch was not confirmed.
- **Node.js release status**: Node 24 leaving Active LTS around October 20, 2026 (still supported through April 2028) and Node 26 becoming LTS around October 28, 2026 come from secondary sources; confirm on the Node.js release schedule. npm `11.x` for Node 24 is assumed.
- That the Node.js download page currently recommends **fnm** for Windows.
- winget IDs not independently confirmed in this session: `Bruno.Bruno`, `Postman.Postman`, `SQLite.SQLite` (the package may be portable or renamed), `JetBrains.Rider`, `Schniz.fnm` (seen in a community guide), `OpenJS.NodeJS.LTS`, `Docker.DockerDesktop`, `Microsoft.WindowsTerminal`, `Microsoft.PowerShell`, `Git.Git`, `GitHub.cli`, `Microsoft.VisualStudioCode`. Confirmed by sources: `Microsoft.DotNet.SDK.10`, `Microsoft.AzureCLI`, `DBBrowserForSQLite.DBBrowserForSQLite`, `Microsoft.VisualStudio.Community`. Check any ID with `winget show --id <Id>`.
- VS Code extension IDs were written from knowledge, not looked up; if one fails, search the Extensions view.
- `dotnet tool install --global dotnet-ef --version 10.*` (floating major) is the documented pattern for a major version; if your SDK rejects it, pin a full version such as `--version 10.0.7` (pick the latest from https://www.nuget.org/packages/dotnet-ef).
- Docker Desktop licensing thresholds (fewer than 250 employees and under $10 million revenue for the free business tier) were confirmed from Docker's license page via search; re-read before any commercial use.
- Windows 10 specifics (end of mainstream support October 14, 2025; per-tool minimum builds).
