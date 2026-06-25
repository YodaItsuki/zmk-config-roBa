# zmk-config-roBa

<img src="keymap-drawer/roBa.svg" >

## roBa_R pointer latency test builds

The right half, `roBa_R`, is the split central and contains the PMW3610
trackball. Flash pointer-latency test firmware to `roBa_R`.

GitHub Actions builds these comparison artifacts:

- `roBa_R-default`: normal build, using 250Hz with low-latency BLE preferences
- `roBa_R-250hz`: explicit 250Hz baseline
- `roBa_R-125hz`: hardware 125Hz polling
- `roBa_R-125sw`: 250Hz sensor polling with software 125Hz reporting
- `roBa_R-250hz-ble-low-latency`: 250Hz plus lower preferred BLE connection latency

The BLE low-latency build requests a 7.5ms preferred connection interval and
zero peripheral latency. Windows may ignore that preference, and battery
consumption can increase if the host accepts it.

To roll back, flash `roBa_R-default` or revert the test-build commits.
