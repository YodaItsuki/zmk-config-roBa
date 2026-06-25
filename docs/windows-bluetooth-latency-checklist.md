# Windows Bluetooth latency checklist

Use the same firmware, CPI, Windows pointer speed, and acceleration settings when comparing results.

## Bluetooth and pairing

- Update the Bluetooth driver from the PC maker or Bluetooth adapter maker.
- Remove the roBa Bluetooth device from Windows.
- Clear the roBa bond from the keyboard side, then pair again.
- Test with a different Bluetooth adapter if available.
- Test with a different PC if available.

## Power management

- Set Windows power mode to best performance while testing.
- In Device Manager, disable power saving for the Bluetooth adapter if that option is shown.
- Disable power saving for USB Root Hub entries connected to the Bluetooth adapter.

## Radio and USB placement

- Put a USB Bluetooth adapter on the desk with a USB 2.0 extension cable.
- Keep the Bluetooth adapter away from USB 3.x hubs, SSDs, docks, and high-speed cables.
- Prefer 5GHz or 6GHz Wi-Fi while testing, and avoid crowded 2.4GHz Wi-Fi.
- Compare adapter positions close to and far from the keyboard.

## Load checks

- Check Task Manager for CPU, GPU, and disk load while reproducing the issue.
- Compare idle, CPU load, GPU load, and disk load conditions.
- If needed, use LatencyMon from Resplendence to check DPC latency. Treat it as a diagnostic tool, not proof that firmware is at fault.

## Pointer tests

- Draw a small circle slowly.
- Draw a large circle at a steady speed.
- Flick the ball quickly.
- Repeat short left-right movements.
- Check whether the cursor keeps moving after the ball stops.
- Move the pointer while clicking.
- Move the pointer while typing normally.
- Compare against USB mode under the same physical conditions.

