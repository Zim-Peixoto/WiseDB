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
________________________________________________________________________________________________________________________________________________________________________________________________________________



## 📋 Itens Pendentes de Validação Externa (Fora do Terminal)

Estes componentes do Setup WiseDB não podem ser coletados via scripts automatizados de leitura e exigem validações gerenciais, contratuais ou testes em ambiente de laboratório:

- [ ] **Definição de Política de RPO/RTO:** Alinhar com o Órgão a estratégia de retenção de backups físicos e escolher a ferramenta centralizada definitiva (`pgBackRest` ou `Barman`).
- [ ] **Homologação de Locale/Collation com o Fabricante:** Validar com as empresas desenvolvedoras dos sistemas (SIGE, Custas, Geter Giro) se as aplicações suportam a alteração de localidade para o padrão brasileiro (`pt_BR.UTF-8`).
- [ ] **Alinhamento de Timeouts de Sessão:** Definir junto à equipe de negócios os limites aceitáveis para derrubada de queries travadas (`statement_timeout`) e sessões ociosas (`idle_in_transaction_session_timeout`).
- [ ] **Homologação Comercial do Monitoramento:** Confirmar se o Órgão contratou o licenciamento da ferramenta *Site24x7* para liberação da instalação do agente oficial.
- [ ] **Execução de Teste de Mesa de Desastre (PITR):** Realizar a restauração física simulada de um dump e dos arquivos de WAL em um servidor isolado de testes para homologar a integridade das cópias de segurança.

________________________________________________________________________________________________________________________________________________________________________________________________________________


## 🔍 Guia Operacional de Descoberta para Itens Externos

Procedimento para levantamento de requisitos não-lógicos, homologações com fornecedores de software e validações contratuais/operacionais fora do terminal.

### 1. Definição de Timeouts e Regras de Retenção de Backup
*   **Ação:** Abrir chamado ou enviar comunicação formal para a **Gestão de TI / Equipe de Governança do Órgão**.
*   **Perguntas Diretivas:**
    *   *Timeout de Execução:* "Qual o tempo máximo tolerável que uma instrução/query de usuário pode reter locks e travar o banco antes de ser abortada automaticamente? (Sugerimos o teto de 2 minutos para operações OLTP comuns)."
    *   *Janela de Retenção:* "Qual o período mínimo de histórico de retenção de backups (dumps e archives) que a instituição exige manter armazenado no Storage local/cloud para fins de compliance?"

### 2. Validação de Compatibilidade de Locale/Collation (`pt_BR` vs `en_US`)
*   **Ação:** Acionar o suporte técnico ou os engenheiros de software responsáveis pelo desenvolvimento das aplicações corporativas (**SIGE, Custas e Geter Giro**).
*   **Item de Validação:**
    *   *Alinhamento de Dicionário:* "Se realizarmos a alteração do parâmetro `LC_COLLATE` do cluster PostgreSQL de `en_US.UTF-8` para `pt_BR.UTF-8` para corrigir a ordenação nativa de acentuações no Brasil, a aplicação de vocês homologa formalmente essa alteração ou há riscos de quebra de comportamento em buscas e relatórios?"

### 3. Validação de Licenciamento do Monitoramento (Site24x7)
*   **Ação:** Consultar o **Contrato de Prestação de Serviços (SLA/Escopo)** firmado entre a consultoria e o Órgão, ou realizar alinhamento interno com a liderança técnica (Fábio).
*   **Item de Validação:**
    *   *Auditoria de Escopo:* Verificar se o fornecimento das licenças do agente *Site24x7* está sob a responsabilidade da consultoria ou se o monitoramento será integrado às ferramentas vigentes do Órgão (ex: topologia de agentes Zabbix ativa na porta `10050`).

### 4. Execução e Homologação do Teste de Restore e PITR
*   **Ação:** Executar validação física e funcional em um ambiente de laboratório isolado (VM de Sandbox).
*   **Roteiro de Execução:**
    1.  Provisionar uma Máquina Virtual temporária isolada da rede de produção, utilizando a mesma distribuição de S.O. e versão exata do motor do banco auditado (PostgreSQL `14.5`).
    2.  Transferir o arquivo de backup gerado pelo script automatizado local (`/WiseDb/scripts/Dumps/Pgdump_Export_Full_All.sh`) para o storage dessa nova VM.
    3.  Efetuar o processo de restauração utilizando o utilitário nativo correspondente (`pg_restore` ou `psql`).
    4.  **Critério de Aceite:** O item será marcado como **Validado e Concluído** se o cluster inicializar sem corrupção física de blocos e os seletores SQL retornarem a leitura íntegra dos dados nas tabelas.


