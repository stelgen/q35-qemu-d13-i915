# q35-qemu-d13-i915 — полный дамп расследования

**Intel N150 (Alder Lake-N "Twin Lake", 8086:46d4) iGPU passthrough → Debian 13 VM, Proxmox VE 9, machine q35.**

Период: 19.09.2026 — 20.09.2026. Цель: удалённый GNOME-десктоп (консоль из морды PVE) + полный QSV/VA-API транскод в одной ВМ. 40+ итераций за 2 дня.

Все персональные данные (имена, хостнеймы, IP, MAC, UUID, ключи) удалены. Аппаратные ID (8086:46d4, 1234:1111) оставлены — без них дамп бесполезен.

---

## TL;DR (итог)

- ✅ **QSV/VA-API полный стек**: VA-API encode 4.71x, QSV/MFX 3.03x realtime. Фикс = пакет `libmfx-gen1.2`
- ✅ **3D GL в госте**: iris на проброшенном i915 (`glxinfo: Mesa Intel ADL-N`, GL 4.6)
- ❌ **Wayland GNOME (mutter 48.3) → noVNC на q35+std-VGA**: глухой. Рут-козь: **bochs-drm не обновляет VRAM для PRIME-imported framebuffer** (родные dumb-буферы блитятся, импортированные — нет)
- ✅ **Финал: откат на базу i440fx + qxl/SPICE + X11** — конфигурация, работавшая годами
- 🐛 Побочные баги по пути: зомби systemd-logind, мёртвый env-override mutter, gdm-runtime-config в неочевидном месте

---

## 1. Стек

| Уровень | Состав |
|---|---|
| Железо | N150 mini-PC (LPDDR5, 5×NVMe ZFS) |
| Хост | PVE 9.2.11, kernel 7.0.14-15-pve, QEMU 11.0.3-3 (pve-qemu-kvm), SeaBIOS |
| Host cmdline | `intel_iommu=on iommu=pt` (исправлено с `iommu=soft` + опечатка `i915. force_probe`) |
| ВМ | q35 (финал: откат на i440fx), `vga std` (bochs-drm, 1234:1111), `hostpci0 0000:00:02.0,pcie=1,rombar=1`, serial0 socket, 3 vCPU, 8GB, virtiofs |
| Гость | Debian 13, kernel 6.16.3+deb13 (апгрейд с 6.12.107 дисплей-проблему не изменил), GNOME 48 / mutter 48.3, Mesa 25.0.7, Xorg (modesetting) |
| Гость grub | `console=ttyS0,115200 console=tty0 transparent_hugepage=never swiotlb=131072 plymouth.enable=0` |
| QEMU device-факты | VGA std на pcie.0 addr=0x1; vfio-pci 00:02.0 → guest 01:00.0; VNC unix-socket; `OpRegion detected on Intel display 46d4` |

Базовая рабочая конфигурация (годами до проекта): **i440fx + SeaBIOS + qxl/SPICE + passthrough iGPU**.

---

## 2. Хронология фаз

### Фаза 0 — хост (19.09)

- `iommu=soft` конфликтовал с passthrough → `intel_iommu=on iommu=pt` (GRUB UEFI+LVM, update-grub).
- `/etc/modprobe.d/vfio-bind.conf` содержал мусорный ID **8086:4642** (не существует) → удалён; реальный GPU биндится динамически через PVE PCI mapping.

### Фаза 1 — «q35 падает» = фейковые смерти

- «q35 умирает при буте» = **тихая консоль** (`quiet` скрывает всё, кроме ошибок) + клин qxl. Не смерть.
- **qxl + проброс i915** = `ttm memset soft lockup` (ttm_move_memcpy←qxl_bo_create←qxl_draw_dirty_fb) + qxl_ring_push D-state → qxl на q35 закрыт.
- **virtio-gpu** = завис на буте после `Initialized virtio_gpu` (features -virgl -resource_blob -host_visible).
- Спасательные меры: serial0 socket (`qm terminal 100`), маскированы plymouth-юниты (зависший plymouthd держал DRM-master и блокировал plymouth-quit → GDM не стартовал), `plymouth.enable=0`.
- `monitors.xml` с фантомным режимом **1961x1098** от qxl-эпохи удалён (mutter не мог применить).
- Шум, не ошибки: `shpchp -16` (PVE bug 6909), `i915 VT-d active` (норма в госте), `Unknown kernel command line parameters BOOT_IMAGE` (норма), `Cannot find any crtc or sizes` (норма для passthrough без мониторов).
- Ядро гостя 6.12.107 → 6.16.3: без изменений в display-поведении.

### Фаза 2 — фиксация display-мистерии

- **fbcon доходит до noVNC** (boot-текст бежит) — труба QEMU↔bochs доказана.
- После старта Wayland-сессии: VNC **замирает на последнем fbcon-кадре**. CRTC активен, Virtual-1 connected, режим установлен, fb «от gnome-shell» — кадр не идёт.
- Ранняя попытка X11 на bochs: «glamor refuses llvmpipe → SIGILL crash-loop» — позже опровергнута: в юзер-сессии Xorg аккуратно отказался от glamor на llvmpipe и поднялся.

### Фаза 3 — ресерч исходников (суб-агенты)

Разобраны: mutter 48.3 (meta-renderer-native.c, meta-onscreen-native.c, meta-render-device-gbm.c, meta-backend-native.c, doc/multi-gpu.md, doc/debugging.md), Mesa 25.0.4/main (gbm_dri.c, loader.c, platform_drm.c, pipe_loader_drm.c, kms_dri_sw_winsys.c), ядро 6.16 (drivers/gpu/drm/tiny/bochs.c, DRM_GEM_SHMEM_DRIVER_OPS), virtgpu_drv.c, QEMU 11 stable (vga.c, vga-pci.c). Срезы исходников — в `docs/sources/`.

Ключевые факты:

1. **Первичный GPU mutter**: udev-тег `mutter-device-preferred-primary` → builtin panel → platform device → boot_vga; сначала только hardware-рендер-кандидаты. `ID_PRIMARY_GPU` udev-свойство mutter **не читает** (это gdm/Xorg-логика).
2. **MUTTER_DEBUG_MULTI_GPU_FORCE_COPY_MODE — мёртвый код для non-accelerated secondary**: env читается только в `set_default_secondary_gpu_copy_mode()` (accelerated-путь); `init_secondary_gpu_data_cpu()` хардкодит `copy_mode=ZERO` без getenv. Подтверждено эмпирически: env был в сессии, а state показал imported=yes (ZERO).
3. **Mesa 25.0.4 gbm_dri.c — автоматический фолбэк**: `dri_screen_create` (bochs → fail) → `dri_screen_create_sw` → **kms_swrast** (llvmpipe + dumb-буферы на том же fd; render-нода не нужна). EGL принимает software-GBM. `eglQueryDevices`/GLX card0 без renderD **не видят** — прошлая диагностика «kms_swrast не поднялся» была методом-невидимкой.
4. **bochs-drm 6.16**: GEM SHMEM + shadow-plane; PRIME import/export полный (`prime_import_sg_table`); VRAM обновляется **memcpy из GEM-буфера** в atomic_update (всегда VRAM offset 0); fb через drm_gem_fb_create_with_dirty; fake-vblank сразу.
5. **virtio-gpu без virgl = тот же llvmpipe** для mutter (имя `virtio_gpu` в таблице Mesa = virgl-драйвер) + баги: multi-VGA vgaarb бут-хенг (фикс 5dd8b53 в 6.15), suspend/resume сломан до коммитов после 6.16 (cad6a87/78e434b), QEMU <11.0.5 guest-triggered abort (бэкпорт g_try_malloc).
6. Симлинк-хак `bochs_dri.so` мёртв (Mesa 22+ всё в libgallium, выбор по таблице имён). `GBM_ALWAYS_SOFTWARE=1` / `MESA_LOADER_DRIVER_OVERRIDE` глобальны — убили бы iris на i915.
7. Публичного конфига «GNOME Wayland + noVNC + iGPU passthrough на q35» **не существует** — недокументированная комбо-зона.

### Фаза 4 — зомби systemd-logind (корень «всё сломалось»)

- Симптомы: gdm active, сессий нет, журнал пуст, `loginctl: Connection timed out`, `0 users`, XDG_RUNTIME_DIR отсутствует.
- Диагноз: logind **active/running, но не отвечает по D-Bus** (зомби с бута).
- Фикс: `systemctl restart systemd-logind` — мгновенно; сессии зарегистрировались, GDM поднял автологин.
- Урок: вся «поломка системы» была одним зомби-демоном.
- После рестарта gdm поднялась **X11**-сессия: oneshot-юнит gdm-wayland-force при ручном рестарте не выполнялся, runtime-config отсутствовал.

### Фаза 5 — QSV: симптом → фикс

- `vainfo` iHD 25.2.3 — жив; первый QSV-фейл был от голого SSH-окружения (XDG_RUNTIME_DIR).
- `h264_vaapi` (ffmpeg, vaapi-девайс напрямую): **4.71x realtime** — медиа-стек жив.
- `h264_qsv` (MFX через vaapi-child): `Error creating a MFX session: -9` — у юзера и root, даже с XDG.
- dpkg: `libvpl2` (диспетчер) есть, **`libmfx-gen1.2` (oneVPL GPU-runtime) отсутствовал**.
- Фикс: `apt install libmfx-gen1.2` (25.1.4-1) → **h264_qsv 3.03x realtime (92 fps @ 720p30)** без ребута.
- Вывод: **ffmpeg-QSV до фикса никогда не работал**; исторический «3.17x» был h264_vaapi; приложения (Jellyfin/frigate) всё это время жили на VA-API — потому и не замечали.

### Фаза 6 — форензика X11-сессии

- Xorg.0.log: `modeset(0)` = BochsCard (modesetting), «Refusing to try glamor on llvmpipe» → аккуратный отказ, БЕЗ SIGILL; `modeset(G0)` = **glamor X acceleration enabled on Mesa Intel ADL-N** — PRIME-рендер на проброшенном i915. Xorg сам собрал multi-GPU схему из коробки.
- debugfs: `fb allocated by = Xorg, imported=NO` (родной dumb), CRTC active 1024x768.
- Скриншот через `import -window root`: **2359 уникальных цветов** — Shell полностью нарисован.
- X11-стек внутри гостя: полностью рабочий и 3D-ускоренный (glamor на i915). Но noVNC всё ещё глухой → разрыв «bochs VRAM → QEMU → noVNC» или VNC-клиент.
- Открытие: `gdm-runtime-config` — ELF-бинарь, пишет в **`/run/gdm3/custom.conf`** (мы смотрели в /run/gdm/ — не туда).

### Фаза 7 — Wayland + муттер-форензика

- `start gdm-wayland-force && restart gdm` → **Wayland-сессия поднялась** (автологин + greeter).
- Env подтверждён в gnome-shell: force-copy + `MUTTER_DEBUG=render,kms` + `G_MESSAGES_DEBUG=all`.
- Отработан надёжный **QMP screendump** (python unix-socket; `qm monitor` ненадёжен при вставках).
- Screendump: **2 цвета**, режим кадра **1280x800** (preferred mutter; fbcon был 1024x768) → **modeset mutter дошёл до QEMU** — регистровый путь жив.
- debugfs: `fb=38, allocated by gnome-shell, imported=yes` → mutter сканаутит **ZERO-copy PRIME-импорт с i915**. forced env проигнорирован (подтверждение факта 2 фазы 3).
- **ftrace**: `drm_vblank_event_delivered` — только seq=0, ровно каждые 60.0с (idle-тики), **ни одного page-flip после initial**.
- Журнал муттера молчит даже с debug-переменными → **Debian release-сборка вырезает debug-топики** (stderr → journald-сокет, но пусто).
- Тестовые окна (xclock/gedit) из SSH не поднялись (Xauthority/отсутствие) — damage-тест не состоялся.

### Фаза 8 — VRAM-гипотеза и её крах

- Расчёт: fbcon ~4.1MB + 3 dumb-фолбэка mutter ~12.3MB = **16.4MB > 16MB default VRAM std-vga** (по докам QEMU).
- `vga: std,memory=128` через морду PVE → в cmdline **не транслируется** (showcmd пуст) — PVE не проводит memory= для std.
- Попытка `--args '-global VGA.vga_mem_mb=128'` → **`Property 'VGA.vga_mem_mb' not found` → старт ВМ упал** (QEMU 11 pve такого свойства не имеет). Аргументы сняты, ВМ возвращена.
- **VRAM-гипотеза снята за отсутствием доказательств**. Задокументировано как антипаттерн: свойство не верифицировали против конкретной сборки QEMU перед выдачей.
- modetest-грабли: модуль = **bochs-drm** (не bochs); `-D` в этой сборке = busid, не путь; верный таргет `Virtual-1 id=37, CRTC 35, plane 33`; при живом gdm — Permission denied (DRM master); неверный коннектор id=34 → «failed to find mode»; «failed to create dumb buffer EINVAL» — артефакт неверных параметров, не VRAM.

### Фаза 9 — финальная теория: imported-fb scanout

Пересборка ВСЕХ улик в одну картину:

- Глазами: «картинка, висящая на буте» = **VRAM до сих пор содержит последний fbcon-кадр**.
- Screendump «2 цвета» = **белый текст на чёрном = тот самый fbcon-кадр**, а не «пустой экран».
- mutter сканаутит **imported** (PRIME dma-buf с i915) fb; ядро bochs обновляет VRAM memcpy'ем **только из родных GEM-объектов** (fbcon — родной dumb, виден всегда; X11-fb — родной, imported=NO).
- **Вердикт**: bochs-drm 6.16 не обновляет VRAM для PRIME-imported framebuffer → mutter ZERO-copy на software-secondary обречён в этой связке; Wayland-путь сломан на стыке ядро+композитор.
- Тест X11 (`WaylandEnable=false`, без wayland-force, restart gdm): Xorg стартовал, но `Xorg.wrap` ушёл в **D-state** (ядро в плохом состоянии после десятков рестартов сессий) → нужен был чистый ребут.

### Фаза 10 — откат на базу (финал)

- ВМ переведена на **i440fx + qxl/SPICE** (проверенная годами конфигурация).
- Гостевые чистки q35-наследия: удалены `/etc/X11/xorg.conf.d/10-bochs.conf`, override-отладка gdm, `~/.config/environment.d/99-mutter-debug.conf`, строки MUTTER/G_MESSAGES из /etc/environment; `WaylandEnable=false` оставлен (X11 — рабочий путь для qxl); gdm-wayland-force отключён.
- **QSV-фикс не зависит от дисплея** (renderD128) и остаётся: VA-API 4.71x + QSV/MFX 3.03x.

---

## 3. Мёртвые пути (проверено — НЕ повторять)

| Путь | Результат |
|---|---|
| qxl + проброс i915 на q35 | ttm memset soft lockup + qxl_ring D-state |
| virtio-gpu (virgl off) | бут-хенг; для mutter = тот же llvmpipe; resume сломан до 6.17 |
| X11 greeter на llvmpipe (ранний тест) | SIGILL crash-loop в greeter-контексте (в юзер-сессии позже — аккуратный отказ) |
| i915.modeset=0 | deprecated в 6.12+, render-ноды нет |
| i915.enable_display=0 | параметр удалён из ядра 6.12+ |
| MUTTER_DEBUG_MULTI_GPU_FORCE_COPY_MODE | мёртвый код для non-accelerated secondary |
| ID_PRIMARY_GPU udev | mutter не читает |
| TAG mutter-device-preferred-primary на bochs | bochs без EGL/DRI → глухо |
| monitors.xml со старым режимом | фантом 1961x1098 ломал режимы |
| symlink kms_swrast→bochs_dri.so | мёртв (Mesa 22+, libgallium) |
| GBM_ALWAYS_SOFTWARE / MESA_LOADER_DRIVER_OVERRIDE | глобальны, убивают iris на i915 |
| `vga: std,memory=128` (морда PVE) | не транслируется в cmdline для std |
| `-global VGA.vga_mem_mb=N` | property не существует в QEMU 11 pve → старт ВМ падает |
| i440fx legacy IGD (GOP ROM) | ADL-N не поддерживается legacy-igd (SNB-CML only) |

---

## 4. Побочные баги и грабли (переиспользуемые знания)

- **Зомби systemd-logind**: active/running, но D-Bus мёртв → `loginctl timeout`, сессии не регистрируются, XDG_RUNTIME_DIR нет, GDM не создаёт сессию, журнал «пуст». Фикс: `systemctl restart systemd-logind`.
- **gdm-runtime-config** пишет в `/run/gdm3/custom.conf`; generate-config (ExecStartPre) мержит его при старте gdm.
- **oneshot-юниты не выполняются при ручном рестарте сервиса** — всегда пара: `start gdm-wayland-force && restart gdm`.
- **Debian mutter = release-сборка**: debug-топики вырезаны; журнал молчит при любых MUTTER_DEBUG/G_MESSAGES_DEBUG.
- **eglinfo -p gbm** всегда открывает первую render-ноду (i915) — card0 без renderD он не проверяет.
- **Xorg-логи гостя**: `~/.local/share/xorg/` (rootless X), не /var/log/Xorg.0.log.
- **Xorg.wrap D-state** после частых рестартов сессий — лечение только чистым ребутом; диагностика: `/proc/<pid>/stack`.
- **journal persistent** включается `mkdir /var/log/journal` — иначе после ребутов диагностика теряется.
- `gkr-pam: couldn't unlock the login keyring` при автологине — шум, не фейл.
- `Failed to start app-gnome-*scope` (keyring/at-spi) — шум сессии.
- qemu-процесс в PVE = `/usr/bin/kvm` (pgrep по 'qemu-system' не находит); CPU ~33% — фоновый QSV/docker, не криминал.
- `linux-image-6.12.105` — кандидат на autoremove после апгрейда ядра.

---

## 5. Тулбокс диагностики (всё проверено в бою)

```bash
# KMS-состояние bochs: кто держит fb, imported?, CRTC active?
sudo cat /sys/kernel/debug/dri/0/state | head -40

# флипы ядра в реальном времени (page-flip активность):
sudo sh -c 'echo 1 > /sys/kernel/debug/tracing/events/drm/enable'
sudo timeout 20 cat /sys/kernel/debug/tracing/trace_pipe | grep -aE "vblank|flip"
# интерпретация: seq=0 каждые 60с = idle-тики; растущий seq = реальные кадры

# QMP screendump (надёжно; qm monitor ненадёжен при вставках):
python3 - <<'EOF'
import socket, json
s = socket.socket(socket.AF_UNIX); s.connect("/var/run/qemu-server/100.qmp")
f = s.makefile("rw"); f.readline()
def cmd(**a):
    s.sendall(json.dumps(a).encode()+b"\n"); return f.readline().strip()
cmd(execute="qmp_capabilities")
print(cmd(execute="human-monitor-command", arguments={"command-line":"screendump /tmp/s.ppm"}))
EOF
# подсчёт цветов: 2 = fbcon-текст (НЕ «чёрный экран»!), >100 = живая картинка
python3 -c "d=open('/tmp/s.ppm','rb').read(); i=d.index(b'255\n')+4; px=d[i:]
print('цветов:', len(set(zip(px[0::3],px[1::3],px[2::3]))))"

# скриншот живого X-десктопа по SSH:
DISPLAY=:0 XAUTHORITY=/run/user/1000/gdm/Xauthority import -window root /tmp/shot.png
convert /tmp/shot.png -format '%k\n' info:

# modetest: модуль = bochs-drm; Virtual-1 id=37, CRTC 35, plane 33;
# при живом gdm — Permission denied (master занят)
sudo modetest -M bochs-drm -c

# env-проверка сессии:
sudo cat /proc/$(pgrep -n gnome-shell)/environ | tr '\0' '\n' | grep -aE "MUTTER|G_MESSAGES"

# где Xorg висит (D-state):
sudo cat /proc/<pid>/stack

# QSV-тесты:
ffmpeg -hide_banner -init_hw_device vaapi=va:/dev/dri/renderD128 \
  -f lavfi -i testsrc2=size=1280x720:rate=30 -vf 'format=nv12,hwupload' \
  -c:v h264_vaapi -frames:v 120 -f null - 2>&1 | tail -3
export XDG_RUNTIME_DIR=/run/user/$(id -u)
ffmpeg -hide_banner -init_hw_device vaapi=va:/dev/dri/renderD128 \
  -init_hw_device qsv=hw@va -f lavfi -i testsrc2=size=1280x720:rate=30 \
  -vf 'format=nv12,hwupload' -c:v h264_qsv -b:v 2M -frames:v 300 -f null - 2>&1 | tail -4
```

---

## 6. Рут-козь (финальный, с уровнем уверенности)

**Wayland-путь mutter 48.3 в связке i915(primary render) + bochs-drm(secondary scanout) не выводит кадры: bochs-drm 6.16 обновляет VRAM memcpy'ем только из родных GEM-объектов; ZERO-copy PRIME-imported fb (который mutter использует для software-secondary) в VRAM не попадает.** fbcon/X11 (родные dumb-буферы) — работают; потому noVNC вечно показывает fbcon-кадр бута.

Уверенность: высокая по механике «VRAM = fbcon-кадр» (глаза юзера + 2-цветный screendump + imported=yes в state); точный отказ (memcpy не вызывается / не работает на foreign sgt) требует подтверждения — QMP `pmemsave` VRAM или патч-эксперимент.

---

## 7. Инцидент-урок процесса

`-global VGA.vga_mem_mb=128` был дан без верификации против конкретной сборки QEMU 11 (pve) → старт ВМ упал (`Property not found`). Урок: **любое свойство QEMU/флаг PVE проверять (`-device VGA,help` / `qm showcmd`) до выдачи**. VRAM-гипотеза при этом снята по существу — доказательств нехватки VRAM не было.

---

## 8. Статус и роадмап

**Статус: откат на базу i440fx + qxl/SPICE + X11 (рабочая конфигурация). QSV/VA-API полный (VA-API 4.71x + MFX 3.03x). 3D в госте: iris/glamor на i915.**

Open items:

1. Верификация X11 после чистого ребута ВМ (screendump >2 цветов на q35, если вернуться) — не завершена из-за D-state Xorg.
2. Прямое доказательство imported-fb: QMP `pmemsave` дамп VRAM (адрес BAR2 из lspci гостя) — сравнить с содержимым dumb.
3. Wayland-десктоп без сканаута: gnome-remote-desktop headless RDP (GNOME 46+: `grdctl --system rdp enable`, mutter --headless --virtual-monitor) — не зависит от bochs вообще.
4. Upstream-репорты: (a) bochs-drm: foreign dma-buf fb не обновляет VRAM; (b) mutter: force-copy env dead code на software-secondary; (c) PVE: vga memory= не транслируется для std.
5. Ресерч virtio-gpu с virgl (host side) — отдельная ветка, не начиналась (бут-хенг).

---

## 9. Ссылки

- mutter multi-gpu doc: gitlab.gnome.org/GNOME/mutter — `doc/multi-gpu.md`
- MR copy-mode override: mutter!4251 (Phoronix: "GNOME Mutter Adds Debug Option To Override Multi-GPU Copy Mode")
- QEMU Standard VGA (16MB fb): qemu.org/docs/master/specs/standard-vga.html
- bochs GEM SHMEM + shadow-plane: dri-devel «drm/bochs: Use GEM SHMEM helpers» (2024)
- Исходные срезы для этого расследования: `docs/sources/`
- Дословные артефакты конфигурации этапа (host/VM/guest, udev-эксперименты, systemd-юниты, замеры): `docs/final-configs.md`
