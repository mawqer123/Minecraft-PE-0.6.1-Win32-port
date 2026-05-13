Drop the single DLL in this folder next to MinecraftPE.exe.

  opengl32.dll  - Mesa3D 17.3.7 (32-bit, self-contained), the last branch
                  of Mesa that pal1000 built explicitly with Windows 2000
                  / XP in mind. ~24 MB; software-rasterised OpenGL via
                  llvmpipe by default.

Do NOT replace C:\WINNT\system32\opengl32.dll - keep the Mesa one *next to
the game exe* so only Minecraft picks it up. Windows resolves DLLs from
the executable's directory first, so this scopes Mesa to just MinecraftPE.

Optional pre-launch env var (cmd: `set GALLIUM_DRIVER=softpipe`):
  - llvmpipe (default): JIT, faster, requires SSE2.
  - softpipe        : pure-C reference renderer, slowest, max compatibility.
