# Flashing Guide (T6E Calibration Flasher / EFI_PROT)

Step-by-step instructions for flashing a calibration (.CPT/.CRP) or full firmware (.CRP) with the T6E Calibration Flasher. See the README's "T6E Calibration Flasher" section for background.

If you are flashing full firmware rather than a quick calibration tune, attach a battery tender first.

## Flashing from LotusECMLogger

1. Sit in the car with both doors closed.
2. Put the key in, but do **not** turn it.
3. With the OBD adapter connected, open LotusECMLogger and select **Tools → T6E Calibration Flasher**.
4. Fill in the three paths, then select **Launch Program**:
   - the path to `EFI_PROT.exe` (`EFI_PROT.dll` must be in the same folder)
   - the path to your `.CRP` (encrypted) or `.CPT` (unencrypted) file
   - the directory containing `EFI_PROT.exe`
5. A command window opens, running EFI_PROT with your file. It asks you to select your OBD adapter. Once you have selected it and it says "waiting for …", turn the key to **on**, but do not start the car (do not press the start button).
6. Flashing starts as soon as the key is on:
   - **Calibration (tune):** takes seconds.
   - **Firmware:** can take several minutes and includes an ECU reboot. The cooling fans may turn on.

   The window closes by itself when finished, after a brief status message. A successful flash shows **100%** and then **"Successful Download"** before closing.

## If a flash fails

A failed firmware flash can lock out flashing for a while. Either wait about 20 minutes, or disconnect the battery, reconnect it, and start again. Turning the ignition off and on does not clear the lockout.

## Running EFI_PROT from the command line

Use this if you hit an error you can't read, or want to see the final status message (the window stays open):

1. Open a command prompt and go to the folder containing `EFI_PROT.exe` and `EFI_PROT.dll`.
2. Run:

   ```
   EFI_PROT <path to the CRP>
   ```

3. Follow steps 5–6 above: select the OBD adapter, then turn the key when prompted.

If you never get the prompt to turn the key, the conditions for flashing have not been met.
