# 🛠️ Esteira de Telemetria e Auditoria de Instâncias PostgreSQL

Procedimento operacional padrão para levantamento de configurações de Kernel, parâmetros lógicos do RDBMS e governança de segurança de rede.

## 🚀 Passo 1: Auditoria de Parâmetros Lógicos (Dentro do `psql`)
Conecte-se como usuário `postgres` (`su - postgres`) e acesse o prompt (`psql`). Execute o bloco unificado abaixo:

```sql
\x
-- 1. Parâmetros de Memória e Performance
SHOW shared_buffers;
SHOW effective_cache_size;
SHOW work_mem;
SHOW maintenance_work_mem;
SHOW huge_pages;

-- 2. Parâmetros de WAL e Checkpoints
SHOW wal_level;
SHOW archive_mode;
SHOW archive_command;
SHOW max_wal_size;
SHOW min_wal_size;
SHOW checkpoint_timeout;
SHOW checkpoint_completion_target;

-- 3. Durabilidade e Integridade
SHOW fsync;
SHOW synchronous_commit;
SHOW full_page_writes;
SHOW data_checksums;

-- 4. Autovacuum
SHOW autovacuum;
SHOW autovacuum_max_workers;
SHOW autovacuum_naptime;
SHOW autovacuum_vacuum_scale_factor;
SHOW autovacuum_analyze_scale_factor;
SHOW log_autovacuum_min_duration;

-- 5. Parâmetros de Log e Queries Lentas
SHOW log_min_duration_statement;
SHOW log_lock_waits;
SHOW log_temp_files;
SHOW logging_collector;
SHOW log_directory;
SHOW log_filename;
SHOW log_rotation_age;
SHOW log_line_prefix;
SHOW log_checkpoints;
SHOW log_connections;
SHOW log_disconnections;

-- 6. Hardening, Segurança e Limites
SHOW shared_preload_libraries;
SHOW listen_addresses;
SHOW password_encryption;
SHOW ssl;
SHOW max_connections;
SHOW superuser_reserved_connections;
SHOW timezone;
SHOW log_timezone;
SHOW idle_session_timeout;
SHOW idle_in_transaction_session_timeout;
SHOW statement_timeout;
SHOW lock_timeout;

-- 7. Listagem de Bancos (Encoding e Collation)
\l
\x
```

## 🌐 Passo 2: Auditoria de Segurança de Rede (No Shell como `root`)
Comando para extrair as regras de permissão de acesso e métodos de criptografia de senhas ativos sem os comentários do arquivo:

```bash
cat /etc/postgresql/14/main/pg_hba.conf | grep -v '^#' | grep -v '^$'
```
*Nota: Substitua o caminho do diretório conforme a versão ativa (ex: `/16/`) se necessário.*

## 💻 Passo 3: Auditoria de Recursos do Sistema Operacional (No Shell)
Garante a coleta de limites de Hardware, agressividade de Swap, Huge Pages e utilitários de suporte:

```bash
free -h
sysctl vm.swappiness vm.overcommit_memory vm.dirty_background_ratio vm.dirty_ratio vm.nr_hugepages
cat /sys/kernel/mm/transparent_hugepage/enabled
systemctl show postgresql --property=LimitNOFILE
which nmon mutt sendmail pgbadger pg_activity pgbackrest barman 2>/dev/null
```
___________________________________________________________________________________________________________________________________________________________________



## 📋 Itens Pendentes de Validação Externa (Fora do Terminal)

Estes componentes do Setup WiseDB não podem ser coletados via scripts automatizados de leitura e exigem validações gerenciais, contratuais ou testes em ambiente de laboratório:

- [ ] **Definição de Política de RPO/RTO:** Alinhar com o Órgão a estratégia de retenção de backups físicos e escolher a ferramenta centralizada definitiva (`pgBackRest` ou `Barman`).
- [ ] **Homologação de Locale/Collation com o Fabricante:** Validar com as empresas desenvolvedoras dos sistemas (SIGE, Custas, Geter Giro) se as aplicações suportam a alteração de localidade para o padrão brasileiro (`pt_BR.UTF-8`).
- [ ] **Alinhamento de Timeouts de Sessão:** Definir junto à equipe de negócios os limites aceitáveis para derrubada de queries travadas (`statement_timeout`) e sessões ociosas (`idle_in_transaction_session_timeout`).
- [ ] **Homologação Comercial do Monitoramento:** Confirmar se o Órgão contratou o licenciamento da ferramenta *Site24x7* para liberação da instalação do agente oficial.
- [ ] **Execução de Teste de Mesa de Desastre (PITR):** Realizar a restauração física simulada de um dump e dos arquivos de WAL em um servidor isolado de testes para homologar a integridade das cópias de segurança.




