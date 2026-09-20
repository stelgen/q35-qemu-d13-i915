# Финальные артефакты конфигурации (дословно, из чата 19-20.09)

Дополнение к README: дословные содержимое файлов/команды, которые применялись.
Проверено в бою; часть работает только на q35-этапе (отмечено).

## Хост (PVE 9.2.11)

### /etc/default/grub (строка CMDLINE, хост)
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```
Изменения: `update-grub` + reboot. Проверка: `cat /proc/cmdline`.

### Удалённые файлы хоста
- `/etc/modprobe.d/vfio-bind.conf` — содержал `options vfio-pci ids=8086:4642` (ID не существует; реальный GPU 8086:46d4). После удаления — `update-initramfs -u -k all`.
- Бэкапы оставлены: `/etc/default/grub.bak`, `/etc/modprobe.d/vfio-bind.conf.bak`.

### VM 100 конфигурация q35-этапа (состояние на момент заморозки)
```
machine: q35
bios: seabios
vga: std
serial0: socket
hotplug: 0
hostpci0: 0000:00:02.0,pcie=1,rombar=1
```
Вариант с PCI-mapping: `hostpci0: mapping=Intel_N150_GPU` (равнозначен по результату).

### Откат на рабочую базу (проверено, годами)
```
qm set 100 --machine pc --vga qxl
qm start 100
```

## Гость (Debian 13)

### /etc/default/grub (гость)
```
GRUB_CMDLINE_LINUX_DEFAULT="console=ttyS0,115200 console=tty0 transparent_hugepage=never swiotlb=131072 plymouth.enable=0"
```
Грабля: одна из sed-правок «потерялась» — после каждой правки проверять:
`grep CMDLINE_LINUX_DEFAULT /etc/default/grub && sudo update-grub`

### Маскировка plymouth (лечение зависшего plymouthd с DRM-master)
```
sudo systemctl mask plymouth-quit-wait.service plymouth-quit.service plymouth-start.service
```
### /etc/systemd/system/gdm.service.d/override.conf
```
[Service]
Environment=MUTTER_DEBUG_MULTI_GPU_FORCE_COPY_MODE=primary-gpu-cpu
```
⚠️ `sudo systemctl edit gdm3` НЕ сохранил файл при пустом вводе
("after editing, new contents are empty, not writing file") — писать файл напрямую
+ `systemctl daemon-reload`. Проверка: `systemctl show gdm.service -p Environment`.
Эффект: переменная применена, но копи-мод на software-secondary не спасает (см. README п.6).

### /etc/environment (добавлялось)
```
MUTTER_DEBUG_MULTI_GPU_FORCE_COPY_MODE=primary-gpu-cpu
```
pam_env в trixie ЕСТЬ (`/etc/pam.d/gdm-launch-environment`: `session required pam_env.so readenv=1`) — доставка работает.

### udev-эксперименты (все в конечном счёте удалены)
- `/etc/udev/rules.d/61-primary-gpu.rules`:
  `SUBSYSTEM=="drm", KERNEL=="card0", ENV{ID_PRIMARY_GPU}="1"` — mutter НЕ читает ID_PRIMARY_GPU (это gdm/Xorg-логика). Удалён.
- `/etc/udev/rules.d/61-mutter-preferred-primary.rules`:
  ```
  SUBSYSTEM=="drm", ENV{DEVTYPE}=="drm_minor", ENV{DEVNAME}=="/dev/dri/card[0-9]", SUBSYSTEMS=="pci", ATTRS{vendor}=="0x1234", ATTRS{device}=="0x1111", TAG+="mutter-device-preferred-primary"
  ```
  TAG применился (проверено `udevadm info --query=property --name=/dev/dri/card0 | grep -i tags`),
  mutter взял bochs primary (fd gnome-shell = только card0), НО VNC осталась глухой
  (Mesa не подняла рендер на bochs). Удалён после эксперимента.
- `/etc/X11/xorg.conf.d/10-bochs.conf` (X11-путь, удалён при откате):
  ```
  Section "Device"
      Identifier "BochsCard"
      Driver     "modesetting"
      BusID      "PCI:0:1:0"
  EndSection
  ```

### gdm-wayland-force.service (форс Wayland, обход 61-gdm.rules)
```
[Unit]
Description=Force Wayland for GDM (q35 + passthrough iGPU)
After=systemd-udev-settle.service
Before=display-manager.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c '/usr/libexec/gdm-runtime-config set daemon WaylandEnable true && rm -f /run/udev/gdm-machine-has-virtual-gpu /run/udev/gdm-machine-has-hardware-gpu /run/udev/gdm-machine-has-hybrid-graphics'

[Install]
WantedBy=display-manager.service
```
⚠️ Грабля: `systemctl edit` с пустым содержимым не пишет файл (см. выше).
⚠️ oneshot-юнит не выполняется при `systemctl restart gdm` — всегда пара:
`systemctl start gdm-wayland-force && systemctl restart gdm`.

### Гостевые grub-предупреждения, которые НЕ ошибки
```
Unknown kernel command line parameters "BOOT_IMAGE=/boot/vmlinuz-...", will be passed to user space.
```
— норма (BOOT_IMAGE дописывает GRUB, ядро его не парсит).
Также норма: shpchp -16 (PVE bug 6909), i915 "VT-d active" в госте,
"Cannot find any crtc or sizes" на passthrough без мониторов.

## Результаты тестов (замеры чата)
- VAAPI encode (h264_vaapi через renderD128): 3.17x realtime (ранний замер, эпоха 6.12) → 4.71x (свежий замер, 20.09; НЕ зависит от libmfx-фикса — это разные стеки)
- QSV/MFX (h264_qsv): до `apt install libmfx-gen1.2` — MFX session -9; после — 3.03x realtime
- glxinfo: "Mesa Intel(R) Graphics (ADL-N)" — iris (не llvmpipe)
