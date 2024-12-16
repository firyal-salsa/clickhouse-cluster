<div align="center">
    <img src="https://res.cloudinary.com/dvehyvk3d/image/upload/v1734323501/clickhouse_icon_cetkgx.png" alt="clickhouse-arsitektur"/>    
</div>
<br>
Clickhouse adalah open source columnar database management system. Sistem ini dirancang untuk pemrosesan data analitik dengan kecepatan tinggi, yang sering digunakan dalam kebutuhan analisis data besar (big data), singkatnya clickhouse ini OLAP (Online Analytical Processing) jadi kurang cocok untuk transaksi OLTP (Online Transaction Processing).

Saya membangun satu cluster yang terdiri dari tiga buah zookeeper untuk mengkoordinasikan replikasi data dan dua buah server untuk copy data, untuk sumber bisa dicek pada link berikut [dokumentasi clickhouse replikasi](https://clickhouse.com/docs/en/architecture/replication) Tidak hanya membangun cluster, tetapi juga menghubungkan dengan postgreSQL, sumber bisa dilihat di link berikut [dokumentasi clickhouse postgresql](https://clickhouse.com/docs/en/integrations/postgresql#using-clickpipes-powered-by-peerdb).

## Arsitektur
<img src="https://res.cloudinary.com/dvehyvk3d/image/upload/v1734333410/Architecture.1S_2R_ReplicatedMergeTree_5-nodes.3.CH.Keeper.nodes.2.CH.nodes-f93c1e2343cf462a8a59b4cb37a6a24e_hz0irg.png" alt="arsitektur-clickhouse"/>
Disini saya menggunakan VM 1 berisi clickhouse-01 dan clickhouse-keeper-01, VM 2 berisi clickhouse-02 dan clickhouse-keeper-02, lalu VM 3 untuk clickhouse-keeper-03. Masing-masing memiliki port yang dijelaskan pada tabel dibawah ini :

### Port
| **Port** | **Default Protocol** | **Fungsi**                                                                                             |
|----------|-----------------------|-------------------------------------------------------------------------------------------------------|
| 9000     | TCP                   | **Native Interface**: Digunakan untuk koneksi client-server dengan protokol ClickHouse Native. Ideal untuk aplikasi yang menggunakan driver resmi ClickHouse atau alat seperti `clickhouse-client`. |
| 8123     | HTTP                  | **HTTP Interface**: Digunakan untuk mengirim query SQL melalui HTTP. Mendukung berbagai format output seperti JSON, CSV, dan TSV. Cocok untuk integrasi ringan atau monitoring. |
| 9181     | TCP                   | **ZooKeeper Coordination**: Port ini digunakan untuk komunikasi dengan server eksternal ZooKeeper yang mengatur replikasi, metadata cluster, dan koordinasi antar node. |
| 9234     | TCP                   | **ClickHouse Keeper**: Digunakan jika ClickHouse Keeper (pengganti bawaan ZooKeeper) diaktifkan. Mengelola koordinasi internal dalam cluster seperti replikasi dan status node. |

## Prerequesite
Saya menggunakan device sebagai berikut :
<ul>
    <li>2 buah server ubuntu 24.04.1 LTS noble</li>
    <li>1 buah server Rocky Linux 8.10 (Green Obsidian)</li>
</ul>
Tidak ada depedensi khusus, hanya install :
<ul>
    <li>ClickHouse local version 24.11.1.2557 (official build)</li>
    <li>ClickHouse Keeper 24.11.1.2557</li>
</ul>

## Konfigurasi clickhouse
Langkah awal adalah menginstall clickhouse server dan client pada VM / instance, berikut panduanya.
Untuk ubuntu

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL 'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key' | sudo gpg --dearmor -o /usr/share/keyrings/clickhouse-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg] https://packages.clickhouse.com/deb stable main" | sudo tee \
    /etc/apt/sources.list.d/clickhouse.list
sudo apt-get update

sudo apt-get install -y clickhouse-server clickhouse-client
```

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
clickhouse-client # or "clickhouse-client --password" if you set up a password.
```

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
Sesuaikan semua konfigurasi pada setiap virtual machine dengan files di folder repo ini seperti konfigurasi di direktori etc. 

## Testing
Setelah selesai konfigurasi, jalankan perintah seperti di gambar, terlihat disini node-03 sebagai leader sementara kedua node lain sebagai follower.
<img src="https://res.cloudinary.com/dvehyvk3d/image/upload/v1734334700/Untitled_design_wpopaa.png" alt="clickhoue-keeper-follower-leader-testing"/>

Sekarang, coba kita buat database, tabel, dan insert data disalah satu node seperti yang ada di gambar berikut.
<img src="https://res.cloudinary.com/dvehyvk3d/image/upload/v1734335829/Untitled_design_1_lymmtu.png" alt="create-db-table"/>

Sekarang kita test menginput data ke salah satu node dan lihat data tersebut ke node lain.
<img src="https://res.cloudinary.com/dvehyvk3d/image/upload/v1734335829/Untitled_design_1_lymmtu.png" alt="ingest-data"/>
Terlihat bahwa replikasi data berhasil dibuat.
