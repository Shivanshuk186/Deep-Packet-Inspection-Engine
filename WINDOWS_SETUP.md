# DPI Engine — Windows Setup Guide

This guide explains how to build and run the **DPI Engine** on Windows.

Multiple setup methods are provided depending on the preferred development environment:

1. **Visual Studio 2022** — Recommended for a native MSVC setup
2. **MinGW-w64 / GCC** — GCC-based Windows development
3. **WSL** — Linux environment inside Windows
4. **Visual Studio Code** — Lightweight editor-based workflow

Follow one of the setup options below.

---

# Option 1: Using Visual Studio

Visual Studio provides the native Microsoft C++ compiler and development environment.

## Step 1: Install Visual Studio

1. Download **Visual Studio 2022 Community**:

   [Visual Studio Downloads](https://visualstudio.microsoft.com/downloads/?utm_source=chatgpt.com)

2. Run the installer.

3. When asked which workloads to install, select:
   - ✅ **Desktop development with C++**

4. Click **Install** and wait for the installation to complete.

---

## Step 2: Open the Project

1. Open **Visual Studio 2022**.
2. Select **Open a local folder**.
3. Navigate to the `packet_analyzer` project directory.
4. Select the folder.
5. Allow Visual Studio to scan the project files.

---

## Step 3: Create a Build Configuration

1. In Solution Explorer, right-click the project folder.
2. Select **Add → New Item**.
3. Select **C++ File (.cpp)** and name it:

```text
CMakeSettings.json
```

4. Replace the contents with:

```json
{
  "configurations": [
    {
      "name": "x64-Release",
      "generator": "Ninja",
      "configurationType": "Release",
      "inheritEnvironments": ["msvc_x64_x64"],
      "buildRoot": "${projectDir}\\build\\${name}",
      "installRoot": "${projectDir}\\install\\${name}",
      "cmakeCommandArgs": "",
      "buildCommandArgs": "",
      "ctestCommandArgs": ""
    }
  ]
}
```

5. Save the file with `Ctrl+S`.

---

## Step 4: Build the Project

### Method A: Using CMake

If the repository contains a working `CMakeLists.txt`:

1. Visual Studio should detect the CMake configuration automatically.
2. Select **Build → Build All**.
3. The generated executable should appear in the configured build directory.

---

### Method B: Manual Build

The project can also be compiled directly using the MSVC compiler.

1. Open **View → Terminal** or press:

```text
Ctrl+`
```

2. Navigate to the project directory:

```cmd
cd packet_analyzer
```

3. Compile the multi-threaded DPI engine:

```cmd
cl /EHsc /std:c++17 /O2 /I include /Fe:dpi_engine.exe ^
    src\dpi_mt.cpp ^
    src\pcap_reader.cpp ^
    src\packet_parser.cpp ^
    src\sni_extractor.cpp ^
    src\types.cpp
```

4. If compilation succeeds, the executable will be generated as:

```text
dpi_engine.exe
```

---

## Step 5: Run the Program

```cmd
dpi_engine.exe test_dpi.pcap output.pcap
```

The engine reads:

```text
test_dpi.pcap
```

and produces:

```text
output.pcap
```

---

# Option 2: Using MinGW-w64

MinGW-w64 provides GCC-based C++ development on Windows.

## Step 1: Install MSYS2

1. Download MSYS2:

   [MSYS2](https://www.msys2.org/?utm_source=chatgpt.com)

2. Run the installer.

3. The default installation directory is:

```text
C:\msys64
```

4. Keep **Run MSYS2 now** enabled.

5. Click **Finish**.

6. In the MSYS2 terminal, update the package database:

```bash
pacman -Syu
```

7. If the terminal closes after the update, open:

**MSYS2 MINGW64**

from the Start Menu.

> Use the **MINGW64** environment rather than the regular MSYS2 shell for this GCC-based setup.

8. Install GCC and Make:

```bash
pacman -S mingw-w64-x86_64-gcc mingw-w64-x86_64-make
```

9. Enter `Y` when prompted.

---

## Step 2: Add MinGW to PATH

1. Press:

```text
Win + R
```

2. Enter:

```text
sysdm.cpl
```

3. Open the **Advanced** tab.
4. Select **Environment Variables**.
5. Under **System variables**, select **Path**.
6. Click **Edit**.
7. Click **New**.
8. Add:

```text
C:\msys64\mingw64\bin
```

9. Click **OK** on all dialogs.
10. Restart Windows so the updated PATH is available to new terminals.

---

## Step 3: Build the Project

1. Open **Command Prompt** or **PowerShell**.
2. Navigate to the project directory:

```cmd
cd C:\path\to\packet_analyzer
```

3. Compile:

```cmd
g++ -std=c++17 -O2 -I include -o dpi_engine.exe ^
    src/dpi_mt.cpp ^
    src/pcap_reader.cpp ^
    src/packet_parser.cpp ^
    src/sni_extractor.cpp ^
    src/types.cpp
```

4. Successful compilation produces:

```text
dpi_engine.exe
```

---

## Step 4: Run

```cmd
dpi_engine.exe test_dpi.pcap output.pcap
```

---

# Option 3: Using WSL

**Windows Subsystem for Linux (WSL)** provides a Linux environment inside Windows and is useful when a Linux-style development workflow is preferred.

## Step 1: Install WSL

1. Open **PowerShell as Administrator**.

2. Run:

```powershell
wsl --install
```

3. Restart Windows when prompted.

4. After installation, Ubuntu should be available.

5. Create a Linux username and password when prompted.

---

## Step 2: Install Build Tools

Open the Ubuntu terminal and run:

```bash
sudo apt update
sudo apt install -y build-essential g++
```

---

## Step 3: Navigate to the Project

Windows drives are accessible from WSL through `/mnt`.

For example, if the project is located at:

```text
C:\Users\YourName\Downloads\packet_analyzer
```

use:

```bash
cd /mnt/c/Users/YourName/Downloads/packet_analyzer
```

---

## Step 4: Build

```bash
g++ -std=c++17 -pthread -O2 -I include -o dpi_engine \
    src/dpi_mt.cpp \
    src/pcap_reader.cpp \
    src/packet_parser.cpp \
    src/sni_extractor.cpp \
    src/types.cpp
```

---

## Step 5: Run

```bash
./dpi_engine test_dpi.pcap output.pcap
```

---

# Option 4: Using Visual Studio Code

Visual Studio Code can be used as the development environment while using MinGW-w64 as the compiler.

## Step 1: Install VS Code

Download Visual Studio Code:

[Visual Studio Code](https://code.visualstudio.com/?utm_source=chatgpt.com)

Install it using the default options.

---

## Step 2: Install Extensions

1. Open VS Code.
2. Open the Extensions panel with:

```text
Ctrl+Shift+X
```

3. Install:

- **C/C++** — Microsoft
- **C/C++ Extension Pack** — Microsoft

---

## Step 3: Install a Compiler

Install MinGW-w64 by following **Option 2** above.

Make sure `g++` is available from the terminal:

```cmd
g++ --version
```

---

## Step 4: Open the Project

1. Select **File → Open Folder**.
2. Open the `packet_analyzer` project directory.

---

## Step 5: Create a Build Task

1. Press:

```text
Ctrl+Shift+P
```

2. Search for:

```text
Tasks: Configure Task
```

3. Select:

```text
Create tasks.json file from template
```

4. Select:

```text
Others
```

5. Replace the generated content with:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Build DPI Engine",
      "type": "shell",
      "command": "g++",
      "args": [
        "-std=c++17",
        "-O2",
        "-I",
        "include",
        "-o",
        "dpi_engine.exe",
        "src/dpi_mt.cpp",
        "src/pcap_reader.cpp",
        "src/packet_parser.cpp",
        "src/sni_extractor.cpp",
        "src/types.cpp"
      ],
      "group": {
        "kind": "build",
        "isDefault": true
      },
      "problemMatcher": ["$gcc"]
    }
  ]
}
```

6. Save the file.

---

## Step 6: Build

Press:

```text
Ctrl+Shift+B
```

VS Code will execute the configured build task.

---

## Step 7: Run

Open the VS Code terminal:

```text
Ctrl+`
```

Then run:

```cmd
.\dpi_engine.exe test_dpi.pcap output.pcap
```

---

# Troubleshooting

## Error: `'g++' is not recognized`

### Cause

MinGW-w64 is either:

- Not installed
- Not added to PATH
- Installed but the terminal has not been restarted

### Fix

1. Verify MinGW-w64 is installed.
2. Verify this directory exists:

```text
C:\msys64\mingw64\bin
```

3. Verify it is present in the system PATH.
4. Restart Windows.
5. Open a new terminal.
6. Test:

```cmd
g++ --version
```

---

# Error: `'cl' is not recognized`

### Cause

The MSVC compiler environment has not been initialized.

### Fix

1. Open:

```text
Developer Command Prompt for VS 2022
```

2. Navigate to the project directory.
3. Run the MSVC build command:

```cmd
cl /EHsc /std:c++17 /O2 /I include /Fe:dpi_engine.exe ^
    src\dpi_mt.cpp ^
    src\pcap_reader.cpp ^
    src\packet_parser.cpp ^
    src\sni_extractor.cpp ^
    src\types.cpp
```

---

# Error: Cannot find include file

### Cause

The compiler cannot locate the project's header files.

### Fix

Make sure the terminal is inside the project directory:

```cmd
cd C:\full\path\to\packet_analyzer
```

Then verify the `include` directory:

```cmd
dir include
```

The directory should contain the project's `.h` files.

Also make sure the compiler command contains:

```text
-I include
```

---

# Error: `undefined reference to std::thread`

### Cause

The threading support has not been enabled/linked correctly when using GCC/MinGW.

### Fix

Compile using the `-pthread` option:

```cmd
g++ -std=c++17 -pthread -O2 -I include -o dpi_engine.exe ...
```

---

# Program Runs but Crashes Immediately

### Possible Cause

The input PCAP file is missing or cannot be opened.

### Fix

Verify that:

```text
test_dpi.pcap
```

exists:

```cmd
dir test_dpi.pcap
```

If the test PCAP is missing, generate it with Python:

```cmd
python generate_test_pcap.py
```

Python must be installed and available through PATH.

---

# Error: Cannot Open Output File

### Possible Causes

- The output file is currently open in another program.
- The directory does not allow writing.
- The selected output filename is already being used.

### Fix

1. Close applications that may have `output.pcap` open.
2. Try a different filename:

```cmd
dpi_engine.exe test_dpi.pcap result.pcap
```

---

# Quick Reference

## Build Commands

| Environment              | Command                                                                                                                                              |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Visual Studio / MSVC** | `cl /EHsc /std:c++17 /O2 /I include /Fe:dpi_engine.exe src\dpi_mt.cpp src\pcap_reader.cpp src\packet_parser.cpp src\sni_extractor.cpp src\types.cpp` |
| **MinGW / GCC**          | `g++ -std=c++17 -O2 -I include -o dpi_engine.exe src/dpi_mt.cpp src/pcap_reader.cpp src/packet_parser.cpp src/sni_extractor.cpp src/types.cpp`       |
| **WSL / Linux**          | `g++ -std=c++17 -pthread -O2 -I include -o dpi_engine src/dpi_mt.cpp src/pcap_reader.cpp src/packet_parser.cpp src/sni_extractor.cpp src/types.cpp`  |

---

## Run Commands

### Basic

```cmd
dpi_engine.exe input.pcap output.pcap
```

### With Blocking Rules

```cmd
dpi_engine.exe input.pcap output.pcap --block-app YouTube --block-ip 192.168.1.50
```

### Custom Thread Configuration

```cmd
dpi_engine.exe input.pcap output.pcap --lbs 4 --fps 4
```

---

# Getting Wireshark Captures

The DPI Engine can also be tested against real PCAP captures.

## Step 1: Install Wireshark

Download Wireshark:

[Wireshark Downloads](https://www.wireshark.org/download.html?utm_source=chatgpt.com)

---

## Step 2: Capture Traffic

1. Open Wireshark.
2. Select the appropriate network interface, such as Wi-Fi or Ethernet.
3. Start a capture.
4. Generate some network traffic.
5. Stop the capture.

---

## Step 3: Save the Capture

Select:

```text
File → Save As
```

Save the capture in PCAP format, for example:

```text
my_capture.pcap
```

---

## Step 4: Process the Capture

Run:

```cmd
dpi_engine.exe my_capture.pcap filtered.pcap
```

The engine will process the captured packets and write the resulting traffic to:

```text
filtered.pcap
```

---

# Recommended Windows Setup

For a native Windows development environment:

```text
Windows
   │
   ├── Visual Studio 2022
   │       │
   │       └── MSVC / C++17
   │
   └── DPI Engine
           │
           ├── PCAP Reader
           ├── Packet Parser
           ├── SNI Extractor
           ├── Flow Tracking
           ├── Rule Manager
           └── Multi-threaded DPI Engine
```

For users already comfortable with Linux tooling, **WSL** provides an alternative environment using GCC and standard Linux build tools.

---

# Setup Flow at a Glance

```text
Choose Environment
       │
       ├── Visual Studio
       │
       ├── MinGW
       │
       ├── WSL
       │
       └── VS Code + MinGW
              │
              ▼
        Install Compiler
              │
              ▼
        Open Project
              │
              ▼
             Build
              │
              ▼
       Generate / Obtain PCAP
              │
              ▼
             Run
              │
              ▼
       output.pcap + Statistics
```

---

# Final Checklist

Before running the engine, verify:

- [ ] C++17 compiler is installed
- [ ] Project directory is correct
- [ ] `include/` directory is available
- [ ] Required `.cpp` files are present
- [ ] `test_dpi.pcap` exists
- [ ] `dpi_engine.exe` was generated successfully
- [ ] Input PCAP can be opened
- [ ] Output location is writable

Basic test:

```cmd
dpi_engine.exe test_dpi.pcap output.pcap
```

If the command completes successfully, the DPI engine has processed the PCAP and generated the output capture.
