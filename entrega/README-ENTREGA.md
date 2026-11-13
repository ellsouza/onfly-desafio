# Entrega do desafio Onfly

## Demonstração

[Acessar o dashboard publicado no GitHub Pages](https://ellsouza.github.io/onfly-desafio/).

## Defeitos corrigidos

1. Cálculo de noites por quarto estava usando apenas o dia do mês em vez do intervalo total entre datas.
   - Causa raiz: a função `calculate_room_nights()` subtraía `checkout.day - checkin.day`, ignorando a diferença total em dias e gerando valores negativos ao cruzar mês/ano.
   - Abordagem: o cálculo passou a usar `(checkout - checkin).days * rooms`, preservando a diferença não inclusiva e multiplicando pelo número de quartos.

2. Conversão para BRL estava invertendo a operação e usando `float`, o que gerava erros de precisão.
   - Causa raiz: `convert_to_brl()` dividia `amount` pela taxa em vez de multiplicar e convertia via `float`, quebrando o cálculo financeiro.
   - Abordagem: a conversão passou a usar `amount * rate` com `Decimal`, mantendo precisão decimal e alinhando com a regra do negócio.

3. Política de faturamento estava permitindo cancelados e sem considerar `NO_SHOW` como não faturável.
   - Causa raiz: `is_billable()` retornava `status != "CANCELLED"`, incluindo `NO_SHOW` e qualquer status diferente de cancelado nas métricas financeiras.
   - Abordagem: `is_billable()` foi ajustada para aceitar somente `CONFIRMED`, exatamente como definido no contrato.

4. Ordenação do ranking de hotéis estava invertida e também incluía o valor DESC em ordem errada.
   - Causa raiz: no `ReportBuilder`, os hotéis eram ordenados por maior valor com sinal invertido e apenas por valor, em vez de `total_brl` decrescente com desempate por `hotel_id` ascendente.
   - Abordagem: a ordenação passou a usar `(-amount, hotel_id)`, mantendo a seleção dos `top_n` após ordenar corretamente.

5. Leitura do CSV de reservas truncava a lista após 200 linhas.
   - Causa raiz: o repositório de reservas interrompia a leitura em 200 registros, causando relatórios incompletos.
   - Abordagem: o corte artificial foi removido para processar todas as reservas do arquivo.

6. Normalização de status e moeda estava inconsistente.
   - Causa raiz: `RowParser` preservava letras minúsculas em `status` e `currency`, o que quebrava comparações e lookup de taxas.
   - Abordagem: status e moeda passaram a ser normalizados para maiúsculas antes de armazenar e validar.

## Como o problema foi investigado e abordado

A investigação começou pelos testes automatizados, que apontaram divergências em conversão cambial, cruzamento de mês, contagem de status, ranking de hotéis e cálculo de noites. Cada falha foi rastreada até sua função de origem e corrigida sem alterar o contrato do CLI, a estrutura de campos do relatório ou os nomes esperados pelo desafio.

## Como validar

Foram executados os testes do projeto e gerados relatórios tanto para a amostra quanto para o arquivo completo:

- `py -3.14 -m pytest -q` -> resultado: `5 passed in 1.29s`
- `py -3.14 -m onfly_report --input data/exemplo-validacao.csv --rates data/taxas_cambio.csv --output output/exemplo.json` -> resumo validado com `total_reservations=12`, `total_amount_brl=7566.00` e `average_nightly_rate_brl=270.21`
- `py -3.14 -m onfly_report --input data/reservas_hoteis.csv --rates data/taxas_cambio.csv --output output/report.json` -> execução completa com 300 reservas, 235 confirmadas, 1.347 noites e total de `396170.00` BRL.

## Resultado do pytest

`5 passed in 1.29s`

## Uso de IA

Houve uso de IA como apoio técnico na análise, implementação assistida, documentação e dashboard. Minha contribuição incluiu definir prioridades e critérios, elaborar prompts com contexto e resultados esperados, avaliar as respostas e conduzir revisões até o material atender ao objetivo.

Na apresentação, dei a direção da identidade visual: selecionei referências, defini o alinhamento à marca Onfly, pedi a paleta azul e a padronização das cores e orientei hierarquia, tipografia, imagens, logo e rodapé. Também solicitei a revisão de responsividade e dos requisitos do README.

Não foi utilizado RAG: o desafio trabalha com CSVs estruturados e regras explícitas, então o processamento determinístico foi suficiente e mais direto de validar. A ferramenta, os modelos e o registro consolidado da sessão estão em [`uso-de-ia/declaracao.md`](uso-de-ia/declaracao.md) e [`uso-de-ia/sessao.md`](uso-de-ia/sessao.md).
