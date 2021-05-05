# Relazione Assignment 2 Gestione di reti

## Analisi di dati rilevati tramite SNMP
A cura di Luca Miglior (580671) e Federico Ramacciotti (582646) - Gruppo 8

---

## Obiettivi dell'esercitazione
L'obiettivo di questa esercitazione è quello di monitorare le performance di macchine locali tramite l'utilizzo di messaggi SNMP e mettere in relazione i valori così ottenuti. Abbiamo per prima cosa identificato gli OID SNMP delle interfacce interessate alla cattura, poi analizzati tramite Telegraf e InfluxDB. I dati vengono poi visualizzati tramite Chronograf, che permette di effettuare query complesse automatiche al database Influx e visualizzarle efficacemente grazie a dei grafici.

## Configurazione dei file
Per abilitare la cattura e l'analisi dei dati, sono stati modificati i file `/etc/snmp/snmpd.conf`, `/etc/telegraf/telegraf.conf`, che rappresentano le configurazioni dei due servizi usati.

#### SNMP
Il file contiene tutte le impostazioni di base del servizio SNMP. In particolare comprende i dati sul dispositivo sul quale è attivo il demone SNMP, tra le quali la posizione e le informazioni di contatto dell'amministratore di sistema.
Al file è possibile aggiungere anche dei comandi da eseguire nel terminale, specificati con due direttive: extend e pass. Abbiamo utilizzato la direttiva pass per eseguire due script python per la misurazione della temperatura della cpu e nella stanza. 
```json
pass .1.3.6.1.2.1.25.1.0 /bin/python3 /home/ubuntu/snmp_scripts/cpu_temp.py
pass .1.3.6.1.2.1.25.1.8 /bin/python3 /home/ubuntu/snmp_scripts/env_temp.py
```
Entrambi gli script stampano l'OID della risorsa, il tipo del dato e il dato effettivo, in modo tale che vengano letti da SNMP ed aggiunti alle tabelle. Per il primo script il dato viene letto dal file di sistema specificato nella open, mentre il secondo viene preso tramite una richiesta http ad un arduino che implementa una semplice API REST per la lettura della temperatura.
```py
temp = int(0)
with open('/sys/class/thermal/thermal_zone0/temp') as tempfile:
    temp = int(tempfile.readline())/1000

print('.1.3.6.1.2.1.25.1.0')
print('integer')
print(int(temp))
```
```py
import json
import requests as http

response = http.get('http://192.168.1.6/get')
serialized_resp = json.loads(response.text)
print('.1.3.6.1.2.1.25.1.1')
print('integer')
print(int(serialized_resp['temp']))
```

#### Telegraf
Il file telegraf contiene le configurazioni per i plugin di Influx, tra cui il monitor SNMP. Abbiamo specificato l'indirizzo sul quale è attivo il servizio SNMP e, per ogni campo da monitorare, il suo OID e il nome.
```json
[[inputs.snmp]]

  agents = ["udp://127.0.0.1:161"]

  [[inputs.snmp.field]]
    oid = "RFC1213-MIB::sysName.0"
    name = "source"
    is_tag = true
  [[inputs.snmp.field]]
    oid = "HOST-RESOURCES-MIB::hrProcessorLoad.196608"
    name = "cpuload_0"
  [[inputs.snmp.field]]
    oid = "HOST-RESOURCES-MIB::hrProcessorLoad.196609"
    name = "cpuload_1"
  [[inputs.snmp.field]]
    oid = "HOST-RESOURCES-MIB::hrProcessorLoad.196610"
    name = "cpuload_2"
  [[inputs.snmp.field]]
    oid = "HOST-RESOURCES-MIB::hrProcessorLoad.196611"
    name = "cpuload_3"
  [[inputs.snmp.field]]
    oid = ".1.3.6.1.2.1.25.1.0"
    name = "cpu_temp"
  [[inputs.snmp.field]]
    oid = "IF-MIB::ifInOctets.2"
    name = "eth0_rx"
  [[inputs.snmp.field]]
    oid = "IF-MIB::ifOutOctets.2"
    name = "eth0_tx"
  [[inputs.snmp.field]]
    oid = ".1.3.6.1.2.1.25.1.8"
    name = "indoor_temp"
```

## InfluxDB e Chronograf
I servizi Influx e Chronograf utilizzano un sistema di query periodiche al database per far monitorare i dati specificati tramite Telegraf.
Le maggior parte delle query vengono generate automaticamente da Chronograf e utilizzate per creare i grafici e le dashboard; in alternativa, si possono creare delle query manualmente per visualizzare i dati in un modo più appropriato al contesto.
Di seguito vengono riportate rispettivamente le query per il monitoring della cpu (temperatura e carico), della rete e della temperatura esterna.
```sql
SELECT mean("cpu_temp") AS "mean_indoor_temp", mean("indoor_temp") AS "mean_indoor_temp" FROM "telegraf"."autogen"."snmp"
```
```sql
SELECT mean("cpuload_3") AS "mean_cpuload_3", mean("cpuload_2") AS "mean_cpuload_2", mean("cpuload_1") AS "mean_cpuload_1", mean("cpuload_0") AS "mean_cpuload_0" FROM "telegraf"."autogen"."snmp" WHERE time > :dashboardTime: AND time < :upperDashboardTime: GROUP BY time(:interval:) FILL(null)
```
```sql
SELECT mean("eth0_rx") AS "mean_eth0_rx", mean("eth0_tx") AS "mean_eth0_tx" FROM "telegraf"."autogen"."snmp" WHERE time > :dashboardTime: AND time < :upperDashboardTime: GROUP BY time(:interval:) FILL(null)
```
```sql
SELECT "indoor_temp" AS "indoor_temp" FROM "telegraf"."autogen"."snmp"
```
