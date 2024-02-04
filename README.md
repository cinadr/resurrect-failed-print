# Restarting a failed print in klipper

I've collected these steps from several sources of the net, klipper forums, reddit, etc. I have a DS18B20 sensor for chamber temperature connected to raspberry gpio pins and the sensor sometimes make klipper shutdown with **MCU 'host' shutdown: DS18B20 out of range** message. That is why it made me an "expert" to restart after a klipper shutdown. My workaround currently is to set min_temp to -273 and max_temp to 2000 in printer.cfg which allows wild values as it only controls a chamber fan. The heater is connected to a correctly working 104NT-4 sensor so no real fire hazard is taken. Still need to debug DS18B20 errors to avoid this. So lets dive in.

## Initial steps, keep your bed warm

Be calm! You need to take every steps to get the most accurate results. You need to start "resurrection" as soon as possible to avoid bed cooldown.

- Before firmware restart **get klippy.log** from your host machine. Usually if you use KIAUH it is located at ~/printer_data/logs folder.
- **Restart klipper**.
- **Move up your head** from the stop point to avoid melting printed part at that point.
- Set your idle timeout to a reasonable time: **SET_IDLE_TIMEOUT TIMEOUT=3600** (3600 sec will give you 1 hour to complete these steps).
- Now **turn on your bed heater** whatever it was during the print to avoid releasing the printed part.

This is where you can take a deep breath and calm down. Proceed after you can regain focus. :smile:

## Modify gcode file

- **Get your gcode** file from the printer (Fluidd main page->Jobs->click file->download). 
- In klippy.log: Search for *Requested toolhead position at shutdown time*. Take **note of your Z** position.
- Search backward for *Upcoming* and *Virtual sdcard* (this should be 10-15 lines above). **Take note of the first gcode command** in *Upcoming (xxxxxxxx):* line
- Within these lines search for the **M73** command which means [Set Print Progress - M73 P&lt;percent&gt; [R&lt;minutes@gt;]](https://marlinfw.org/docs/gcode/M073.html).
- If this cannot be found in your gcode file than turn off "Disable set remaining print time" in orcaslicer under printer "settings/Basic Information/Advanced" section (search for similar if your slicer is different - should be in printer settings).
- Take note of the **exact params of the M73** command (copy to clipboard) and search &lt;M73 Pxx Rxxxx&gt; string in your gcode file. This is the last progress section within the printing stopped - note: percent increases, time decreases while printing.
- **Search down** from M73 to the first gcode that you've found at **Upcoming (xxxxxxx):** line in *klippy.log*.
- **Scroll up 1-5 lines** above from this command that you want to start to print. You will erase code from here to top, but *do not erase now!* **Note:** You can calculate the last movement which contains the position at line *Requested toolhead position at shutdown time* but it needs some trigonometry. I usually I do not bother with this.
- Search more up from here to get the last **G0/G1 Z** command to get last **Z** position. This might be different from shutdown time Z. Use whichever you want. I found it *more precise to use this one*.
- From the previous position now **erase gcode up to PRINT_START line (included)**. **Note:** You need to remove PRINT_START to avoid homing, bed_mesh calibrate, KAMP runs, etc. These are already done at the very start of your failed print. We need to perform homing and temp settings only. See below.
- I usually **leave EXCLUDE_OBJECT_DEFINE and all above** (thumbnail code, etc) intact. (???? This might help with keeping one gcode file (overwrite) on printer. ????)
- **Enter** manually your **heating and Z position** parameters inside the gcode. You can get help from your PRINT_START macro. Use gcode below, edit xxxx for your needs:

```G-code
RESPOND MSG="Heating up bed
M190 Sxx                            ;Wait for bed tem. Takes no time as you are already there.
RESPOND MSG="Waiting for extruder temp"
M109 Sxxx T0                        ;Wait for reach hotend temperature.
RESPOND MSG="Loading skew profile"
SKEW_PROFILE LOAD=my_skew_profile   ;Load if you have one saved. X/Y compensation needed to the top.
;LOAD_BED_MESH LOAD=my_mesh_profile ;I usually don't reload my bed profile as I set up fade_start/fade_end in
                                    ;[bed_mesh] in printer.cfg so the whole surface above fade_end layer 
                                    ;your mesh has no meaning. If your print failed below fade_end you 
                                    ;might want to restart the whole print.
SET_KINEMATIC_POSITION Z=xx.xx      ;Set your Z level from <Requested toolhead position at shutdown time>
G92 E0                              ;Reset extruder.
G11                                 ;Unretract -> This will oose some filament but can be removed later
                                    ;of the print if finished. I found this important to avoid 
                                    ;printing thin air
```

- **Save your gcode file** and upload to printer. **Do not start printing yet.**

## Set Z level

- Now while your extruder is cooled down **set your z level manually**. Find a position at the top of the print that is easy to visualize while your nozzle is above it. **Tip:** Try to use one eye to exactly look horizontally (make sure you don't see any printed surface behind the nozzle and its tip is fully visible). It helped a lot to have a white sheet of paper behind the whole thing as a background.
- Set your nozzle height rotating your Z stepper to be as close as possible. **Tip:** Rotate one or few steps (one pop only in motor you can feel with your fingers - *NOT a whole rotation!*) lifting the nozzle approximately a layer height (usually 0.2 mm for my prints). (This can be calculated exactly from stepper_z microsteps and rotation_distance parameters exatly but I do not bother with these.)
- Perform a **home on X and Y** only. Leave Z alone.
- **Move your head** in X-Y plane to a position where you will be able **to purge** some filament after heating up your nozzle. Use your klipper user interface to do that (Fluidd or Klipperscreen) - you cannot move head by hand as it is homed already. You need to do this with *cold extruder* as hitting your top layer will be melted with a hot one. If you find that your Z height is too low or too high you can restart Z leveling by stopping your motors (M84) and restart from the line after uploading your gcode above.
- **Set your nozzle temp** on UI to the printing temperature. Wait for heat up.
- **Purge** some filament while not above your print. Remove the purged filament from nozzle with a tweezers.

## Start printing

- **Start the print**.
- **Adjust Z offset** from UI to perfect layer adhesion if you made a mistake setting manual Z level before. **Tip:** You can use SET_GCODE_OFFSET Z_ADJUST=+-0.01 MOVE=1 on console but it is really slow. **Note:** If you have a nozzle camera you can watch your printing closely. I found useful to *lower speed to 50-70%* to be able to visualize (M220 S50). You can set 100% again if finished (M220 S100).
- You can set idle timeout to default now (10 minutes). **SET_IDLE_TIMEOUT TIMEOUT=600**

**Note:** Print progress and time estimation will be wide off as the already printed part is not taken into calculations.
