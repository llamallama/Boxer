This is a fork of Boxer's leopard_legacy branch with a third-party PowerPC
dynrec JIT (patch-r4301.diff) backported into the embedded DOSBox tree.
The Legacy Release configuration now produces a JIT-enabled PPC binary.

The patch is jmarsh's PPC backend for DOSBox's core_dynrec, originally
posted on Vogons and packaged with precompiled binaries by Dr. Cameron
Kaiser on SourceForge:
- https://www.vogons.org/viewtopic.php?f=32&t=65057
- https://sourceforge.net/projects/dosbox-ppcjit/

See PPC_JIT_NOTES.md for the integration details. Build instructions
below are unchanged from Boxer.


Some notes on building Boxer
============================

The Boxer XCode project is designed to be a painless one-click build, but there are a few caveats explained below, so please read this before you get to work.


Requirements
------------

To build the Boxer project you will need OS X 10.6 or higher and XCode 3.2 or higher. (See below for notes about XCode 4 compatibility.)

All required frameworks are included in the Boxer project, so the project itself is all you need.


Build Configurations
--------------------

The Boxer project has 3 build configurations: Legacy Release, Release and Debug.
- Legacy Release compiles an optimized 32-bit universal binary for PowerPC and i386 using LLVM in GCC 4.2 mode.
- Release compiles an optimized 32-bit binary for i386 using the LLVM 3 compiler.
- Debug does the same as Release but also turns on console debug messages and additional error-checking.

Unless you want to distribute your build to others, you should stick with Release or Debug rather than Legacy Release, since a Universal binary takes twice as long to build.


XCode 4 Caveats
---------------

XCode versions 4.0 and later do not come with PowerPC compilers, which will normally prevent you from building the Legacy Release configuration. However: if you are still using OS X 10.6, then you can install XCode 3.2.x alongside XCode 4.x and compile the Legacy Release configuration using that. (XCode 3.2 cannot be installed on Lion, so you're out of luck there.)


Other things to be aware of
---------------------------
Boxer's OS X hotkey override option ("Reserve function keys and arrow keys for games") relies on a keyboard event tap, and these *do not play nice at all* with the XCode debugger. I strongly recommend turning off that option in Boxer's preferences if you're testing through XCode: Otherwise, when pausing in the debugger or hitting a breakpoint, the mouse and keyboard may stop responding altogether and you'll have to restart your Mac to get them back.


Having trouble?
---------------
If you have any problems building the Boxer project, or questions about Boxer's code, please get in touch with me at abestor@boxerapp.com and I'll help out as best I can.

