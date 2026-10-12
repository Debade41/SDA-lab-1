# План создания VM в KVM

## Статус

На ноутбуке проверены macOS 26.6.2, Apple M2, arm64.
kern.hv_support=1, но /dev/kvm отсутствует.
Результат сохранён в host-diagnostics.txt.

Команды ниже предлагаются для подходящего Linux-хоста.
Создание VM и её работа здесь не проверены.

## Предполагаемая среда

Хост: Ubuntu 24.04 LTS, x86_64, аппаратная виртуализация
включена в BIOS/UEFI, доступен /dev/kvm.

Гость: Ubuntu Server 24.04 LTS amd64.
Ресурсы VM: 2 CPU, 2048 МБ RAM, диск 20 ГБ.

## Подготовка Linux-хоста

    sudo apt update
    sudo apt install -y cpu-checker qemu-kvm libvirt-daemon-system libvirt-clients virtinst virt-viewer
    sudo systemctl enable --now libvirtd
    sudo kvm-ok
    sudo virsh net-list --all

Если сеть default неактивна, запустить её:

    sudo virsh net-start default

Включить её автоматический запуск:

    sudo virsh net-autostart default

## Создание VM

Скачать ISO Ubuntu Server 24.04 LTS amd64 с официальной страницы:
https://releases.ubuntu.com/24.04/

Поместить ISO на Linux-хосте в:
 /var/lib/libvirt/images/ubuntu24.04-server-amd64.iso

Установить права чтения:

    sudo chmod 0644 /var/lib/libvirt/images/ubuntu24.04-server-amd64.iso

Создать VM:

    sudo virt-install \
      --connect qemu:///system \
      --virt-type kvm \
      --name sda-lab2-vm \
      --memory 2048 \
      --vcpus 2 \
      --disk path=/var/lib/libvirt/images/sda-lab2-vm.qcow2,size=20,format=qcow2,bus=virtio \
      --cdrom /var/lib/libvirt/images/ubuntu24.04-server-amd64.iso \
      --network network=default,model=virtio \
      --graphics spice,listen=127.0.0.1 \
      --os-variant ubuntu24.04 \
      --noautoconsole

Открыть установщик в графической сессии Linux-хоста:

    virt-viewer --connect qemu:///system sda-lab2-vm

При установке создать пользователя [NAME] с правом sudo
и включить OpenSSH Server. Настроить вход публичным SSH-ключом
ноутбука. Приватный ключ хранить только на ноутбуке.

После установки получить адрес в терминале гостя:

    hostname -I

## Доступ с ноутбука

Сеть default использует NAT. Адрес VM не считается напрямую
доступным с Mac. Подключение выполняется через Linux-хост.

Перед запуском SSH-команд заменить:

- `[NAME]` — именем пользователя внутри VM;
- `LINUX_USER` — именем пользователя Linux-хоста;
- `LINUX_HOST` — адресом Linux-хоста;
- `VM_IP` — адресом VM.

    ssh -J LINUX_USER@LINUX_HOST [NAME]@VM_IP

После запуска приложения в VM открыть туннель:

    ssh -N -J LINUX_USER@LINUX_HOST -L 127.0.0.1:8081:127.0.0.1:8080 [NAME]@VM_IP

На ноутбуке приложение будет доступно по:
http://localhost:8081

Схема: ноутбук → SSH через Linux-хост → VM → nginx → бэкенд.
