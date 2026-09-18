# 7. Métricas

Números medidos, com a fonte de cada um. Sem valores financeiros (regra deste repositório). Datas de 2026.

## Prazo

| Marco | Data | Dias desde o início |
|---|---|---|
| Documentação da API recebida | 27/07 | 0 |
| Pedidos homologados de ponta a ponta | 29/07 | 2 |
| Nota fiscal em ciclo completo | 31/07 | 4 |
| Carga real de estoque e produto nos dois sentidos | 03/08 | 7 |
| Edição espelhada, cancelamento reverso, fotos, cadastro | 07/08 | 11 |
| **Go-live em produção** | 10/08 | 14 |
| Leader lease na nuvem; painel de relatórios do atacado | 18–21/08 | 22–25 |
| Segunda empresa (Bling) espelhada e com painel | 24/08 | 28 |
| Acesso único das três empresas | 25/08 | 29 |
| Guarda de estoque, retomada automática, réplica, agente, simulação | 31/08 | 35 |
| 95 pais órfãos migrados em lote; trava manual; planilha passiva | 08/09 | 43 |
| Pedidos, notas fiscais, financeiro e relatórios imprimíveis nas 3 empresas | 14/09 | 49 |
| ERP principal sem saldo negativo nem reserva presa; saúde vigiando telas; folgas com livro-razão | 18/09 | 53 |

## Escala

| Item | Valor | Fonte |
|---|---|---|
| Empresas no sistema | 3 | — |
| Plataformas integradas | Teceo, Tiny/Olist v3, Bling v3 (2 contas), Nuvemshop (como canal) | — |
| Computadores | 4 (1 host + 3 standby, papéis rotativos) | painel de computadores |
| Pessoas com login | 5, com papéis e permissões por empresa | banco de acesso |
| SKUs na plataforma B2B (produção) | ~1.330 | listagem por cursor, 28 páginas |
| Produtos no catálogo do ERP principal | ~2.700 | catálogo local |
| Saldos sincronizados | ~1.400 | `stock_state` |
| Segunda empresa | 2.563 produtos · 311 contatos · 797 pedidos · 707 notas (80 NF-e + 627 NFC-e) | espelho do Bling |
| Notas fiscais em cache (empresa principal) | 623 desde jan/2025 · 16.215 itens · 1.897 SKUs distintos | `nf_cache` |
| Fotos no catálogo | 454 (PNG; 57% cabem no limite de 2 MB da API) | medição por script |

## Código

| Item | Valor |
|---|---|
| Dependências de runtime | 0 |
| Testes unitários | 49 (de 40 na primeira semana) |
| Simuladores de mecanismo em banco descartável (setembro) | trava manual 13 · espelhamento 22 · página 12 · cota 9 · manifesto 8 · sincronização de cadastro 4 · guarda de produto novo 12 |
| Rotas vigiadas pela aba Saúde | 43 (43/43 em 7,4 s) |
| Cenários do harness com ERP falso | 7 (3 falharam na primeira rodada; 7/7 depois) |
| Cenários da auditoria da guarda | 5/5 |
| Cenários da retomada automática | 8 |
| Testes da guarda de produto novo | 12/12 |
| Scripts operacionais versionados | ~270 |
| Arquivos no pacote do instalador | 102 (semana 1) |
| Módulos em `src/sync` | ~15 (pedidos, clientes, estoque, NF, edições, fotos/cadastro, financeiro, históricos, caches, réplica, update, lease, guarda, alerta) |
| Documentos de trabalho datados | 100 (27/07 → 18/09) |

## Desempenho

| Medição | Antes | Depois | Como |
|---|---|---|---|
| Listar todo o estoque na Teceo | 966 chamadas SKU a SKU, ~2h30 | 28 páginas, **14 s** | cursor de paginação decifrado |
| Busca por descrição no painel | 30 s a 3 min | **0,02 s** | catálogo local + prioridade interativa |
| Grade de um modelo tamanho a tamanho (saldo ao vivo) | minutos | 3,1 s | idem |
| Esperas de 429 do Tiny | 356 numa hora | raras | limitador local + varredura fora do boot |
| Espelho completo da segunda empresa (Bling) | — | ~34 s | fila a 2 req/s, paginação de 100 |
| Failover de host | descoberta por rede local a cada 30 min | até 5 min (lease) | Redis |
| Perda residual no failover | banco do standby "parado há dias" | ≤ 2 min de escritas | réplica por `VACUUM INTO` |
| Instalação/atualização dos standbys | zip distribuído à mão | automática, arquivo a arquivo, no máx. 1×/30 min | manifest + hash |
| Varredura Tiny → Teceo (1.355 SKUs) com outros PCs ligados | 0 SKUs em 70 min (pendurada) | 25 SKUs/min; 3 reinícios no meio = 0 perdidos | PM15: cota, 429, checkpoint |
| Enumerar o catálogo do ERP com os dois campos de tipo | 2.656 chamadas | 27 chamadas | paginação de 100 |
| Conferência relatórios × 2 ERPs × 3 empresas, 90 dias, dia a dia | manual | ~2 min, script só leitura, 0 divergências | conferência diária |

## Incidentes medidos e reparados

| Incidente | Extensão | Reparo |
|---|---|---|
| Baixa por nota em dobro (PM1) | 60 SKUs, 115 peças a mais, 3 notas | 57 SKUs, 110 peças devolvidas, 0 problemas (3 excluídos por contagem posterior) |
| `disponivel` em vez de `saldo` (PM5) | 242 SKUs sobrescritos numa rodada | 7 modelos / 30 SKUs alinhados à mão; ciclo corrigido |
| Reserva fantasma (PM2) | 288 SKUs com reserva > 0; 124 batendo com pedidos faturados | 31 peças devolvidas em 20 SKUs; 75 notas na fila |
| Pais órfãos (tipo "com variações" sem variação) | 19 produtos (14 + 5 pela faxina) | todos recriados como simples, mantendo o SKU |
| SKU duplicado no Tiny (PM6) | 2 SKUs | 1 isolado; sync congela ambos |
| Estoque fantasma de cadastro (PM9) | 128 SKUs novos num dia, 27 com 20–99 unidades e 0 notas de entrada | 10 zerados; ~32 em auditoria; guarda de produto novo |
| Contagem física importada | 966 SKUs comparados | 304 corrigidos (+1.206 peças), 659 já corretos, 3 erros |
| Divergências de cadastro de clientes | 183 vinculados → 45 divergentes (16 endereço, 9 contato, 20 cosméticas) | decisão caso a caso pelo dono |
| Fotos externas → internas | 454 | 290 convertidas; 226 mantidas externas (> 2 MB); 1 erro |
| Pais órfãos escondidos por `tipo` × `tipoVariacao` (PM14) | 95 | 95/95 migrados em 5 lotes, 0 erro; 94 antigos excluídos |
| Lançamento tardio de nota (PM13) | 113 SKUs negativos, −384 peças; 100 sem dedução nossa | lote v2: 91 notas, 739 reservas soltas com saldo idêntico; 12 pedidos cancelados, 348 reservas; 114 negativos zerados (−375) |
| Reservas indevidas (varredura de 09/09) | 2.020 peças em 518 SKUs, 0 legítimas | 813 resolvidas em 14/09; ERP sem reserva presa em 18/09 |
| Folga do destravar (PM19) | 77 desbloqueios, +172 peças na plataforma | 55/56 alinhados ao vivo; 0 resíduo; livro-razão + passe de 20 min |
| Quase-acidente da baixa em massa (02/09) | primeira conta: 618 notas / 32.587 peças | escopo real: 65 notas / 3.170 peças — e mesmo assim não aplicado (físico ≥ sistema) |
| Compensação errada do cancelamento (PM17) | 11 SKUs | 6 ficaram com 1 peça a menos (decisão pendente); código corrigido para consultar autoria |

## Achados de negócio que só o sistema enxergou

- No varejo da segunda empresa, **43% das peças** que saíram por nota (4.008 de 9.233) foram lançadas em código genérico — a peça de verdade não fica registrada, e o relatório de "peças paradas" precisa dizer isso na tela.
- Na mesma empresa, a **importação era maior que o varejo** em valor e estava contaminando todas as análises até ser separada pelo modelo da nota (NF-e × NFC-e, lido da chave de acesso).
- No atacado, **a grade furada no tamanho do meio** tem custo mensurável: perda direta (o que o tamanho furado vendia) mais venda casada (peças de outros tamanhos vendidas nas mesmas notas). Top 20 modelos por perda, com filtro de período.
- Régua de inatividade do atacado é **1 ano**, não 60 dias — a régua "de livro" marcaria clientes normais como perdidos.
- No financeiro do ERP principal, a maioria das contas a receber aparece vencida e em aberto — não é bug: parcelas recebidas e nunca baixadas no ERP. O caixa do painel existe para isso.
- Um "cliente" com ciclo de compra de 8 dias era na verdade uma temporada de 8 pedidos em 3 semanas seguida de 405 dias de silêncio: a fórmula precisava agrupar compras próximas e exigir 3 compras separadas antes de afirmar um ritmo.
