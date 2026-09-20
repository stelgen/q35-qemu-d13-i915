# Апдейт 20.09.2026 — upstream-ресёрч (суб-агенты): кто не свапает, какой патч лечит

Доп-расследование поверх README (фаза 3 углублена + новая верификация). Суб-агенты разобрали: mutter 48.3/49.0/main (клон gitlab + тарболы), ядро 6.12/6.16/6.17/6.18/6.19/master (drivers/gpu/drm/tiny/bochs.c + drm-core helpers), virtio-gpu 6.16, QEMU vga.c/bochs-display.c, git-истории torvalds + drm-misc. Все срезы кода ниже сверены с локальными копиями в `docs/sources/` (m-rn.c = mutter 48.3 `meta-renderer-native.c`, m-onscreen.c = `meta-onscreen-native.c`).

## Итог в одну строку

**Ядро НЕ сломано: bochs-drm 6.16 блитит PRIME-imported fb корректно на каждом atomic-коммите (прямой `dma_buf_vmap` чужого i915-буфера). Заморозка = mutter 48.3 перестаёт слать кадры в bochs после initial-коммита. Лечащий upstream-патч: mutter `bbeb8bdca` (MR !4576, начиная с 49.0) — делает `MUTTER_DEBUG_MULTI_GPU_FORCE_COPY_MODE` рабочим и в non-accelerated (CPU) пути.**

## 1. Опровергнуто: «bochs не обновляет VRAM для imported fb»

Трасса v6.16 (`drivers/gpu/drm/tiny/bochs.c`, в репо: `docs/sources/bochs-616.c`):

- `bochs_primary_plane_helper_atomic_update` (L440–470): damage-итератор (`drm_atomic_helper_damage_iter_init` → `drm_atomic_for_each_plane_damage`) → `drm_fb_memcpy(dst = bochs->fb_map (ioremap_wc BAR0), src = shadow_plane_state->data, damage)`.
- L463–467: `/* Always scanout image at VRAM offset 0 */ bochs_hw_setbase(..., 0)`.
- **«Shadow» — это НЕ промежуточная копия**: `shadow_plane_state->data` = прямые vmap'ы fb-объектов; для imported shmem это `dma_buf_vmap(obj->import_attach->dmabuf)` — живой vmap чужого i915-буфера (`drm_gem_shmem_helper.c` L352).
- Race исключён: `commit_tail` сначала ждёт i915 render-fence (`drm_atomic_helper_wait_for_fences`, drm_atomic_helper.c L1857), только потом blit. `prepare_fb` = extraction exclusive-fence из `obj->resv`; ошибка vmap абортит коммит с errno (не «тихо»).
- Dirty-гейта, блокирующего повторные обновления, нет. Единственный kernel-edge: **пустой damage-blob (num_clips=0) при неизменном src → 0 итераций memcpy** (drm_damage_helper.c L223–251); bochs не валидирует damage в atomic_check и, в отличие от virtio-gpu (virtgpu_plane.c L118 форсирует `ignore_damage_clips=true` при смене fb), не делает full-update принудительно.
- У bochs НЕТ vblank/IRQ/таймеров/self-refresh: VRAM меняется **только** в commit-tail. Если mutter не шлёт коммиты — VRAM замерзает. Ядро автономно обновлять VRAM не может по дизайну.

## 2. Переинтерпретация ftrace (важно)

- У bochs нет `drm_vblank_init` → ядро само ставит `no_vblank=true` (drm_atomic_helper.c L700–703) → каждый флип завершается **fake-vblank мгновенно с seq=0** (drm_vblank.c L1135: `seq = 0`).
- ⇒ Наблюдавшиеся «seq=0 ровно раз в 60.0с» = **mutter реально коммитит ~раз в минуту**, ядро НЕ блокирует пайплайн. Это норма для bochs — подтверждено апстримом: `bde44378397b` «drm/bochs: Use vblank timer» (6.19): «Bochs' virtual hardware does not provide vblank interrupts, so DRM sends each event ASAP».
- Для честной детекции коммитов трейсить `bochs_primary_plane_helper_atomic_update` (function_graph/printk), а не `drm_vblank_event_delivered`.

## 3. Устойчивая часть исходного вердикта

Mutter 48.3, ZERO-copy путь (m-onscreen.c):

- L1349–1355 (`acquire_front_buffer`): `case ZERO: imported_fb = import_shared_framebuffer(...); if (imported_fb) return imported_fb;` — **если dma-buf импортнулся успешно, foreign-буфер уходит прямо в scanout**.
- L1280–1305 (`pre_swap_buffers`): ZERO + `import_status == OK` → ничего не копируется. Комментарий «falls back to PRIMARY as needed» правдив только при **провале импорта** — а у bochs импорт проходит. Фолбэк никогда не разряжается.
- L2043–2049 (m-rn.c) `init_secondary_gpu_data_cpu()` — весь боди:

```c
  /* First try ZERO, it automatically falls back to PRIMARY as needed */
  renderer_gpu_data->secondary.copy_mode =
    META_SHARED_FRAMEBUFFER_COPY_MODE_ZERO;
```

getenv в CPU-пути НЕТ (подтверждено кодом в репо). Env читается только в `set_default_secondary_gpu_copy_mode()` (L1918), вызываемой лишь из accelerated-пути `init_secondary_gpu_data_gpu()` (L2019).

**Root cause (уточнённый):** архитектурная мина mutter — ZERO-copy дефолт для non-accelerated secondary + фолбэк только при провале импорта; в связке i915+bochs импорт успешен, mutter сканаутит foreign-буфер и после initial не гонит новые кадры. Ядро и QEMU вычеркнуты по коду (QEMU stdvga redraw — dirty-based, vga.c L1696: гость не пишет VRAM → dirty нет → surface заморожен).

## 4. «6.12→6.16 одинаково» — совпадение симптома, не ядра

- v6.12 bochs = GEM **VRAM** helpers + `drm_simple_display_pipe`: PRIME-импорт невозможен (`drm_gem_vram` не имеет prime_import_sg_table) → mutter там фолбэкал в CPU-copy и работал.
- v6.13+ (`2037174993c8` «Use regular atomic helpers» + `c3ac343c1448` «Use GEM SHMEM helpers», обе 2024-09): universal plane, GEM SHMEM, damage-clips, `drm_plane_enable_fb_damage_clips` — именно с 6.13 открылся путь mutter-ZERO-copy → bochs.

## 5. Лечащий патч

| Патч | Что делает | Версия |
|---|---|---|
| **mutter `bbeb8bdca0f27f9e071b41f0f346f11214eb7e9b` / MR !4576** «renderer/native: Unify copy mode initialization» (Daniel van Vugt, ветка `make-displaylink-testable-v3`, merged 28.08.2025) | удаляет `init_secondary_gpu_data_cpu()`; CPU-путь идёт через `set_copy_mode()` = тот же getenv-оверрайд → **env работает и для non-accelerated secondary**; дефолт ZERO не меняет | **49.0**; в gnome-48 НЕ бэкпортирован (проверено `merge-base --is-ancestor` на 48.8) |
| mutter MR !4251 (José Expósito, merged 05.02.2025 → GNOME 48; коммиты 5d07e6946/8245f9f79/a95644dbd/b65209856/fea6abb4f) | добавил env, но **только в accelerated-путь** — потому в 48.x он «мёртв» для bochs | 48.0 |
| ядро `bde44378397b` «drm/bochs: Use vblank timer» | реальные seq флип-ивентов + пейсинг; на заморозку не влияет | 6.19 |
| ядро: фикс imported-scanout | **не существует** — полный git log bochs.c (torvalds + drm-misc) после 6.16: только `a629feabb53b` (6.17, drm_panic), `306c8959b5fd` (6.18, drm_err), `bde44378397b` (6.19, vblank timer), `80da96d73509` (6.15, DPMS fix), `b15838b03cd0`/`03ebb1ede056` (2026, мелочи). Фиксить нечего — ядро по коду работает | — |
| mutter MR !5219 / `2853853c09` «Handle cross GPU buffer scanout» | другой баг (FB-ID коллизии при direct scanout), не наш случай | 2026 |
| virtio-gpu: `5dd8b536bbda` (vgaarb, 6.13), `cad6a879a7fb`+`78e434bdc0ab`+`54a970048296` (suspend/resume, 6.20), `df4dc947c46b` (6.20, PRIME import при virgl) | опознаны точные хеши; virtio-gpu умеет imported scanout потому что **хост** читает dmabuf-страницы (`dma_buf_pin` → `set_scanout_blob`, guest-side zero-copy); bochs по дизайну обязан копировать сам — и копирует | — |

Дефолт CPU-пути **ZERO остаётся даже в main (51.0)** — апстрим не считает это багом.

## 6. Udev-тег mutter-device-preferred-primary на bochs — подтверждённо не работает

В 48.3 (m-rn.c): тег действительно пробивается через hardware-фильтр — проверка `META_KMS_DEVICE_FLAG_PREFERRED_PRIMARY` идёт первой в `choose_primary_gpu_unchecked()` (L2346, лог «GPU %s selected primary given udev rule», L2361–2371, вне allow_sw-цикла). Но затем `choose_primary_gpu()` (L2440–2457) жёстко требует EGL-дисплей у primary:

```c
  if (meta_render_device_get_egl_display (render_device) == EGL_NO_DISPLAY)
    {
      g_set_error (error, ..., "The GPU %s chosen as primary is not supported by EGL.", ...);
      return NULL;
    }
```

→ `meta_renderer_native_initable_init()` (L2496) возвращает FALSE → renderer init фатален (`meta_backend_create_renderer`) → **gnome-shell/gdm сессия не стартует вовсе**. То же в 49.0 и main — не патчилось до сих пор.

## 7. Community-сигнал

Прямые issue с этим симптомом не найдены (GitLab search API — 401 без auth, веб-поиск пуст). Косвенный, но жирный сигнал: мотивация MR !4576 — **DisplayLink** (тот же класс non-accelerated secondary GPU): «So that DisplayLink can also use MUTTER_DEBUG_MULTI_GPU_FORCE_COPY_MODE to test different copy modes». Публичной конфигурации «GNOME Wayland + noVNC + iGPU passthrough на q35» не существует — недокументированная комбо-зона (как и записано в README).

## 8. Патч-эксперимент для верификации

`docs/sources/bochs-6.16-debug-reblit.patch` (156 строк, против 6.16) — 4 модуль-параметра `bochs.*`:

1. `debug=1` (дефолт) — каждый atomic_update логирует `update #N: fb=… obj=… imported=… WxH` → видно, приходят ли коммиты и imported ли fb.
2. `force_full_update=1` — игнор damage-клипов, полный blit (тест «пустой blob/кривой damage»).
3. `reblit_ms=N` — delayed work: полный ре-блит текущего fb в VRAM **без коммитов**. **Ключевой тест** (N=500): VNC ожил при молчащем mutter → ядро живо, не шлёт коммиты mutter; живой стол → foreign vmap отдаёт правильные байты; чёрное → mutter рендерит не в тот буфер, что импортировал bochs (сверить `/sys/kernel/debug/dri/*/state`).
4. `reject_imported=1` — `-EINVAL` на imported fb в atomic_check → принудительный фолбэк mutter. Осторожно: mutter может вместо фолбэка отключить выход — надёжнее env через `systemctl edit gdm3`.

## 9. Решения (итоговый роадмап поверх README §8)

1. **Верификация (5 минут):** пересборка bochs с патчем → `reblit_ms=500`.
2. **Лечебный путь (q35+Wayland):** mutter 49+ ИЛИ cherry-pick `bbeb8bdca` в 48.3 (10 строк, см. §3). Затем env **строго** через drop-in юнита:
   ```
   systemctl edit gdm3   # [Service] Environment=MUTTER_DEBUG_MULTI_GPU_FORCE_COPY_MODE=primary-gpu-cpu
   ```
   (`primary-gpu-gpu` тоже сработает: GL-копия fail → фолбэк CPU, но с лишней попыткой.)
3. **Headless-обход:** gnome-remote-desktop headless RDP — не зависит от bochs-сканаута.
4. Текущий статус (i440fx+qxl+X11) — валидный вариант; QSV/VA-API не зависит от дисплея.

## 10. Методология

Суб-агенты: (A) mutter-исходники 48.3/49.0/main + git-истории + MR-диффы; (B) ядро bochs/drm-core/virtio-gpu/QEMU + git log torvalds+drm-misc; (C) community-поиск (упал на tool-call лимите — добит вручную, результат §7). Все цитаты кода сверены с локальными срезами `docs/sources/` (строки 1918/2019/2043/2346/2361/2440/2455/2496 m-rn.c; 1280/1349–1370 m-onscreen.c) — расхождений нет.
