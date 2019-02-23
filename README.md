THIS IS AN UNSUPPORTED, POTENTIALLY UNSTABLE BUILD OF LUA. DO NOT EXPECT ANY FIXES OR CHANGES.  
NOTE THAT I HAVE NOT YET TESTED IF THE FULL FUNCTIONALITY OF THE LIBRARY ACTUALLY WORKS USING THIS. AS OF RIGHT NOW, I KNOW IT COMPILES AND CAN INSTATIATE A `lua_State` OBJECT. I WILL REMOVE THIS NOTICE ONCE I'VE VERIFIED THE REST OF THE FUNCTIONALITY.

# Overview
While attempting to integrate LuaJIT into an Unreal Engine 4.19.2 project, I ran into an issue with the custom built Lua library I had as it seemed to export the symbol  
`_vsnprintf`  
which was also defined in the Unreal Engine "default" third-party library `libcurl`. I'm familiar with libcurl on Linux, but have never checked it's exports. I do not know if this is a problem only in the context of Unreal, or with any integration of libcurl.

This repo contains a version of LuaJIT compiled *WITH THE `DLUAJIT_ENABLE_LUA52COMPAT` KEY*. That loads properly both when Hot-Reloading Unreal and Launching/Compiling from VS. This readme contains the solution I used and the steps I took to isolate and fix the problem.

## Solution
- Download the LuaJIT Windows source
- Open the Makefile and
    - Set the mode to `dynamic`
    - Enable the `DLUAJIT_ENABLE_LUA52COMPAT` flag
- Compile using `mingw32-make`
- Copy the `lua51.dll` file to another folder.
- Using the `x64 Native Tools Comand Prompt for VS 2017`, export the external symbols found in the DLL
    - `dumpbin /EXPORTS lua51.dll > lua51.exports`
- From the exports file, create a separate `.def` file pointing to the dll containing all the symbol references (dlltolib/lua51.def)
- Using the `x64 Native Tools Comand Prompt for VS 2017`, generate the lib and exp files based on the def file
    - `lib /def:lua51.def /out:lua51.lib`

You now have a DLL and a LIB that links to it!

- You may now copy the dll, lib and exp file to the location you want to install Lua to.
- Finish following the `Installation` instructions on the LuaJIT website, *for my own paths*, that was
    - Copy all files from `K:/Git Repos/luajit205-52-libdll/luajit205-52-dll/src` to `C:/LUA`
    - Copy the files from `K:/Git Repos/luajit205-52-libdll/luajit205-52-dll/src/jit` to `C:/LUA/lua/jit`

### Requirements
- mingw-w64 | their repos are a bit of a mess IMO, but I the one I downloaded was:
    - https://sourceforge.net/projects/mingw-w64/files/Toolchains%20targetting%20Win32/Personal%20Builds/mingw-builds/installer/mingw-w64-install.exe/download
- LuaJIT 2.0.5 source
    - https://luajit.org/download/LuaJIT-2.0.5.zip
- Visual Studio 2017
- Masochism


## Steps Taken
This was honestly one of the most convoluted things I have experienced as a programmer, so I'll go over in detail how it all fit together.

Originally, I wrote most of the Lua code I was planning on integrating to my Unreal project in a Linux VM. Everything worked great, but I realized I needed a custom build of Lua with the extra 5.2 compatibility due to a small shortcoming in the `luajson` package concerning the use of the `#` operator. Overriding this functionality using the `__len` metatable was one of the helpful extras included in LuaJIT using the `DLUAJIT_ENABLE_LUA52COMPAT` key. So I built it myself on Linux with the key enabled and everything worked, no problem.

Once I had a decent amount of code written and functional, I wanted to start seeing what it would look like in-game. So I downloaded Unreal, set up a project based on the Third-Person example template, and openend the Editor. So far so good.

Knowing that I would have to add Lua as a library dependency to Unreal very early in the process, I added the include paths and the additional library paths to Unreal's Build.cs file and attempted to recompile it. This is where things starting going downhill.

At first, it worked fine. I hit `Compile` in the Unreal Editor and it passed without trouble. The next day, when I tried to reopen the Editor, it no longer let me, telling me I should attempt to compile it from source. Okay...? I open VS2017 with my project and attempt to compile it.  
`lua51.lib(lua51.dll) : error LNK2005: _vsnprintf already defined in libcurl_a.lib(cryptlib.obj)`

luigihands

In general, this was already a bad sign. These were two librarys who's source I had very little control over and they had confilcting symbols. As one of the stackoverflow posts I read put it:
- First step: swear loudly

`Note: I'm still completely clueless as to WHY the Unreal Editor's Hot-Reload with "Compile" worked, but building it from scrath did not. I can only assume that the order or manner in which the LIB/DLLs are parsed and added is different in the two cases, but I never found any documentation reguarding that. Which is on par with Unreal having terrible documentation as a whole.`

I briefly looked in to the possibility of removing libcurl from Unreal's build. While it seems like it could be done, it would involve editing the engine source code. As someone completely new to Unreal, I was hesitant to make changes that could have an impact beyond my comprehension, and I had no idea how deep the dependency rabbit-hole went. So instead, I took a long hard look at my LuaJIT files.

The first step was to make sure that _vsnprintf was indeed actually defined in my lib. A quick search had me using `dumpbin.exe /ALL lua51.lib` to quickly list the symbols. Indeed, it was there. Out of curiosity, I then went and performed the equivalent on the static library file I had been linking with on my Linux machine (using `nm`), and it was *not* there. Now at that time, I still wasn't completely sure what `vsnprintf` was suppose to mean, but this make be look into it in more detail. The discrepancy in the static libraries may have made sense if it was a Windows-only kind of wrapper or something. Nope, it's just some print helper function. That leads to the first big problem: `clang`.

Since there was a discrepancy between the Linux and Windows versions of the library, I then starting looking through the build proceedure I had used for each. On Linux, I had used `GCC 4.5.8 -> make`, and on windows, as suggested by the LuaJIT website, I used the `msvcbuild.bat` file.

DO NOT USE THE `MSVCBUILD` METHOD, is all i can say. I hadn't paid attention to it in the past, unfortunately, but it *does not take any of the makefile parameters/flags into account* (most likely because it simply cannot due to clang limitations or something). This brought up another issue, in that it meant my addition of `DLUAJIT_ENABLE_LUA52COMPAT` was not actually sticking on Windows anyway. So I went ahead and downloaded one of LuaJIT's suggested Windows compilers, mingw (and then realized I should probably be using mingw-w64, downloaded the wrong one, and then finally got it right) and recompiled LuaJIT on Windows using `mingw32-make`.

Cool, it ran the Makefile, used the right flags, and generated a DLL and the EXE... *But no lib*. Unreal *requires* you to link libraries at compile time, and there seems to be very little/highly unstable support for integrating DLLs indiviually. Apparently because of "security concerns". Bullshit. Okay, that aside for now, I take a look at the DLL's symbols. No `_vsnprintf` anywhere. Awesome. Now I just need the lib.

I look into the Makefile for LuaJIT again and notice that there are three different "buildmodes": Static, Dynamic and Mixed. They note that Dynamic will only produce the DLL, static will only produce the LIB and mixed *isn't supported on Windows*. Fantastic. And also I call bullshit since that's exactly what the output of `msvcbuild.bat` is, unless I'm much mistaken.

I actually briefly wondered if the static library generated by using the static buildmode could be used to link to the DLL. BUT WAIT! Static building on Windows with LuaJIT 2.0.5 is broken! So even if that would be the case, I can't generate it unless maybe I change versions of LuaJIT. While that *could* have been a solution, I had no guarentee the next beta-release version had it fixed yet (though a note in a PR for LuaJIT said it was), and I didn't want to use an older version either. Instead, I started looking into generating my own lib based off the DLL that dynamic mode had produced.

By this point I had been using mingw-w64 for all my build utility needs, so I tried looking into the DLL using `nm`, as I would on Linux. This showed me absolutely nothing. Instead, I then went back to the VS2017 tools and used `dumpbin`, which showed me exactly what I was hoping for. I'm not sure if I was missing a parameter with `nm`, but the exactly same command had showed me what I wanted on Linux, so I'm assuming the DLL I produced using mingw isn't compatible with it for some reason.

Using the output of dumpbin, I generated a `.def` file that contained all the exported symbols present in the DLL and stupidly attempted to use mingw's `dlltool` to forge the lib. Of course, since mingw tools seem to be incompatible with the DLL they produced, that failed miserably, so I instead went back one last time to the VS2017 tools and generated the lib with the lib command `lib /def:lua51.def /out:lua51.lib`. This generated a LIB file and an EXP file. Checking the LIB file with `dumpbin`, it seemed to show all the symbols I needed. Phew.

Then all that was left was replacing my current Lua install with the one I had regenerated with mingw and paste my DLL, LIB and EXP files in the directories they belonged in.
