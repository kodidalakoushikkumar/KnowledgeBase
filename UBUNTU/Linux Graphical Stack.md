
# 1️⃣7️⃣ Linux Graphical Stack

`Display Manager → Display Server → Window Manager / Compositor → Applications`

### 1. Display Manager (Login / Lock screen)

The **Display Manager** starts the graphical session and shows the **login screen**.
Examples:
- GDM
- SDDM
- LightDM

What it does:
- Shows **login screen**
- Lets you choose **X11 or Wayland session**
- Starts the desktop environment or window manager after login

Example flow:  `Boot → GDM → login → start GNOME / i3 / Hyprland`

### 2. Display Server (Graphics system)

This is the system that **talks to the GPU and draws windows**. Two main ones:

- X Window System
- Wayland

They handle:
- drawing windows
- keyboard input
- mouse input
- screen rendering

### 3. Window Manager / Compositor

This controls **how windows behave**.
Examples:
- i3
- Openbox
- Hyprland

What it controls:
- window tiling
- moving/resizing windows
- animations
- workspaces

Note:
- In **X11**, window manager is separate.
- In **Wayland**, the **compositor + window manager are usually the same program**.

## Architecture

### X11 system

```
Display Manager (GDM)  
        ↓  
X Server (X11)  
        ↓  
Window Manager (i3 / Openbox)  
        ↓  
Applications
```

### Wayland system

```
Display Manager (GDM)  
        ↓  
Wayland Compositor (Hyprland / GNOME)  
        ↓  
Applications
```

**Simple way to remember**

|Component|Role|
|---|---|
|Display Manager|Login screen|
|Display Server|Draws graphics|
|Window Manager|Controls windows|
