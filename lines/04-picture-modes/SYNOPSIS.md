# Лінія 04 — режими картинки: від 1080p 50/60 до справжніх можливостей ⏳ (патч спроєктований, не згенерований)

**Статус: корінь доведений бінарно; патч повністю специфікований і НЕ виконаний.**

## Підсумок
- **Що Android бачить**: рівно 2 режими — 1920×1080@50 (літерал 20 000 000 нс) і @60 (глобал 0x00FE502A = 16 666 666 нс) — **два addConfig у конструкторі HWCDisplayDevicePrimary, літерали @ file offset 0x3BC88** (`INDEPENDENT_HY4_HY3_DISPLAY_FORENSIC_REVIEW`). EDID, панельні INI, PcModeTimingTable, SKU — не беруть участі.
- **Що платформа вміє**: повна 37-рядкова таблиця таймінгів у HWC .rodata — 1080p@24/25/30 (enum 6–12), 4K@24/25/30/50/60 (13–17), 4096×2160@24–60 (18–22)… **нічого вище 60 Гц** (4K120/1080p120 спростовані).
- **Тип імагера**: 1080p-клас DLP (DLPC6540) + XPR pixel-shift до 3840×2160; `m_wPanelWidth=3840` НЕ доводить нативні 4K.
- **Хронологія вердиктів**: «двох HWC-патчів досить» → СПРОСТОВАНО незалежним рев'ю (setActivePanelFrequency мапить усе≠0 → FreeRunConfig 2 = 60 Гц; VRR недосяжний — лише cfg 0, який HWC ніколи не шле) → справжній шлагваут таймінгу = ioctl `0xc008122c` на `/dev/mik!disp` (payload {0x4354, timingEnum}), який HWC ніколи не викликає.
- **Фінальний дизайн патчу** (`PATCH_FEASIBILITY_FINAL.md`): метод A, HWC сам відкриває `/dev/mik!disp`; C1 = cave ~100 Б (4× addConfig 4K 24/30/50/60), C2 = cave ~70–80 Б (мап cfgId→timingEnum + ioctl). ~230–260 Б дельти, libmi3/kernel не чіпаються. **Hard STOP — авторизація на генерацію не давалась.**
- **Runtime-фактор**: `tv_input_hdmi_edid_version` — рантайм-глобал, який пінає EDID 1.4 поверх конфігу 2.1 (пояснює «чому не 4K» без патчів для HDMI-RX).
- **Два шари налаштувань**: Android `tv_picture_*` писабельні, vendor `g_video__*`/`g_fusion_picture__*` ні → PQ «Save» — no-op, Color Space ресетиться в Auto.
- **Factory menu** (`FACTORY_MENU_CHEATSHEET.md`): PWM floor 29.9%, m_PanelBitNums=2 (10 біт), CUSTOMER_PQ криві — база для DV/«молока»-роботи (лінія 03).

## Файли
12 документів `display_edid_investigation/` (в т.ч. 117 КБ `REPORT_EDID_DISPLAY_MODES.md` — 5 документів в одному) + FINDING_edid_*/hwc/modes/video_settings + cheatsheets.

## Перехрестя
- VRR/HDR капс (MI_DISP_GetCaps bitmap) → лінія 03.
- HDMI-RX EDID-біни (60 профілів) — витягнуті, локально + 2 шт. в platform-tools.
