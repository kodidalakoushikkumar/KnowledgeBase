# Manually adding a screen resolution

**manually adding a screen resolution that Ubuntu does not currently detect**.

### 1. Check the current display modes

```
xrandr
```

`xrandr` = **X Resize and Rotate**.
It asks the graphics/display system: "What displays do I have, what resolution are they using, and what resolutions are available?"
You saw: `eDP-1 connected primary 1366x768`. So:

- `eDP-1` = your laptop's internal screen
- `connected` = Ubuntu detects it
- `primary` = main display
- `1366x768` = current resolution

And importantly, `1920x1080` wasn't in the list.

### 2. Calculate a 1920×1080 display mode

```
cvt 1920 1080 60
```

`cvt` means **Coordinated Video Timings**.
You're telling it: "Calculate the technical timing parameters required to display 1920×1080 at 60 Hz."
It produced: `Modeline "1920x1080_60.00" 173.00 1920 2048 2248 2576 1080 1083 1088 1120 -hsync +vsync`
The important part for you is: `1920 1080`
The rest are **timing parameters** used by the display system.
Think of the Modeline as: "Here are the exact instructions for how to generate a 1920×1080 @ 60 Hz video signal."

### 3. Create that mode inside xrandr

```
xrandr --newmode "1920x1080_60.00" 173.00 1920 2048 2248 2576 1080 1083 1088 1120 -hsync +vsync
```

This tells `xrandr`: "Create a new display mode called `1920x1080_60.00` using these timing parameters."
After this, the mode exists in xrandr, but it isn't attached to your screen yet.
Think:

```
1920x1080 mode
       ↓
created in xrandr
       ↓
but not assigned to eDP-1 yet
```

### 4. Attach the mode to your laptop display

```
xrandr --addmode eDP-1 "1920x1080_60.00"
```

This tells xrandr: "Make this newly-created mode available on my `eDP-1` display."
Now the relationship is:

```
eDP-1
 ├── 1366x768
 ├── 1280x720
 ├── 1024x768
 └── 1920x1080_60.00  ← manually added
```

You can check with: `xrandr`
### 5. Actually switch to it

```
xrandr --output eDP-1 --mode "1920x1080_60.00"
```

This tells Ubuntu: "Use the 1920×1080 mode on the eDP-1 display now."
So the complete process is:

```
xrandr
  ↓
Check available resolutions

cvt 1920 1080 60
  ↓
Calculate timing information

xrandr --newmode ...
  ↓
Create 1920×1080 mode

xrandr --addmode eDP-1 ...
  ↓
Attach that mode to your screen

xrandr --output eDP-1 --mode ...
  ↓
Actually switch the screen to it
```
