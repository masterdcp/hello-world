# My configuration 
Just for fun

This is just test gitgub :-)
sudo apt-get install mysql-client mysql-client-5.7 mysql-client-core-5.7 mysql-common mysql-common-5.6 mysql-mmm-agent mysql-mmm-common mysql-mmm-monitor mysql-mmm-tools mysql-sandbox mysql-server mysql-server-5.7 mysql-server-core-5.7 mysql-source-5.7 mysql-testsuite mysql-testsuite-5.7 mysql-utilities mysql-workbench mysql-workbench-data mysqltcl mysqltuner


TS3 Server

Tworzenie użytkownika dla serwera TeamSpeak3

sudo useradd -d /opt/teamspeak3-server -m teamspeak3-user

Instalacja serwera TeamSpeak3 dla systemu Linux 64bit

sudo wget http://dl.4players.de/ts/releases/3.0.12/teamspeak3-server_linux_amd64-3.0.12.tar.bz2
sudo tar -jxvf teamspeak3-server_linux_amd64-3.0.12.tar.bz2
sudo mkdir /opt/teamspeak3-server
sudo mv teamspeak3-server_linux_amd64/* /opt/teamspeak3-server
sudo chown teamspeak3-user:teamspeak3-user /opt/teamspeak3-server -R
sudo rm -fr teamspeak3-server_linux_amd64-3.0.12.tar.bz2 teamspeak3-server_linux_amd64

Instalacja serwera TeamSpeak3 dla systemu Linux 32bit

sudo wget http://dl.4players.de/ts/releases/3.0.12/teamspeak3-server_linux_x86-3.0.12.tar.bz2
sudo tar -jxvf teamspeak3-server_linux_x86-3.0.12.tar.bz2
sudo mkdir /opt/teamspeak3-server
sudo mv teamspeak3-server_linux_x86/* /opt/teamspeak3-server
sudo chown teamspeak3-user:teamspeak3-user /opt/teamspeak3-server -R
sudo rm -fr teamspeak3-server_linux_x86-3.0.12.tar.bz2 teamspeak3-server_linux-x86

Firewall – konfiguracja iptables

-A INPUT -p udp --dport 9987 -j ACCEPT
-A INPUT -p tcp --dport 10011 -j ACCEPT
-A INPUT -p tcp --dport 30033 -j ACCEPT

 
Konfiguracja serwera DNS – Bind9 – dla serwera TeamSpeak3
Dodajemy do strefy naszej domeny dwa wpisy. Zmieniamy we wpisach nazwę domeny domain.com na swoją własną nazwę domeny

_ts3._udp.domain.com. 86400 IN SRV 0 5 9987 domain.com.
_tsdns._tcp.domain.com. 86400 IN SRV 0 5 41144 domain.com.

Test konfiguracji serwera DNS

sudo nslookup -type=SRV _ts3._udp.domain.com
 lub
sudo nslookup -q=SRV _ts3._udp.domain.com

 
Konfiguracja bazy danych MySQL – MariaDB
Tworzenie bazy danych teamspeak3 i użytkownika teamspeak3 dla serwera TeamSpeak3
Zmieniamy HASŁO do bazy danych na własne hasło

sudo mysql -u root -p

create database teamspeak3;
GRANT ALL PRIVILEGES ON teamspeak3.* TO teamspeak3@localhost IDENTIFIED BY 'HASŁO';
flush privileges;
quit

Linkujemy bibliotekę z serwera redist do katalogu głównego

sudo ln -s /opt/teamspeak3-server/redist/libmariadb.so.2 /opt/teamspeak3-server/libmariadb.so.2

Uruchamiamy program ldd aby sprawdzić, czy wszystkie biblioteki są dostępne i nie  posiadają błędów

sudo ldd /opt/teamspeak3-server/libts3db_mariadb.so

Tworzenie plików konfiguracyjnych serwera TeamSpeak3

Pliki konfiguracyjne serwera TeamSpeak3 można utworzyć automatycznie skryptem

./ts3server_minimal_runscript.sh z opcją createinifile=1.

Ten sposób pozwala na uruchomienie serwera z użyciem bazy banych SQL

Ponieważ serwer będzie skonfigurowany z bazą danych MySQL-MariaDB, pliki konfiguracyjne stworzymy recznie:
Tworzenie czarnej listy

sudo touch /opt/teamspeak3-server/query_ip_blacklist.txt

Tworzenie białej listy

sudo cat  << EOT > /opt/teamspeak3-server/query_ip_whitelist.txt

127.0.0.1
EOT

Tworzenie głównego pliku konfiguracyjnego z obsługą bazy danych MySQL-MariaDB

sudo cat << EOT > /opt/teamspeak3-server/ts3server.ini

machine_id=
default_voice_port=9987
voice_ip=0.0.0.0
licensepath=
filetransfer_port=30033
filetransfer_ip=0.0.0.0
query_port=10011
query_ip=0.0.0.0
query_ip_whitelist=query_ip_whitelist.txt
query_ip_blacklist=query_ip_blacklist.txt
dbsqlpath=sql/
dbplugin=ts3db_mariadb
dbsqlcreatepath=create_mariadb/
dbpluginparameter=ts3db_mariadb.ini
dbconnections=10
logpath=logs
logquerycommands=0
dbclientkeepdays=30
logappend=0
query_skipbruteforcecheck=0
EOT

Tworzenie pliku konfiguracyjnego do bazy danych

sudo cat << EOT > /opt/teamspeak3-server/ts3db_mariadb.ini

[config]
host=127.0.0.1
port=3306
username=teamspeak3
password=HASŁO
database=teamspeak3
socket=
EOT

 

Zmieniamy ponownie uprawnienia do plików konfiguracyjnych

sudo chown teamspeak3-user:teamspeak3-user /opt/teamspeak3-server -R

Tworzenie skryptu startowego serwera TeamSpeak3
Tworzenie skryptu startowego który będzie zawierał lokalizację katalogu domowego, skrypt startowy i nazwę użytkownika serwera TeamSpeak3

sudo nano /etc/init.d/ts3

#! /bin/sh
### BEGIN INIT INFO
# Provides:          ts3
# Required-Start:    $network mysqld
# Required-Stop:     $network
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: TeamSpeak3 Server Daemon
# Description:       Starts/Stops/Restarts the TeamSpeak Server Daemon
### END INIT INFO

set -e

PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
DESC="TeamSpeak3 Server"
NAME=teamspeak3-server
USER=teamspeak3-user
DIR=/opt/teamspeak3-server
OPTIONS=inifile=ts3server.ini
DAEMON=$DIR/ts3server_startscript.sh
#PIDFILE=/var/run/$NAME.pid
SCRIPTNAME=/etc/init.d/$NAME

# Gracefully exit if the package has been removed.
test -x $DAEMON || exit 0

sleep 2
sudo -u $USER $DAEMON $1 $OPTIONS

 

Zmiana uprawnień skryptów na wykonywalny.
sudo chmod a+x /etc/init.d/ts3
sudo chmod a+x /opt/teamspeak3-server/ts3server_startscript.sh
sudo chmod a+x /opt/teamspeak3-server/ts3server_minimal_runscript.sh
sudo update-rc.d ts3 defaults

Uruchomienie serwera TeamSpeak3 z bazą danych MySQL – MAriaDB.
sudo /etc/init.d/ts3 start

Dostępne komendy skryptu ts3:
sudo /etc/init.d/ts3 {start|stop|restart|status}
Usage: ./ts3server_startscript.sh {start|stop|restart|status}

 
Pomocne informacje:
Uruchomienie serwera bez używania skryptu init.d z użyciem autoutworzonej bazy danych SQL:
su - teamspeak3-user ./ts3server_minimal_runscript.sh start

Uruchomienie serwera bez używania skryptu init.d z użyciem pliku konfiguracyjnego ts3server.ini z wcześniej utworzoną bazą danych MySQL:
su - teamspeak3-user ./ts3server_minimal_runscript.sh start inifile=ts3server.ini 

Uruchomienie serwera bez używania skryptu init.d z odświerzeniem bazy danych MySQL:
su - teamspeak3-user ./ts3server_minimal_runscript.sh start inifile=ts3server.ini clear_database=1

Zmiana hasła Administratora.
Edytujemy skrypt ./ts3server_startscript.sh i upewniamy się że przy COMMANDLINE_PARAMETERS= jest parametr „${2}”.

sudo nano /opt/teamspeak3-server/ts3server_startscript.sh
su - teamspeak3-user ./ts3server_startscript.sh start serveradmin_password=NOWE_HASŁO

Logi serwera TeamSpeak3 znajdują się w folderze:
/opt/teamspeak3-server/logs

Krótka ale za to pomocna dokumentacja znajduje sie w folderze:
/opt/teamspeak3-server/doc

Test nasłuchiwania serwera TeamSpeak3.
sudo netstat -lnp | grep ts3



Run Install TS3 Client 
Open your Terminal (you can press default shortcut of Ctrl+Alt+T), and go to directory where the file is located, eg:

cd Downloads

And run the installer, eg. like this:
./TeamSpeak3-Client-linux_amd64-3.0.16.run

Keep System Clean
TeamSpeak will be installed in current directory, and its probably a good idea to move it somewhere - /opt is good place to keep additional software like this
sudo mv TeamSpeak3-Client-linux_amd64 /opt/

Run TeamSpeak
To run installed TeamSpeak enter:
/opt/TeamSpeak3-Client-linux_amd64/ts3client_runscript.sh

Create Launcher
You can permanently create a Launcher, for yourself:

gedit ~/.local/share/applications/TeamSpeak3.desktop

...or for all users on your system like this:

sudo gedit /usr/share/applications/TeamSpeak3.desktop

Put a content to this Launcher like this:

[Desktop Entry]
Name=TeamSpeak 3
Comment=TeamSpeak 3 VoIP Communicator
Exec=/opt/TeamSpeak3-Client-linux_amd64/ts3client_runscript.sh
Terminal=false
Type=Application
Categories=Network;Application;
Icon=/opt/TeamSpeak3-Client-linux_amd64/styles/default/logo-128x128.png

Remember to replace file and directory names accordingly to TeamSpeak version (here 3.0.16) and target architecture (here amd64).




