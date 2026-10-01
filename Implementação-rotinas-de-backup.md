# 🚀 GUIA DE IMPLANTAÇÃO: BACKUP LÓGICO POSTGRESQL (DPCE)

## 📌 1. REGRAS DE NEGÓCIO E PREMISSAS
* **Escopo Estrito:** Executar as configurações exclusivamente nos servidores de Produção (PRD).
* **Usuário Dono:** Toda a estrutura de scripts, logs e execuções pertence e é controlada pelo usuário oficial `postgres`.
* **Armazenamento:** Os dumps são salvos localmente na pasta montada via rede NFS `/backup` (907 GB úteis) e os relatórios são enviados para a nuvem da Oracle Cloud via `rclone`.
* **Sistema Operacional:** Rocky Linux 9.6. A home do usuário `postgres` é `/home/postgres` (e não `/var/lib/postgresql`, padrão do Ubuntu/Debian). O `rclone` foi instalado manualmente (sem pacote RPM) e o binário fica em `/usr/bin/rclone`.
* **Atenção:** o Linux diferencia maiúsculas de minúsculas. O ponto de montagem é `/backup` (minúsculo).

---

## 🔎 2. VERIFICAÇÕES INICIAIS

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `root`

Antes de configurar a rotina, verificar o espaço em disco:
```bash
df -h
```

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `postgres`

Verificar o rclone:
```bash
which rclone
rclone version
```

Verificar os bancos da instância:
```bash
psql
```

```sql
SELECT datname,
       pg_size_pretty(pg_database_size(datname)) AS size
  FROM pg_database
 WHERE datistemplate = false
 ORDER BY pg_database_size(datname) DESC;
```

> Cada banco listado deve ter um diretório de backup (seção 3) e uma linha no script orquestrador (seção 6).

---

## 📁 3. CRIAÇÃO DE PASTAS E PERMISSÕES

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `root`
```bash
mkdir -p /WiseDb/scripts/logs
mkdir -p /WiseDb/scripts/Dumps
mkdir -p /backup/$(hostname)/Bkp_Logico/postgres
mkdir -p /backup/$(hostname)/Bkp_Logico/tauge
mkdir -p /backup/$(hostname)/Bkp_Logico/multi
chown postgres:postgres -R /WiseDb /backup/$(hostname)
chmod 755 -R /WiseDb
chmod 700 -R /backup/$(hostname)
```

> O caminho dos dumps precisa ser o mesmo em dois lugares: aqui e na variável `DIR` do script (seção 5). Confira com `ls /backup`.  
> Os dumps contêm os dados dos bancos, por isso `/backup/$(hostname)` fica com permissão `700` (somente o `postgres` acessa).

---

## 🌐 4. CONFIGURAÇÃO DO CONFIG DO RCLONE

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `postgres`
```bash
mkdir -p /home/postgres/.config/rclone/
```

O arquivo pode ser criado de forma interativa:
```bash
rclone config
```

Ou editado diretamente:

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `postgres`  
**Arquivo:** `/home/postgres/.config/rclone/rclone.conf`
```text
[WiseDb]
type = s3
provider = Other
access_key_id = <ACCESS_KEY_ID>
secret_access_key = <SECRET_ACCESS_KEY>
region = sa-saopaulo-1
endpoint = https://<NAMESPACE>.compat.objectstorage.sa-saopaulo-1.oraclecloud.com
```

```bash
chmod 600 /home/postgres/.config/rclone/rclone.conf
```

> **Nunca publique as chaves reais no repositório.** Se alguma chave já foi exposta, gere uma nova no console da OCI e revogue a antiga.

---

## 📜 5. CONSTRUÇÃO DO SCRIPT PRINCIPAL (`Pgdump_Export_Full.sh`)

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
export DIR=/backup/$(hostname)/Bkp_Logico/$DATABASE
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

## 🎼 6. CONSTRUÇÃO DO SCRIPT ORQUESTRADOR (`Pgdump_Export_Full_All.sh`)

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

## 🧪 7. TESTE MANUAL DO BACKUP

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `postgres`

Executar o backup do banco leve `postgres`, sem passar pelo orquestrador:
```bash
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh postgres
```

Conferir o arquivo gerado. Deve existir o `.dmp.tar.gz` e o `.dmp` solto deve ter sido removido:
```bash
ls -lh /backup/$(hostname)/Bkp_Logico/postgres/
```

Conferir o log da execução. Não deve haver erro de conexão ou de permissão:
```bash
tail -n 30 /WiseDb/scripts/logs/Dump_Full_postgres_*.log
cat /WiseDb/scripts/logs/Check_Dump_postgres.log
```

Conferir o relatório. A última linha deve estar com `SUCESSO` e com tamanho maior que zero:
```bash
tail -n 3 /WiseDb/scripts/logs/Report_Dump_History_$(hostname).csv
```

Conferir se o relatório chegou no bucket:
```bash
rclone ls WiseDb:Backup-Report/DPCE/
```

Validar a integridade do dump. Primeiro listar o conteúdo do `.tar.gz`, depois extrair em `/tmp` e listar os objetos do dump:
```bash
tar -tzf /backup/$(hostname)/Bkp_Logico/postgres/Dump_Full_*_postgres_*.dmp.tar.gz
tar -xzf /backup/$(hostname)/Bkp_Logico/postgres/Dump_Full_*_postgres_*.dmp.tar.gz -C /tmp
pg_restore -l /tmp/backup/$(hostname)/Bkp_Logico/postgres/Dump_Full_*_postgres_*.dmp | head
rm -rf /tmp/backup
```

> Se o `pg_restore` não for encontrado, use o caminho completo: `/usr/local/postgres_1610/bin/pg_restore`.  
> O `tar` do script guarda o caminho completo do arquivo (sem a barra inicial), por isso a extração recria `backup/<hostname>/Bkp_Logico/postgres/` dentro de `/tmp`.  
> Só depois de este teste passar, siga para o agendamento no cron.

---

## ⏰ 8. AGENDAMENTO AUTOMÁTICO (CRONTAB)

**Ambiente:** Servidor de Banco de Dados PRD  
**Usuário:** `postgres`
```bash
crontab -e
```

```text
00 22 * * * /WiseDb/scripts/Dumps/Pgdump_Export_Full_All.sh > /dev/null 2>&1
```

---

## 🗂️ ANEXO A. EXEMPLO DE INSTÂNCIA COM MAIS BANCOS (12 BANCOS)

Use este anexo quando a instância tiver mais bancos do que `postgres`, `tauge` e `multi`. Conferir também a variável `BINARY` do script (caminho do `pg_dump` do servidor, por exemplo `/usr/bin`).

Exemplo de saída da consulta de bancos (seção 2):
```text
       datname       |  size
---------------------+---------
 multi               | 98 GB
 sige_new            | 3760 MB
 sige_old            | 635 MB
 custas              | 181 MB
 sige_dev            | 116 MB
 giro                | 34 MB
 terceirizados_stage | 30 MB
 flowise             | 27 MB
 taskdev             | 26 MB
 dag                 | 14 MB
 taskdev_old         | 9833 kB
 postgres            | 8569 kB
(12 rows)
```

**Usuário:** `root` — criar um diretório para cada banco (no lugar das três linhas de `mkdir` da seção 3):
```bash
for DB in multi sige_new sige_old custas sige_dev giro terceirizados_stage flowise taskdev dag taskdev_old postgres
do
  mkdir -p /backup/$(hostname)/Bkp_Logico/$DB
done
chown postgres:postgres -R /backup/$(hostname)
chmod 700 -R /backup/$(hostname)
```

**Usuário:** `postgres` — orquestrador (no lugar do conteúdo da seção 6):
```bash
#!/usr/bin/bash

/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh postgres
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh taskdev_old
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh dag
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh taskdev
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh flowise
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh terceirizados_stage
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh giro
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh sige_dev
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh custas
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh sige_old
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh sige_new
/WiseDb/scripts/Dumps/Pgdump_Export_Full.sh multi
```
