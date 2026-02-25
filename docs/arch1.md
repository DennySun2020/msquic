# DebugPilot: Architecture Diagrams

These diagrams illustrate how the **DebugPilot** AI agent drives real debuggers (GDB, LLDB, CDB) to find bugs end-to-end.

---

## 1. End-to-End Architecture

Shows all 6 layers: Developer → DebugPilot Agent → TCP Client → TCP Server → Debugger Backend → Target Program.

```mermaid
%%{ init: { 'flowchart': { 'nodeSpacing': 70, 'rankSpacing': 70 } } }%%
flowchart TB
    subgraph DEV["👤 Developer"]
        D1["'Debug my_program.c — it crashes on the second call'"]
        D2["'Put a breakpoint at funcA(), print the callstack and check all variables'"]
        D1 ~~~ D2
    end

    subgraph PILOT["🤖 DebugPilot Agent (AI Reasoning Layer)"]
        direction TB
        P1["1. Read & Understand Source Code"]
        P2["2. Detect Platform & Toolchain (info → JSON report)"]
        P2b["3. Discover Repo Build System (repo-context → build system, docs, source dirs, test dirs)"]
        P2c["4. Locate Target Binary (find-binary → search build dirs)"]
        P3["5. Auto-Compile with Debug Symbols (compile → MSVC/GCC/Clang)"]
        P4["6. Launch TCP Debug Server (serve → background process) OR: serve --attach PID OR: serve --core dump"]
        P5["7. Set Strategic Breakpoints (cmd b function / file:line)"]
        P6["8. Start Execution (cmd start)"]
        P7["9. Analyze JSON Response (location, variables, stack)"]
        P8["10. Explain Findings to Developer"]
        P9["11. Decide Next Action (step/continue/inspect/evaluate)"]
        P10["12. Report Root Cause & Suggest Fix"]

        P1 --> P2 --> P2b --> P2c --> P3 --> P4 --> P5 --> P6 --> P7 --> P8
        P8 --> P9 --> P7
        P8 --> P10
    end

    subgraph CLIENT["📟 TCP Client - Skills"]
        C1["Open TCP socket → 127.0.0.1:5678"]
        C2["Send JSON command {'action':'step_over','args':''}"]
        C3["Receive JSON response"]
        C4["Close socket & exit"]
        C1 --> C2 --> C3 --> C4
    end

    subgraph SERVER["🖥️ TCP Debug Server"]
        S1["Accept connection"]
        S2["Parse JSON command"]
        S3["Dispatch to debugger backend"]
        S4["Build JSON response"]
        S5["Send response || close connection"]
        S1 --> S2 --> S3 --> S4 --> S5
        S5 -.->|"wait for next connection"| S1
    end

    subgraph BACKEND["⚙️ Debugger"]
        direction TB
        B1["GdbDebugger"]
        B2["LldbDebugger"]
        B3["CdbDebugger"]
    end

    subgraph TARGET["🎯 Target Program"]
        T1["Compiled Binary with Debug Symbol"]
    end

    DEV -->|"natural language"| PILOT
    PILOT -->|"terminal command: cmd --port 5678 step_over"| CLIENT
    CLIENT -->|"TCP JSON"| SERVER
    SERVER -->|"debugger commands: -exec-next/next/p"| BACKEND
    BACKEND -->|"control subprocess"| TARGET
    TARGET -->|"breakpoint hit/signal"| BACKEND
    BACKEND -->|"events"| SERVER
    SERVER -->|"JSON response"| CLIENT
    CLIENT -->|"stdout JSON"| PILOT
    PILOT -->|"plain language explanation"| DEV

    style DEV fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#1B5E20
    style PILOT fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#0D47A1
    style CLIENT fill:#FFF3E0,stroke:#E65100,stroke-width:2px,color:#BF360C
    style SERVER fill:#F3E5F5,stroke:#6A1B9A,stroke-width:2px,color:#4A148C
    style BACKEND fill:#FCE4EC,stroke:#B71C1C,stroke-width:2px,color:#880E4F
    style TARGET fill:#FFFDE7,stroke:#F57F17,stroke-width:2px,color:#E65100
```

---

## Key Design Decisions

| Decision | Rationale |
|---|---|
| **TCP server/client split** | Copilot agents can only run terminal commands — TCP lets a stateless CLI control a persistent debugger subprocess |
| **One connection per command** | Simple, no state leaks, easy error recovery — if a command fails the next one still works |
| **Polymorphic backends** | `GdbDebugger`, `LldbDebugger`, `CdbDebugger` share the same `cmd_*()` interface — the server doesn't know which is running |
| **Unified JSON schema** | Every command returns the same fields (`status`, `current_location`, `call_stack`, `local_variables`) regardless of backend |
| **Auto-compile** | Developer hands a `.c` file to the agent — the system compiles with debug symbols automatically before launching the debugger |
| **Platform auto-detection** | `ToolchainInfo` searches PATH, Visual Studio directories, Windows SDK, and xcrun to find the best tools for the current OS |
| **CDB preferred on Windows** | CDB natively reads PDB symbols from MSVC — no format mismatch, works with the same debug engine as Visual Studio and WinDbg |
| **Repo discovery** | `build-info`, `find-binary`, `repo-context` let the agent understand unfamiliar projects — detect build system, locate binaries, find docs |
| **Attach mode** | `--attach PID` enables debugging running processes — essential for deadlocks, hangs, and production issues |
| **Core dump mode** | `--core PATH` enables post-mortem crash analysis — no need to reproduce the bug |
| **Auto source path mapping** | Server auto-detects repo root via `.git` and adds it as a source path — source-level debugging works even for out-of-tree builds |
