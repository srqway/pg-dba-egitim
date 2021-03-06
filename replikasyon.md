## Streaming Replication

Postgresql'de başka paket kurmadan hazır olarak replication gelmektedir. Bu replikasyon active-passive/master/slave çalışmaktadır ve istenildiği zaman multiple slave mimariye dönüştürülebilmektedir.

2 adet sunucu kuruyoruz. Ubuntu 16.04 üzerinde Postgresql 9.6 temel kurulumu linkinden erişilebilir.

Master sunucu üzerinde /etc/postgresql/9.6/main/postgresql.conf açıyoruz ve aşağıdaki bilgileri değiştiriyoruz.
```
listen_addresses = '*'
wal_level = replica
max_wal_senders = 8 #
wal_keep_segments = 100
max_replication_slots = 2
#
hot_standby = on
```
/etc/postgresql/9.6/main/pg_hba.conf dosyasının en altına gerekli değişiklikleri yaparak ekliyoruz.
```
host replication <replicationuser> <ipaddressofreplica> <connection_method>
```
<replicationuser>: seçeceğiniz replication kullanıcı adı, altta kullanıcıyı oluşturacağız.
<ipaddressofreplica>: slave/passive sunucunun ip adresi
<connection_method>: postgresql connection metodlarından birisi ( link )
psql ile servise bağlanıp replication kullanıcısını oluşturuyoruz ve slave sunucusunun transaction log dosyalarını masterdan çekmesini sağlamak için replication_slot oluşturuyoruz. Link
 ```
create user <replicationuser> replication;
select pg_create_physical_replication_slot('<a replication slot name>',true);
```

Servisi yeniden başlatıyoruz.
```
systemctl restart postgresql
```

Slave SunucuSlave'de servisi durduruyoruz ve data klasörünü siliyoruz.

```
systemctl stop  postgresql
rm -rf <postgresql_data_directory>
```

postgresql kullanıcı hesabına geçerek pg_basebackup başlatıyoruz. Aşağıdaki script database protokolü üzerinden vt dosyalarını ve transaction loglarını slave sunucusuna aktaracaktır. Vt cluster'ınının büyüklüğüne göre zaman alacaktır. Eğer uzaktan bağlanıyorsak öncesinde "screen" komutuyla varolan bağlantıdan ayırmamız gerekmektedir.
# session ı ayırmak için
```
screen
pg_basebackup -h <ipofmaster> -p 5432 -U <replicationuser> -D <postgresql_data_directory> -R -S <a replication slot name> -X stream -P -v > log 2>&1; date;
```
Eğer cluster üzerindeki tablespace'i başka bir dizine adreslemek istersek aşağıdaki parametreyi ekliyoruz.
```
--tablespace-mapping=<ESKI_DIZIN>=<YENI_DIZI>
```


İşlem bittikten sonra bulunulan klasördeki log dosyasını herhangi bir hata için incelemekte yarar var. <postgresql_data_directory> içerisinde recovery.conf adında bir dosya oluşacaktır. Bunun içerisinde aşağıdakine benzer bir tanım oluşacaktır. Bir şey değiştirmeye gerek yoktur.
```
standby_mode = 'on'
primary_conninfo = 'user=<replicationuser> host=<ipofmaster> port=5432 sslmode=prefer sslcompression=1 krbsrvname=postgres'
primary_slot_name = '<a replication slot name>'
```


Ubuntu sistemlerde postgresql.conf dosyası <postgresql_data_directory> dizininden ayrık bir yerde olacağından bu dosyanın ilgili satırını sadece okunur bağlantıları kabul etsin diye aşağıdaki şekilde değiştiriyoruz.
```
hot_standby = on
```
Slave üzerindeki servisi yeniden başlatıyoruz.
```
systemctl restart postgresql
```
Slave sunucusu tüm transaction logları işleyip açılınca aşağıdaki sorgu t olarak dönecektir. Master sunucusunda f olarak dönecektir.
```
select pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 t
(1 row)

```

Master sunucuda aşağıki sorgular çalıştırılarak replikasyonun çalışıp çalışmadığı kontrol edilebilir.

```
select * from pg_replication_slots ;
-[ RECORD 1 ]-------+----------
slot_name           | r1
plugin              |
slot_type           | physical
datoid              |
database            |
active              | t
active_pid          | 28152
xmin                |
catalog_xmin        |
restart_lsn         | 0/3006D00
confirmed_flush_lsn |


select * from pg_stat_replication ;
-[ RECORD 1 ]----+------------------------------
pid              | 28152
usesysid         | 16384
usename          | repuser
application_name | walreceiver
client_addr      | <ip_address>
client_hostname  |
client_port      | 54362
backend_start    | 2017-02-14 23:43:44.850637+03
backend_xmin     |
state            | streaming
sent_location    | 0/3006D00
write_location   | 0/3006D00
flush_location   | 0/3006D00
replay_location  | 0/3006D00
sync_priority    | 0
sync_state       | async
```
Replica sunucularının byte olarak ne kadar geride olduklarıyla ilgili bilgi almak için (master sunucusunda):
```
SELECT client_hostname, client_addr,
pg_wal_location_diff(pg_stat_replication.sent_location,
pg_stat_replication.replay_location) AS byte_lag
FROM pg_stat_replication;
```
Slotu kaldırmak için
```
select pg_drop_replication_slot('<slotname>');
```

Özel-1: Eğer [senkron replikasyon](https://www.postgresql.org/docs/current/runtime-config-replication.html) yapmak istiyorsak

1- replikada recovery.conf içerisindeki aşağıdaki satıra ```application_name``` tanımı ekleyeceğiz.
```
primary_conninfo = 'user=<username> host=<primary_ip> application_name=<replika_adi>'
```
sonra postgresql.conf içerisindeki aşağıdaki satırı etkin hale getiriyoruz.
```
synchronous_standby_names = '<replika_adi>'
```

```
psql -c "select application_name,client_addr,state,sent_location,write_location,flush_location,replay_location
 from pg_catalog.pg_stat_replication order by 1"
psql  -c "select slot_name,active,xmin,restart_lsn from pg_catalog.pg_replication_slots"
psql -c "SELECT * from pg_stat_archiver;"
psql -c "SELECT pg_current_xlog_location(), pg_xlogfile_name(pg_current_xlog_insert_location()) insert_location, pg_xlogfile_name(pg_current_xlog_location()) write_location, pg_xlogfile_name(pg_current_xlog_flush_location()) flush_location;"

```

* Bir sonraki:
[Yedekleme](yedekleme.md)
