**⚠️ Early Proof of Concept**

This project is a working proof of concept, not a polished product. I got it 
working on my setup — your mileage may vary, especially depending on your 
network configuration (IPv6 routing, NAT66, VM networking all matter).

**Known limitations to be aware of:**
- Uses the ESP32-C6's internal PCB antenna — RF range is limited, keep it 
  close to both your WiFi router and your Thread devices
- WiFi and Thread share a single RF path — performance is worse than a 
  dedicated two-chip border router
- Has not been extensively tested beyond the author's own setup

**AI disclosure:** I'm just an enthusiast, not an embedded systems expert. 
This project was built with very heavy assistance from 
[Claude Sonnet 4.5](https://www.anthropic.com/claude) (Anthropic), including 
the firmware patches, troubleshooting, and this writeup. The AI was essential 
— I could not have done this without it. Use the code accordingly and please 
improve on it!

# ESP32-C6 Thread Border Router for Home Assistant

Turn a $5 XIAO ESP32-C6 into a fully functional Thread Border Router that connects Matter/Thread devices (like the IKEA Vindstyrka / Alpstuga air quality sensor) to Home Assistant — with no additional hardware required.

![ESP32-C6 Thread Border Router](https://img.shields.io/badge/ESP32--C6-Thread%20Border%20Router-blue)
![Home Assistant](https://img.shields.io/badge/Home%20Assistant-Matter%2FThread-green)
![License](https://img.shields.io/badge/license-CC0-lightgrey)

## What this does

- Runs a **Thread Border Router** on the ESP32-C6's built-in 802.15.4 radio
- Connects to your **WiFi network** (no USB data connection needed — power only)
- Exposes an **OTBR-compatible REST API** on port 8080 so Home Assistant can discover and use it
- Lets you commission and control **Matter over Thread** devices from Home Assistant

## Hardware required

- [Seeed Studio XIAO ESP32-C6](https://www.seeedstudio.com/Seeed-Studio-XIAO-ESP32C6-p-5884.html) (~$5)
- USB-C cable for power (charge-only cable is fine after flashing)
- A PC/Mac/Linux machine for flashing (one time only)

## Compatibility

Tested with:
- IKEA Vindstyrka / Alpstuga (Matter over Thread air quality sensor)
- Home Assistant running in a VM (with bridged networking)
- pfsense router (see [Network Notes](#network-notes) for IPv6 requirements)

Should work with any Matter over Thread device.

## Prerequisites

- Home Assistant with the **Matter (BETA)** integration and **Matter Server** add-on installed
- A router that supports IPv6 (required for Thread/Matter)
- **NAT66 must be disabled** on your router — Matter commissioning requires real IPv6 end-to-end connectivity

---

## Step 1 — Install ESP-IDF

ESP-IDF is Espressif's development framework. You only need to do this once.

### Windows

1. Download and run the [ESP-IDF Windows Installer](https://dl.espressif.com/dl/esp-idf/)
2. During installation, select **ESP32-C6** as the target chip
3. Or clone manually:
   ```cmd
   git clone -b v5.4.2 --recursive https://github.com/espressif/esp-idf.git
   cd esp-idf
   install.bat esp32c6
   ```

### Linux / macOS

```bash
git clone -b v5.4.2 --recursive https://github.com/espressif/esp-idf.git
cd esp-idf
./install.sh esp32c6
```

> **Important:** Always use ESP-IDF **v5.4.2** — other versions may conflict with the toolchain.

---

## Step 2 — Set up the environment

Every time you open a new terminal you must activate the ESP-IDF environment first.

### Windows
```cmd
cd path\to\esp-idf
export.bat
```

### Linux / macOS
```bash
cd path/to/esp-idf
. ./export.sh
```

---

## Step 3 — Copy the project files

Navigate to the border router example:

```
cd examples/openthread/ot_br
```

Copy the following files from this repository into the example folder:

| File | Destination |
|------|-------------|
| `partitions_4mb_otbr.csv` | `examples/openthread/ot_br/` |
| `sdkconfig.defaults` | append to `examples/openthread/ot_br/sdkconfig.defaults` |
| `main/esp_ot_br.c` | replace `examples/openthread/ot_br/main/esp_ot_br.c` |

> **For sdkconfig.defaults:** Open the existing file and paste the contents of this repo's `sdkconfig.defaults` at the **bottom**. Do not replace the whole file.

---

## Step 4 — Configure WiFi and target

Set the build target:
```
idf.py set-target esp32c6
```

Open the configuration menu:
```
idf.py menuconfig
```

Navigate to **Example Configuration** and set your WiFi SSID and password.

Also verify:
- `Component config → OpenThread → Thread Core Features → Thread 15.4 Radio Link` → **Native 15.4 radio**
- `Serial flasher config → Flash size` → **4 MB**

Save with **S**, quit with **Q**.

---

## Step 5 — Build and flash

Connect your XIAO ESP32-C6 via USB, then:

### Windows
```cmd
idf.py build flash -p COM11 monitor
```
(replace `COM11` with your actual COM port — check Device Manager)

### Linux / macOS
```bash
idf.py build flash -p /dev/ttyUSB0 monitor
```
(replace `/dev/ttyUSB0` with your actual port)

The build takes ~10 minutes the first time. After flashing, watch the monitor output. After ~20 seconds you should see the device connect to WiFi and get an IP address.

---

## Step 6 — Verify the REST API

Once connected to WiFi (check your router's DHCP list for the ESP's IP), open these URLs in your browser:

```
http://<ESP_IP>:8080/node
```
Should return: `{"State":4}`

```
http://<ESP_IP>:8080/networks/dataset/active
```
Should return: `{"ActiveDataset":"0e08..."}`

If both work, your border router is ready.

---

## Step 7 — Configure Home Assistant

### Add the Thread network dataset

1. In HA go to **Settings → System → Thread**
2. Your ESP border router should appear — click it and add the Thread dataset from `http://<ESP_IP>:8080/networks/dataset/active`

### Commission your Matter device

1. Install the **Home Assistant Companion app** on your phone
2. Go to **Settings → Companion app → Troubleshooting → Sync Thread credentials**
3. Go to **Settings → Devices & Services → Add Integration → Matter**
4. Scan the QR code on your device or enter the setup code
5. Follow the prompts — the device will join the Thread network and appear in HA

---

## Network Notes

Thread/Matter requires proper **end-to-end IPv6 connectivity**. Several common network setups need extra configuration:

### NAT66
**Disable NAT66** on your router. Matter commissioning uses direct IPv6 and will fail if packets are being NAT'd.

### Home Assistant in a VM
Make sure the VM uses **bridged networking** (not NAT). The VM must be on the same network segment as the ESP border router.

### Static route for Thread prefix
The ESP advertises the Thread network prefix (`fd55:ec6e:b588::/48` by default) via Router Advertisements. If your router blocks or doesn't propagate these, add a static IPv6 route:

- **Destination:** `fd55:ec6e:b588::/48` (check your actual prefix from the dataset)
- **Gateway:** ESP's link-local IPv6 address (visible in router's neighbor table)

### pfsense specific
1. Add a static IPv6 gateway pointing to the ESP's link-local address
2. Add a firewall rule on LAN (above the default IPv6 rule) routing `fd55:ec6e:b588::/48` via the ESP gateway
3. Disable NAT66

---

## Troubleshooting

### WiFi won't connect
- Check SSID and password in menuconfig
- The onboard antenna is small — keep the ESP within 5m of your router
- Signal below -85 dBm causes instability with Thread coexistence

### Build fails with "Tool doesn't match supported version"
You have multiple ESP-IDF versions installed. Always open a **fresh terminal** and run `export.bat` / `. ./export.sh` from your **cloned v5.4.2** folder before running any `idf.py` command.

### LAN IPv6 stops working when ESP is powered on
The ESP's border routing manager sends Router Advertisements on WiFi which can conflict with your router. The `suppress_backbone_ra_task` in the firmware handles this automatically after 20 seconds. If you still see issues, check that `otBorderRouterRemoveOnMeshPrefix` is being called successfully in the logs.

### Matter commissioning fails with "PASESession timed out"
- Check that NAT66 is disabled
- Verify the static route to the Thread prefix is in place
- Check firewall rules aren't blocking IPv6 between HA and the Thread prefix
- Make sure HA has a working IPv6 address and default gateway

### "No Thread border router" error during commissioning
- Verify `/node` returns `{"State":4}` — if State is 1, Thread hasn't attached yet, wait 30 seconds and try again
- Do **Settings → Companion app → Troubleshooting → Sync Thread credentials** on your phone before each commissioning attempt

### ESP crashes after ~30 seconds (SW_CPU reset)
This was caused by calling `otBorderRoutingSetEnabled()` too early. Make sure you are using the version of `esp_ot_br.c` from this repository which uses `otBorderRouterRemoveOnMeshPrefix()` instead.

---

## How it works

The ESP32-C6 contains both a 2.4GHz WiFi radio and an 802.15.4 Thread radio on the same chip, sharing a single RF path. The ESP-IDF `ot_br` example connects these together to form a border router.

On top of the base example, this project adds:

1. **OTBR-compatible REST API** — Home Assistant's Matter integration expects specific endpoints (`/node`, `/networks/dataset/active`) that the base example doesn't provide. We add a minimal HTTP server implementing these endpoints.

2. **RA suppression** — By default the border routing manager floods the WiFi network with Router Advertisements, breaking existing IPv6 setups. We remove the on-mesh prefix after initialization to prevent this.

3. **4MB partition table** — The default partition layout assumes 8MB+ flash. This custom layout fits everything into 4MB by removing OTA and RCP update slots.

---

## Known limitations

- **Single RF path:** WiFi and Thread share one antenna. This reduces throughput compared to a two-chip setup (e.g. ESP32-S3 + ESP32-H2). Fine for sensor polling, not ideal for high-bandwidth devices.
- **No RCP auto-update:** Disabled to save flash space.
- **No web GUI:** Disabled to save flash space.
- **WiFi credentials in NVS:** Stored in flash. The backup `.bin` file contains your credentials — don't share it.

---

## Backup and restore

To back up the full flash (including WiFi credentials and Thread dataset):
```
python -m esptool --chip esp32c6 -p <PORT> read_flash 0x0 0x400000 backup.bin
```

To restore:
```
python -m esptool --chip esp32c6 -p <PORT> write_flash 0x0 backup.bin
```

> Keep your backup `.bin` private — it contains your WiFi password.

---

## Contributing

This is an early proof of concept. Known areas for improvement:

- ESPHome integration (would make deployment much easier)
- Two-chip variant with dedicated Thread radio for better RF performance
- Automatic IPv6 route advertisement to upstream routers
- OTA update support

Pull requests welcome!

---

## Credits

Built with [ESP-IDF](https://github.com/espressif/esp-idf) and the `openthread/ot_br` example as a base. Thanks to the Home Assistant and OpenThread communities.

---

## License

The modifications in this repository are released under CC0 (public domain), matching the license of the original Espressif example code.
