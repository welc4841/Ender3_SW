# Enderwire Mini12864 Modular Menu

This package adds an `Enderwire` entry at the top of Klipper's normal main menu. It does not modify Klipper's installed `menu.cfg`, so Klipper updates will not overwrite these files.

## Files

- `mini12864-menu.cfg`: loader and top-level Enderwire menu
- `mini12864-menu-preheat.cfg`: heater targets and cooldown
- `mini12864-menu-motion.cfg`: homing, park, and motors off
- `mini12864-menu-print.cfg`: pause, resume, cancel, speed, and flow
- `mini12864-menu-calibration.cfg`: screws tilt, bed mesh, probe/PID calibration, and save
- `mini12864-menu-lighting.cfg`: Mini12864 RGB presets
- `mini12864-menu-system.cfg`: Klipper restart commands and emergency stop

## Install

1. Copy all `.cfg` files into `~/printer_data/config/`.
2. Add this after your existing includes in `printer.cfg`:

   ```ini
   [include mini12864-menu.cfg]
   ```

3. Run `RESTART` from the Klipper console.
4. Open the encoder menu and select `Enderwire`.

## Important notes

- The hardware display configuration stays in your existing `klipper-mini12864.cfg`.
- The temperature presets are suggestions and should be adjusted for your filament and hotend.
- `Probe Calibrate` calls Klipper's generic `PROBE_CALIBRATE`. If your Beacon workflow uses a different calibration command, edit that one menu entry.
- `SAVE_CONFIG` and PID calibration trigger configuration changes/restarts. Use them intentionally.
- The emergency-stop item is deliberately last in the System submenu.
