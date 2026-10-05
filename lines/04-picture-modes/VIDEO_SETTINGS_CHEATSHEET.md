# Шпора: «Розширені налаштування відео» — TD98 Pro / C50A

**Порядок як у меню.** `EN` — оригінальна назва вендора (українські підписи — кривий переклад).
Усі значення живуть у `settings` namespace **`global`**: `settings get global <key>` /
`settings put global <key> <value>` (працює з root). Стан знято 2026-10-01.

---

### 1. Color Temperature · «Температура кольорів» · `picture_color_temperature=2`
**Підменю:** Standard colours / **Custom** + **Red / Green / Blue boost** (`picture_red_gain`,
`picture_green_gain`, `picture_blue_gain` = 0/0/0, діапазон до −50).
*Що це:* підгонка балансу білого — підсилення окремих каналів.

### 2. Dolby Vision Notification · «Сповіщення Dolby Vision» · `sound_dolby_notification=1` · **TOGGLE (ввімкнено)**
**Вмикає/вимикає:** баннер «Dolby Vision» на екрані, коли грається DV-контент. Це **лише сповіщення** — сам Dolby Vision цим не вмикається.

### 3. (Dynamic) Noise Reduction · «Динамічне зменшення шуму» · `tv_picture_advance_video_dnr=4`
**Режими:** Off / Low / Medium / High / **Auto** (зараз Auto).
*Що це:* прибирає шум між кадрами; Auto оцінює його сам.

### 4. (MPEG) Noise Reduction · «Зменшення шумів MPEG» · `tv_picture_advance_video_mpeg_nr=2`
**Режими:** Off / Low / Medium / High (зараз Medium).
*Що це:* прибирає «косички» і блочність від стиснення відео.

### 5. Sharpness · «Найвища чіткість» · `picture_sharpness=4`
**Режими:** Off / On — **усього два** (зараз Off).
*Що це:* підсилення країв. Яскравіше ≠ чіткіше: робить картинку «різкішою» з артефактами.

### 6. Adaptive Luma Control · «Адаптивне керування яскравістю» · `tv_picture_video_adaptive_luma_control=2`
**Режими:** Off / Low / Medium / High (зараз Medium).
*Що це:* частина DV-конвеєра — локально підлаштовує яскравість під сцени.

### 7. Local Contrast Control · «Керування локальним контрастом» · `tv_picture_video_local_contrast_control=2`
**Режими:** Off / Low / Medium / High (зараз Medium).
*Що це:* локальний контраст (яскравіше світле, темніше темне) без зміни яскравості.

### 8. Dynamic Color Booster · «Dynamic Color Booster» · ключа немає в settings · **Off**
**Режими:** Off / Low / Middle / High.
*Що це:* «бустер» кольору DV — підсилює насиченість/барвистість на локальних ділянках.

### 9. Director Mode · «Режим режисера» · ключа немає
**Режими:** Director Mode / **Автоперемикання**.
*Що це:* режим «як у режисера» — вимикає майже всю обробку і шанує дані майстер-публікації. Автоперемикання вмикає його лише на DV.

### 10. Flesh Tone · «Відтінок шкіри» · `tv_picture_video_flesh_tone=0`
**Режими:** Off / Low / Medium / High (зараз Off).
*Що це:* тримає природний тон шкіри в обробці.

### 11. Film Mode · «Режим кіно DI» · `tv_picture_video_di_film_mode=0`
**Режими:** Off / **SLOW_PIC** / **ACTION_PIC**.
*Що це:* кінематографічний режим руху: повільніші переходи / «плівковий» фільм.

### 12. Blue Stretch · «Вирівнювання синього» · `tv_picture_video_blue_stretch=0` · **TOGGLE**
**Вмикає/вимикає:** розтягування синього каналу (корекція білого на вищих яскравостях).

### 13. Gamma · «Гама» · `picture_gamma=2`
**Режими:** Dark / Medium / Light (зараз Medium).
*Що це:* крива гами — світлотінь.

### 14. Game Mode · «Режим гри» · `tv_picture_video_game_mode=0` · **СІРИЙ у Android, працює в OSD проектора** · TOGGLE
**Вмикає/вимикає:** ігровий режим із низькою затримкою (зменшує input lag).

### 15. ALLM · `tv_picture_video_allm=0` · **TOGGLE**
**Вмикає/вимикає:** Auto Low Latency Mode — домовленість HDMI із джерелом про автоматичне зниження затримки (для ігор).

### 16. PC Mode · «Режим ПК» · `tv_picture_video_pc_mode=0` · **СІРИЙ у Android, працює в OSD** · TOGGLE
**Вмикає/вимикає:** режим для роботи з комп'ютером (яскравість/контраст під робочу графіку).

### 17. De-Counter · `tv_picture_video_de_counter=0`
**Режими:** Off / Low / Medium / High (зараз Off).
*Що це:* анти-джуддер — «розгладжує» смики в русі.

### 18. AISR · `tv_picture_video_aisr=1`
**Режими:** Low / Medium / High (зараз Low) — **вимкненого стану немає**.
*Що це:* ШІ-апскейл (до-апскейл картинки алгоритмом).

### 19. MJC (MEMC) · «MJC» · `tv_picture_advance_video_mjc_effect=0`
**Ефект:** Off / Low / Medium / High (зараз Off).
**Розділення демо-режиму:** All / Right / Left — яка половина екрана показує демо.
**Демо-режим** — окремий показ.
*Що це:* компенсація руху — вставляє проміжні кадри (як у «розширених» LCD-телеках).
**Умова:** керується лише коли «Automatic playback optimization» вимкнено.

### 20. HDMI RGB Range · «Діапазон RGB для HDMI» · `tv_picture_video_hdmi_rgb_range=0` · **СІРИЙ**, значення Auto
*Що це:* full/limited RGB на виході HDMI.

### 21. Low Blue Light · «Слабке блакитне світло» · `tv_picture_video_low_bluelight=0` · **TOGGLE**
**Вмикає/вимикає:** фільтр синього світла / придушення ореолів.

### 22. Color Space · «Колірний простір» · `tv_picture_video_color_space=0`
**Режими:** **Auto** / Off / On / sRGB-BT.709 / **BT.2020** / Adobe RGB / DCI.
*Що це:* колірний простір на виході. Dolby Vision працює у **BT.2020**.

### 23. Automatic playback optimization · «Автоматична оптимізація відтворення» · `tv_picture_video_automatic_playback_optimization=0` · **TOGGLE**
**Вмикає/вимикає:** автоматичну оптимізацію відтворення. **Це не AFR.** Серед іншого керує
доступністю ручного керування MJC — але це лише один із наслідків, а не єдине призначення.

### 24. Dolby Vision PQ Calibration · «Калібрування Dolby Vision PQ» · ключа немає в settings
**Підменю:** **View Mode 0–9** (10 слотів) → **End-user calibration: Tmax, Tmin, screen Gamma,
Rx, Ry, Gx, Gy, Bx, By, Wx, Wy — усі 0.0**.
*Що це:* вимірювальні дані дисплея для DV PQ. **Нулі = панель ніколи не вимірювали** → тон-мапінг DV не має орієнтиру.

### 25. Light Sensor · «Датчик світла» · `tv_picture_video_light_sense=0` · **TOGGLE**
**Вмикає/вимикає:** датчик зовнішнього освітлення (автоматична яскравість).

### 26. 3D Mode · `tv_picture_advance_video_3d_mode=0` · **СІРИЙ (справді заблокований)** · TOGGLE
**Вмикає/вимикає:** обробку 3D (SBS / Top-Bottom).

### 27. 3D-to-2D · `tv_picture_advance_video_3d_to_2d=0` · **СІРИЙ** · TOGGLE
**Вмикає/вимикає:** перетворення 3D у 2D.

### 28. L/R Switch · `tv_picture_advance_video_3d_lr_switch=0` · **TOGGLE**
**Вмикає/вимикає:** перемикання лівого/правого ока в 3D.

### 29. Color Tuner · «Налаштування кольорів» · `tv_picture_color_tune_*` (31 ключ, усі 50)
**Підменю:** Enable + **Tint / Saturation / Brightness / Offset / Gain**.
За кожною групою — 7 компонентів: **Red, Green, Blue, Yellow, Magenta, Cyan, Flesh tone**
(hue/saturation/brightness), gain/offset — для RGB.
*Що це:* повна матриця грейдингу — ручний спосіб виправити колір, якщо DV-калібрування нульове.

### 30. 11 Point White Balance Correction · «Корекція балансу білого за 11 параметрами» · `picture_white_balance11_*`
**Підменю:** Enable / Gain (5%) / Red 50 / Green 50 / Blue 50.
*Що це:* корекція балансу білого. Попри назву — **самі 11 точок не редагуються**.

---

## Скоро: усі перемикачі (toggles) і що вони вмикають

| # | Toggle | Що вмикає/вимикає | Стан |
|---|---|---|---|
| 2 | Dolby Vision Notification | баннер DV | on |
| 12 | Blue Stretch | розтягування синього каналу | off |
| 14 | Game Mode | ігровий режим низької затримки | off (сірий у Android, є в OSD) |
| 15 | ALLM | авто-низька затримка HDMI | off |
| 16 | PC Mode | режим для роботи з ПК | off (сірий у Android, є в OSD) |
| 19 | MJC Effect + Demo | вставлення проміжних кадрів | off |
| 21 | Low Blue Light | фільтр синього / ореоли | off |
| 23 | Automatic playback optimization | автооптимізація відтворення; **готує MJC** | off |
| 25 | Light Sensor | датчик освітлення | off |
| 26 | 3D Mode | обробка 3D | off (справді заблокований) |
| 27 | 3D-to-2D | перетворення 3D→2D | off (заблокований) |
| 28 | L/R Switch | ліве/праве око в 3D | off |

## Три головні висновки

1. **DV PQ калібрування нульове** — єдине, що реально псує Dolby Vision. Решта — звичайні
   перемикачі обробки, які працюють.
2. **Color Tuner (31 параметр)** — ручний інструмент, яким можна замінити калібрування, якщо
   колориметра немає.
3. **Color Space має BT.2020** — режим, який Dolby Vision очікує; зараз стоїть Auto.