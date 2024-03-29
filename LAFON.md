# TP1-open-source

#TP1 Systemd

Apres avoir installé Fedora 31, docker et désactiver selinux

1. First steps

On vérifie la version de systemd qui doit etre supérieure à 241, ici version 243 :

[root@localhost ~]# systemctl --version 
systemd 243 (v243.4-1.fc31)
+PAM +AUDIT +SELINUX +IMA -APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4+SECCOMP +BLKID 
+ELFUTILS +KMOD +IDN2 -IDN +PCRE2 default-hierarchy=unified

Pour vérifier que systemd est PID 1, on fait la commande ps -ef : 
zaphir      1120       1  0 12:40 ?        00:00:00 /usr/lib/systemd/systemd --user

et il est bien PID 1

Ensuite pour donner le nom de 5 processus systeme on peut faire la commande ps -ef ou alors la commande 
ps --ppid 2 -p 2 --deselect : 

    774 ?        00:00:00 ModemManager -> processus système qui permet de gerer la, 2g, 3g et 4g
    776 ?        00:00:00 bluetoothd -> processus système du bluetooth machine 
    779 ?        00:00:00 firewalld -> processus système du pare feu machine
    839 ?        00:00:00 sshd -> processus système lié au service ssh 
    930 ?        00:00:00 dockerd -> processus système lié a Docker

2. Gestion du temps

Voici le résultat de la première commande :

[root@localhost ~]# timedatectl
               Local time: ven. 2019-11-29 15:08:19 CET
           Universal time: ven. 2019-11-29 14:08:19 UTC
                 RTC time: ven. 2019-11-29 14:08:19
                Time zone: Europe/Paris (CET, +0100)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
         
Local Time : Heure locale 
Universal Time : Heure basée sur le GMT
RTC Time : Heure de la pile Bios 

Il est pertinent d'utiliser le RTC car si on utilise notre machine sans connexion, notre machine sera 
quand même à l'heure grâce à la pile Bios.

j'ai changé la timezone pour Rome en faisant ceci : 

[root@localhost ~]# timedatectl set-timezone Europe/Rome
[root@localhost ~]# 

On nous demande ensuite de désactiver l'utilisation de la synchronisation NTP :

[root@localhost ~]# timedatectl set-ntp 0
[root@localhost ~]# 

Pour vérifier que c'est coupé on refait la première commande : 

[root@localhost ~]# timedatectl
               Local time: ven. 2019-11-29 15:34:20 CET
           Universal time: ven. 2019-11-29 14:34:20 UTC
                 RTC time: ven. 2019-11-29 14:34:20
                Time zone: Europe/Rome (CET, +0100)
System clock synchronized: yes
              NTP service: inactive
          RTC in local TZ: no
          
3. Gestion de noms

Le changement de nom : 

[root@localhost ~]# hostnamectl set-hostname VMZaphir
[root@localhost ~]# exit
déconnexion
[zaphir@localhost ~]$ hostname
VMZaphir

La différence entre : 
--pretty = Nom sans regle de nommage adapté pour l'utilisateur mais pas pour la machine 
--static = Nom donné de base en général localhost
--transient = Nom donné par le MDNS ou DHCP

Ensuite nous pouvons mettre un nom de deployement : 

[root@VMZaphir ~]# hostnamectl set-deployment Bonjour

Puis on vérifie 

[root@VMZaphir ~]# hostnamectl 
   Static hostname: VMZaphir
         Icon name: computer-vm
           Chassis: vm
        Deployment: Bonjour
        Machine ID: 6c3ea4afea054d059105d365cce6d0b9
           Boot ID: f42dda2d58fa4ab29d639da6c58177b2
    Virtualization: vmware
  Operating System: Fedora 31 (Server Edition)
       CPE OS Name: cpe:/o:fedoraproject:fedora:31
            Kernel: Linux 5.3.12-300.fc31.x86_64
      Architecture: x86-64
      
4. Gestion du réseau (et résolution de noms)

on essaye la commande, on obtient ceci : 

[root@VMZaphir ~]# nmcli con show
NAME     UUID                                  TYPE      DEVICE  
ens33    53acbbeb-b190-3c6d-aab6-fff92545632c  ethernet  ens33   
docker0  c24980a6-b255-447a-8656-1f0eed3fd9f3  bridge    docker0 

Pour afficher les infos DHCP avec network manager voici la commande :

[root@VMZaphir ~]# cat /var/lib/NetworkManager/internal-53acbbeb-b190-3c6d-aab6-fff92545632c-ens33.lease 
# This is private data. Do not parse.
ADDRESS=192.168.43.3
NETMASK=255.255.255.0
ROUTER=192.168.43.145
SERVER_ADDRESS=192.168.43.145
NEXT_SERVER=192.168.43.145
BROADCAST=192.168.43.255
T1=1800
T2=3150
LIFETIME=3600
DNS=192.168.43.145
HOSTNAME=VMZaphir
CLIENTID=01000c29ed14eb
VENDOR_SPECIFIC=414e44524f49445f4d455445524544

Désactiver NetworkManager : 

[root@VMZaphir ~]# systemctl disable NetworkManager 
Removed /etc/systemd/system/multi-user.target.wants/NetworkManager.service.
Removed /etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service.
Removed /etc/systemd/system/network-online.target.wants/NetworkManager-wait-online.service.

Activer systemd-networkd :

[root@VMZaphir ~]# systemctl enable systemd-networkd
Created symlink /etc/systemd/system/dbus-org.freedesktop.network1.service → /usr/lib/systemd/system/systemd-networkd.service.
Created symlink /etc/systemd/system/multi-user.target.wants/systemd-networkd.service → /usr/lib/systemd/system/systemd-networkd.service.
Created symlink /etc/systemd/system/sockets.target.wants/systemd-networkd.socket → /usr/lib/systemd/system/systemd-networkd.socket.
Created symlink /etc/systemd/system/network-online.target.wants/systemd-networkd-wait-online.service → /usr/lib/systemd/system/systemd-networkd-wait-online.service.

Pour check : 

[root@VMZaphir ~]# systemctl is-enabled NetworkManager
disable
et
[root@VMZaphir ~]# systemctl is-enabled systemd-networkd
enabled

Pour pouvoir configurer le réseau avec systemd, j'ai créé un fichier ens33.network dans /etc/systemd/network/ :

  GNU nano 4.3                              /etc/systemd/network/ens33.network                              Modifié  
[Match]
Key=ens33 

[Network]
Address=192.168.43.100/24
DNS=1.1.1.1

System resolvd :

Pour start et start au démarrage systemd-resolved on fait les deux commandes suivantes : 

[root@VMZaphir ~]# systemctl start systemd-resolved
[root@VMZaphir ~]# systemctl enable  systemd-resolved
Created symlink /etc/systemd/system/dbus-org.freedesktop.resolve1.service → /usr/lib/systemd/system/systemd-resolved.service.
Created symlink /etc/systemd/system/multi-user.target.wants/systemd-resolved.service → /usr/lib/systemd/system/systemd-resolved.service.

On vérifie qu'il est bien lancé : 

[root@VMZaphir ~]# systemctl status systemd-resolved
● systemd-resolved.service - Network Name Resolution
   Loaded: loaded (/usr/lib/systemd/system/systemd-resolved.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2019-12-05 12:15:36 CET; 1min 17s ago
     Docs: man:systemd-resolved.service(8)
           https://www.freedesktop.org/wiki/Software/systemd/resolved
           https://www.freedesktop.org/wiki/Software/systemd/writing-network-configuration-managers
           https://www.freedesktop.org/wiki/Software/systemd/writing-resolver-clients
 Main PID: 1740 (systemd-resolve)
   Status: "Processing requests..."
    Tasks: 1 (limit: 2307)
   Memory: 11.5M
   CGroup: /system.slice/systemd-resolved.service
           └─1740 /usr/lib/systemd/systemd-resolved

On vérifie que un DNS tourne localement sur notre machine : 

[root@VMZaphir ~]# ss -ltunp.  avec l pour afficher que ce qui ecoute, t pour tcp, u pour udp, n pour ne pas affciher le domaine et p pour affcher les processus associer au port d'écoute.
Netid State  Recv-Q Send-Q        Local Address:Port   Peer Address:Port                                             
udp   UNCONN 0      0             127.0.0.53%lo:53          0.0.0.0:*     users:(("systemd-resolve",pid=1740,fd=18)) 
udp   UNCONN 0      0        192.168.43.3%ens33:68          0.0.0.0:*     users:(("NetworkManager",pid=1354,fd=20))  
udp   UNCONN 0      0                   0.0.0.0:68          0.0.0.0:*     users:(("dhclient",pid=1110,fd=9))         
udp   UNCONN 0      0                   0.0.0.0:5355        0.0.0.0:*     users:(("systemd-resolve",pid=1740,fd=12)) 
udp   UNCONN 0      0                      [::]:5355           [::]:*     users:(("systemd-resolve",pid=1740,fd=14)) 
tcp   LISTEN 0      128                 0.0.0.0:5355        0.0.0.0:*     users:(("systemd-resolve",pid=1740,fd=13)) 
tcp   LISTEN 0      128           127.0.0.53%lo:53          0.0.0.0:*     users:(("systemd-resolve",pid=1740,fd=19)) 
tcp   LISTEN 0      128                 0.0.0.0:22          0.0.0.0:*     users:(("sshd",pid=873,fd=5))              
tcp   LISTEN 0      128                       *:9090              *:*     users:(("systemd",pid=1,fd=124))           
tcp   LISTEN 0      128                    [::]:5355           [::]:*     users:(("systemd-resolve",pid=1740,fd=15)) 
tcp   LISTEN 0      128                    [::]:22             [::]:*     users:(("sshd",pid=873,fd=7))              

On fait une résolution DNS locale :

[root@VMZaphir resolve]# dig 127.0.0.53

; <<>> DiG 9.11.13-RedHat-9.11.13-2.fc31 <<>> 127.0.0.53
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28254
;; flags: qr aa rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;127.0.0.53.			IN	A

;; ANSWER SECTION:
127.0.0.53.		0	IN	A	127.0.0.53

;; Query time: 67 msec
;; SERVER: 192.168.43.8#53(192.168.43.8)
;; WHEN: jeu. déc. 05 14:40:07 CET 2019
;; MSG SIZE  rcvd: 44

On fait une résolution DNS avec dig : 

[root@VMZaphir ~]# dig www.facebook.com

; <<>> DiG 9.11.13-RedHat-9.11.13-2.fc31 <<>> www.facebook.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 21080
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1280
;; QUESTION SECTION:
;www.facebook.com.		IN	A

;; ANSWER SECTION:
www.facebook.com.	1210	IN	CNAME	star-mini.c10r.facebook.com.
star-mini.c10r.facebook.com. 31	IN	A	157.240.21.35

;; Query time: 36 msec
;; SERVER: 192.168.43.15#53(192.168.43.15)
;; WHEN: jeu. déc. 05 12:44:31 CET 2019
;; MSG SIZE  rcvd: 90

On tente une résolution avec la commande systemd-resolve :

[root@VMZaphir resolve]# systemd-resolve 127.0.0.53
127.0.0.53: localhost                          -- link: lo

-- Information acquired via protocol DNS in 1.2ms.
-- Data is authenticated: yes

On fait un lien symbolique entre /etc/resolv.conf et /run/systemd/resolve/stub-resolv.conf :

[root@VMZaphir resolve]# ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf 

On modifie le fichier /etc/systemd/resolved.conf en ajoutant des DNS : 

  GNU nano 4.3                                  /etc/systemd/resolved.conf                                           
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.
#
# Entries in this file show the compile time defaults.
# You can change settings by editing this file.
# Defaults can be restored by simply deleting this file.
#
# See resolved.conf(5) for details

[Resolve]
DNS=208.67.222.222 208.67.220.220
#FallbackDNS=1.1.1.1 8.8.8.8 1.0.0.1 8.8.4.4 2606:4700:4700::1111 2001:4860:4860::8888 2606:4700:4700::1001 2001:486>
#Domains=
#LLMNR=yes
#MulticastDNS=yes
#DNSSEC=allow-downgrade
#DNSOverTLS=no
#Cache=yes
#DNSStubListener=yes
#ReadEtcHosts=yes

On redemarre le service avec systemctl restart systemd-resolved puis,
Ensuite on vérifie la modification avec resolvectl :

[root@VMZaphir resolve]# resolvectl
Global
       LLMNR setting: yes
MulticastDNS setting: yes
  DNSOverTLS setting: no
      DNSSEC setting: allow-downgrade
    DNSSEC supported: yes
         DNS Servers: 208.67.222.222
                      208.67.220.220
Fallback DNS Servers: 1.1.1.1
                      8.8.8.8
                      1.0.0.1
                      8.8.4.4
                      2606:4700:4700::1111
                      2001:4860:4860::8888
                      2606:4700:4700::1001
                      2001:4860:4860::8844
          DNSSEC NTA: 10.in-addr.arpa
                      16.172.in-addr.arpa
                      168.192.in-addr.arpa
                      17.172.in-addr.arpa
                      18.172.in-addr.arpa
                      19.172.in-addr.arpa
                      20.172.in-addr.arpa
                      21.172.in-addr.arpa
                      22.172.in-addr.arpa
                      23.172.in-addr.arpa
                      24.172.in-addr.arpa
                      25.172.in-addr.arpa
                      26.172.in-addr.arpa
                      27.172.in-addr.arpa
                      28.172.in-addr.arpa
                      29.172.in-addr.arpa
                      30.172.in-addr.arpa
                      31.172.in-addr.arpa
                      corp
                      d.f.ip6.arpa
                      home
                      internal
                      intranet
                      lan
                      local
                      private
                      test

Link 3 (docker0)
      Current Scopes: none
DefaultRoute setting: no
       LLMNR setting: yes
MulticastDNS setting: no
  DNSOverTLS setting: no
      DNSSEC setting: allow-downgrade
    DNSSEC supported: yes

Link 2 (ens33)
      Current Scopes: DNS LLMNR/IPv4 LLMNR/IPv6
DefaultRoute setting: yes
       LLMNR setting: yes
MulticastDNS setting: no
  DNSOverTLS setting: no
      DNSSEC setting: allow-downgrade
    DNSSEC supported: yes
  Current DNS Server: 192.168.43.8
         DNS Servers: 192.168.43.8
                      2a04:cec0:1130:223b::b0
          DNS Domain: ~.

Nous allons maintenant mettre en place un DNS over TLS, ce qui permet d'encrypter les données échangées.

En mode opportunistic :

[Resolve]
DNS=208.67.222.222 208.67.220.220
#FallbackDNS=1.1.1.1 8.8.8.8 1.0.0.1 8.8.4.4 2606:4700:4700::1111 2001:4860:4860::8888 2606:4700:4700::1001 2001:486>
#Domains=
#LLMNR=yes
#MulticastDNS=yes
#DNSSEC=allow-downgrade
DNSOverTLS=opportunistic 
#Cache=yes
#DNSStubListener=yes
#ReadEtcHosts=yes

On test voir si cela fonctionne : 

[root@VMZaphir resolve]# resolvectl query facebook.com
facebook.com: 2a03:2880:f130:83:face:b00c:0:25de -- link: ens33
              157.240.21.35                    -- link: ens33

-- Information acquired via protocol DNS in 51.2ms.
-- Data is authenticated: no

On fait un test en capturant que les flux avec le port 853 :

[root@VMZaphir resolve]# tcpdump -i ens33 port 853
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens33, link-type EN10MB (Ethernet), capture size 262144 bytes
15:49:06.473933 IP6 VMZaphir.44006 > 2a04:cec0:1130:223b::b0.domain-s: Flags [S], seq 3604177154, win 64320, options [mss 1340,sackOK,TS val 3484711830 ecr 0,nop,wscale 7,tfo  cookiereq,nop,nop], length 0
15:49:06.479454 IP6 2a04:cec0:1130:223b::b0.domain-s > VMZaphir.44006: Flags [R.], seq 0, ack 3604177155, win 0, length 0
15:49:06.480609 IP VMZaphir.42238 > _gateway.domain-s: Flags [S], seq 3398631558, win 64240, options [mss 1460,sackOK,TS val 1348520437 ecr 0,nop,wscale 7,tfo  cookiereq,nop,nop], length 0
15:49:06.487765 IP _gateway.domain-s > VMZaphir.42238: Flags [R.], seq 0, ack 3398631559, win 0, length 0
^C
4 packets captured
12 packets received by filter
0 packets dropped by kernel
[root@VMZaphir resolve]# 

Ensuite on doit activer DNSSEC : 

[Resolve]
DNS=1.1.1.1
#FallbackDNS=1.1.1.1 8.8.8.8 1.0.0.1 8.8.4.4 2606:4700:4700::1111 2001:4860:4860::8888 2606:4700:4700::1001 2001:486>
#Domains=
#LLMNR=yes
#MulticastDNS=yes
DNSSEC=yes            
DNSOverTLS=opportunistic
#Cache=yes
#DNSStubListener=yes
#ReadEtcHosts=yes

5. Gestion de sessions logind

Dans cette partie on découvre loginctl : 

[root@VMZaphir resolve]# loginctl
SESSION  UID USER   SEAT  TTY  
      1    0 root   seat0 tty1 
      3 1000 zaphir       pts/0
      5 1000 zaphir       pts/1

3 sessions listed.
[root@VMZaphir resolve]# 

6. Gestion d'unité basique (services)

On veut lister les services présent dans systemd : 

systemctl list-units -t service
UNIT                                 LOAD   ACTIVE SUB     DESCRIPTION                                              
abrt-journal-core.service            loaded active running Creates ABRT problems from coredumpctl messages          
abrt-oops.service                    loaded active running ABRT kernel log watcher                                  
abrt-xorg.service                    loaded active running ABRT Xorg log watcher                                    
abrtd.service                        loaded active running ABRT Automated Bug Reporting Tool                        
atd.service                          loaded active running Deferred execution scheduler                             
auditd.service                       loaded active running Security Auditing Service                                
bluetooth.service                    loaded active running Bluetooth service                                        
dbus-broker.service                  loaded active running D-Bus System Message Bus                                 
docker.service                       loaded active running Docker Application Container Engine                      
dracut-shutdown.service              loaded active exited  Restore /run/initramfs on shutdown                       
firewalld.service                    loaded active running firewalld - dynamic firewall daemon                      
getty@tty1.service                   loaded active running Getty on tty1                                            
gssproxy.service                     loaded active running GSSAPI Proxy Daemon                                      
import-state.service                 loaded active exited  Import network configuration from initramfs              
iscsi-shutdown.service               loaded active exited  Logout off all iSCSI sessions on shutdown                
kmod-static-nodes.service            loaded active exited  Create list of static device nodes for the current kernel
lvm2-monitor.service                 loaded active exited  Monitoring of LVM2 mirrors, snapshots etc. using dmeventd
lvm2-pvscan@8:2.service              loaded active exited  LVM event activation on device 8:2                       
mcelog.service                       loaded active running Machine Check Exception Logging Daemon                   
ModemManager.service                 loaded active running Modem Manager                                            
NetworkManager.service               loaded active running Network Manager                                          
polkit.service                       loaded active running Authorization Manager                                    
rngd.service                         loaded active running Hardware RNG Entropy Gatherer Daemon                     
rpc-statd-notify.service             loaded active exited  Notify NFS peers of a restart                            
rsyslog.service                      loaded active running System Logging Service                                   
smartd.service                       loaded active running Self Monitoring and Reporting Technology (SMART) Daemon  
sshd.service                         loaded active running OpenSSH server daemon                                    
sssd.service                         loaded active running System Security Services Daemon                          
systemd-journal-flush.service        loaded active exited  Flush Journal to Persistent Storage                      
systemd-journald.service             loaded active running Journal Service                                          
systemd-logind.service               loaded active running Login Service                                            
systemd-modules-load.service         loaded active exited  Load Kernel Modules                                      
systemd-networkd-wait-online.service loaded active exited  Wait for Network to be Configured                        
systemd-networkd.service             loaded active running Network Service                                          
systemd-random-seed.service          loaded active exited  Load/Save Random Seed                                    
systemd-remount-fs.service           loaded active exited  Remount Root and Kernel File Systems                     
systemd-resolved.service             loaded active running Network Name Resolution                                  
systemd-sysctl.service               loaded active exited  Apply Kernel Variables                                   
systemd-tmpfiles-setup-dev.service   loaded active exited  Create Static Device Nodes in /dev                       
systemd-tmpfiles-setup.service       loaded active exited  Create Volatile Files and Directories                    
systemd-udev-settle.service          loaded active exited  udev Wait for Complete Device Initialization             
systemd-udev-trigger.service         loaded active exited  udev Coldplug all Devices                                
systemd-udevd.service                loaded active running udev Kernel Device Manager                               
systemd-update-utmp.service          loaded active exited  Update UTMP about System Boot/Shutdown                   
systemd-user-sessions.service        loaded active exited  Permit User Sessions                                     
user-runtime-dir@0.service           loaded active exited  User Runtime Directory /run/user/0                       
user-runtime-dir@1000.service        loaded active exited  User Runtime Directory /run/user/1000                    
user@0.service                       loaded active running User Manager for UID 0                                   
user@1000.service                    loaded active running User Manager for UID 1000                                
vgauthd.service                      loaded active running VGAuth Service for open-vm-tools                         
vmtoolsd.service                     loaded active running Service for virtual machines hosted on VMware            

LOAD   = Reflects whether the unit definition was properly loaded.
ACTIVE = The high-level unit activation state, i.e. generalization of SUB.
SUB    = The low-level unit activation state, values depend on unit type.

51 loaded units listed. Pass --all to see loaded but inactive units, too.
To show all installed unit files use 'systemctl list-unit-files'.

Pour obtenir plus de détails sur une unitée donnée :

détermine si l'unité est actuellement en cours de fonctionnement 
-> [root@VMZaphir resolve]# systemctl is-active sshd.service 
   active
   
détermine si l'unité est liée à un target (généralement, on s'en sert pour activer des unités au démarrage)
-> [root@VMZaphir resolve]# systemctl is-enabled sshd.service 
   enabled

affiche l'état complet d'une unité donnée,
comme le path où elle est définie, si elle a été enable, tous les processus liés, etc.
-> [root@VMZaphir resolve]# systemctl status sshd.service 
● sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2019-12-05 11:24:12 CET; 5h 26min ago
     Docs: man:sshd(8)
           man:sshd_config(5)
 Main PID: 873 (sshd)
    Tasks: 1 (limit: 2307)
   Memory: 5.7M
   CGroup: /system.slice/sshd.service
           └─873 /usr/sbin/sshd -D -oCiphers=aes256-gcm@openssh.com,chacha20-poly1305@openssh.com,aes256-ctr,aes256->

déc. 05 15:48:15 VMZaphir unix_chkpwd[2443]: password check failed for user (root)
déc. 05 15:48:15 VMZaphir sshd[2439]: pam_succeed_if(sshd:auth): requirement "uid >= 1000" not met by user "root"
déc. 05 15:48:17 VMZaphir sshd[2439]: Failed password for root from 192.168.43.173 port 53941 ssh2
déc. 05 15:48:27 VMZaphir unix_chkpwd[2444]: password check failed for user (root)
déc. 05 15:48:27 VMZaphir sshd[2439]: pam_succeed_if(sshd:auth): requirement "uid >= 1000" not met by user "root"
etc ...


affiche les fichiers qui définissent l'unité
->[root@VMZaphir resolve]# systemctl cat sshd.service 
# /usr/lib/systemd/system/sshd.service
[Unit]
Description=OpenSSH server daemon
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target sshd-keygen.target
Wants=sshd-keygen.target

[Service]
Type=notify
EnvironmentFile=-/etc/crypto-policies/back-ends/opensshserver.config
EnvironmentFile=-/etc/sysconfig/sshd-permitrootlogin
EnvironmentFile=-/etc/sysconfig/sshd
ExecStart=/usr/sbin/sshd -D $OPTIONS $CRYPTO_POLICY $PERMITROOTLOGIN
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target

On doit rechercher l'unité associé a chronyd : 

[root@VMZaphir resolve]# ps -e -o pid,cmd,unit | grep "chronyd"
   2791 grep --color=auto chronyd   session-3.scope.    on voit ici que c'est session-3.scope.

II. Boot et Logs

On veut savoir combien de temps a mis le service sshd a se lancer au démarrage : 

[root@VMZaphir resolve]# cat graphe.svg | grep "sshd.service"
  <text class="right" x="1576.785" y="4794.000">sshd.service (166ms)</text>

Donc ici on voit 166ms

III. Mécanismes manipulés par systemd

1. cgroups

On veut identifier le cgroup utilisé par votre session SSH : 

[root@VMZaphir cgroup]# ps -e -o pid,cmd,cgroup | grep "sshd"
    873 /usr/sbin/sshd -D -oCiphers 10:devices:/system.slice/sshd.service,6:memory:/system.slice/sshd.service,4:pids:/system.slice/sshd.service,1:name=systemd:/system.slice/sshd.service,0::/system.slice/sshd.service
   1175 sshd: zaphir [priv]         10:devices:/user.slice,6:memory:/user.slice/user-1000.slice/session-3.scope,4:pids:/user.slice/user-1000.slice/session-3.scope,1:name=systemd:/user.slice/user-1000.slice/session-3.scope,0::/user.slice/user-1000.slice/session-3.scope
   1187 sshd: zaphir@pts/0          10:devices:/user.slice,6:memory:/user.slice/user-1000.slice/session-3.scope,4:pids:/user.slice/user-1000.slice/session-3.scope,1:name=systemd:/user.slice/user-1000.slice/session-3.scope,0::/user.slice/user-1000.slice/session-3.scope
   2445 sshd: zaphir [priv]         10:devices:/user.slice,6:memory:/user.slice/user-1000.slice/session-5.scope,4:pids:/user.slice/user-1000.slice/session-5.scope,1:name=systemd:/user.slice/user-1000.slice/session-5.scope,0::/user.slice/user-1000.slice/session-5.scope
   2449 sshd: zaphir@pts/1          10:devices:/user.slice,6:memory:/user.slice/user-1000.slice/session-5.scope,4:pids:/user.slice/user-1000.slice/session-5.scope,1:name=systemd:/user.slice/user-1000.slice/session-5.scope,0::/user.slice/user-1000.slice/session-5.scope
   3183 grep --color=auto sshd      10:devices:/user.slice,6:memory:/user.slice/user-1000.slice/session-3.scope,4:pids:/user.slice/user-1000.slice/session-3.scope,1:name=systemd:/user.slice/user-1000.slice/session-3.scope,0::/user.slice/user-1000.slice/session-3.scope
   
On veut identifier la RAM maximale à votre disposition (dans /sys/fs/cgroup):

[root@VMZaphir cgroup]# cat /sys/fs/cgroup/memory/memory.max_usage_in_bytes 
1741504512

On veut ensuite limiter la RAM à 512 Mo : 

[root@VMZaphir cgroup]# systemctl set-property user-1000.slice MemoryMax=512M
[root@VMZaphir cgroup]# cat /sys/fs/cgroup/memory/user.slice/user-1000.slice/memory.limit_in_bytes 
536870912

En faisant la deuxième commande, on voit bien qe la limite est à 512Mo.

Ensuite on vérifie le fichier créé dans /etc/systemd/system.control/ :

[root@VMZaphir cgroup]# cat /etc/systemd/system.control/user-1000.slice.d/50-MemoryMax.conf 
# This is a drop-in unit file extension, created via "systemctl set-property"
# or an equivalent operation. Do not edit.
[Slice]
MemoryMax=536870912

Donc c'est bien créé avec la bonne valeur dedans cet à dire 512Mo.
