# Rocky Linux on WSL – Development Environment Manual (kikancen)

This manual sets up a **development** environment for the Kikan System (`kikancen`) inside Rocky Linux on WSL, so you can edit, run and debug the Java backend and the Angular frontend without Eclipse/Tomcat on Windows.

> **Releasing / deploying?** Use [JAVA Application Deployment on WSL - Rocky Linux](JAVA-Application-Deployment-on-WSL---Rocky-Linux). That guide builds a WAR and serves it through the system Tomcat + Apache. This guide is for day-to-day development.

The manual has two parts:

| Part | When |
|---|---|
| [Part 1: Fresh Setup](#part-1-fresh-setup) (sections 1–16) | **Once**, on a new PC or a new WSL distribution. Installs everything and ends with a check that it all works |
| [Part 2: Daily Development](#part-2-daily-development) (sections 17–28) | **Every day**: start, develop, debug, stop, commit, troubleshoot |

### What you end up with

```text
 Windows                                   WSL (Rocky Linux 9)
 -------                                   -------------------
 VS Code window  ──── "WSL" extension ───> VS Code Server + Java extension
                                           source code in ~/kikancen
 Browser ── http://localhost:4200 ───────> ng serve  (Angular 13, Node 16)   ── live reload
         ── http://localhost:8080 ───────> Tomcat 9  (/kikancen, Java 8)     ── debug port 8000
                                                 │
                                                 └──> dev MySQL (db.properties)
```

| | Release guide | This guide (development) |
|---|---|---|
| Backend | `mvn clean install` → copy WAR to system Tomcat | Your own Tomcat in `~/dev/tomcat`, serving straight from `target/classes`. Save a `.java` file and Tomcat reloads |
| Frontend | Built `dist` served by Apache | `npm start` (ng serve) with live reload |
| Apache / AJP | Required | Not used |
| Maven | 3.0.5 | 3.9.x (same as GitLab CI) |
| sudo | Needed for every deploy | Only during the one-time setup |

### How to read the commands

- Commands are meant to be copied and pasted as they are.
- A line marked `# ← change ...` holds a value you must edit before running it.
- Text in `<angle brackets>` outside code blocks (e.g. `/home/<user>/...`) stands for your own value. Never type the `<` `>` into the shell: bash treats them as redirects and fails with `syntax error near unexpected token`.

### Versions used (match the project)

| Tool | Version | Why |
|---|---|---|
| Rocky Linux | 9.x | |
| JDK | OpenJDK 1.8 (`java-1.8.0-openjdk-devel`) | `pom.xml` targets 1.8; CI uses Temurin 8 |
| Maven | 3.9.16 | CI image is `maven:3.9-eclipse-temurin-8` |
| Tomcat | 9.0.122 | Tomcat 9 = `javax.*` API. **Do not use Tomcat 10+** (uses `jakarta.*`, the app will not start) |
| Node.js | 16.10.0 (via nvm) | Angular 13 needs Node 16 |

---

# Part 1: Fresh Setup

Do this part **once**. Go through the sections in order; most of them end with a check, so don't continue until the check passes. Section 16 starts everything for the first time and tests live reload.

## 1. Windows Preparation

### 1.1 Check your WSL distribution

In PowerShell:

```powershell
wsl -l -v
```

Example:

```text
  NAME             STATE     VERSION
* RockyLinux-9     Running   2
```

Use the name shown here wherever this guide says `RockyLinux-9` (in some setups it is `Rocky`).

### 1.2 Give WSL enough memory

Tomcat, `ng serve` and the VS Code Java extension together need about 5–6 GB. With the default or a small limit (e.g. `memory=3GB`), `ng serve` crashes with *JavaScript heap out of memory*. With too much, Windows itself runs out of memory and the whole PC slows down.

Check your RAM in *Settings → System → About* and choose `memory=`:

| PC RAM | `memory=` | Note |
|---|---|---|
| 12 GB | `6GB` | Close Eclipse and Docker Desktop on Windows while developing |
| 16 GB | `8GB` | |
| 32 GB or more | `12GB` | |

> Rule: about half of your PC's RAM, but not less than 6GB.

Edit `C:\Users\<you>\.wslconfig` (create it if it does not exist):

```ini
[wsl2]
memory=6GB
processors=4
swap=8GB

[experimental]
autoMemoryReclaim=gradual
```

| Setting | Why |
|---|---|
| `memory` | Upper limit for WSL (see the table above) |
| `swap` | A file on disk, used only when `memory` is full. Prevents crashes instead of slowing you down |
| `autoMemoryReclaim=gradual` | WSL keeps memory it has used as cache. This makes it give unused memory back to Windows over time |

Apply it. This stops **all** WSL distributions (and anything running in them):

```powershell
wsl --shutdown
wsl -d RockyLinux-9
```

Check inside Rocky:

```bash
free -g
```

The `total` column for `Mem:` should now show the value you set (e.g. 6, or 5 because of rounding).

> Changing `.wslconfig` later, while Tomcat / `ng serve` are running? Follow section 25 (*Restart WSL*).

### 1.3 Stop the Windows-side servers

The browser on Windows reaches WSL through `localhost`. If something on Windows already listens on the same port, the browser talks to **that** instead of WSL.

- In Eclipse on Windows: stop **"Anac Server"** (port 8080).
- Stop any `npm start` / `ng serve` running on Windows (port 4200).

Check in PowerShell that nothing is listening:

```powershell
netstat -ano | findstr ":8080 :4200"
```

### 1.4 Docker Desktop (only if it is installed)

Docker Desktop runs its own WSL distribution (`docker-desktop`) inside the **same** WSL memory limit, so it takes memory away from Tomcat and `ng serve`.

If you don't use Docker, stop it from starting at login: *Task Manager → Startup apps → Docker Desktop → Disable*. Or uninstall it in *Settings → Apps → Installed apps*.

### 1.5 Install the VS Code "WSL" extension

In VS Code on Windows, install **WSL** (`ms-vscode-remote.remote-wsl`) from the Extensions panel.

---

## 2. Base Packages

Open the Rocky Linux terminal and run:

```bash
sudo dnf update -y
sudo dnf install -y git wget tar unzip which openssh-clients fontconfig dejavu-sans-fonts
```

> `fontconfig` and a base font are needed by JasperReports (PDF output) and Apache POI (Excel output) on Linux. Without them, report screens fail with font / `FontConfiguration` errors.

---

## 3. Turn Off the Release Services (only if you followed the release guide)

The release guide installs the system Tomcat (`tomcat` service) on ports **8080 / 8005** and Apache on port 80. The development Tomcat needs the same ports, so turn the release services off:

```bash
sudo systemctl disable --now tomcat httpd
```

Check that the development ports are free (no output means free):

```bash
sudo ss -lntp | grep -E ':(8005|8080|8000|4200)\s'
```

> To go back to the release setup later: `sudo systemctl enable --now tomcat httpd` (stop the dev Tomcat first with `tc-stop`, see section 7).

---

## 4. Install Java 8 (JDK)

```bash
sudo dnf install -y java-1.8.0-openjdk-devel
```

Check that the **compiler** is there (a JRE alone is not enough):

```bash
javac -version
```

Expected:

```text
javac 1.8.0_xxx
```

---

## 5. Install Maven 3.9 (same as GitLab CI)

Maven is installed in your home directory, so the Maven 3.0.5 installed by the release guide (if any) is left untouched.

```bash
mkdir -p ~/dev && cd ~/dev
MVN_VER=3.9.16
wget https://dlcdn.apache.org/maven/maven-3/${MVN_VER}/binaries/apache-maven-${MVN_VER}-bin.tar.gz \
  || wget https://archive.apache.org/dist/maven/maven-3/${MVN_VER}/binaries/apache-maven-${MVN_VER}-bin.tar.gz
tar -xzf apache-maven-${MVN_VER}-bin.tar.gz
ln -sfn ~/dev/apache-maven-${MVN_VER} ~/dev/maven
```

> `dlcdn.apache.org` only keeps the newest release; when a newer Maven comes out, the command falls back to `archive.apache.org`, which keeps every version but can be slow. The download can take a few minutes.

Maven 3.9 uses HTTPS by default, so the `settings.xml` mirror and `MAVEN_OPTS=-Dhttps.protocols=TLSv1.2` from the release guide are **not** needed.

---

## 6. Install Tomcat 9 for Development

This Tomcat lives in your home directory and runs as your user, so you need no `sudo` to deploy, read logs or debug.

### 6.1 Download

```bash
cd ~/dev
TC_VER=9.0.122
wget https://dlcdn.apache.org/tomcat/tomcat-9/v${TC_VER}/bin/apache-tomcat-${TC_VER}.tar.gz \
  || wget https://archive.apache.org/dist/tomcat/tomcat-9/v${TC_VER}/bin/apache-tomcat-${TC_VER}.tar.gz
tar -xzf apache-tomcat-${TC_VER}.tar.gz
ln -sfn ~/dev/apache-tomcat-${TC_VER} ~/dev/tomcat
```

Optional: remove the sample applications (faster startup, less log noise):

```bash
rm -rf ~/dev/tomcat/webapps/{docs,examples,host-manager,manager}
```

### 6.2 Tomcat settings (`setenv.sh`)

```bash
cat > ~/dev/tomcat/bin/setenv.sh << 'EOF'
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
export CATALINA_PID="$CATALINA_BASE/temp/tomcat.pid"
export CATALINA_OPTS="-Xms256m -Xmx1536m -Dfile.encoding=UTF-8 -Djava.awt.headless=true"
EOF
chmod +x ~/dev/tomcat/bin/*.sh
```

| Option | Why |
|---|---|
| `-Xmx1536m` | Enough for the app, leaves memory for `ng serve` and VS Code |
| `-Dfile.encoding=UTF-8` | Japanese text in CSV / Excel / PDF output |
| `-Djava.awt.headless=true` | WSL sets `DISPLAY=:0` (WSLg). Without this, PDF/Excel generation tries to use the Windows display |
| `CATALINA_PID` | Lets `tc-stop` force-stop Tomcat if it hangs |

### 6.3 Log folder used by the application

`src/main/resources/log4j2.xml` writes the request/response JSON log to `/console/brycen_request.log` (on Windows this is `C:\console`). Create the folder and make it yours:

```bash
sudo mkdir -p /console
sudo chown $USER: /console
```

---

## 7. Environment Variables and Shortcuts

Add one block to the end of `~/.bashrc`:

```bash
cat >> ~/.bashrc << 'EOF'

# ---- kikancen development ----
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
export MAVEN_HOME=$HOME/dev/maven
export CATALINA_HOME=$HOME/dev/tomcat
export PATH=$JAVA_HOME/bin:$MAVEN_HOME/bin:$PATH
alias tc-start='$CATALINA_HOME/bin/catalina.sh jpda start'
alias tc-stop='$CATALINA_HOME/bin/catalina.sh stop 10 -force'
alias tc-log='tail -f $CATALINA_HOME/logs/catalina.out'
# ------------------------------
EOF
source ~/.bashrc
```

Check:

```bash
which mvn
mvn -version
```

Expected:

```text
/home/<user>/dev/maven/bin/mvn
Apache Maven 3.9.16 (...)
Java version: 1.8.0_xxx, vendor: Red Hat, Inc., runtime: /usr/lib/jvm/java-1.8.0-openjdk-...
```

> If `which mvn` shows `/usr/bin/mvn` or `/usr/share/maven/bin/mvn`, the block was not loaded. Open a new terminal or run `source ~/.bashrc` again.

| Shortcut | What it does |
|---|---|
| `tc-start` | Starts Tomcat in the background with the Java debugger port **8000** open |
| `tc-stop` | Stops Tomcat (force-kills it after 10 s) |
| `tc-log` | Follows `catalina.out`. `Ctrl+C` stops watching; Tomcat keeps running |

---

## 8. Install Node.js 16 (nvm)

Check first. Skip this section if both commands work and Node is `v16.10.0`:

```bash
nvm --version
node --version
```

Otherwise:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 16.10.0
nvm alias default 16.10.0
```

> The nvm install script already adds its lines to `~/.bashrc`. You don't need to add them again by hand.

Check:

```bash
node --version   # v16.10.0
npm --version
```

---

## 9. Git Settings in WSL

Git in WSL has its own settings, separate from Git for Windows.

```bash
git config --global user.name  "Your Name"
git config --global user.email "your.name@brycenmyanmar.com.mm"
git config --global core.autocrlf input
git config --global credential.helper "/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager.exe"
```

| Setting | Why |
|---|---|
| `core.autocrlf input` | The repository stores LF line endings. Windows Git converts to CRLF on checkout; in WSL we keep LF |
| `credential.helper` | Reuses the Git Credential Manager from Git for Windows, so you log in to GitLab the same way as on Windows and the password is not stored in plain text. (Requires Git for Windows installed in the default folder) |

---

## 10. Get the Source Code (in the Linux file system)

> **Important:** keep the project under your Linux home (`~/kikancen`), **not** under `/mnt/c/...`.
> Files on `/mnt/c` are many times slower for `npm`, Maven and the Java extension, and file watching does not work there (no live reload, no Tomcat auto-reload).

If you don't have the repository in WSL yet:

```bash
cd ~
git clone https://gitlab.sge-dev.com/anac_psy/purchase/kikancen
```

If `~/kikancen` already exists (e.g. from the release guide), update it:

```bash
cd ~/kikancen
git status
git fetch
```

Switch to your working branch (see the branch rules in the repository `README.md`):

```bash
BRANCH=feature/PSYREP2-1234-short-summary    # ← change to your Backlog key + summary
git switch develop
git pull
git switch -c "$BRANCH"      # new branch
# or
git switch "$BRANCH"         # existing branch from GitLab
```

### Moving unfinished work from your Windows checkout

`ng serve` and Tomcat in WSL only see `~/kikancen`. Work that is still in your Windows checkout (e.g. `C:\Users\<you>\Desktop\Anac-new\kikancen`) has to be moved over through GitLab:

1. On **Windows**, in the Windows checkout: commit your changes and push the branch.
2. In **WSL**:

   ```bash
   cd ~/kikancen
   git fetch
   git switch bugfix/PSYREP2-1234-short-summary     # ← change to the branch you pushed
   ```

3. From now on, edit only the WSL copy (section 17). Do not work on the same branch in both copies at the same time.

To open the WSL files in Windows Explorer:

```bash
explorer.exe .
```

(or type `\\wsl.localhost\RockyLinux-9\home\<user>\kikancen` in the Explorer address bar)

---

## 11. Local Configuration Files

All files are in `~/kikancen/src/main/resources/`.

| File | What to do |
|---|---|
| `db.properties` | MySQL connection for the backend. Check it points to the dev DB you use. If your Windows copy has local changes, copy it over (below). **Never commit changes to this file** |
| `system.properties` | Only needed for the `spvw00131` screen (MSSQL). Not in Git; copy it from Windows if you need that screen |
| `mode.properties` | Must stay `mode=local`. This enables CORS, so the Angular dev server (port 4200) can call Tomcat (port 8080) |
| `config.properties` | Batch server settings. The SSH key path is `~/.key/bcm-dev` (see below). Only needed for screens that run batch jobs |

### 11.1 Copy the files from Windows

`WIN_HOME` is your Windows user folder (e.g. `/mnt/c/Users/kghte`). If your Windows checkout is not in `Desktop\Anac-new\kikancen`, change the second line:

```bash
WIN_HOME=$(wslpath "$(cmd.exe /c 'echo %USERPROFILE%' 2>/dev/null | tr -d '\r')")
WIN_REPO="$WIN_HOME/Desktop/Anac-new/kikancen"
ls "$WIN_REPO/src/main/resources/"                 # must list the files, otherwise fix WIN_REPO
cp "$WIN_REPO/src/main/resources/db.properties"     ~/kikancen/src/main/resources/
cp "$WIN_REPO/src/main/resources/system.properties" ~/kikancen/src/main/resources/   # optional
```

### 11.2 Batch SSH key (optional)

Only if you test screens that call the batch server:

```bash
KEY_FILE=/mnt/d/path/to/bcm-dev-key    # ← change to where the key is on Windows (D:\path\to\... = /mnt/d/path/to/...)
mkdir -p ~/.key
cp "$KEY_FILE" ~/.key/bcm-dev
chmod 700 ~/.key && chmod 600 ~/.key/bcm-dev
```

### 11.3 Check that WSL can reach the database

This reads the DB server from the `url=` line in `db.properties` and tries to connect to it. Copy and paste it as it is:

```bash
DB_HOSTPORT=$(sed -nE 's#^url=jdbc:mysql://([^/?]+).*#\1#p' ~/kikancen/src/main/resources/db.properties)
echo "DB server: $DB_HOSTPORT"
timeout 3 bash -c "</dev/tcp/${DB_HOSTPORT%:*}/${DB_HOSTPORT##*:}" && echo "DB reachable" || echo "DB NOT reachable"
```

Expected:

```text
DB server: 10.95.101.6:3306
DB reachable
```

If Windows can reach the DB but WSL cannot (common with VPNs), see *Troubleshooting → DB not reachable from WSL* (section 27).

---

## 12. First Backend Build

```bash
cd ~/kikancen
mvn compile dependency:copy-dependencies -DincludeScope=runtime
```

This creates:

| Folder | Contents |
|---|---|
| `target/classes` | Compiled classes + everything from `src/main/resources` |
| `target/dependency` | Runtime jars from `pom.xml` (the same jars the WAR gets in `WEB-INF/lib`) |

The first run downloads all dependencies into `~/.m2` and takes several minutes. Expected end:

```text
[INFO] BUILD SUCCESS
```

> **Why not `mvn package`?** The WAR plugin copies the whole `src/main/webapp` folder, including `angular/node_modules` (hundreds of MB) once you have run `npm ci`. For development, Tomcat serves the files directly instead (section 13), so there is nothing to package.

---

## 13. Register kikancen in Tomcat

Create a context file. It tells Tomcat to serve `/kikancen` directly from your source folder, the way Eclipse's "Anac Server" does:

```bash
mkdir -p ~/dev/tomcat/conf/Catalina/localhost
cat > ~/dev/tomcat/conf/Catalina/localhost/kikancen.xml << EOF
<Context docBase="$HOME/kikancen/src/main/webapp" reloadable="true">
  <Resources>
    <PreResources className="org.apache.catalina.webresources.DirResourceSet"
                  base="$HOME/kikancen/target/classes" webAppMount="/WEB-INF/classes" />
    <PostResources className="org.apache.catalina.webresources.DirResourceSet"
                   base="$HOME/kikancen/target/dependency" webAppMount="/WEB-INF/lib" />
  </Resources>
</Context>
EOF
cat ~/dev/tomcat/conf/Catalina/localhost/kikancen.xml
```

Check that the printed file contains real paths like `/home/<user>/kikancen/...` (not `$HOME`).

| Part | Meaning |
|---|---|
| File name `kikancen.xml` | Context path `/kikancen` |
| `docBase` = `src/main/webapp` | `web.xml`, `healthz.txt` and the jars that are kept in Git under `WEB-INF/lib` (JasperReports, MySQL driver, …) |
| `target/classes` → `/WEB-INF/classes` | Your compiled code and resources |
| `target/dependency` → `/WEB-INF/lib` | Maven dependencies |
| `reloadable="true"` | When a class in `target/classes` changes, Tomcat reloads `/kikancen` automatically (about 10–20 s) |

---

## 14. Install the Frontend Packages

```bash
cd ~/kikancen/src/main/webapp/angular
node --version    # must be v16.10.0, otherwise: nvm use 16.10.0
npm ci
```

`npm ci` installs exactly the versions in `package-lock.json` into `node_modules`. Run it again whenever `package-lock.json` changes (section 19).

---

## 15. VS Code Setup (WSL)

### 15.1 Create the workspace file

Create a VS Code workspace file **next to** the repository (outside it, so nothing shows up in `git status`):

```bash
cat > ~/kikancen.code-workspace << 'EOF'
{
  "folders": [
    { "path": "kikancen" }
  ],
  "settings": {
    "java.configuration.runtimes": [
      { "name": "JavaSE-1.8", "path": "/usr/lib/jvm/java-1.8.0-openjdk", "default": true }
    ],
    "java.configuration.updateBuildConfiguration": "automatic",
    "java.compile.nullAnalysis.mode": "automatic",
    "java.debug.settings.hotCodeReplace": "auto",
    "files.watcherExclude": {
      "**/node_modules/**": true,
      "**/target/**": true,
      "**/dist/**": true
    },
    "search.exclude": {
      "**/node_modules": true,
      "**/target": true,
      "**/dist": true
    }
  },
  "launch": {
    "version": "0.2.0",
    "configurations": [
      {
        "type": "java",
        "request": "attach",
        "name": "Attach to Tomcat (8000)",
        "hostName": "localhost",
        "port": 8000
      },
      {
        "type": "chrome",
        "request": "launch",
        "name": "Angular in Chrome (Windows)",
        "url": "http://localhost:4200/kikancen/",
        "webRoot": "${workspaceFolder:kikancen}/src/main/webapp/angular",
        "browserLaunchLocation": "ui"
      }
    ]
  }
}
EOF
code ~/kikancen.code-workspace
```

The first time, VS Code installs its server inside WSL (takes a minute). The bottom-left corner shows **WSL: RockyLinux-9** when you are connected.

> Always open the project this way (or *File → Open Workspace from File…* inside a WSL window). If you open `\\wsl.localhost\...` from a normal Windows VS Code window, everything runs on Windows and is slow.

### 15.2 Extensions (install them "in WSL")

Extensions installed on Windows are **not** active in WSL. In the Extensions panel, click **Install in WSL: RockyLinux-9** for:

| Extension | ID | For |
|---|---|---|
| Extension Pack for Java | `vscjava.vscode-java-pack` | Java editing, Maven, debugger |
| Angular Language Service | `Angular.ng-template` | Angular templates |
| EditorConfig | `EditorConfig.EditorConfig` | Uses the project's `.editorconfig` (2 spaces, UTF-8, …) |
| ESLint | `dbaeumer.vscode-eslint` | TypeScript lint |
| XML | `redhat.vscode-xml` | `pom.xml`, `.jrxml` report layouts |

### 15.3 Java project import

1. Open any `.java` file. When VS Code asks to import the Java project, choose **Yes / Always**.
2. Wait until the status bar shows **Java: Ready**. The first import of this project takes several minutes.
3. Check with *Command Palette → Java: Configure Java Runtime*: the project should use **JavaSE-1.8**.

> The Java extension runs on its own bundled Java; you don't need to install another JDK. The `java.configuration.runtimes` setting makes it compile this project as Java 8.

---

## 16. First Run: Check That Everything Works

You need two Rocky Linux terminals (two Windows Terminal tabs, or two terminals in VS Code's WSL window).

### 16.1 Backend

Terminal 1:

```bash
tc-start
tc-log
```

Wait for:

```text
... org.apache.catalina.startup.Catalina.start Server startup in [xxxx] milliseconds
```

Press `Ctrl+C` to stop following the log (Tomcat keeps running), then check:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/kikancen/healthz.txt
```

Expected: `200`

### 16.2 Frontend

Terminal 2:

```bash
cd ~/kikancen/src/main/webapp/angular
npm start
```

Wait for `Compiled successfully`, then open in the **Windows** browser:

```text
http://localhost:4200/kikancen/
```

The dev build calls the backend at `http://localhost:8080` (`src/environments/environment.ts`), which is the WSL Tomcat from 16.1.

### 16.3 Log in and check CORS

Log in to the app. After the first API call, the Tomcat log must show that CORS local mode is on:

```bash
grep "ローカルモード" ~/dev/tomcat/logs/catalina.out
# CORSフィルタを起動：ローカルモード:true
```

### 16.4 Test live reload

1. In VS Code (the **WSL: RockyLinux-9** window), open the `.html` file of a screen you have open in the browser (`src/main/webapp/angular/src/app/components/...`).
2. Add some text and save. The `npm start` terminal rebuilds and the browser reloads with your text.
3. Undo the change and save again.

If the browser does not reload, see *Troubleshooting → Saved a file but the browser does not reload* (section 27).

### 16.5 Stop

```bash
tc-stop
# and Ctrl+C in the npm start terminal
```

**Setup is complete.** From now on, use Part 2.

---

# Part 2: Daily Development

## 17. The One Rule: Edit Only the WSL Copy

`ng serve` and Tomcat run from **`~/kikancen` in WSL**. They cannot see a Windows checkout such as `C:\Users\<you>\Desktop\Anac-new\kikancen`. If you edit files there:

- nothing reloads, and
- the browser keeps showing whatever branch is checked out in WSL, not the one you are editing.

So:

- Always open the project with `code ~/kikancen.code-workspace`. VS Code's bottom-left corner must show **WSL: RockyLinux-9**, and file paths start with `/home/<user>/kikancen`.
- Close the Windows checkout in VS Code / Eclipse while you work in WSL.
- To bring unfinished work over from Windows, see section 10 (*Moving unfinished work from your Windows checkout*).

---

## 18. Start the Day

**Step 1 – Open Rocky Linux.** Windows Terminal → *RockyLinux-9*, or in PowerShell:

```powershell
wsl -d RockyLinux-9
```

**Step 2 – Get your branch ready.** Starting a new task or need the latest code? Do section 19 first.

**Step 3 – Start the backend** (Terminal 1):

```bash
tc-start && tc-log
```

Wait for `Server startup in [xxxx] milliseconds`, then press `Ctrl+C`. Tomcat keeps running in the background.

**Step 4 – Start the frontend** (Terminal 2):

```bash
cd ~/kikancen/src/main/webapp/angular && npm start
```

Wait for `Compiled successfully`. Keep this terminal open: it shows compile errors while you work.

**Step 5 – Open the editor:**

```bash
code ~/kikancen.code-workspace
```

Check that the bottom-left corner shows **WSL: RockyLinux-9**.

**Step 6 – Open the app** in the Windows browser: `http://localhost:4200/kikancen/`

---

## 19. Branches: New Task, Update, Switch

Branch names and commit rules are in the repository `README.md`.

### Start a new task

```bash
cd ~/kikancen
BRANCH=feature/PSYREP2-1234-short-summary    # ← change to your Backlog key + summary
git switch develop
git pull
git switch -c "$BRANCH"
```

### Continue an existing branch

```bash
cd ~/kikancen
git fetch
git switch feature/PSYREP2-1234-short-summary    # ← change to the branch name
git pull
```

> `git status` always shows `db.properties` as modified. That is expected; `git switch` keeps your local version. If git refuses with *Your local changes to the following files would be overwritten*, run `git stash`, switch, then `git stash pop`.

### After a pull or switch

Check whether the dependencies changed:

```bash
git diff --name-only HEAD@{1} HEAD | grep -E 'package-lock.json|pom.xml'
```

| Output | What to do |
|---|---|
| Nothing | Nothing. `ng serve` rebuilds the frontend; VS Code compiles the Java files and Tomcat reloads |
| `package-lock.json` | `Ctrl+C` in the npm terminal, then `npm ci`, then `npm start` |
| `pom.xml` | See the `pom.xml` row in section 20 |

---

## 20. After a Change: What to Do

| You changed | What to do |
|---|---|
| Angular (`.ts`, `.html`, `.scss`) | Nothing. `ng serve` rebuilds and the browser reloads |
| `angular.json`, `package.json`, `tsconfig*.json` | `Ctrl+C` in the npm terminal, then `npm start` again |
| `package-lock.json` | `Ctrl+C` in the npm terminal, then `npm ci`, then `npm start` |
| Java (`.java`) | Save in VS Code. The Java extension compiles into `target/classes` and Tomcat reloads `/kikancen` in 10–20 s (the log shows `Reloading Context with name [/kikancen] is completed`). Without VS Code: `mvn -o -q compile` |
| Resources (`.properties`, `.jrxml`, other files in `src/main/resources`) | `mvn -o -q compile` (copies them to `target/classes`). Settings that are read only at startup need `tc-stop && tc-start` |
| `pom.xml` | `tc-stop`, then `rm -rf target/dependency && mvn compile dependency:copy-dependencies -DincludeScope=runtime`, then `tc-start` |
| `web.xml` | Nothing. Tomcat reloads automatically |
| Something is strange / after a big merge | `tc-stop`, then `mvn clean compile dependency:copy-dependencies -DincludeScope=runtime`, then `tc-start` |

Run the `mvn` commands in `~/kikancen`.

> Always stop Tomcat **before** `mvn clean`. `clean` deletes `target/classes` and `target/dependency`, which the running Tomcat is using.

> `-o` (offline) makes Maven skip checking the internet. It works once the first online build has filled `~/.m2`.

---

## 21. Debug the Backend (Java)

`tc-start` already opens the debug port **8000**.

1. In VS Code: **Run and Debug** (`Ctrl+Shift+D`) → select **"Attach to Tomcat (8000)"** → `F5`.
2. Set a breakpoint: click left of the line number in a `.java` file (a red dot appears).
3. Use the screen in the browser. VS Code stops at the breakpoint.
4. `F10` step over, `F11` step into, `F5` continue. Hover over variables or use the *Variables* panel.
5. `Shift+F5` disconnects the debugger. Tomcat keeps running.

While the debugger is attached, small changes inside a method body are applied as soon as you save (Hot Code Replace, set to `auto` in the workspace file) without waiting for a reload.

> If the automatic reload after every save disturbs your debugging, change `reloadable="true"` to `reloadable="false"` in `~/dev/tomcat/conf/Catalina/localhost/kikancen.xml` and restart Tomcat. Then you rely on Hot Code Replace and restart Tomcat yourself for bigger changes.

Check the logs for exceptions:

```bash
tc-log                                                        # follow (Ctrl+C to stop)
grep -n "Exception" ~/dev/tomcat/logs/catalina.out | tail     # last exceptions
tail -f /console/brycen_request.log                           # request/response JSON
```

---

## 22. Debug the Frontend (Angular)

**In the browser** (`F12`):

| Tab | Use it for |
|---|---|
| *Sources* | `Ctrl+P` → type the file name (e.g. `spmt01501ac.component.ts`) → click a line number to set a breakpoint. Source maps are on in dev mode; the files are under `webpack://` |
| *Network* | Filter `webapi` to see the API calls to `localhost:8080`: status, request and response |
| *Console* | Runtime errors |

**In VS Code:** **Run and Debug** → **"Angular in Chrome (Windows)"** → `F5` while `npm start` is running. Set breakpoints directly in the `.ts` files in VS Code.

You can attach the Java and the Chrome debugger at the same time.

**Compile errors** show in the `npm start` terminal and as an overlay in the browser. The browser does not reload until the error is fixed.

---

## 23. Stop for the Day

```bash
tc-stop          # backend
# Ctrl+C in the npm start terminal (frontend)
```

Optional, to give all of WSL's memory back to Windows (PowerShell):

```powershell
wsl --shutdown
```

---

## 24. Commit and Push

Use the commit message rules in the repository `README.md` (`refs #<redmine-no> : PSYREP2-<n> <title>`).

```bash
cd ~/kikancen
git status                                                   # db.properties / system.properties must NOT be added
git add src/main/webapp/angular/src/app/components/xxx/       # ← change to your files
git commit -m "refs #12345 : PSYREP2-1234 Short title"       # ← change
git push -u origin "$(git branch --show-current)"
```

> Don't use `git add .` or `git add -A`: they would add your local `db.properties`.

---

## 25. Restart WSL

Do this after changing `.wslconfig`, or when WSL behaves strangely.

1. Stop the apps in WSL:

   ```bash
   # Ctrl+C in the npm start terminal (and in tc-log if it is running), then:
   tc-stop
   ```

2. In PowerShell:

   ```powershell
   wsl --shutdown
   wsl -l -v
   ```

   Wait until `RockyLinux-9` shows **Stopped** (a few seconds). An open VS Code WSL window disconnects; that is expected.

3. Open Rocky again and check the memory:

   ```powershell
   wsl -d RockyLinux-9
   ```

   ```bash
   free -g     # Mem: total = the memory= value in .wslconfig
   ```

4. Start everything again: section 18, steps 3–6. In VS Code, click **Reload Window** when it asks.

---

## 26. Building Release Artifacts (only if you need them)

Release builds are made by GitLab CI from the `deploy/*` branches. You normally do **not** need to build them during development.

- **Angular build like CI** (checks that the production-style build still compiles):

  ```bash
  cd ~/kikancen/src/main/webapp/angular
  NODE_OPTIONS=--max-old-space-size=4096 npx ng build --configuration dev-server
  ```

- **WAR like CI:** build it from a **separate clean clone**. In your development copy the WAR would include `src/main/webapp/angular/node_modules`. Only committed changes are included:

  ```bash
  # builds the branch you currently have checked out in ~/kikancen
  git clone -b "$(git -C ~/kikancen branch --show-current)" ~/kikancen /tmp/kikancen-war
  cd /tmp/kikancen-war
  cp ~/kikancen/src/main/resources/db.properties src/main/resources/
  mvn -B -DskipTests clean package       # -> target/kikancen.war
  ```

---

## 27. Troubleshooting

### Saved a file but the browser does not reload

1. **You are editing a different copy** (most common). List the files changed in WSL in the last 10 minutes:

   ```bash
   find ~/kikancen/src/main/webapp/angular/src -type f -mmin -10
   ```

   If the file you just saved is not listed, you saved it somewhere else, e.g. in the Windows checkout. Open the project with `code ~/kikancen.code-workspace` (section 17).
2. **Compile error.** Look at the `npm start` terminal. The browser does not reload until the error is fixed.
3. **`ng serve` is not running** (e.g. after `wsl --shutdown`). Start it again (section 18, step 4).
4. **The project is under `/mnt/c`.** File watching does not work there (section 10).
5. **`ENOSPC` in the npm terminal.** See below.

### The browser shows a different branch than the one I'm working on

The browser shows what is checked out in `~/kikancen`:

```bash
git -C ~/kikancen branch --show-current
```

If that is not your branch, see section 17 and section 19.

### Browser shows the old Windows app or cannot connect

- Stop Eclipse's "Anac Server" / any Windows `ng serve` (section 1.3).
- Restart WSL (section 25), then start Tomcat and `npm start` again.

### Port already in use / Tomcat does not start

```bash
sudo ss -lntp | grep -E ':(8005|8080|8000)\s'
```

- `systemctl is-active tomcat` prints `active` → the release Tomcat is still running: `sudo systemctl disable --now tomcat`.
- Otherwise it is your own Tomcat from an earlier start → `tc-stop`.

### Windows becomes slow / low on memory

- `memory=` in `.wslconfig` is too high for your PC → see the table in section 1.2.
- Add `autoMemoryReclaim=gradual` (section 1.2).
- Close Eclipse and stop Docker Desktop on Windows (section 1.4).
- At the end of the day, `wsl --shutdown` gives all of WSL's memory back.

### `ng serve`: "JavaScript heap out of memory"

WSL has too little memory. Increase `memory=` in `.wslconfig` (section 1.2) and restart WSL (section 25). `npm start` already gives Node a 4 GB heap.

### `ENOSPC: System limit for number of file watchers reached`

```bash
echo "fs.inotify.max_user_watches=524288" | sudo tee /etc/sysctl.d/90-inotify.conf
sudo sysctl --system
```

### DB not reachable from WSL (but works from Windows)

This often happens with VPN connections. Turn on WSL mirrored networking in `C:\Users\<you>\.wslconfig`:

```ini
[wsl2]
memory=6GB
processors=4
swap=8GB
networkingMode=mirrored

[experimental]
autoMemoryReclaim=gradual
```

Then restart WSL (section 25) and test again (section 11.3).

### `/kikancen/webapi/...` returns 404, or Tomcat logs errors at startup

```bash
cat ~/dev/tomcat/conf/Catalina/localhost/kikancen.xml   # real paths?
ls ~/kikancen/target/classes/jp/co/brycen               # compiled?
ls ~/kikancen/target/dependency | wc -l                 # jars copied?
less ~/dev/tomcat/logs/catalina.out
less ~/dev/tomcat/logs/localhost.$(date +%F).log         # startup exceptions of the webapp
```

### Log4j error: "Unable to create file /console/brycen_request.log"

Create the folder (section 6.3).

### PDF / Excel output fails with font or X11 errors

```bash
sudo dnf install -y fontconfig dejavu-sans-fonts
grep headless ~/dev/tomcat/bin/setenv.sh      # must contain -Djava.awt.headless=true
```

Then `tc-stop && tc-start`.

### VS Code: red errors everywhere / Java never becomes "Ready"

- Check `java.configuration.runtimes` in `~/kikancen.code-workspace`.
- *Command Palette → Java: Clean Java Language Server Workspace* → Restart.

### `git status` shows `.project` modified after opening VS Code

The Java extension can rewrite the Eclipse `.project` file. Discard it:

```bash
git restore .project
```

### Useful log locations

| Log | Path |
|---|---|
| Tomcat + application (stdout) | `~/dev/tomcat/logs/catalina.out` |
| Webapp startup errors | `~/dev/tomcat/logs/localhost.<date>.log` |
| Request/response JSON log | `/console/brycen_request.log` |

---

## 28. Quick Command Summary

```bash
# ======== Part 1: one-time setup ========
# Windows: .wslconfig (memory=6GB ...), wsl --shutdown, VS Code "WSL" extension
sudo dnf install -y git wget tar unzip which openssh-clients fontconfig dejavu-sans-fonts java-1.8.0-openjdk-devel
sudo systemctl disable --now tomcat httpd          # only if the release guide was used
# Maven 3.9.16 -> ~/dev/maven, Tomcat 9.0.122 -> ~/dev/tomcat, setenv.sh, /console, ~/.bashrc block
# nvm + Node 16.10.0, git config, clone to ~/kikancen, db.properties
cd ~/kikancen && mvn compile dependency:copy-dependencies -DincludeScope=runtime
# ~/dev/tomcat/conf/Catalina/localhost/kikancen.xml
cd ~/kikancen/src/main/webapp/angular && npm ci
# ~/kikancen.code-workspace + extensions "in WSL"

# ======== Part 2: every day ========
tc-start && tc-log                                         # backend  -> http://localhost:8080/kikancen/
cd ~/kikancen/src/main/webapp/angular && npm start         # frontend -> http://localhost:4200/kikancen/
code ~/kikancen.code-workspace                             # editor (bottom-left: WSL: RockyLinux-9)
tc-stop                                                    # stop backend (Ctrl+C stops npm start)

# ---- branches ----
cd ~/kikancen && git switch develop && git pull && git switch -c feature/PSYREP2-1234-short-summary   # new task
cd ~/kikancen && git fetch && git switch feature/PSYREP2-1234-short-summary && git pull               # existing branch
git diff --name-only HEAD@{1} HEAD | grep -E 'package-lock.json|pom.xml'                              # deps changed?

# ---- after changes ----
cd ~/kikancen && mvn -o -q compile                         # resources / Java without VS Code
cd ~/kikancen && rm -rf target/dependency && mvn compile dependency:copy-dependencies -DincludeScope=runtime   # pom.xml changed (tc-stop first)
cd ~/kikancen/src/main/webapp/angular && npm ci            # package-lock.json changed (stop npm start first)

# ---- restart WSL (PowerShell, after stopping the apps) ----
wsl --shutdown
```

| What | Where |
|---|---|
| Source code (the only copy you edit) | `~/kikancen` |
| Tomcat | `~/dev/tomcat` (→ `~/dev/apache-tomcat-9.0.122`) |
| Maven | `~/dev/maven` (→ `~/dev/apache-maven-3.9.16`) |
| kikancen context file | `~/dev/tomcat/conf/Catalina/localhost/kikancen.xml` |
| VS Code workspace | `~/kikancen.code-workspace` |
| Maven repository | `~/.m2/repository` |
| WSL memory settings (Windows) | `C:\Users\<you>\.wslconfig` |

---

## Appendix A. Using Eclipse inside WSL (optional)

If you prefer Eclipse's "Servers" view over the `tc-start` workflow, you can run the Linux version of Eclipse inside WSL. Windows 11 shows Linux GUI apps through WSLg.

1. Install GUI libraries and Japanese fonts (for Japanese comments and messages):

   ```bash
   sudo dnf install -y gtk3 libXtst google-noto-sans-cjk-ttc-fonts
   ```

2. Download **Eclipse IDE for Enterprise Java and Web Developers**, **Linux x86_64**, from <https://www.eclipse.org/downloads/packages/>. Extract it to `~/dev` and start it:

   ```bash
   tar -xzf ~/Downloads/eclipse-jee-*-linux-gtk-x86_64.tar.gz -C ~/dev    # adjust the path to the downloaded file
   ~/dev/eclipse/eclipse &
   ```

   Use a workspace inside Linux, e.g. `~/eclipse-workspace`.

3. *Window → Preferences → Java → Installed JREs → Add…* → `/usr/lib/jvm/java-1.8.0-openjdk`, and in *Execution Environments* select it for **JavaSE-1.8**.
4. *File → Import → Maven → Existing Maven Projects* → `~/kikancen`.
5. *Servers* view → *New → Server → Apache → Tomcat v9.0* → installation directory `~/dev/tomcat`, JRE `java-1.8.0-openjdk` → add `kikancen`.

> Eclipse runs Tomcat with its own configuration, so the `kikancen.xml` context file from section 13 is not used. Do not run Eclipse's server and `tc-start` at the same time: they use the same ports.
