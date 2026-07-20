# axon-jvmti Phase 1 — developer setup

Machine-specific steps that cannot be automated from the repo alone.

## Automated in repo (after tools installed)

- Workspace stubs for placeholder crates
- `axon-jvmti-agent` cdylib scaffold
- `axon-jvmti attach` CLI scaffold
- `fixtures/jvm-test-app` Java harness
- `scripts/jvmti-phase1.ps1` integration script

## You must install manually

### 1. Rust 1.95 + MSVC build tools

```powershell
winget install Rustlang.Rustup
winget install Microsoft.VisualStudio.2022.BuildTools
# Select "Desktop development with C++" in the installer UI
```

Restart terminal, then:

```powershell
cd C:\Users\Shaurya\Documents\GitHub\Operation-NSP
rustup default 1.95.0
cargo check --workspace
```

### 2. JDK (not JRE)

You currently have **Java 8 JRE only** (`java` works, `javac` does not).

Install **Eclipse Temurin JDK 17** for daily dev:

```powershell
winget install EclipseAdoptium.Temurin.17.JDK
```

Set user environment variable:

```powershell
[System.Environment]::SetEnvironmentVariable(
  "JAVA_HOME",
  "C:\Program Files\Eclipse Adoptium\jdk-17.0.x-hotspot",
  "User"
)
```

Restart terminal; verify:

```powershell
java -version
javac -version
echo $env:JAVA_HOME
Test-Path "$env:JAVA_HOME\include\jni.h"
Test-Path "$env:JAVA_HOME\lib\jvm.lib"
```

### 3. Full §4.4 matrix JDKs (before day 10)

Install side-by-side: Temurin 8, 11, 17, 21 + IBM Semeru 17, 21.

Copy `fixtures/jdk-paths.ps1.example` → `fixtures/jdk-paths.ps1` with your paths.

### 4. GitHub CLI (for PRs from terminal)

```powershell
winget install GitHub.cli
gh auth login
```

Or open PRs in the browser.

### 5. Nelieo dev dashboard

Request access from Entropis — **daily shipping logs are mandatory**.

## Quick validation

```powershell
cargo build -p axon-jvmti-agent -p axon-jvmti
cd fixtures\jvm-test-app
.\build.ps1
# In another terminal:
cargo run -p axon-jvmti --bin attach -- --pid <PID>
```

## Draft PR

Branch: `feature/shaurya-jvmti-phase1`

After bootstrap commit is pushed:

https://github.com/Nelieo/Operation-NSP/compare/main...feature/shaurya-jvmti-phase1
