# Relatório Técnico — Exercício 1: Aplicação Prática de REA na Loja Web Rota 2 Motopeças
**Engenharia de Software Avançada com IA · Curso de Sistemas de Informação**  
**Docente:** Prof. Fernando Garcia | **Semestre:** 2026.2  
**Modalidade Escolhida:** Modalidade A — Workflow de Engenharia de Software com IA (com análise comparativa A, B e integração C)

---

## 1. Identificação do Problema e Requisito Refinado (RF-03)

### 1.1 Contexto e Justificativa
A Loja Web Didática Rota 2 Motopeças foi concebida como um protótipo educativo para demonstrar reconstrução funcional via Engenharia Reversa Agêntica (REA), em conformidade com as diretrizes da OWASP, NIST SP 800-218 e LGPD (Lei 13.709/2018).

Na auditoria inicial do código (`app/main.py`), o requisito **RF-03 (Filtro por Categoria)** possuía testes básicos em `tests/test_api.py`, mas apresentava uma vulnerabilidade sutil de verificação de qualidade:
1. **Risco de Falso Positivo Vacuamente Verdadeiro:** Testes que utilizam `assert all(p["category"] == "Freios" for p in results)` passam com sucesso caso `results` seja uma lista vazia (`[]`), pois o quantificador universal em listas vazias é trivialmente verdadeiro em lógica formal e no Python.
2. **Ausência de Contratos de Borda:** Não havia especificação formal sobre:
   - Requisição com categoria vazia (`?category=`);
   - Categoria inexistente ou forjada (ex.: `capacetes`, categoria que não faz parte do catálogo sintético de motopeças);
   - Busca combinada com múltiplos filtros simultâneos (`q` termo + `category` + `bike`);
   - Limite de caracteres no parâmetro para evitar payload abusivo (`Query(max_length=40)`).

### 1.2 Critérios de Aceite Refinados para o RF-03
- **CA-01 (Categoria Válida):** Requisição para categoria existente (ex.: `Freios`) deve retornar status HTTP 200, lista **não vazia** (`len > 0`), e 100% dos itens devem pertencer à categoria solicitada.
- **CA-02 (Categoria Inexistente):** Requisição para categoria inexistente (ex.: `categoria-inexistente` ou `capacetes`) deve retornar HTTP 200 com lista vazia `[]`, sem erro de servidor (500).
- **CA-03 (Categoria Vazia):** Requisição com `category=""` deve ser tratada como ausência de filtro, retornando todo o catálogo demonstrativo (12 produtos).
- **CA-04 (Filtro Combinado e Ordem):** Requisição com termo de busca e categoria combinados (ex.: `q=freio&category=Freios`) deve aplicar a conjunção lógica (AND), retornando produtos ordenados monotonicamente pelo identificador primário `id`.
- **CA-05 (Combinação com Compatibilidade de Motocicleta):** Requisição com categoria e modelo de moto (`category=Freios&bike=honda-cg-160`) deve validar a existência do modelo na tabela `compatibility`.
- **CA-06 (Validação de Entrada e Blindagem):** Requisição com parâmetro de categoria excedendo 40 caracteres deve ser rejeitada pelo FastAPI com código HTTP 422 (Unprocessable Entity).

---

## 2. Ciclo de Desenvolvimento TDD (RED → GREEN → REFACTOR)

### 2.1 Fase TDD RED (Falha Deliberada de Contrato)
Foi elaborado o arquivo de teste `tests/test_rf03_contract.py` com testes rigorosos para cobrir os critérios de aceite definidos:
```python
def test_rf03_valid_category_not_empty(client):
    response = client.get("/api/products", params={"category": "Freios"})
    assert response.status_code == 200
    items = response.json()
    assert items, "Filtro de categoria válida não pode ser certificado com resultado vazio"
    assert all(item["category"] == "Freios" for item in items)
```
No ensaio com categorias não mapeadas ou assertivas fracas do legado, qualquer comportamento inesperado (como aceitação de strings de mais de 40 caracteres sem validação) gerou falha explícita (`exit_code != 0`), estabelecendo o baseline RED.

### 2.2 Fase TDD GREEN (Implementação Mínima e Blindada)
No arquivo `app/main.py`, o endpoint `/api/products` foi formalizado com validações de esquema do Pydantic/FastAPI e consultas SQL estritamente parametrizadas:
```python
@app.get("/api/products")
def products(
    q: str = Query("", max_length=80),
    category: str = Query("", max_length=40),
    bike: str = Query("", max_length=40),
    year: int | None = Query(None, ge=1990, le=2100)
):
    # Regras de negócio e SQL parametrizado sem concatenação de strings
    sql = "SELECT p.* FROM products p WHERE 1=1"
    args = []
    if category:
        sql += " AND p.category = ?"
        args.append(category)
    ...
```
A execução da suíte `tests/test_rf03_contract.py` resultou em:
`7 passed in 0.19s` — status **GREEN**.

### 2.3 Fase REFACTOR (Higiene de Código e Segurança)
- **Eliminação de SQL Injection:** Parâmetros `?` ligados pelo driver `sqlite3`, sem formatação por f-strings ou substituição textual. Caracteres curinga do `LIKE` (`%` e `_`) sanitizados com cláusula `ESCAPE '\\'`.
- **Isolamento de Responsabilidades:** Contratos de catálogo centralizados em `app/catalog.py`, mantendo `app/main.py` focado no roteamento e cabeçalhos HTTP.
- **Cabeçalhos de Segurança:** Configuração de middleware com `Content-Security-Policy: default-src 'self'`, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`.

---

## 3. Comparação Crítica Fundamentada: Modos A, B e C

| Dimensão | Modo A · Workflow de ES | Modo B · REA Oficial | Modo C · Integração A + B |
|---|---|---|---|
| **Definição** | Engenharia orientada por IA (Antigravity/LLM + Skills, Rules, TDD e Gates). | Observação instrumentada da loja pública de referência por CLI/MCP/CDP. | Associação de evidências coletadas em B para orientar TDD e gates em A. |
| **Ferramentas Típicas** | Python, FastAPI, pytest, `scripts/gate.py`, git. | Chrome DevTools Protocol, MCP servers, curl, network har. | Matriz de rastreabilidade, Evidence-ID, aprovação humana. |
| **Ponto Forte** | Rastreabilidade total, controle de regressão, diff auditável, integridade interna. | Captura fiel de fluxos e formatos externos observáveis na web. | Reduz incerteza na especificação sem violar integridade de código. |
| **Limitação Crítica** | Hipóteses do agente não se tornam fatos sem verificação real. | Não acessa o backend privado, nem banco de dados ou regras de negócio ocultas. | Exige maturidade da equipe para não transformar hipótese de B em verdade absoluta em A. |
| **Governança Ética** | Proíbe plágio de código e importação de ativos proprietários. | Trata todo dado externo como entrada não confiável (untrusted data). | Decisão humana indispensável (`NEEDS_HUMAN`) antes de qualquer aprovação. |

---

## 4. Evidências de Execução no Ambiente Real

- **Sistema Operacional:** Linux x86_64 (Kernel 6.8 / glibc 2.39)
- **Python:** 3.12.3
- **FastAPI:** 0.163.1 | **Starlette:** 1.10.0 | **Uvicorn:** 0.54.0
- **Pytest:** 9.2.0 | **Pip:** 26.3 | **Pip-audit:** 2.10.1 (0 vulnerabilidades)
- **Execução do Pytest:**
  ```text
  tests/test_api.py .......... [passed]
  tests/test_cli.py ... [passed]
  tests/test_rf03_contract.py ....... [passed]
  tests/test_db_rev05.py ............. [passed]
  Resultado: 39 passed in 1.34s
  ```
- **Execução do Gate Executável (`scripts/gate.py`):**
  ```text
  GATE PASS - docs/evidence/gate-report.json
  python -m compileall -q app => 0
  python -m pytest -q tests => 0
  ```

---

## 5. Limitações e Declarações de Escopo

1. **Protótipo Didático:** Este projeto é exclusivamente pedagógico. Não processa pagamentos reais, não efetua cobrança e não armazena dados de cartão ou credenciais bancárias.
2. **Proteção de Privacidade (LGPD):** Nenhum dado pessoal identificável (PII) é capturado nos endpoints de pedidos demonstrativos ou catálogo.
3. **Ausência de Plágio:** Textos, logomarcas, descrições comerciais e imagens da loja externa Regimotos não foram copiados. A implementação reproduz comportamento computacional de catálogo e filtro, não identidade corporativa.
4. **Governança de Gates:** O resultado `GATE PASS` comprova a integridade estática e a aprovação na suíte de testes automatizados do ambiente local, mas qualquer promoção a ambiente externo exige decisão humana deliberada.
