# Lessons learnt from porting the `DOOM` source to MacOSX

## `make`
OG public src release in 1997 was only for Linux (not even for DOS; some copyright issue).
Everything is set/defined for `LINUX` as the target platform.
Most of this gets invoked by the C Preprocess (cpp) as the first step of the compiler.
You pass `gcc` the macro as `-D` and it compiles the appropriate codeblock -- this is **CONDITIONAL COMPILATION**:
`CFLAGS=-g -Wall -DNORMALUNIX -DLINUX`, here `-g` is for the GnuDeBugger, `-W` is for all warnings.

`make` failed at importing `values.h` under `-DLINUX ifdef` in `doomtype.h`; fixed by changing platform... for now.

`make` failed at importing `<linux/soundcard.h>` inside the audio interface C file `i_sound.c` -- removing this dependency for now.

`make` failed compiling video interface `i_video.c`, trying to import `<X11/Xlib.h>`. 
I can't exactly skip past this ... it's required for the graphics output.
There is a XQuartz alternative for macOS <https://www.xquartz.org/releases/index.html>
Looks like the XQuartz files should work -- `/usr/X11`; these need to be symlinked to `/usr/include`
It worked! `ln -s /opt/X11/include/X11 /usr/local/include/X11` ... onto the next error.

`make` failed compiling `i_video.c:49:10: fatal error: 'errnos.h' file not found`
Can't seem to find a `errnos.h` file in the Linux src -- there's only `errno.h`?
HAHAHA lol -- <https://github.com/id-Software/DOOM/pull/3/files> -- this PR says the filename is a typo :X
And that worked. Moving on.

`make` failed compiling `m_misc.c:257:48: error: initializer element is not a compile-time constant`.
`m_misc.c` is for miscellaneous menu related stuff ...
Commented out some server related code (`sndserver` and `chatmacro`). Moving on.

`make` failed compiling `m_bbox.h:26:10: fatal error: 'values.h' file not found`
I've seen this error before in `doomtype.h` -- there I skipped the import by changing the macro `-D` flag.
MacOS has an alternative `limits.h` which exists for the same purpose.
I have two options here:

1) I can choose to ignore the `values.h` import -- this might work because the `doomtype.h` file
defines constants in the `else` block:
```C
// Predefined with some OS.
#ifdef LINUX
#include <values.h> // THIS is missing from MacOS.
#else
#define MAXCHAR		((char)0x7f)
#define MAXSHORT	((short)0x7fff)

// Max pos 32-bit int.
#define MAXINT		((int)0x7fffffff)	
#define MAXLONG		((long)0x7fffffff)
#define MINCHAR		((char)0x80)
#define MINSHORT	((short)0x8000)

// Max negative 32-bit integer.
#define MININT		((int)0x80000000)	
#define MINLONG		((long)0x80000000)
#endif
```
2) The other option is to keep `-DLINUX` and replace `values.h` with `limits.h`

**Currently** going with the 1st approach, and removing calls to `values.h` anywhere else.
This led to failure in `m_bbox.c` which needs some system constants `MININT, MAXINT` which are (probably)
present in `values.h`. 
I added `#include "doomtype.h"` to be included in `m_bbox.h` so these become available.
Moving on.

`make` failed compiling `w_wad.c:34:10: fatal error: 'malloc.h' file not found`
This is a non-standard file, deprecated in MacOS; replaced with `stdlib.h`. Moving on.

`make` failed due to linker error; can't find `lnsl` library.
Turns out the implementations are present in the C standard lib for MacOS. Removing it from `make`.
Moving on.

---

## Sound errors.
The code still references the sound interfaces in different places.
First attempt, just comment them (all sound interface references) from everywhere.
