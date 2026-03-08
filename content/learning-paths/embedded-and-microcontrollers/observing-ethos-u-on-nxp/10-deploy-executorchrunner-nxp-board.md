---
title: Deploy and test on FRDM-IMX93
weight: 11

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## Prerequisites

Before deploying, ensure the following are in place on your FRDM-IMX93 board:

1. **Ethos-U kernel driver is loaded.** The Linux `ethosu` driver must be bound so the NPU is powered and clocked. Verify with:

   ```bash { command_line="root@frdm-imx93" output_lines="2" }
   ls /dev/ethosu*
   /dev/ethosu0
   ```

   If `/dev/ethosu0` does not exist, the NPU is not powered and the firmware will hang at NPU initialization.

2. **DDR memory is reserved for the CM33.** The device tree must reserve 4MB of DDR at `0xC0000000` for the model and NPU scratch buffer:

   ```dts
   reserved-memory {
       ethosu_mem: ethosu@c0000000 {
           reg = <0 0xc0000000 0 0x400000>;
           no-map;
       };
   };
   ```

   The NXP BSP for the FRDM-IMX93 includes this reservation by default.

## Load the model to DDR via U-Boot

The executor_runner firmware reads the `.pte` model from DDR address `0xC0000000`. Because this memory region is marked `no-map` in the device tree, it cannot be written from Linux. You must load the model using U-Boot before Linux boots.

1. Insert the SD card containing the `.pte` files (copied in the previous step) into the FRDM-IMX93 board.

2. Power on the board and interrupt U-Boot by pressing a key when you see the autoboot countdown in the serial console.

3. Load the model into DDR:

   ```text
   u-boot=> fatload mmc 0:1 0xc0000000 model_u65.pte
   ```

   You should see output confirming the bytes loaded:

   ```text
   3872 bytes read in 12 ms (315.4 KiB/s)
   ```

4. Continue booting Linux:

   ```text
   u-boot=> boot
   ```

{{% notice Note %}}
The model remains in DDR across Linux boot because the region is marked `no-map`. You only need to reload via U-Boot if the board is fully power-cycled. To load the larger MobileNet V2 model instead, use `fatload mmc 0:1 0xc0000000 mobilenetv2_u65.pte`.
{{% /notice %}}

## Connect to the FRDM-IMX93 board

Once Linux has booted, find your board's IP address using the serial console or check your router's DHCP leases.

Connect via SSH:

```bash
ssh root@<board-ip>
```

Replace `<board-ip>` with your board's actual IP address.

{{< tabpane code=false >}}
{{< tab header="Windows" >}}
You can also use PuTTY:
- Host: `<board-ip>`
- Port: `22`
- Connection type: SSH
- Username: `root`
{{< /tab >}}
{{< tab header="Linux/macOS" >}}
```bash
ssh root@<board-ip>
```
{{< /tab >}}
{{< /tabpane >}}

## Copy the firmware to the board

Copy the built firmware file to the board's firmware directory:

```bash
scp debug/executorch_runner_cm33.elf root@<board-ip>:/lib/firmware/
```

Verify the file was copied:

```bash { command_line="root@frdm-imx93" output_lines="2" }
ls -lh /lib/firmware/executorch_runner_cm33.elf
-rw-r--r-- 1 root root 424K Mar  8 10:30 /lib/firmware/executorch_runner_cm33.elf
```

## Load and start the firmware on Cortex-M33

The Cortex-M33 firmware is managed by the RemoteProc framework running on Linux.

Stop any currently running firmware:

```bash { command_line="root@frdm-imx93" }
echo stop > /sys/class/remoteproc/remoteproc0/state
```

{{% notice Note %}}
If no firmware is running, this command prints an error. That is expected and can be ignored.
{{% /notice %}}

Set the firmware filename and start it:

```bash { command_line="root@frdm-imx93" }
echo executorch_runner_cm33.elf > /sys/class/remoteproc/remoteproc0/firmware
echo start > /sys/class/remoteproc/remoteproc0/state
```

Verify the firmware loaded successfully:

```bash { command_line="root@frdm-imx93" output_lines="2-5" }
dmesg | grep remoteproc | tail -n 5
[12345.678] remoteproc remoteproc0: powering up imx-rproc
[12345.679] remoteproc remoteproc0: Booting fw image executorch_runner_cm33.elf, size 434280
[12345.680] remoteproc remoteproc0: registered virtio0 (type 7)
[12345.681] remoteproc remoteproc0: remote processor imx-rproc is now up
```

The message "remote processor imx-rproc is now up" confirms successful loading.

## View inference output

The executor_runner writes all output to the remoteproc **trace buffer**, which is readable from Linux. Wait a few seconds for inference to complete, then read the trace:

```bash { command_line="root@frdm-imx93" }
sleep 3
cat /sys/kernel/debug/remoteproc/remoteproc0/trace0
```

You should see output similar to:

```output
I: Initializing NPU: base_address=0x4a900000, fast_memory=0, secure=0, privileged=0
I: Soft reset NPU
I: New NPU driver registered (handle: 0x0x20009210, NPU: 0x0x4a900000)
I: Optimizer config. product=1, cmd_stream_version=0, macs_per_cc=8, shram_size=48, custom_dma=0
I: Optimizer config. arch version: 1.0.6
I: Ethos-U config. product=1, cmd_stream_version=0, macs_per_cc=8, shram_size=48, custom_dma=0
I: Ethos-U. arch version=1.0.6
I: Test Case 16: handle_optimizer_config: NPU config match
I: Test Case 17: handle_optimizer_config: NPU arch match
I: Test Case 14: cmd_end_reached 0x1
1 inferences finished
Output[0]: dtype=6, numel=1, nbytes=4
  [0]=1073741824 (float raw)
Model run: 1
Program complete, exiting.
```

The key indicators of a successful inference run:

| Output | Meaning |
|--------|---------|
| `NPU config match` | The compiled model's NPU configuration matches the hardware |
| `NPU arch match` | The compiled model's architecture version matches the hardware |
| `cmd_end_reached 0x1` | The NPU executed the full command stream without errors |
| `bus_status_error 0x0` | No AXI bus errors during NPU memory access |
| `1073741824 (float raw)` | Output value `0x40000000` = `2.0` in IEEE 754 float (correct for the add model: 1.0 + 1.0 = 2.0) |

{{% notice Note %}}
If the trace buffer shows `Program identifier '' != expected 'ET12'`, the `.pte` model was not loaded into DDR. Power-cycle the board and load the model via U-Boot as described above.
{{% /notice %}}

## Test inference

The executor_runner automatically runs inference when the firmware starts. To re-run inference, reload the firmware:

```bash { command_line="root@frdm-imx93" }
echo stop > /sys/class/remoteproc/remoteproc0/state
sleep 2
echo start > /sys/class/remoteproc/remoteproc0/state
sleep 3
cat /sys/kernel/debug/remoteproc/remoteproc0/trace0
```

The trace buffer is reset at the start of each inference run, so you always see fresh output.

## Verify deployment success

Confirm your deployment is working correctly:

1. **RemoteProc status shows "running":**

```bash { command_line="root@frdm-imx93" output_lines="2" }
cat /sys/class/remoteproc/remoteproc0/state
running
```

2. **Firmware is loaded:**

```bash { command_line="root@frdm-imx93" output_lines="2" }
cat /sys/class/remoteproc/remoteproc0/firmware
executorch_runner_cm33.elf
```

3. **Trace buffer shows inference output** with `cmd_end_reached 0x1` and model results.

## Update the firmware

To deploy a new version of the firmware:

1. Build the updated firmware on your development machine.
2. Copy to the board:

```bash
scp debug/executorch_runner_cm33.elf root@<board-ip>:/lib/firmware/
```

3. Restart RemoteProc:

```bash { command_line="root@frdm-imx93" }
echo stop > /sys/class/remoteproc/remoteproc0/state
sleep 2
echo start > /sys/class/remoteproc/remoteproc0/state
sleep 3
cat /sys/kernel/debug/remoteproc/remoteproc0/trace0
```

## Troubleshooting

**RemoteProc fails to load firmware:**

Check that the file exists and has correct permissions:

```bash { command_line="root@frdm-imx93" }
ls -la /lib/firmware/executorch_runner_cm33.elf
chmod 644 /lib/firmware/executorch_runner_cm33.elf
```

**`Program identifier '' != expected 'ET12'`:**

The `.pte` model is not present at DDR address `0xC0000000`. Power-cycle the board and load the model via U-Boot:

```text
u-boot=> fatload mmc 0:1 0xc0000000 model_u65.pte
u-boot=> boot
```

**Firmware hangs (no trace output):**

Verify the Ethos-U kernel driver is loaded:

```bash { command_line="root@frdm-imx93" }
ls /dev/ethosu*
dmesg | grep ethosu
```

If `/dev/ethosu0` does not exist, the NPU is not powered and the firmware cannot initialize it.

**BUS FAULT or vtable corruption:**

The SDK linker script patch has not been applied. Run the patch script and rebuild:

```bash
./patches/apply_patches.sh
cmake --preset debug
cmake --build debug
```

**Firmware crashes or hangs after NPU init:**

Check kernel logs for errors:

```bash { command_line="root@frdm-imx93" }
dmesg | grep -i error | tail
```

This might indicate memory configuration issues. Verify that the DDR region `0xC0000000-0xC03FFFFF` is reserved in the device tree.
