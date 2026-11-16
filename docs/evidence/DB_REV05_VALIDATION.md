# Registro de Evidências Locais — Validação DB REV05 (Exercício 2)

**Execução realizada em 16/11/2026 no ambiente isolado Linux x86_64.**  
**Ambiente técnico verificado:** Python 3.12.3, SQLite 3.45.1, FastAPI 0.163.1, pytest 9.2.0, pip 26.3, pip-audit 2.10.1.

---

## 1. Tabela Consolidada de Execução e Verificação

| Verificação Técnica | Critério / Procedimento | Evidência / Resultado Observado | Status |
|---|---|---|---|
| **Ensaio RED Deliberado** | Alteração de estoque sem movimento correspondente em `inventory_movements`. | `verify` acusa `ledger discrepancies: 1` e finaliza com exit code 2. | **PASS (RED comprovado)** |
| **Ensaio GREEN Instrumentado** | `UPDATE products` e `INSERT inventory_movements` na mesma transação `BEGIN IMMEDIATE`. | `verify` acusa `ledger discrepancies: 0` e integridade `ok`. | **PASS (GREEN comprovado)** |
| **Migração 001 Idempotente** | Executar `migrate` duas vezes seguidas no mesmo banco de laboratório. | Primeira execução cria tabelas e T0; segunda retorna "Migração já aplicada; sem alterações". | **PASS** |
| **Online Backup e Restore** | `sqlite3.Connection.backup()` em arquivo de laboratório e restauração em `_restored.db`. | `RESTORE PASS`: 12 produtos, `integrity_check = ok`, 0 violações de chave estrangeira (FK). | **PASS** |
| **Suíte de Testes Automatizados** | `pytest tests/` cobrindo API, CLI, RF-03 e DB REV05. | **39 testes aprovados** (0 falhas) em 1.34s. | **PASS** |
| **Gate Executável (`scripts/gate.py`)** | Verificação de compilação estática (`compileall`) e execução integral da suíte (`pytest`). | `GATE PASS` com exit code 0 em ambos os comandos. | **PASS** |
| **Auditoria de Vulnerabilidades** | Execução de `pip-audit` no ambiente de dependências. | `No known vulnerabilities found` (0 vulnerabilidades). | **PASS** |
| **Revisão de Agente Independente** | Auditoria separada sem autoaprovação conforme `.agents/skills/rea-verify/SKILL.md`. | Executado via subagente com parecer documentado em `independent-review.md`. | **AUDITADO** |
| **Deploy / Alteração em Produção** | Homologação para modificação do banco de produção `data/motopecas.db`. | Bloqueado por governança (Fail-Closed). | **NEEDS_HUMAN** |

---

## 2. Logs Reais da Sessão de Laboratório (16/11/2026)

### 2.1 Baseline e Backup Inicial
```bash
$ python scripts/lab_db_extension.py init --db data/lab_rota2.db
Base pronta: 12 produtos

$ python scripts/lab_db_extension.py backup --db data/lab_rota2.db
Backup completo: /home/chroma/Área de trabalho/rea-motopecas-antigravity/data/lab_rota2_backup.db
sha256: 9172b46dbb749584d5c32766b0563785412b90e651d588afce73476372cf8434
```

### 2.2 Restauração e Verificação de Integridade
```bash
$ python scripts/lab_db_extension.py restore --db data/lab_rota2.db
RESTORE PASS: /home/chroma/Área de trabalho/rea-motopecas-antigravity/data/lab_rota2_restored.db produtos: 12 FK violadas: 0
```

### 2.3 Aplicação da Migração 001 e Teste de Idempotência
```bash
$ python scripts/lab_db_extension.py migrate --db data/lab_rota2.db
Migração aplicada. ATENÇÃO: API e CLI devem ser instrumentadas antes de novas transações.

$ python scripts/lab_db_extension.py migrate --db data/lab_rota2.db
Migração já aplicada; sem alterações
```

### 2.4 Conciliação do Livro-Razão e Ajuste Auditável
```bash
$ python scripts/lab_db_extension.py verify --db data/lab_rota2.db
SQLite: 3.45.1 integrity: ok FK violations: 0 ledger discrepancies: 0
PASS: conciliação e integridade. Este PASS não prova integração com API/CLI.

$ python scripts/lab_db_extension.py adjust --db data/lab_rota2.db --sku R2-FR-001 --quantity 25 --reason inventario-aula-conferido
SKU: R2-FR-001 de: 20 para: 25 delta: 5

$ python scripts/lab_db_extension.py verify --db data/lab_rota2.db
SQLite: 3.45.1 integrity: ok FK violations: 0 ledger discrepancies: 0
PASS: conciliação e integridade. Este PASS não prova integração com API/CLI.
```

---

## 3. Comprovação dos Gates Técnicos G-DB1 a G-DB5

- **G-DB1 (Conciliação sem divergências):** Todos os 12 produtos possuem saldo idêntico à soma de seus movimentos.
- **G-DB2 (Integridade referencial e física):** `PRAGMA integrity_check` retorna `ok` e `foreign_key_check` não retorna violações.
- **G-DB3 (Rollback transacional):** Inserção rejeitada no ledger ou saldo insuficiente desfaz todas as alterações de estoque e pedidos (`test_failed_ledger_insert_rolls_back_order_and_stock`).
- **G-DB4 (Idempotência sem duplo débito):** Reenvio da mesma `Idempotency-Key` com o mesmo payload reexibe o pedido anterior sem novo movimento no livro-razão (`test_idempotency_replay_does_not_duplicate_sale`).
- **G-DB5 (Registro transacional na API e CLI):** Vendas pela API gravam `SALE` e ajustes pela CLI gravam `ADJUSTMENT` com motivo auditável (`test_cli_stock_adjustment_and_reason`).
