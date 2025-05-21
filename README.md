# Orange-Pi-M.2-NVMe-SSD Debiam

переноса вашей системы Orange Pi OS (Bookworm, ядро 5.15.147-sun55iw3) с microSD/eMMC (`mmcblk1p1`) на M.2 NVMe-SSD (`nvme0n1p1`). После этого U-Boot и `/boot` остаются на SD/eMMC, а корень (`/`) — на SSD.

---

### Шаг 0. Подготовка

* Загрузитесь с SD-карты или eMMC, как обычно.
* Убедитесь, что SSD виден как `/dev/nvme0n1` и что раздел `/dev/nvme0n1p1` создан (если нет — см. пункт 1).
* Имейте права `sudo`.

---

### 1. Разметка и форматирование SSD

```bash
# 1.1. Удаляем старые подписи и создаём новую GPT
sudo fdisk /dev/nvme0n1
# — нажмите Y на "remove signature"
# — в меню fdisk: g → n → (Enter, Enter по умолчанию) → w

# 1.2. Форматируем в ext4
sudo mkfs.ext4 /dev/nvme0n1p1

# 1.3. Сохраняем UUID раздела
sudo blkid /dev/nvme0n1p1
# записываем строку после UUID=", например 13260bd5-474d-4aab-be2a-878f7005520a
```

---

### 2. Копирование корня на SSD

```bash
# 2.1. Подготовка staging-папки
sudo mkdir -p /mnt/nvme/{dev,proc,sys,run,tmp}
sudo chmod 1777 /mnt/nvme/tmp

# 2.2. Монтируем SSD
sudo mount /dev/nvme0n1p1 /mnt/nvme

# 2.3. Копируем всю файловую систему, кроме виртуальных
sudo rsync -aAXv / /mnt/nvme \
  --exclude={"/mnt/nvme","/proc","/sys","/dev","/run","/tmp"}
```

---

### 3. Настройка `fstab` на SSD

```bash
sudo nano /mnt/nvme/etc/fstab
```

Замените (или добавьте) строку для корня на:

```
UUID=13260bd5-474d-4aab-be2a-878f7005520a  /  ext4  defaults,discard,errors=remount-ro  0 1
```

Удалите или закомментируйте любые другие монтирования того же раздела (например `/var/log.hdd`).

---

### 4. Правка загрузочных переменных U-Boot

```bash
sudo nano /boot/orangepiEnv.txt
```

Добавьте или поправьте:

```
rootdev=UUID=13260bd5-474d-4aab-be2a-878f7005520a
rootfstype=ext4
```

Сохраните (`Ctrl+O`, Enter) и выйдите (`Ctrl+X`).

---

### 5. Пересборка бинарного скрипта U-Boot

```bash
sudo apt update
sudo apt install -y u-boot-tools
sudo mkimage -C none -A arm -T script \
    -d /boot/boot.cmd /boot/boot.scr
```

---

### 6. Восстановление init и пересборка initramfs

```bash
# 6.1. Привязываем виртуальные FS
sudo mount --bind /dev  /mnt/nvme/dev
sudo mount -t proc proc /mnt/nvme/proc
sudo mount -t sysfs sys  /mnt/nvme/sys
sudo mount --bind /run  /mnt/nvme/run

# 6.2. Заходим в chroot
sudo chroot /mnt/nvme /bin/bash

# 6.3. Проверяем init (должно быть /sbin/init → systemd)
if [ ! -e /sbin/init ]; then
  ln -s ../lib/systemd/systemd /sbin/init
fi

# 6.4. Переустанавливаем systemd-sysv и пересобираем initramfs
apt update
apt install --reinstall systemd-sysv
update-initramfs -u -k all

# 6.5. Выходим и отмонтируем
exit
sudo umount /mnt/nvme/{dev,proc,sys,run}
```

---

### 7. Очистка и финальная перезагрузка

```bash
# 7.1. Отмонтируем staging-папку
sudo umount /mnt/nvme

# 7.2. (Если есть) снимите лишние монтирования
sudo umount /var/log.hdd        # если осталось

# 7.3. Перезагружаемся
sudo reboot
```

---

### Шаг 8. Проверка

После загрузки выполните:

```bash
mount | grep nvme0n1p1
```

Ожидаемый выход:

```
/dev/nvme0n1p1 on / type ext4 (rw,noatime,errors=remount-ro,commit=600)
```

Если так — поздравляю, система успешно перенесена!
