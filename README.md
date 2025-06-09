# Orange Pi OS Root Migration to NVMe‑SSD

Этот репозиторий содержит подробный скрипт и инструкцию для переноса корневой файловой системы (/) Orange Pi OS (Bookworm, ядро 5.15.147‑sun55iw3) с microSD/eMMC на M.2 NVMe‑SSD. После переноса загрузчик U‑Boot и раздел /boot остаются на SD/eMMC, а корень системы монтируется с SSD.

---

## Содержание

1. [Описание](#описание)
2. [Требования](#требования)
3. [Подготовка](#шаг-0-подготовка)
4. [Разметка и форматирование SSD](#шаг-1-разметка-и-форматирование-ssd)
5. [Копирование корня на SSD](#шаг-2-копирование-корня-на-ssd)
6. [Конфигурация `fstab`](#шаг-3-настройка-etcfstab-на-ssd)
7. [Настройка U‑Boot](#шаг-4-правка-переменных-u-boot)
8. [Пересборка скрипта U‑Boot](#шаг-5-пересборка-скрипта-u-boot)
9. [chroot и сборка initramfs](#шаг-6-chroot-и-пересборка-initramfs)
10. [Очистка и перезагрузка](#шаг-7-очистка-и-перезагрузка)
11. [Финальная проверка](#шаг-8-финальная-проверка)
12. [Советы и примечания](#советы-и-примечания)

---

## Описание

Миграция корня системы на быстрый NVMe‑SSD позволяет существенно ускорить загрузку и работу Orange Pi. U‑Boot и `/boot` остаются на SD/eMMC для совместимости с заводским загрузчиком, а основной корень (`/`) используется с SSD.

---

## Требования

* Плата Orange Pi с поддержкой NVMe‑SSD (Sun55iw3).
* Orange Pi OS (Bookworm) с ядром 5.15.147‑sun55iw3.
* Карта microSD или eMMC с рабочей системой.
* M.2 NVMe‑SSD и адаптер (если требуется).
* Подключение по UART, SSH или физический монитор + клавиатура.
* Права root (`sudo -i`).

---

## Шаг 0. Подготовка

1. **Загрузка с SD/eMMC**

   ```bash
   mount | grep mmcblk1p1
   # Должно быть: /dev/mmcblk1p1 on / типа …
   ```

2. **Получение прав root**

   ```bash
   sudo -i
   ```

3. **Проверка SSD**

   ```bash
   lsblk -o NAME,SIZE,TYPE,MOUNTPOINT | grep nvme0n1
   # Должно показать /dev/nvme0n1 без данных
   ```

---

## Шаг 1. Разметка и форматирование SSD

1. Сбросить старые таблицы и создать GPT:

   ```bash
   wipefs -a /dev/nvme0n1        # при необходимости
   fdisk /dev/nvme0n1
   # Во входном меню fdisk:
   #  Y → g → n → Enter, Enter → w
   ```

2. Форматирование в ext4:

   ```bash
   mkfs.ext4 -F -L rootfs /dev/nvme0n1p1
   ```

3. Сохранить UUID раздела:

   ```bash
   blkid /dev/nvme0n1p1
   ```

   Запишите значение `UUID="..."`.

---

## Шаг 2. Копирование корня на SSD

1. Создать временные точки монтирования:

   ```bash
   mkdir -p /mnt/nvme/{dev,proc,sys,run,tmp}
   chmod 1777 /mnt/nvme/tmp
   ```

2. Смонтировать SSD:

   ```bash
   mount /dev/nvme0n1p1 /mnt/nvme
   ```

3. Скопировать всё содержимое корня:

   ```bash
   rsync -aAXHv --numeric-ids \
     --exclude={/mnt/nvme,/proc,/sys,/dev,/run,/tmp} \
     / /mnt/nvme
   ```

4. Проверить наличие основных директорий:

   ```bash
   for d in bin etc usr var home lib sbin; do
     [ -d /mnt/nvme/$d ] && echo "OK: $d" || echo "MISSING: $d"
   done
   ```

---

## Шаг 3. Настройка `fstab` на SSD

Откройте `/mnt/nvme/etc/fstab`:

```bash
nano /mnt/nvme/etc/fstab
```

Добавьте или замените строку для корня:

```ini
UUID=<ваш-UUID>  /  ext4  defaults,discard,errors=remount-ro,commit=600  0 1
```

* `discard` — TRIM для SSD.
* `errors=remount-ro` — при ошибках.
* `commit=600` — реже синхронизирует метаданные.

Закомментируйте старые строки с тем же UUID.

---

## Шаг 4. Правка переменных U‑Boot

Откройте `/boot/orangepiEnv.txt` на SD/eMMC и обновите:

```makefile
rootdev=UUID=<ваш-UUID>
rootfstype=ext4
rootwait
```

* `rootwait` — ждёт появления SSD.

---

## Шаг 5. Пересборка скрипта U‑Boot

```bash
apt update
apt install -y u-boot-tools
mkimage -C none -A arm -T script \
  -d /boot/boot.cmd /boot/boot.scr
```

---

## Шаг 6. `chroot` и пересборка initramfs

1. **Bind‑монтирование виртуальных FS**:

   ```bash
   mount --bind /dev  /mnt/nvme/dev
   mount -t proc proc /mnt/nvme/proc
   mount -t sysfs sysfs /mnt/nvme/sys
   mount --bind /run  /mnt/nvme/run
   ```

2. **Проверка и вхождение**:

   ```bash
   mountpoint -q /mnt/nvme/dev  && echo OK dev
   # тоже для proc, sys, run
   chroot /mnt/nvme /bin/bash
   ```

3. **Проверка `/sbin/init`**:

   ```bash
   [ -L /sbin/init ] \
     && readlink /sbin/init | grep -q systemd \
     && echo "OK: init → systemd" \
     || ln -s ../lib/systemd/systemd /sbin/init
   ```

4. **Переустановка и сборка initramfs**:

   ```bash
   apt update
   apt install --reinstall systemd-sysv
   update-initramfs -u -k all
   ```

5. **Проверка артефактов**:

   ```bash
   echo "initrd:"; ls /boot/initrd.img-*
   echo "vmlinuz:"; ls /boot/vmlinuz-*
   dpkg -l systemd-sysv | grep ^ii || echo "systemd-sysv missing"
   grep -E '^[^#].*UUID=.*\s+/\s+' /etc/fstab
   lsblk /dev/nvme0n1p1
   mount | grep -E ' on /(proc|sys|run) '
   ```

6. **Выход и отмонтирование**:

   ```bash
   exit
   umount /mnt/nvme/{dev,proc,sys,run}
   ```

---

## Шаг 7. Очистка и перезагрузка

```bash
umount /mnt/nvme
systemctl daemon-reload
mount -a
reboot
```

---

## Шаг 8. Финальная проверка

После загрузки проверьте:

```bash
mount | grep nvme0n1p1
```

Ожидаемый вывод:

```text
/dev/nvme0n1p1 on / type ext4 (rw,noatime,discard,errors=remount-ro,commit=600)
```

Если корень на SSD и активны `noatime,discard` — миграция прошла успешно.

---

## Советы и примечания

* Всегда сохраняйте резервные копии важных данных.
* При возникновении ошибок изучайте `dmesg` и логи `journalctl`.
* Если система не загружается, подключитесь по UART и проверьте переменные U‑Boot.

---


