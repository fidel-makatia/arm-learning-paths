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

2. **DDR memory is reserved for the CM33.** The NXP BSP for the FRDM-IMX93 reserves two DDR regions by default:

   ```dts
   reserved-memory {
       model@c0000000 {
           reg = <0 0xc0000000 0 0x400000>;   /* 4MB for .pte model */
           no-map;
       };
       ethosu_region@A8000000 {
           reg = <0 0xa8000000 0 0x8000000>;  /* 128MB for NPU working memory */
           no-map;
       };
   };
   ```

## Load the model to DDR

The executor_runner firmware reads the `.pte` model from DDR address `0xC0000000`.

### Option A — From Linux via `/dev/mem` (no reboot required)

Copy the `.pte` file to the board and write it directly to DDR:

```bash
scp mobilenetv2_u65.pte root@<board-ip>:/tmp/
```

On the board, write the model to the reserved DDR address:

```bash { command_line="root@frdm-imx93" output_lines="8" }
python3 -c "
import mmap, os
pte = open('/tmp/mobilenetv2_u65.pte', 'rb').read()
fd = os.open('/dev/mem', os.O_RDWR | os.O_SYNC)
m = mmap.mmap(fd, len(pte), mmap.MAP_SHARED, mmap.PROT_WRITE, offset=0xC0000000)
m.write(pte)
m.close()
os.close(fd)
print(f'Wrote {len(pte)} bytes to 0xC0000000')
"
Wrote 3507872 bytes to 0xC0000000
```

### Option B — Via U-Boot (requires reboot)

If you have the `.pte` files on the SD card's first partition, load the model in U-Boot before booting Linux:

1. Power on the board and press a key when you see the autoboot countdown in the serial console.

2. Load the model into DDR:

   ```text
   u-boot=> fatload mmc 0:1 0xc0000000 mobilenetv2_u65.pte
   ```

   You should see output confirming the bytes loaded:

   ```text
   3507872 bytes read in 58 ms (57.7 MiB/s)
   ```

3. Continue booting Linux:

   ```text
   u-boot=> boot
   ```

{{% notice Note %}}
The model remains in DDR across Linux boot because the region is marked `no-map`. You only need to reload the model if the board is fully power-cycled.
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
-rw-r--r-- 1 root root 716K Mar  8 10:30 /lib/firmware/executorch_runner_cm33.elf
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

The executor_runner writes all output to the remoteproc **trace buffer**, which is readable from Linux. MobileNet V2 takes approximately 10 seconds to complete inference. Wait for it to finish, then read the trace:

```bash { command_line="root@frdm-imx93" }
sleep 15
cat /sys/kernel/debug/remoteproc/remoteproc0/trace0
```

You should see output similar to:

```output
I: Optimizer config. product=1, cmd_stream_version=0, macs_per_cc=8, shram_size=48, custom_dma=0
I: Optimizer config. arch version: 1.0.6
I: Ethos-U config. product=1, cmd_stream_version=0, macs_per_cc=8, shram_size=48, custom_dma=0
I: Ethos-U. arch version=1.0.6
I: Test Case 16: handle_optimizer_config: NPU config match
I: Test Case 17: handle_optimizer_config: NPU arch match
I: Test Case 14: cmd_end_reached 0x1
I: Test Case 10: bus_status_error 0x0
1 inferences finished
Output[0]: dtype=6, numel=1000, nbytes=4000
    [0]=0 (float raw)
    [1]=1058744595 (float raw)
    [2]=-1081274863 (float raw)
    ... (1000 total values)
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
| `numel=1000, nbytes=4000` | MobileNet V2 output: 1000 float32 values, one score per ImageNet class |

The model was compiled with random input data, so the output scores do not correspond to a real image classification. To get meaningful predictions, you would feed a real 224x224 RGB image as input.

{{% notice Note %}}
If the trace buffer shows `Program identifier '' != expected 'ET12'`, the `.pte` model was not loaded into DDR. Reload the model using one of the methods described above.
{{% /notice %}}

## Re-run inference

The executor_runner runs inference once when the firmware starts. To re-run, reload the firmware:

```bash { command_line="root@frdm-imx93" }
echo stop > /sys/class/remoteproc/remoteproc0/state
sleep 2
echo start > /sys/class/remoteproc/remoteproc0/state
sleep 15
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

3. **Trace buffer shows inference output** with `cmd_end_reached 0x1` and 1000 output values.

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
sleep 15
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

The `.pte` model is not present at DDR address `0xC0000000`. Reload the model using either method:

From Linux:
```bash { command_line="root@frdm-imx93" }
python3 -c "
import mmap, os
pte = open('/tmp/mobilenetv2_u65.pte', 'rb').read()
fd = os.open('/dev/mem', os.O_RDWR | os.O_SYNC)
m = mmap.mmap(fd, len(pte), mmap.MAP_SHARED, mmap.PROT_WRITE, offset=0xC0000000)
m.write(pte)
m.close()
os.close(fd)
print(f'Wrote {len(pte)} bytes to 0xC0000000')
"
```

Or via U-Boot (requires reboot):
```text
u-boot=> fatload mmc 0:1 0xc0000000 mobilenetv2_u65.pte
u-boot=> boot
```

**Firmware hangs (no trace output):**

Verify the Ethos-U kernel driver is loaded:

```bash { command_line="root@frdm-imx93" }
ls /dev/ethosu*
dmesg | grep ethosu
```

If `/dev/ethosu0` does not exist, the NPU is not powered and the firmware cannot initialize it.

**Memory allocation failed for planned buffer:**

This occurs when a large model's activation tensors exceed the DTCM method allocator. The firmware automatically uses DDR for models that need more than 12KB of planned buffers. If you still see this error, verify the `ethosu_region@A8000000` (128MB) is reserved in the device tree.

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

This might indicate memory configuration issues. Verify that both DDR regions (`0xC0000000-0xC03FFFFF` and `0xA8000000-0xAFFFFFFF`) are reserved in the device tree.
