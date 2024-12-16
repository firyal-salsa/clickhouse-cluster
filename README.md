# Clickhouse

## Arsitektur
Disini saya akan membuat satu cluster dimana cluster tersebut memiliki 3 zookeeper dan 2 replica. Berikut arsitekturnya.
<img src="" alt="clickhouse-arsitektur"/>

## Port
| **Port** | **Default Protocol** | **Fungsi**                                                                                             |
|----------|-----------------------|-------------------------------------------------------------------------------------------------------|
| 9000     | TCP                   | **Native Interface**: Digunakan untuk koneksi client-server dengan protokol ClickHouse Native. Ideal untuk aplikasi yang menggunakan driver resmi ClickHouse atau alat seperti `clickhouse-client`. |
| 8123     | HTTP                  | **HTTP Interface**: Digunakan untuk mengirim query SQL melalui HTTP. Mendukung berbagai format output seperti JSON, CSV, dan TSV. Cocok untuk integrasi ringan atau monitoring. |
| 9181     | TCP                   | **ZooKeeper Coordination**: Port ini digunakan untuk komunikasi dengan server eksternal ZooKeeper yang mengatur replikasi, metadata cluster, dan koordinasi antar node. |
| 9234     | TCP                   | **ClickHouse Keeper**: Digunakan jika ClickHouse Keeper (pengganti bawaan ZooKeeper) diaktifkan. Mengelola koordinasi internal dalam cluster seperti replikasi dan status node. |

## Konfigurasi clickhouse
Langkah awal adalah menginstall clickhouse server dan client pada VM / instance, berikut panduanya.
Untuk ubuntu

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL 'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key' | sudo gpg --dearmor -o /usr/share/keyrings/clickhouse-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg] https://packages.clickhouse.com/deb stable main" | sudo tee \
    /etc/apt/sources.list.d/clickhouse.list
sudo apt-get update

sudo apt-get install -y clickhouse-server clickhouse-client```

Untuk rocky
```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo

sudo yum install -y clickhouse-server clickhouse-client
```
Selanjutnya, mari kita install clickhouse keepernya
Untuk ubuntu
```bash
sudo apt-get install -y clickhouse-keeper

sudo systemctl enable clickhouse-keeper
sudo systemctl start clickhouse-keeper
sudo systemctl status clickhouse-keeper
```

Untuk rocky
```bash
sudo yum install -y clickhouse-server clickhouse-client

sudo systemctl enable clickhouse-server
sudo systemctl start clickhouse-server
sudo systemctl status clickhouse-server
clickhouse-client # or "clickhouse-client --password" if you set up a password.```



Setelah selesai, hal yang harus diperhatikan adalah membackup file /etc/clickhouse-server/users.xml dan /etc/clickhouse-server/config.xml, bisa menggunakan perintah berikut
```bash
cp /etc/clickhouse-server/config.xml /tmp/config.xml.bak
cp /etc/clickhouse-server/users.xml /tmp/users.xml.bak
```
Hal ini dilakukan sebagai preventif jika kedua file diatas tereplace. Disini, kita akan menggunakan files di direktori config.d dan users.d
Jika terjadi error saat menjalankan clickhouse-keeper, coba jalankan clickhouse-keeper manual tanpa menggunakan systemd dan lihat log. 
```bash
clickhouse keeper --config-file=/etc/clickhouse-server/<nama file config>
```
Selanjutnya, ganti file keeper_config.xml dengan config.xml saja.

