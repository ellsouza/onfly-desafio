# Onfly — Consolidador de reservas de hotel

Você recebeu a exportação de reservas de hotel de uma empresa cliente da Onfly. O CLI deste projeto deveria gerar um relatório JSON consolidado, mas **entrega números errados**.

**Sua tarefa:** encontrar e corrigir os defeitos do CLI, fazer os testes fornecidos passarem e
entregar o projeto.

## Como rodar

Requisitos: Python 3.11+ e pytest para os testes (`requirements-dev.txt`).

```bash
python3.11 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt

PYTHONPATH=src python -m onfly_report \
  --input data/reservas_hoteis.csv \
  --rates data/taxas_cambio.csv \
  --output output/report.json
```

O relatório é gravado em `output/report.json`; o CLI cria o diretório. Um resumo é impresso no
stdout.

Para visualizar o resultado no dashboard executivo, gere o relatório e inicie um servidor estático
na raiz do projeto:

```bash
py -m http.server 8000
```

No Linux/macOS, use `python3 -m http.server 8000`. Abra `http://localhost:8000/dashboard.html`.
O painel carrega `output/report.json` e mostra os indicadores, gráficos mensais e ranking de hotéis.
Também inclui a narrativa da investigação e correção dos defeitos. A identidade visual usa a fonte
Poppins, a paleta azul e a logo do site oficial da Onfly. Se o JSON não estiver disponível, exibe a
amostra de validação.

Ao publicar o projeto no GitHub, o workflow `.github/workflows/pages.yml` gera o relatório completo
e publica o dashboard no GitHub Pages após um push para `main` (ou execução manual do workflow).

Os testes já estão em `tests/`. No início, todos falham — é esperado. Rode:

```bash
pytest -q
```

Corrija o código até todos passarem. **Não altere os testes fornecidos.** Se quiser, adicione
testes novos em arquivos `test_*.py`.

## Estrutura

```
.
├── README.md              # este arquivo
├── dashboard.html         # painel visual do relatório
├── pyproject.toml
├── requirements*.txt
├── data/
│   ├── reservas_hoteis.csv
│   ├── taxas_cambio.csv
│   └── exemplo-validacao.csv
├── src/onfly_report/       # CLI
├── tests/                 # testes fornecidos
├── entrega/               # sua entrega
└── output/                # relatório gerado
```

## Regras de negócio

Os dados fornecidos estão consistentes, os defeitos estão no CLI.

As colunas do CSV são autoexplicativas. O relatório usa hotel, check-in, check-out, quartos, status, moeda e valor; as demais colunas não afetam os totais.

### Status

* Só `CONFIRMED` é faturável.
* `CANCELLED` e `NO_SHOW` contam em `total_reservations` e nas contagens por status, mas não entram
em noites, valores, `by_month` nem `top_hotels`.

### Noites

```
room_nights = (checkout_date - checkin_date).days * rooms
```

A diferença é não inclusiva: check-in 10/01 e check-out 13/01 = 3 noites. Vale também na virada de
mês/ano.

### Câmbio

* `amount` está na moeda de `currency`.
* Converta com `amount * rate_to_brl`, usando a tabela `data/taxas_cambio.csv`.
* Todas as moedas do dataset têm taxa.

### Totais

* `total_reservations` = número de reservas no arquivo.
* `confirmed_reservations`, `cancelled_reservations` e `no_show_reservations` = contagens por status.
* `total_room_nights`, `total_amount_brl` e `average_nightly_rate_brl` usam apenas `CONFIRMED`.

### Ordem do processamento

```
ler → converter tipos → contar status → filtrar CONFIRMED → agregar
```

### Exatidão

* Use `Decimal`, nunca `float`, no dinheiro.
* Some os valores exatos e arredonde **só na serialização**, para 2 casas, com half-up.
* `average_nightly_rate_brl = total_amount_brl / total_room_nights`; sem noites, `"0.00"`.
* Check-in/check-out são datas civis (`YYYY-MM-DD`).
* `generated_at` usa offset fixo `-03:00` e é ignorado na comparação.

### Ordenação

* `by_month`: `month` ascendente.
* `top_hotels`: `total_brl` decrescente; empate por `hotel_id` ascendente.

## Contrato do CLI

```bash
python -m onfly_report \
  --input data/reservas_hoteis.csv \
  --rates data/taxas_cambio.csv \
  --output output/report.json \
  --top 5
```

|Argumento|Obrigatório|Padrão|
|-|-|-|
|`--input`|sim|—|
|`--rates`|sim|—|
|`--output`|não|`output/report.json`|
|`--top`|não|`5` (inteiro ≥ 1)|

Códigos de saída: `0` sucesso, `2` entrada/argumento inválido, `1` erro inesperado.

O relatório deve ter exatamente este formato:

```json
{
  "report_version": "1.0",
  "generated_at": "2025-07-01T10:00:00-03:00",
  "summary": {
    "total_reservations": 0,
    "confirmed_reservations": 0,
    "cancelled_reservations": 0,
    "no_show_reservations": 0,
    "total_room_nights": 0,
    "total_amount_brl": "0.00",
    "average_nightly_rate_brl": "0.00"
  },
  "by_month": [
    {"month": "2025-01", "reservations": 0, "room_nights": 0, "total_brl": "0.00"}
  ],
  "top_hotels": [
    {"hotel_id": "H01", "hotel_name": "Hotel Exemplo", "room_nights": 0, "total_brl": "0.00"}
  ]
}
```

* Dinheiro é string com exatamente 2 casas (`"1234.50"`), sem separador de milhar.
* `report_version` é sempre `"1.0"`.
* `top_hotels` tem `--top` itens (ou menos, se houver menos hotéis).
* `by_month` usa o mês do check-in.
* Não altere nomes, tipos ou semântica dos campos.

## Amostra de validação

Rode o CLI na amostra e compare com o resultado abaixo:

```bash
PYTHONPATH=src python -m onfly_report \
  --input data/exemplo-validacao.csv \
  --rates data/taxas_cambio.csv \
  --output output/exemplo.json
```

<!-- AMOSTRA:INICIO -->

|Campo|Valor esperado|
|-|-|
|`total_reservations`|12|
|`confirmed_reservations`|9|
|`cancelled_reservations`|2|
|`no_show_reservations`|1|
|`total_room_nights`|28|
|`total_amount_brl`|7566.00|
|`average_nightly_rate_brl`|270.21|

|`month`|`reservations`|`room_nights`|`total_brl`|
|-|-|-|-|
|2025-01|8|25|6684.00|
|2025-02|1|3|882.00|

|`hotel_id`|`hotel_name`|`room_nights`|`total_brl`|
|-|-|-|-|
|H01|Hotel Aurora Paulista|3|1200.00|
|H02|Hotel Mirante|2|1084.00|
|H12|Hotel Miami Beach|1|1000.00|
|H10|Hotel Madrid, Gran Vía|4|900.00|
|H09|Hotel Lisboa, Centro|3|882.00|

<!-- AMOSTRA:FIM -->

A amostra cobre apenas parte dos cenários; use as regras acima para os demais.

## Entrega

Suba o projeto, incluindo a pasta `entrega/`, em um repositório público no GitHub e compartilhe o
acesso com nosso time via forms.

```
entrega/
├── README-ENTREGA.md
└── uso-de-ia/                 # apenas se usou IA
    ├── sessao.md (ou .txt/.jsonl; várias: sessao-01.md, sessao-02.md...)
    └── declaracao.md
```

**`README-ENTREGA.md`** deve ter: defeitos corrigidos, causa raiz em uma frase por defeito, como
investigou/abordou o problema, como validar e resultado do `pytest`.

**Caminho com IA:** use assistentes de IA se quiser. Se usar, inclua a exportação da(s) sessão(ões)
no formato nativo da ferramenta (`/export` do Claude Code, export/JSON de outros assistentes, ou log
consolidado de prompts e respostas) e uma `declaracao.md` com:

* ferramenta e modelo usados;
* o que a IA fez e o que você fez.

**Caminho sem IA:** não é necessária sessão nem declaração. A investigação fica no
`entrega/README-ENTREGA.md`.

## O que não pode mudar

* O comando e os argumentos do CLI.
* O schema e a semântica do relatório acima.
* Os arquivos `data/reservas_hoteis.csv` e `data/taxas_cambio.csv`.

Você pode reorganizar o código interno, adicionar arquivos, melhorar logs e adicionar testes. Os
testes fornecidos devem permanecer inalterados.

## Como será avaliado

A correção verifica se o relatório segue este documento, se os defeitos foram corrigidos e se os
testes fornecidos passam sem alteração. Testes adicionais são considerados.

## Checklist

* [ ] O CLI roda com Python 3.11 (`PYTHONPATH=src python -m onfly_report ...`).
* [ ] O comando de execução gera `output/report.json` sem erro.
* [ ] O relatório segue o contrato acima.
* [ ] `pytest -q` passa com os testes fornecidos.
* [ ] `entrega/README-ENTREGA.md` preenchido.
* [ ] Sessão + declaração apenas no caminho com IA; investigação no README-ENTREGA nos dois caminhos.
* [ ] Repositório público no GitHub com o projeto e o link enviado.
