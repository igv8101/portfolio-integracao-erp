# 6. Linha do tempo

Todas as datas de 2026. Uma pessoa, em paralelo com a operação diária do estoque e do atendimento.

## Semana 0 — 27/07 a 02/08 · do zero à homologação

| Dia | O que aconteceu |
|---|---|
| 27/07 | Documentação e Swagger da Teceo recebidos. Reunião de levantamento de requisitos marcada. |
| 29/07 | **Pedidos homologados de ponta a ponta**: 14 pedidos ao vivo, SUCCESS com número do ERP reportado à Teceo; ERROR e reprocesso comprovados. |
| 31/07 | **Nota fiscal em ciclo completo**: NF do Tiny → invoice + XML na Teceo. |
| 03/08 | Carga real de estoque Teceo → Tiny (1.013 SKUs, 15.435 unidades, 734 ajustados). Produto Tiny → Teceo funcionando (import assíncrono + estoque automático pós-import). 20/20 clientes corrigidos. Plano de API do Tiny validado (60 req/min, tratamento de 429). **Decisão: Tiny é a fonte da verdade do estoque.** |

## Semana 1 — 03/08 a 09/08 · o que a integração "de livro" não previa

| Dia | O que aconteceu |
|---|---|
| 07/08 manhã | **Edição espelhada de pedidos** Tiny → Teceo (status e itens, reverse sync assíncrono), 3/3 ao vivo. Auditoria completa: base de homologação da Teceo detectada **resetada** (1.013 → 27 SKUs) — não fomos nós; sentinela adicionada. Pedido com 961 tentativas → backoff. |
| 07/08 tarde | **Quatro integrações novas no mesmo dia**: cancelamento Teceo → Tiny (proteção a faturado, anti-eco); financeiro Tiny → Teceo (pronto, bloqueado por 403 do lado deles); fotos Teceo → Tiny (body do anexo descoberto: array de `{url, externo}`); cadastro Tiny → Teceo. |
| 07/08 fim | Auto-recuperação de "skus not found": painel mostra os SKUs faltantes com link já preenchido e tenta o envio automático do produto (1×/24h por SKU). Instalador regenerado e validado a partir do próprio zip (suíte 40/40). |
| 08–09/08 | Fim de semana: PC desligado, token OAuth do Tiny morreu. Motivou o keep-alive rotativo e a reautorização assistida. |

## Semana 2 — 10/08 a 16/08 · go-live e a primeira crise de estoque

| Dia | O que aconteceu |
|---|---|
| 10/08 manhã | **Chaves de produção recebidas** (leitura + escrita), validadas sem escrita antes da virada; suporte a chave dupla. **Atualização automática pela rede** (manifest + hash) testada. Históricos de alteração de produto e cliente; relatórios com Chart.js; backup diário; keep-alive de token. Instalador v2. Primeira carga de vendas (12 meses). |
| 10/08 tarde | **Produção ativa.** A Teceo ainda precisava ativar os serviços de pedidos/clientes/produtos do lado dela; estoque já funcionava. Anti-duplicação de pedidos; contagem manual no painel gravando nos dois sistemas; contagem física importada (966 SKUs comparados, 304 corrigidos). **Saga das duplicatas** (PM4): UUID recriado na reaprovação. Limpeza de homologação com backup. |
| 10/08 noite | Pedido do dono: eliminar dependências da fornecedora. Swagger completo extraído do bundle JS. **Cursor de paginação do estoque decifrado** (`base64("<createdAt>_<id>")`, primeira página com cursor de época, inclusivo): 966 chamadas / ~2h30 → 28 páginas / 14 s. Preço de produto contornado via re-import (UPSERT). Vínculos presos contornados com "externalCode efetivo". Ticket para a fornecedora reduzido a um item. |
| 11/08 | Busca por descrição na contagem; painel como PWA para o celular; redirecionador dos standbys; firewall e DHCP. |
| 12/08 | Contagem física. |
| 13–14/08 | **Fotos**: envio direcionado; exclusão de anexo (rota e formato descobertos por sondagem); conversão de externa para **interna** (hospedada pelo Tiny) driblando o limite não documentado de 2 MB e a exigência de extensão na URL (fragmento `#foto.jpg` passa no validador e é descartado no download): 290 convertidas de 454. Tabela `fotos_convertidas` para o ciclo não desfazer. |
| 14/08 | **Mistério da baixa resolvido** (10 de 10 notas): o Tiny se recusa a lançar estoque de nota com "pai órfão". **Bug do `disponivel`** (PM5) corrigido; ciclo de estoque religado a 6 h; **baixa por nota fiscal** pela integração (10 min). Descoberta: `POST /produtos/{pai}/variacoes` funciona; import da Teceo aceita produto existente quando o `integrationCode` bate. |

## Semana 3 — 17/08 a 23/08 · robustez multi-PC e os relatórios do atacado

| Dia | O que aconteceu |
|---|---|
| 17/08 | Busca por descrição corrigida (filtro do Tiny casa só prefixo → catálogo local com todas as palavras). Painel reorganizado: visão de todos em cima, histórico técnico recolhido. Clientes presos em ERROR por token morto: rodada aborta sem reportar ERROR, e ERROR passa a ser reprocessado com respiro. Chart.js do CDN devolvia 404 → servido localmente. Migração dos primeiros pais órfãos para produtos simples. |
| 18/08 | NF sem "E" → 3 pais órfãos novos migrados; `DELETE /produtos/{id}` funciona mesmo com nota emitida (a API ignora `situacao` no PUT). **Cota do Tiny e lentidão do painel resolvidas** (PM10). **Cadeado de host na nuvem** (leader lease em Redis). Chegada de mercadoria lançada nos dois sistemas. |
| 19/08 | Botão "Forçar integração de pedidos" (diagnóstico mostrou que o pedido "não integrado" aguardava o 2º clique de aprovação na Teceo). Endereço congelado no pedido pela Teceo → regra de usar o cadastro quando o cliente tem um único endereço. |
| 20/08 | Cadeado publica também o IP do host. **Sincronia de catálogo/saldos host → standby** com prioridade por data. Devolução automática à cesta em cancelamento (só se houve contagem depois da integração). Relatórios: inativos por período, estoque parado **por nota fiscal**; cache de notas desde jan/2025 (623 notas, 16.215 itens, 1.897 SKUs); **cache de vendas parado 9 dias** achado e corrigido (PM11). Divergências de cadastro de clientes (183 vinculados, 45 divergentes) em planilha para decisão caso a caso. Roadmap v2 revisado com o dono: régua de inatividade do atacado é 1 ano, não 60 dias; reposição não existe (importação limitada por cota) — indicadores servem para priorizar importação e alocar; **ruptura de grade é o item mais importante**. Pesquisa de automação de pagamento (Pix com webhook). |
| 21/08 | **Onda 1**: ruptura de grade (FURADA/PONTAS/ACABANDO/COMPLETA/ESGOTADO, ordenada pelo que o buraco vendia); corte de pedido; faxina de cadastro (achou 5 pais órfãos novos, 2 SKUs duplicados, 21 ativos sem SKU); régua de clientes; saúde de todos os ciclos com **alerta nativo do Windows**. **Onda 2** no mesmo dia: perfil de compra por cliente com frases por regra, sugestões do estoque atual, **previsão de perdas** por grade furada (perda direta + venda casada), página de clientes. Correção de categorias (descrições de NF sem acento; "Pijama body" separado). Feedback: "ficou perfeito". |

## Semana 4 — 24/08 a 30/08 · as outras duas empresas

| Dia | O que aconteceu |
|---|---|
| 24/08 | **Segunda empresa conectada ao Bling** (OAuth v3, fila serializada a 2 req/s, espelho de produtos/contatos/pedidos/NF em ~34 s; banco físico separado, separação provada por script). Login rígido com scrypt. Relatórios, grades e clientes espelhados e adaptados ao **varejo** (pedidos como fonte). Descobertas: situação 12 = cancelado; NFC-e em endpoint separado (80 notas viraram 707); NF-e é importação, não varejo; três sinais para identificar venda de feira; produtos genéricos de balcão; canais de venda (loja física × site Nuvemshop). NCM sugerido pelos irmãos da categoria. Auditoria com hostname/IP/origem. Bug do fuso (PM12). Bug do "ritmo de compra" (temporada lida como ciclo). WAL + `busy_timeout` no banco da segunda empresa. Alerta ativo próprio; backup das duas. |
| 25/08 | **Acesso único das três empresas**: banco de identidade separado, papéis e permissões por empresa, tela de usuários, migração automática de senhas. Terceira empresa estruturada (banco, OAuth, callback distinto de propósito). Fábricas de cliente e espelho do Bling por empresa. **Brecha 1 achada e fechada** (PM7). Achado: 43% das peças saídas por nota no varejo em código genérico. |
| 28/08 | Planilha de contagem **offline** (baixar, preencher, subir com prévia; linha em branco não vira zero). Variações do Bling deduzidas do nome (grade sem gastar API). Histórico de nomes nas três empresas. Janela de CMD nos standbys (duas causas: freio de 5 min e ação da tarefa agendada). **"Onde está a peça"** (situações do Tiny conferidas nos dados; espelho de reserva mentia — consulta ao vivo). **Reserva fantasma** (PM2): experimento controlado, receita, ferramenta de reparo com 4 erros num dia, 31 peças devolvidas, 0 pendentes. **Brecha 2 e login antigo fechados**. Terceira empresa ligada (54 produtos; depósito de fulfillment excluído da contagem). |

## Semana 5 — 31/08 a 01/09 · blindagem

| Dia | O que aconteceu |
|---|---|
| 31/08 | **PM1** (baixa em dobro): livro por item, reparo de 110 peças com 0 problemas. **Guarda de estoque** com chave obrigatória; **retomada automática** (8 cenários); **réplica dos bancos** host → standby (o standby que assumiu de manhã tinha banco parado há dias). **ERP falso e 7 cenários** de simulação — 3 falharam, corrigidos, 7/7. **Agente de bandeja** em C# compilado no Windows; senha mestre; painel de computadores; instalador refeito para subir pelo agente. Login como primeira tela; cada pessoa entra na empresa dela. |
| 01/09 | Senha mestre autoritativa no host (a primeira versão trancava a manutenção remota). **Dois portões de login** (PM7, brecha 3). **SKU duplicado com failover** (PM6). Banco corrompido (PM8) → **WAL**. **Auditoria de erros de estoque** com harness seguro: 5/5. **Auditoria de erros incomuns**: split-brain no lease (PM3) provado e corrigido; multi-depósito, clock skew, float, réplica velha verificados. **Estoque fantasma** (PM9): prova de que a automação não inventa estoque; **guarda de produto novo** (12/12 testes). `scripts/` fora do manifesto de distribuição. Redesign do painel: casca compartilhada com paleta por empresa. |

## Semana 6 — 02/09 a 08/09 · o que a contagem física ensinou

| Dia | O que aconteceu |
|---|---|
| 02/09 | A plataforma B2B **não expõe histórico** de estoque (6 rotas sondadas, todas 404) — o histórico real é o nosso, em três camadas com datas de início documentadas. Estoque fantasma localizado na coleção certa (todos os 34 SKUs nascidos com saldo alto eram de uma coleção; a suspeita inicial era outra). **Quase-acidente evitado**: a primeira contagem de "notas sem baixa" misturou notas importadas do ERP anterior (553, identificáveis pelo bloco de ids) com as emitidas pelo ERP atual (83) — aplicar aquilo deduziria 32 mil peças em dobro. Escopo corrigido para 65 notas / 3.170 peças, e mesmo assim **não aplicado**: a contagem física da primeira página mostrou físico acima do sistema e muito acima do pós-baixa. Regra confirmada: todo número contado é o da cesta, já sem as peças separadas. |
| 03/09 | Pedido de um depósito secundário de feira na empresa de varejo (levantamento antes de código, por causa do quase-acidente). |
| 08/09 | NF nova travada explicada: pedido anterior à migração guarda o **id** do produto velho (PM14). Bug do "2 → 4" no painel: contagem e baixa gravavam o campo errado em `stock_state`. **Trava manual**: alteração manual vence a automática por 30 min. **Planilha passiva** nos 4 PCs de hora em hora, custo de API zero. **95 pais órfãos** escondidos por dois campos com nomes parecidos — 95/95 migrados em lote, zero erro. **Lançamento tardio come o estoque** (PM13): 113 SKUs negativos, 100 sem nenhuma dedução nossa. |

## Semana 7 — 09/09 a 15/09 · reservas, varredura e as telas das três empresas

| Dia | O que aconteceu |
|---|---|
| 09/09 | Varredura de reservas pelos 3 estados do dono (em cesto / finalizado / nem separado) nos 782 SKUs com pedido: 2.020 peças reservadas em 518 SKUs, nenhuma legitimamente; o ERP usa só 4 situações de pedido, então "em cesto" não existe no sistema. Livro-razão no script depois de perder 35 min de consulta num desligamento. |
| 11/09 | Estoque parado há 3 dias (PM15): fome artificial do "painel em uso", efeito manada no 429 sem teto, cedência por SKU. Checkpoint de varredura com retomada. Renomeação dos PCs. |
| 14/09 | Passe completo pela primeira vez desde 08/09: 1355/1355. Catraca de mão única corrigida; standbys em código antigo eram a fonte das recusas; versão passa a excluir `.env`. **Decisão documentada: ERP próprio × continuar a integração** (com a leitura jurídica da Lei 9.609/98 e o precedente do TST) — recomendação: nem um nem outro; produto pequeno em cima dos ERPs, clean-room. Redesign: início em grade, aba de saúde, histórico paginado. **Tela de pedidos** nas 3 empresas com edição gravando no ERP de varejo (prévia obrigatória; armadilha do PUT com parcelas), selo de estoque por item, emblema de NF e impressão A4. **Aba de notas fiscais** nas 3 (situações de NF diferem entre os dois ERPs — mapas separados, constante nomeada). Auditoria de canceladas por três portas (PM20). **Financeiro** nas 3 empresas: contas a receber, a pagar e caixa, espelho do ERP + lançamentos manuais. Relatórios com período personalizado, impressão por relatório, motor de relatórios do ERP de varejo, "só NFC-e", ajuste +/−/= de estoque. Divergência de desconto na nota (PM20). Lote de reservas v2 com invariante: 813 reservas resolvidas no dia sem tocar saldo. |
| 15/09 | Conferência diária automatizada: relatórios × os dois ERPs, dia a dia, 90 dias, 3 empresas — 0 divergências depois de achar que o espelho da terceira empresa nunca rodava sozinho. **Login caiu** (PM16): host do dia com banco de acesso vazio; blindagem na adoção de réplica; pedido de "tirar a senha" recusado. |

## Semana 8 — 16/09 a 18/09 · fechar brechas

| Dia | O que aconteceu |
|---|---|
| 18/09 | Sincronização do cadastro linha a linha nos standbys — e a descoberta de que a blindagem de 15/09 tinha sido sobrescrita pelo ciclo de atualização (PM16). Análise de impacto da nova versão da plataforma B2B (nada a mudar na integração; catálogos dinâmicos não convertem os existentes; conversor CSV seguro por ler colunas por nome). Configuração de exportação de pedido que resolve nomes cortados (sem código). Últimos 12 pedidos "produto pai" cancelados, 348 reservas soltas; **cancelamento devolveu estoque** — era a regra da cesta, e a compensação foi o erro (PM17). 114 SKUs negativos zerados: o ERP não tem mais saldo negativo. Faturamento por dia na empresa principal; modal que abria sozinho (`hidden` × `display:flex` inline). Home das 3 empresas com os logos reais na casca compartilhada. Dias zerados nos relatórios. **`server.ts` regredido por cópia velha** (PM18) → verificador de 43 rotas como ciclo de saúde. **Folga do destravar** (PM19): livro-razão e passe de 20 min. |

## Estado em 18/09

Três empresas no mesmo sistema com pedidos, notas fiscais, financeiro, estoque e relatórios por empresa; quatro PCs sob agente com código, bancos e cadastro sincronizados; ERP principal sem saldo negativo nem reserva presa; a saúde vigia ciclos **e** telas; lease, guarda de estoque, retomada, cesta e folgas, cada um com livro-razão próprio. Pendências registradas: falhar barulhento ao virar host com cadastro vazio; devolver 1 peça em 6 SKUs (PM17); reavaliar catálogos dinâmicos após a virada da plataforma; roadmap com pagamento automatizado (Pix com webhook) e webhooks do ERP.
