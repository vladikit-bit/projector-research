# Пристрої дослідження

## Основний: проектор
- **Thundeal TD.98 Pro**, модель C50A (рекон-скрипт 2026-08-08 назвав його TD98 Pro; у файлах також згадується alias «H50»)
- SoC **MStar MT5889** (аудіо-підсистема MT9615-alias), платформа з ознаками TCL-заводу T615T
- **Android 11, kernel 4.19.116** (за звітами дослідження; застереження: окремі getprop-знімки містять 4.4.4-fingerprint — див. CROSS_REFERENCES §4)
- ODM-прошивка **Ntech** (ntechserver, NtechSettings/NtechSystemUI/NtechLocalMedia)
- Імагер: DLP DLPC6540 1080p-клас + XPR pixel-shift → 3840×2160 адресація; панель CSOT UD_VB1_16LANE (4K INI), в образі є 3D-DLP INI 120 Гц
- HDMI: без TX (SupportHdmiTxCount=0; строкові 4K-RES у SoC — є); HDMI-RX з 60 EDID-профілями
- root: userdebug/test-keys, `adb root` працює

## Побічні пристрої (не плутати!)
- **RK3328 Android 10 TV-box** (device `H50_box_32`, build.prop у platform-tools) — окрема лінія (bt-switcher, MX9 Pro dts/dtb на C:\)
- **TCL T615T V083/V098/V474** — донорські прошивки для дифів → `tcl-t615t-firmware-analysis`
- AVR стенда: **Pioneer VSX-817** (оптичний SPDIF)
