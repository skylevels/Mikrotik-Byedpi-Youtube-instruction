# Mikrotik-Byedpi-Youtube-instruction
Инструкция для установки контейнера с byedpi для обхода блокировки Youtube

Данная инструкция работает для микротиков на arm, arm64 процах, с RoS 7.18 и выше

Локальный интерфейс (чаще всего это bridge1 или bridge_local) добавьте в interface-list "LAN", так придется меньше адаптировать инструкцию под свой конфиг

### 1. Включим режим использования контейнеров на микротике
Посмотреть список режимов можно так: ``` /system/device-mode print ```

Включаем режим контейнера, за одно и генерации трафика, который может пригодиться для этой инструкции: [wiktorbgu/Mikrotik WireGuard anti DPI.md](https://gist.github.com/wiktorbgu/1f2dfe99837d8f2803483be95814d2e5)

Команда включения: ``` /system/device-mode/update container=yes traffic-gen=yes ```

После включения нужно ребутнуть микрот по питанию, либо физической кнопкой ресет. Программный ребут не сработает, режим не переключится. 
[Официальная справка по device-mode](https://help.mikrotik.com/docs/display/ROS/Device-mode).

Убедимся, что режимы включились: ``` /system/device-mode print ```

### 2. Устанавливаем модуль для работы с контейнерами
На некоторых прошивках модуль автоматически загружается в микротик при включении режимов.
Заходим в System - Packages и смотрим (Олды, не пугайтесь, именно так выглядит интерфейс нового WinBox): ![Скрин с установленными пакетами](screen1.jpg)
Тут осталось только включить контейнер и отправить микротик на ребут

Если же модуль не загрузился, загрузим его вручную командой (только версию своей прошивки подправьте, если у вас другая): 

Для arm64: ```/tool fetch https://upgrade.mikrotik.com/routeros/7.18.1/container-7.18.1-arm64.npk ```

Для arm ``` /tool fetch https://upgrade.mikrotik.com/routeros/7.18.1/container-7.18.1-arm.npk ```

После установки контейнера не забудте сделать ребут. Когда контейнер встал, в главном меню появиться пункт: ![Скрин с контейнером ](screen2.png)

### 3. Создаем интерфейс для будущего контейнера
Тут всё просто, делаем интерфейс, и сразу задаем будущему контейнеру ip адрес: 

``` /interface veth add address=192.168.254.2/24 gateway=192.168.254.1 name=byedpi_interface ```

На другом конце интерфейса задаем ip адрес нашему микротику: 

``` /ip address add address=192.168.254.1/24 disabled=no interface=byedpi_interface ```

### 3. Загружаем контейнер в byedpi
Заливаем на свой микрот свежий контейнер от Виктора: [Полезные контейнеры для Mikrotik](https://teletype.in/@wiktorbgu/containers-mikrotik)

```
/container config set registry-url=https://registry-1.docker.io tmpdir=docker
/container/add remote-image=wiktorbgu/byedpi-hev-socks5-tunnel:mikro interface=byedpi_interface cmd="-s1 -q1 -Y -Ar -s5 -o1+s -At -f-1 -r1+s -As -s1 -o1 +s -s-1 -An -b+500" root-dir=/docker/byedpi-hev-socks5-tunnel-mikro start-on-boot=yes
```
Контейнер лажет на внутреннюю флеш память микротика, в папку /docker, если хотите (или на вашем микроте нет свободного места), можно записать его на флешку, года подправьте путь в переменной root-dir=/usb1/docker...

Обратите внимания на переменную cmd="-s1 -q1 -Y -Ar -s5 -o1+s -At -f-1 -r1+s -As -s1 -o1 +s -s-1 -An -b+500" именно она задает параметры работы byedpi контейнера. После установки можно ее отредактировать, например если у вашего провайдера данные параметры работают плохо.

Редактируется эта переменная в свойствах контейнера через winbox: 

![Скрин с переменной byedpi ](screen3.png)

Разные ключи для подбора можно взять вот отсюда: https://raw.githubusercontent.com/romanvht/ByeDPIAndroid/refs/heads/master/app/src/main/assets/proxytest_cmds.txt

Также можно полазить на форумах, в обсуждениях репозитория byedpi, а еше спросить в телеграм канале: t.me/it_network_people

### 4. Создаем таблицу роутинга и прописываем маршрут до byedpi контейнера

```
/routing/table add disabled=no fib name=byedpi_table
/ip route add dst-address=0.0.0.0/0 gateway=192.168.254.2 routing-table=byedpi_table
```

### 5. Делаем настройки DNS для форварда
Поскольку РКН активно начал блокировать DoH протокол, не советую использовать его для всего трафика, а только для доменов, которым требуется обход блокировки. К тому-же DoH работает гораздо медленнее классического DNS, и бывает подлагивает, если использовать на постоянку.

Ниже приведен пример с CloudFlare сервером, но Вы можете выбрать другой, менее популярный.

Тут мы добавляем форвард DoH сервер: ``` /ip/dns/forwarders add doh-servers="https://1.1.1.1/dns-query " name="doh 1.1.1.1" ```
Тут загружаем сертификат CloudFlare: ``` /tool fetch https://cacerts.digicert.com/DigiCertGlobalRootG2.crt.pem ```
Тут этот сертификат устанавливаем на микрот: ``` /certificate import file-name=DigiCertGlobalRootG2.crt.pem passphrase="" ```

### 6. Создаем список доменов для обхода блокировки

/ip/dns set address-list-extra-time=1d

### 6. Создаем правила на фаерволе
Первое правило будет 



### X. Запретим DNS трафик из вашей локальной сети в обход роутера
```
/ip firewall nat
 add action=redirect chain=dstnat comment="dns redirect" dst-port=53 in-interface-list=LAN protocol=udp
add action=redirect chain=dstnat comment="dns redirect" dst-port=53 in-interface-list=LAN protocol=tcp

#for_ipv6
/ipv6 firewall nat
add action=redirect chain=dstnat comment="dns redirect" dst-port=53 in-interface-list=LAN protocol=tcp
add action=redirect chain=dstnat comment="dns redirect" dst-port=53 in-interface-list=LAN protocol=udp
```

