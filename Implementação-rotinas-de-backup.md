# 🚀 GUIA DE IMPLANTAÇÃO: BACKUP LÓGICO POSTGRESQL (DPCE)

## 📌 1. REGRAS DE NEGÓCIO E PREMISSAS
* **Escopo Estrito:** Executar as configurações exclusivamente nos servidores de Produção (PRD).
* **Usuário Dono:** Toda a estrutura de scripts, logs e execuções pertence e é controlada pelo usuário oficial `postgres`.
* **Armazenamento:** Os dumps são salvos localmente na pasta montada via rede NFS `/backup` (907 GB úteis) e os relatórios são enviados para a nuvem da Oracle Cloud via `rclone`.

---

## 📁 2. CRIAÇÃO DE PASTAS E PERMISSÕES

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `root`
```bash
mkdir -p /WiseDb/scripts/logs
mkdir -p /WiseDb/scripts/Dumps
mkdir -p /backup/postgresql/Bkp_Logico/postgres
mkdir -p /backup/postgresql/Bkp_Logico/tauge
mkdir -p /backup/postgresql/Bkp_Logico/multi
chown postgres:postgres -R /WiseDb /backup/postgresql
chmod 755 -R /WiseDb /backup/postgresql
```

---

## 🌐 3. CONFIGURAÇÃO DO CONFIG DO RCLONE

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `postgres`
```bash
mkdir -p /home/postgres/.config/rclone/
```

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `postgres`  
**Arquivo:** `/home/postgres/.config/rclone/rclone.conf`
```text
[WiseDb]
type = s3
provider = Other
access_key_id = 2818584bc49497a4bce03a743f4a7ce87df6356a
secret_access_key = AakmrOl6nb+lOudGcDHkTBEJPmZ5DK6OqW1wV/fdebU=
region = sa-saopaulo-1
endpoint = https://greyjk43kczz.compat.objectstorage.sa-saopaulo-1.oraclecloud.com
```

---

## 📜 4. CONSTRUÇÃO DO SCRIPT PRINCIPAL (`Pgdump_Export_Full.sh`)

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `postgres`  
**Arquivo:** `/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh`
```bash
#!/usr/bin/bash

export CLIENTE=DPCE
export DATABASE=$1
HOST="192.168.0.208"
PORT="5432"
USERNAME="postgres"
export BINARY=/usr/local/postgres_1610/bin
export TIPO="DIARIO"
export DIR=/backup/postgresql/Bkp_Logico/$DATABASE
export DATA=`date +%d%m%Y_%H%M`
export HORAINI=`date +%H:%M:%S`
export PATH=$PATH:/WiseDb/bin
export ERROR_COUNT=0

export REPORT_HOSTNAME=$(hostname)
export REPORT_DATABASE="$DATABASE"
export REPORT_PGDUMP="FULL"
export REPORT_DATAINI=$(date +%d-%m-%Y)
export REPORT_HORAINI=$(date +%H:%M:%S)

# Se for o Primeiro dia do mes muda o TIPO para MENSAL
DAYPLUS1=`date --date="1 day" +%e`
if [ $DAYPLUS1 -eq 1 ]; then
  TIPO="MENSAL"
fi

export LOGNAME=/WiseDb/scripts/logs/Dump_Full_"$DATABASE"_"$DATA".log
export LOGAUX=/WiseDb/scripts/logs/LOGAUX_DataPump_Full_"$TIPO"_"$DATABASE"_"$DATA".log

# Exclusao dos Arquivos Diarios (Serao Mantidos Apenas os Ultimos 2 Exports Diario)
find $DIR -iname "Dump_Full_DIARIO_"$DATABASE"*.dmp.tar.gz" -mtime +2 -exec rm -fv {} ";"       >> $LOGNAME
find $DIR -iname "Dump_Full_DIARIO_"$DATABASE"*.log" -mtime +15 -exec rm -fv {} ";"             >> $LOGNAME

# Exclusao dos Arquivos MENSAIS (Serao Mantidos Apenas os Ultimos 1 Exports Mensal)
find $DIR -iname "Dump_Full_MENSAL_"$DATABASE"*.dmp.tar.gz" -mtime +62 -exec rm -fv {} ";"

echo  "Iniciando Export Full - $DATABASE | `date`"      >> $LOGNAME

cd $DIR

# COMENTADO PARA EVITAR ESTOURO DE DISCO (Substituido pelo formato customizado binario)
#$BINARY/pg_dump -U postgres $DATABASE -f Dump_Full_"$TIPO"_"$DATABASE"_"$DATA".sql  >> $LOGNAME

$BINARY/pg_dump dbname="$DATABASE" --format=custom --file="$DIR"/Dump_Full_"$TIPO"_"$DATABASE"_"$DATA".dmp --verbose 2>> $LOGNAME

#Backup remoto
#$BINARY/pg_dump  -h $HOST -p $PORT -U $USERNAME  $DATABASE  Dump_Full_"$TIPO"_"$DATABASE"_"$DATA".sql

if [ $? -ne 0 ];
then
        export REPORT_STATUS="FALHA"
        #/WiseDb/bin/SendEmail.sh "ATENCAO: Dump Export Full Database $DATABASE Falhou no Cliente: $CLIENTE" "ATENCAO: Dump Export Full Database $DATABASE Falhou no Cliente: $CLIENTE" $LOGNAME
else
        export REPORT_STATUS="SUCESSO"
        echo "DUMP  Database $DATABASE Encerrado com Sucesso on `date`" >> /WiseDb/scripts/logs/Check_Dump_$DATABASE.log
fi

export REPORT_OBJ_EXPORTADOS=$(grep dumping "$LOGNAME" | wc -l)
export REPORT_DATAFIM=$(date +%d-%m-%Y)
export REPORT_HORAFIM=$(date +%H:%M:%S)
#export REPORT_SIZE=$(du -h "$DIR"/Dump_Full_"$TIPO"_"$DATABASE"_"$DATA".dmp | awk '{print $1}')
export REPORT_SIZE=$(du -b "$DIR"/Dump_Full_"$TIPO"_"$DATABASE"_"$DATA".dmp | cut -f1)

echo "$REPORT_PGDUMP;$REPORT_DATABASE;$REPORT_DATAINI;$REPORT_HORAINI;$REPORT_DATAFIM;$REPORT_HORAFIM;$REPORT_OBJ_EXPORTADOS;$REPORT_SIZE;$REPORT_STATUS;$REPORT_HOSTNAME" >> /WiseDb/scripts/logs/Report_Dump_History_$(hostname).csv

/usr/bin/rclone -v copy /WiseDb/scripts/logs/Report_Dump_History_$(hostname).csv WiseDb:Backup-Report/DPCE/

echo "**************************************"                                                           | tee -a $LOGAUX
echo "* Rotina de Compactacao do Dump"                                                                  | tee -a $LOGAUX
echo "**************************************"                                                           | tee -a $LOGAUX

echo "Iniciando rotina de compactacao" >> $LOGNAME
cd $DIR
tar -czvf $DIR/Dump_Full_"$TIPO"_"${DATABASE}"_$DATA.dmp.tar.gz  $DIR/Dump_Full_"$TIPO"_"${DATABASE}"_"$DATA".dmp  | tee -a $LOGAUX
rm -fv $DIR/Dump_Full_"$TIPO"_"${DATABASE}"_"$DATA".dmp  | tee -a $LOGAUX

################################################################################
# Fim da rotina de backup Lógico.
#################################################################################
echo "Fim do backup logico | `date`" >> $LOGNAME
```

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `postgres`
```bash
chmod 755 /WiseDb/scripts/Dumps/Pgdump_Export_Full.sh
```

---

## 🎼 5. CONSTRUÇÃO DO SCRIPT ORQUESTRADOR (`Pgdump_Export_Full_All.sh`)

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `postgres`  
**Arquivo:** `/WiseDb/scripts/Dumps/Pgdump_Export_Full_All.sh`
```bash
#!/usr/bin/bash

/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh postgres
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh tauge
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh multi
```

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `postgres`
```bash
chmod 755 /WiseDb/scripts/Dumps/Pgdump_Export_Full_All.sh
```

---

## ⏰ 6. AGENDAMENTO AUTOMÁTICO (CRONTAB)

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `postgres`
```text
00 22 * * * /WiseDb/scripts/Dumps/Pgdump_Export_Full_All.sh > /dev/null 2>&1
```
