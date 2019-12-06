
On Mac OS X, I setup the environment with following steps,

```text
$ virtualenv --python /usr/bin/python2.7 ~/venv/xxx
$ export PYTHONPATH=/Applications/kicad/kicad.app/Contents/Frameworks/python/site-packages
$ export LD_LIBRARY_PATH=/Applications/KiCad/kicad.app/Contents/Frameworks
$ source ~/venv/xxx/bin/activate
$ git clone https://github.com/johnbeard/kiplot.git /tmp/kiplot
$ cd /tmp/kiplot
$ pip install -e .
```

And then run `kiplot --help`, and crashes.

```text
Fatal Python error: PyThreadState_Get: no current thread
[1]    1046 abort      kiplot --help
```

Adding some trace logs to those python modules to be loaded by kiplot, I found the crash happens in this line (`/Applications/KiCad/kicad.app/Contents/Frameworks/python/site-packages/pcbnew.py`):

```python
# Import the low-level C/C++ module
if __package__ or '.' in __name__:
    from . import _pcbnew
else:
    import _pcbnew
```

Above python codes try to load `_pcbnew.so` (the C/C++ module). This native library has following dependencies:

```text
$  otool -L /Applications/KiCad/kicad.app/Contents/Frameworks/python/site-packages/_pcbnew.so
/Applications/KiCad/kicad.app/Contents/Frameworks/python/site-packages/_pcbnew.so:
	/System/Library/Frameworks/IOKit.framework/Versions/A/IOKit (compatibility version 1.0.0, current version 275.0.0)
	/System/Library/Frameworks/Carbon.framework/Versions/A/Carbon (compatibility version 2.0.0, current version 158.0.0)
	/System/Library/Frameworks/Cocoa.framework/Versions/A/Cocoa (compatibility version 1.0.0, current version 22.0.0)
	/System/Library/Frameworks/AudioToolbox.framework/Versions/A/AudioToolbox (compatibility version 1.0.0, current version 492.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.0.0)
	/System/Library/Frameworks/OpenGL.framework/Versions/A/OpenGL (compatibility version 1.0.0, current version 1.0.0)
	@rpath/libwx_osx_cocoau_gl-3.0.0.dylib (compatibility version 5.0.0, current version 5.0.0)
	@rpath/libwx_osx_cocoau-3.0.0.dylib (compatibility version 5.0.0, current version 5.0.0)
	@rpath/Python.framework/Python (compatibility version 2.7.0, current version 2.7.0)
	@rpath/libkicad_3dsg.2.0.0.dylib (compatibility version 2.0.0, current version 0.0.0)
	@rpath/libGLEW.2.1.dylib (compatibility version 2.1.0, current version 2.1.0)
	@rpath/libcairo.2.dylib (compatibility version 11603.0.0, current version 11603.0.0)
	@rpath/libpixman-1.0.dylib (compatibility version 39.0.0, current version 39.4.0)
	/usr/lib/libcurl.4.dylib (compatibility version 7.0.0, current version 9.0.0)
	/usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 400.9.0)
	/System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation (compatibility version 150.0.0, current version 1450.15.0)
```

By searching `Fatal Python error: PyThreadState_Get: no current thread` on Google, a similar issue was found in [caffe2#854](https://github.com/facebookarchive/caffe2/issues/854), and [pytorch@73f6715](https://github.com/pytorch/pytorch/commit/73f6715f4725a0723d8171d3131e09ac7abf0666) fixed this issue, with these commit messages:

>
> Summary:
> our cmake build used to link against libpython.so with its absolute path (instead of `-LSOME_LIB_PATH -lpython`), so at runtime loader will think it needs the libpython.so at that specific path, and so load in an additional libpython.so, which causes the python binding built with one python installation not reusable by another (maybe on same machine or sometimes even not on same machine). The solution is quite simple, which is we don't link against libpython, leave all the python related symbols unresolved at build time, they will be resolved at runtime when imported into python.
>

To re-compile `pcbnew` native library may be a too long way to fix. So, I write a wrapper script in BASH to run **kiplot** with python executable (e.g. `/Applications/KiCad/kicad.app/Contents/Frameworks/Python.framework/Versions/2.7/bin/python`) and assign site-package path (e.g. `export PYTHONPATH=/Applications/kicad/kicad.app/Contents/Frameworks/python/site-packages`), and then it works. The wrapper script is [kiplot_macosx_wrapper](./kiplot_macosx_wrapper).


Using the wrapper script is almost same as `kiplot` python script:

```text
$ /tmp/kiplot/scripts/kiplot_macosx_wrapper --help

usage: macosx.py [-h] [-v] -b BOARD_FILE -c PLOT_CONFIG [-d OUT_DIR]

Command-line Plotting for KiCad

optional arguments:
  -h, --help            show this help message and exit
  -v, --verbose         show debugging information
  -b BOARD_FILE, --board-file BOARD_FILE
                        The PCB .kicad-pcb board file
  -c PLOT_CONFIG, --plot-config PLOT_CONFIG
                        The plotting config file to use
  -d OUT_DIR, --out-dir OUT_DIR
                        The output directory (cwd if not given)
```

You can specify the path of Kicad.app to wrapper script:

```text
$ KICAD_APP_PATH=~/Downloads/kicad/Kicad.app /tmp/kiplot/scripts/kiplot_macosx_wrapper
```

Or, to enable more verbose messages for debugging the wrapper script:


```text
$ WRAPPER_VERBOSE=true /tmp/kiplot/scripts/kiplot_macosx_wrapper --help

assume kicad.app is located at /Applications/KiCad/kicad.app ...
found python at kicad.app (/Applications/KiCad/kicad.app/Contents/Frameworks/Python.framework/Versions/2.7/bin) ...
running command:
	PYTHONPATH=/tmp/kiplot/src:/Applications/KiCad/kicad.app/Contents/Frameworks/python/site-packages:/Applications/KiCad/kicad.app/Contents/Frameworks/Python.framework/Versions/2.7/lib/python2.7/site-packages:/tmp/kiplot/venv/macosx/lib/python2.7/site-packages /Applications/KiCad/kicad.app/Contents/Frameworks/Python.framework/Versions/2.7/bin/python /tmp/kiplot/scripts/macosx.py --help
usage: macosx.py [-h] [-v] -b BOARD_FILE -c PLOT_CONFIG [-d OUT_DIR]

Command-line Plotting for KiCad

optional arguments:
  -h, --help            show this help message and exit
  -v, --verbose         show debugging information
  -b BOARD_FILE, --board-file BOARD_FILE
                        The PCB .kicad-pcb board file
  -c PLOT_CONFIG, --plot-config PLOT_CONFIG
                        The plotting config file to use
  -d OUT_DIR, --out-dir OUT_DIR
                        The output directory (cwd if not given)
```
