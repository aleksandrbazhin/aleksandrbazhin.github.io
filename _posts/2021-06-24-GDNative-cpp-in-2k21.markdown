---
layout: post
title:  "How to organize and build your GDNative project with CMake and MinGW in 2k21"
date:   2021-06-24 11:43:24 +0300
categories: godot
---

This a tutorial / memo for myself how to setup GDNative and godot-cpp with CMake and MinGW in June 2021. When you just have to follow instructions, this setup is easy. But for now all the tutorial repos are out of date, confusing ```godot-headers``` with ```godot_headers``` and something like that.

I prefer CMake to SCons because it's more widespread (In reality I hate all of the C++ build tools but the build process is even less fun without them). More widespread means that more of the external libraries use it.

## Step 0: setup your tools

You will need to download and install before the start (all x64 versions):

1. [Git](https://git-scm.com/downloads)
2. [Python](https://www.python.org/downloads/)
3. [MinGW](http://mingw-w64.org/doku.php/download/mingw-builds)
4. [CMake](https://cmake.org/download/)
5. [Godot](https://godotengine.org/download)

All of the above should be searchable in your path. I believe in 2021 all the installers add themselves to the PATH, except for Godot, which is distributed as a single binary, which is certainly cool. But we still have to add it to the PATH variable so that Godot_v3.3.2-stable_win64.exe would be callable from anywhere. 

Also as a terminal emulator on Windows I use [Cmder](https://cmder.net/) and I like it.

## Step 1: organize your project

To make life more interesting let's go with some unorthodox project structure:

<pre>
<b>/export</b>                    //game binaries built with godot
<b>/lib</b>                       //dependencies for our gdnative library
<b>/lib/godot-cpp</b>             //you always need it
<b>/lib/cpp_dependency1</b>
<b>/lib/cpp_dependency2</b>
<b>/native/</b>                   //our c++ libraries
<b>/native/world</b>              //our world generation gdnative library
<b>/godot</b>                     //godot project sources
<b>/godot/native</b>              //gdnative godot libraries
<b>/godot/native/world</b>        //gdnative files (.gdns and .gdnlib) for pur library
<b>/godot/native/bin</b>          //compiled gdnative libraries and dependencies (.dll)
<b>build.py</b>                   //simple build script for godot project
<b>.gitignore</b>
</pre>

## Step 2: setup git repository

We will use git to copy latest godot-cpp and godot-headers 
* Init repo 

    ```bash
git init
    ```
* Create file named "**.gitgnore**" for git not to track the files generated during build process or by Godot editor
<details>
<summary>sample .gitgnore:</summary>
<pre><code># VSCode specific
*.vscode/*

# C++
*.o
*.a
*.dll
lib/*
native/*/build/*

# Godot-specific ignores
logs/
.import/
godot/.import/
godot/export.cfg
godot/export_presets.cfg
*.translation
godot/addons/*
godot/logs/*
godot/native/bin/*
*.lnk

# Export directory
export/*
</code></pre>
</details>


* Add and commit .gitgnore to git:

    ```bash
git add . && git commit -m "added gitgnore"
    ```

## Step 3: build godot-cpp dependency 

* Clone and build godot-cpp dependency to our **lib/** subdirectory
    * ```bash
cd lib
    ```
    * ```bash
git submodule add https://github.com/godotengine/godot-cpp
    ```
    * ```bash
git submodule update --init --recursive
    ```
    this adds [https://github.com/godotengine/gpdot-headers](https://github.com/godotengine/gpdot-headers) dependency
    * ```bash
mkdir godot-cpp\build\win64\debug && cd godot-cpp\build\win64\debug
    ```
    * ```bash
cmake ../../.. -G "MinGW Makefiles" -DCMAKE_CXX_COMPILER=g++ -DBITS=64
    ```
    * ```bash
cmake --build . -- -j4
    ```
* {:.continue_list}Test GDNative with a test project provided with ```godot-cpp```
    * Navigate to **godot-cpp/test/**
    ```bash
cd ../../../test
    ```
    * {:.continue_list} Create a **bin/win64/** directory for test library windows .dll 
    ```bash
mkdir bin\win64
    ```    
    * {:.continue_list} Build the object file from provided **init.cpp**
    ```bash
g++ -fPIC -o src/init.o -c src/init.cpp -g -O3 -std=c++14 -I../include -I../include/core -I../build/win64/debug/include/gen -I../godot-headers
    ```
    First, notice that tutorials sometimes use godot_headers instead of godot-headers since that how it used to be named. Second, **-I../build/win64/debug/include/gen** is used instead of **-I../include/gen** the second one is where the bindings are generated if you use SCons and what is mentioned in all the tutorials, but CMake in godot-cpp repo generates bindings in the first path. 
    **Possible errors:**
        * {:.possible_errors} ```init.cpp:1:10: fatal error: Godot.hpp: No such file or directory``` - Sorry, I don’t remember what it was. Must be the wrong path to the lib\godot-cpp\include\core\Godot.hpp passed here -I../include/core
        * {:.possible_errors} ```gdnative_api_struct.gen.h not found``` - godot-headers repo is not found in godot-cpp. It was first named godot_headers, not godot-headers so older manuals have this error
        * {:.possible_errors} ```...Reference.hpp not found...``` - the Godot bindings headers have not been found in **../build/win64/release_shared/include/gen** These bindings are autogenerated by CMake when you build godot-cpp on step 3. However if you use SCons for building it, they are generated in the **../include/gen** and all manuals mention this directory
    * {:.continue_list}Link it with our gdnative and c++ runtime libraries 
    ```bash
g++ -o bin/win64/libgdexample.dll -shared src/init.o -static -static-libgcc -static-libstdc++ -L../bin/ -lgodot-cpp.windows.debug.64
    ```
    **Possible errors:**
        * {:.possible_errors} ```Error: Can’t open dynamic library platform/windows/os.windows.cpp``` - **libgodot-cpp.windows.debug.64.dll** is not found in the **../bin/** directory
    * {:.continue_list}Run the test godot project 
    ```bash
Godot_v3.3.2-stable_win64.exe -s script.gd
    ```
    (Godot_v3.3.2-stable_win64.exe should be in the PATH or provide full path to it). 
    If everything is fine you will see that Godot ran script.gd and saw our GDNative library: 
    ```
Native Script [NativeScript:1183]
Library [GDNativeLibrary:1181]
Reference [Reference:1185]
Reference name SimpleClass
Reference value 0
Call method 1
    ```

        **Possible errors:**

        - {:.possible_errors} ```ERROR: Can't open dynamic library: .../libgdexample.dll, error: Error 126``` - Godot can’t find your compiled dll. Check the path in **gdexample.gnlib** in **Windows.64=** parameter of section **[entry]** and if it contains the correct path to your library

<!-- * Step 5.d:
    * ```Error: Can’t open dynamic library platform/windows/os.windows.cpp``` - .dlls from the step 5.a.ii are not found in the /test/ directory -->


## Step 4: Compile godot-cpp as a release library
Debug library is cool for debugging, but it’s unoptimised and has a lot of bloat (it’s ~140Mb large). Who needs debugging anyway? (jokes aside, you probably wouldn’t want to debug Godot itself).
From now on I will skip most of navigation (especially back to the project root). But now you should go to the root by ```cd ..\..\.. ```
-  Create release directory in build
```bash
mkdir lib\godot-cpp\build\win64\release && cd lib\godot-cpp\build\win64\release
```
- Modify the CMakeLists.txt in **lib/godot-cpp/**. Disable debug information in the builds so the Godot library build would be not 140Mb, but 5Mb. On line 110 replace:
```cmake
set(GODOT_COMPILE_FLAGS "-fPIC -g -Wwrite-strings")
```
with:
```cmake
set(GODOT_COMPILE_FLAGS "-fPIC -Wwrite-strings")
```
- prepare build for the release version
```bash
cmake ../../.. -G "MinGW Makefiles" -DCMAKE_CXX_COMPILER=g++ -DBITS=64 -DCMAKE_BUILD_TYPE=Release
```
- build it
```bash
cmake --build . -- -j4
```

Cool, we now have the release version of godot-cpp library, which is around 5Mb large. The libraries with .a extension are supposed to be statically linked with the executable or library using them, i.e. copied inside them. Our resulting library should be around 7Mb.

## Step 5: Compile godot-cpp as a shared .dll library 
*This step is optional and is not the supposed way GDNative is used.* There is also a possibility to build a godot-cpp library that can be referenced from the outside library as a .dll, dynamic link library. Why would you do it? If you want to make several modules, statically linking with godot-cpp would mean it’s copied into every library you build. On the downside, you’ll have to distribute not a single gdnative .dll, but several .dll files alongside with it and default Godot exporter won't copy them for you. But you still are distributing your .dll separately from your executable, so why not make even more of a mess?

Another possible scenario for distributing .dlls separately is when some library dependency explicitly forbids embedding it into your binary. This is not the case for Godot obviously, but is, for example, the case for Qt.

- Modify the CMakeLists.txt in **lib/godot-cpp/**. On line 172 replace:
```
add_library(${PROJECT_NAME} 
```
with:
```
add_library(${PROJECT_NAME} SHARED
``` 
- prepare build of godot-cpp release library
```
cmake ../../.. -G "MinGW Makefiles" -DCMAKE_CXX_COMPILER=g++ -DBITS=64 -DCMAKE_BUILD_TYPE=Release
```
- and build it
```
cmake --build . -- -j4
```
At this stage in lib/godot-cpp/bin there should be 3 libraries, 2 for static linking, **libgodot-cpp.windows.debug.64.a**, **libgodot-cpp.windows.debug.64.a** and **libgodot-cpp.windows.debug.64.dll** for dynamic.
- {:.continue_list} Test GDNative as a dynamic library with a provided test project
    - Copy all .dll dependencies to the **godot-cpp/test/bin/win64** folder:
        - our godot-cpp library **libgodot-cpp.windows.release.64.dll**
        - gcc runtime dlls for mingw32 from mingw32 **bin/** folder (my mingw installation is in C:\Program Files\mingw-w64\x86_64-7.3.0-posix-seh-rt_v5-rev0\mingw64)
            - **libstdc++-6.dll**
            - **libgcc_s_seh-1.dll**
            - **libwinpthread-1.dll** (to use threads possibly)
    - Navigate to **godot-cpp/test** by running ```cd ../../../test```
    - Compile init.cpp
    ```bash
g++ -fPIC -o src/init.o -c src/init.cpp -O3 -std=c++14 -I../include -I../include/core -I../build/win64/release/include/gen -I../godot-headers
    ```
    - {:.continue_list} Build a GDNative .dll by linking it with our godot-cpp library 
    ```bash
g++ -o bin/win64/libgdexample.dll -shared src/init.o -Lbin/win64 -lgodot-cpp.windows.release.64
    ```
    **Possible errors:**
        - {:.possible_errors} ```Error: Can’t open dynamic library platform/windows/os.windows.cpp``` - one of the .dll libraries, which you should copy, is not found in the test/ directory
    - {:.continue_list}Run the tes Godot project
    ```bash
Godot_v3.3.2-stable_win64.exe -s script.gd
    ```
    **Possible errors:**    
        - {:.possible_errors} ```ERROR: Can't open dynamic library: C:/Users/user/Documents/sailing_west/lib/godot-cpp/test/lib/libgdexample.dll, error: Error 126 …``` -
Either Godot can not find your compiled dll. Check the path in gdexample.gnlib in the section [entry] under parameter Windows.64= and if that path contains your compiled .dll
Or one of the dll dependencies mentioned in 5.a is absent. They should be in the same directory with your compiled library .dll.


Now our Godot library is only 72kb in size itself with additional dlls of around 7Mb of which 5 is godot-cpp.



## Step 6: Create your own Godot & GDNative project
Now we can move on to setting up our first library project. Let it be named "world" as I was going to generate the world with it.
- *If you went with step 5*, copy your mingw standard library dlls (**libstdc++-6.dll**, **libgcc_s_seh-1.dll**, **libwinpthread-1.dll**) into **/lib/std** - just to have them among dependencies explicitly
- Navigate to the project root and ```cd native && mkdir world\src && cd world```
- Copy **init.cpp** from **lib/godot-cpp/test/src** into **native/world/src**
- Create **CMakeLists.txt**. I came up with the following one, which builds everything for me

```cmake
cmake_minimum_required(VERSION 3.16)
project(world VERSION 0.1.0)
 
set(CMAKE_CXX_STANDARD 17)
 
if(CMAKE_BUILD_TYPE STREQUAL "")
    set(CMAKE_BUILD_TYPE Release)
endif()
  
string(TOLOWER "${CMAKE_SYSTEM_NAME}" SYSTEM_NAME)
string(TOLOWER "${CMAKE_BUILD_TYPE}" BUILD_TYPE)
set(MY_SYSTEM_OUTPUT_PATH "win64")
 
set(BUILD_PATH ${CMAKE_SOURCE_DIR}/../../godot/native/bin/${MY_SYSTEM_OUTPUT_PATH})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${BUILD_PATH}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${BUILD_PATH}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${BUILD_PATH}")
SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG "${BUILD_PATH}")
SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE "${BUILD_PATH}")
SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY_DEBUG "${BUILD_PATH}")
SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY_RELEASE "${BUILD_PATH}")
SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_DEBUG "${BUILD_PATH}")
SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE "${BUILD_PATH}")
 
set(MY_LINKER_FLAGS "-Wl,-R,'$$ORIGIN'")
set(MY_COMPILER_FLAGS "-fPIC -Wwrite-strings")
set(LINK_DIRS )
 
add_library(${PROJECT_NAME} SHARED src/init.cpp)
 
file(REMOVE_RECURSE ${BUILD_PATH})
file(MAKE_DIRECTORY  ${BUILD_PATH})
 
if(CMAKE_BUILD_TYPE MATCHES Release)
    add_definitions(-DNDEBUG)
    add_custom_command(TARGET ${PROJECT_NAME} PRE_LINK
        COMMAND ${CMAKE_COMMAND} -E copy_directory
        ${CMAKE_SOURCE_DIR}/../../lib/std/ ${BUILD_PATH})
    add_custom_command(TARGET ${PROJECT_NAME} PRE_LINK
        COMMAND ${CMAKE_COMMAND} -E copy
        ${CMAKE_SOURCE_DIR}/../../lib/godot-cpp/bin/libgodot-cpp.${SYSTEM_NAME}.${BUILD_TYPE}.64.dll ${BUILD_PATH})
    set(LINK_DIRS "${BUILD_PATH}")
else()
    add_definitions(-D_DEBUG)
    set(COMPILER_FLAGS "-g")
    set(LINKER_FLAGS "-static -static-libgcc -static-libstdc++")
    set(LINK_DIRS ${CMAKE_SOURCE_DIR}/../../lib/godot-cpp/bin/)
endif(CMAKE_BUILD_TYPE MATCHES Release)
 
target_include_directories(${PROJECT_NAME} PUBLIC
    ../../lib/godot-cpp/godot-headers
    ../../lib/godot-cpp/include
    ../../lib/godot-cpp/include/core
    ../../lib/godot-cpp/build/win64/release/include/gen
)
 
set_property(TARGET ${PROJECT_NAME} APPEND_STRING PROPERTY COMPILE_FLAGS ${COMPILER_FLAGS})
set_property(TARGET ${PROJECT_NAME} APPEND_STRING PROPERTY LINK_FLAGS ${LINKER_FLAGS})
 
target_link_directories(${PROJECT_NAME} PRIVATE ${LINK_DIRS})
target_link_libraries(${PROJECT_NAME} PRIVATE "godot-cpp.${SYSTEM_NAME}.${BUILD_TYPE}.64")
 ```

- {:.continue_list} Now you can build
    - debug statically linked “self-contained” library when you want to debug everything and don't care about it being 140Mb
        ```bash
mkdir build\win64\debug && cd build\win64\debug
        ```
        ```bash
cmake ../../.. -G "MinGW Makefiles" -DCMAKE_CXX_COMPILER=g++ -DCMAKE_BUILD_TYPE=Debug -DCMAKE_SYSTEM_NAME=Windows -DBUILD_SHARED=False
        ```
        ```bash
cmake --build . -- -j4
        ```
    - OR release statically linked library without debug bloat included into godot-cpp
        ```bash
mkdir build\win64\release && cd build\win64\release
        ```
        ```bash
cmake ../../.. -G "MinGW Makefiles" -DCMAKE_CXX_COMPILER=g++ -DCMAKE_BUILD_TYPE=Release -DCMAKE_SYSTEM_NAME=Windows -DBUILD_SHARED=False
        ```
        ```bash
cmake --build . -- -j4
        ```
    - OR release dynamically linked library with .dll dependencies distributed separately
        ```bash
mkdir build\win64\release && cd build\win64\release
        ```
        ```bash
cmake ../../.. -G "MinGW Makefiles" -DCMAKE_CXX_COMPILER=g++ -DCMAKE_BUILD_TYPE=Release -DCMAKE_SYSTEM_NAME=Windows -DBUILD_SHARED=True
        ```
        ```bash
cmake --build . -- -j4
        ```
            
    **Possible errors:**
    - {:.possible_errors} ```cannot open output file ... \godot\native\lib\win64\libworld.dll: Permission denied``` - When rewriting the library dll, if the Godot editor is open it can block the writing of your library .dll. It won't happen when you test by running **script.gd**, but it will if you working further, using Godot editor.
- {:.continue_list} Test with the same Godot test project
    - copy the sample project files (**script.gd**, **gdexample.gnlib**, **gdexample.gdns** and **project.godot**) to the godot/ 
    - replace **Windows.64="res://bin/win64/libgdexample.dll"** to **Windows.64="res://native/bin/win64/libworld.dll"** in gdexample.gnlib in the section **[entry]**.
    - navigate to your godot (```cd ..\..\..\..\..\godot```) run the same 
    ```
Godot_v3.3.2-stable_win64.exe -s script.gd
    ```


## Step 7: Prepare and export godot project
Finally we have to test that our game would work after exported from Godot. Godot built-in exporter will copy GDNative libraries' dlls it depends on while exporting, but won’t copy our additional dependencies, if used them as shared libraries. 
- Open the project we have at **godot/** with Godot editor. 
- Create a scene
- Create a script that that scene would use
- Copy the contents of **_initialize()** in **script.gd** to our new script **_ready()**
- Save and run the project, selecting the default scene
- Export project in the Godot editor UI to the **export/win64/** with the name, say **Native_project.exe**
- *(if you used shared while build)* To copy all the additional dlls, you can use Windows **robocopy** utility, like this.
```bash
robocopy godot/native/bin/win64 export/win64 *.dll
```

Now you can run your game and test 
- ```cd export/win64```
- ```Native_project.exe```
At this point you should see your project running


## Final words

That's more of a memo for myself, not to forget how to deal with errors that arise during the process. I am not super proficient with CMake, so the whole process caused a bit of trouble. There are some incorrect tutorials and obsolete sample projects out there, that won't work out-of-the-box now (June 2021). I went through all of the steps before posting (except for step 0) just to test that they work exactly as they are described.