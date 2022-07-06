# Домашняя работа "Мосты, туннели и VPN"

<details>
<summary>Описание</summary>

1. Между двумя виртуалками поднять vpn в режимах:  
- tun  
- tap  
Описатьв чём разница, замерить скорость между виртуальными машинами в туннелях, сделать вывод об отличающихся показателях скорости.
2. Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключитьсā с локальной машины на виртуалку.  

</details>

1. Реализация первой части задания реализована ролью [vpn](./roles/vpn/)

Автоматически разворачиваются интерфейсы типа tap. Для реализации tun интерфейса достаточно изменить тип в конфиге 
[server.conf](./roles/vpn/files/server.conf.j2) 

Отличия tun и tap нитерфесов:  

+ tap - уровень L2, tun - L3  
+ tun - не умеет в мультикаст => не позволяет использовать OSPF ( link-state протоколы динамической маршрутизации). tap - умеет.  

Замеры:  

+ tun  

 ![tun](https://i.ibb.co/9pK0Q0X/tun.png)

+ tap  

![tap](https://i.ibb.co/hYTQQnq/tap.png)

2. Реализация второй части задания выполнена вручную

+ Оставляем только 1 ВМ. Разворачиваем 
+ Устанавливаем epel репозиторий  
```
yum install -y epel-release
```
+ Устанавливаем пакет openvpn, easy-rsa
```
yum install -y openvpn easy-rsa
```
+ Переходим в директорию /etc/openvpn/ и инициализируем pki  
```
cd /etc/openvpn/
/usr/share/easy-rsa/3.0.3/easyrsa init-pki
```
+ Генерируем необходимые ключи и сертификаты для сервера и клиента  
```
echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa build-ca nopass  
echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa gen-req server nopass  
echo 'yes' | /usr/share/easy-rsa/3.0.8/easyrsa sign-req server server  
/usr/share/easy-rsa/3.0.8/easyrsa gen-dh  
openvpn --genkey --secret ta.key  
echo 'client' | /usr/share/easy-rsa/3/easyrsa gen-req client nopass  
echo 'yes' | /usr/share/easy-rsa/3/easyrsa sign-req client client  
```

+ Создаем конфигурационный файл  /etc/openvpn/server.conf для сервера  
```
port 1207
proto udp
dev tun
ca /etc/openvpn/pki/ca.crt
cert /etc/openvpn/pki/issued/server.crt
key /etc/openvpn/pki/private/server.key
dh /etc/openvpn/pki/dh.pem
server 10.10.10.0 255.255.255.0
route 192.168.10.0 255.255.255.0
push "route 192.168.10.0 255.255.255.0"
ifconfig-pool-persist ipp.txt
client-to-client
client-config-dir /etc/openvpn/client
keepalive 10 120
#comp-lzo
persist-key
persist-tun
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```
+ конфигурационный файл  /etc/openvpn/server.conf для клиента  
```
dev tun
proto udp
remote 192.168.10.10 1207
client
resolv-retry infinite
#ca ./ca.crt
#cert ./client.crt
#key ./client.key
route 192.168.10.0 255.255.255.0
persist-key
persist-tun
comp-lzo
verb 3
<ca>
-----BEGIN CERTIFICATE-----
-----END CERTIFICATE-----
</ca>
<cert>
-----BEGIN CERTIFICATE-----
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN PRIVATE KEY-----
-----END PRIVATE KEY-----
</key>
```
+ Два варианта подключения к серверу:  
out1'''
openvpn --config client.conf  
or  
systemctl start openvpn@client  
```