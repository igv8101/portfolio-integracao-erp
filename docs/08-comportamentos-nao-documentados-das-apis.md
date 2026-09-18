# 8. O que as APIs fazem e a documentação não diz

Comportamentos descobertos por sondagem, experimento controlado ou na pele, entre julho e setembro de 2026. Podem ter mudado desde então. Estão aqui porque custaram horas e porque outra pessoa integrando as mesmas plataformas vai esbarrar neles.

## Tiny / Olist ERP — API v3

**Cota e bloqueio.** 60 req/min no plano usado. Ao exceder, 429 com `x-ratelimit-reset`; a espera pode chegar a 60 s. A cota é da conta: o ciclo de fundo e o painel interativo disputam a mesma janela. Solução: limitador local (janela deslizante ~55/min), prioridade para chamadas dentro de requisição HTTP e catálogo local para busca.

**Token OAuth.** O refresh token expira em 1 dia sem uso e é rotativo — um keep-alive a cada 4 h mantém a sessão viva indefinidamente enquanto algum PC estiver ligado. Se morrer, só uma pessoa reautoriza (OAuth de usuário).

**Situações de pedido.** A tabela genérica da API não bateu com os dados: conferindo 119 pedidos contra `dataFaturamento`, só a situação **0 (em aberto)** segura peça; 1 = faturado; 2 = cancelado; 6 = entregue. Assumir "3 = aprovado" contaria pedidos faturados como reserva.

**`idNotaFiscal` no detalhe do pedido vem preenchido até em pedido cancelado e em aberto** (112 de 112 tinham id). Não prova faturamento. O que vale é `dataFaturamento` no pedido ou a própria nota em situação 6 (autorizada) / 7 (DANFE emitida).

**Reserva de estoque.** Não existe endpoint de reserva (criar, listar, remover). Quem consome a reserva de um pedido é o **lançamento de estoque da nota fiscal** — não a situação do pedido. Marcar como entregue não solta nada. `POST /pedidos/{id}/lancar-estoque` responde 400 quando a nota já existe; `POST /notas/{id}/lancar-estoque` → 204 e a reserva cai exatamente a quantidade da nota (e o saldo cai junto).

**Nota que não lança estoque.** O Tiny se recusa, sem mensagem, a lançar estoque de nota que contenha produto do tipo "com variações" (`tipo=V`) **sem nenhuma variação**. Uma linha dessas trava a nota inteira. Não há campo na API que diga se a nota lançou (o JSON de uma com e uma sem é idêntico) — mas a regra "tem item tipo V → não lança" previu 10 de 10 notas.

**Tipo e situação do produto são imutáveis pela API.** `PUT /produtos/{id}` responde 204 e ignora `tipo` e `situacao`. Na tela, o tipo fica bloqueado depois da criação. O caminho para "inativar" é `DELETE /produtos/{id}` → 204, situação E, funciona mesmo com nota emitida e a nota não perde a referência. `PUT` aceita mudar o SKU. `PATCH /produtos/{id}` responde 400 "método não compatível" — atualização é PUT com o corpo completo (sem o bloco de estoque).

**Criar variação em produto existente funciona:** `POST /produtos/{pai}/variacoes` com `grade: [{chave: 'Tamanho', valor: 'RN'}]` → 201.

**Unicidade de SKU não é garantida.** Dois produtos ativos podem ter o mesmo código. Na tela, variações agrupadas dentro da grade escondem o duplicado; a API mostra os dois.

**Filtro `nome=` em `GET /produtos` casa só o prefixo** da descrição. Palavra no meio do nome devolve zero.

**Balanço (`tipo: 'B'`) é recusado quando o alvo é negativo** ("Ocorreram erros de validação"). Entrada (`'E'`) e saída (`'S'`) relativas funcionam com qualquer saldo.

**Anexos de produto.**
- Listar: `GET /produtos/{id}/anexos`. Criar: `POST` com **array** de `{url, externo}` (objeto solto → "corpo mal formatado"). Apagar: `DELETE /produtos/{id}/anexos` com **objeto** `{id, externo}` no body — o `id` é o inteiro que a listagem devolve, não o UUID do nome do arquivo; `DELETE .../anexos/{id}` não existe (404). `externo` no DELETE tem que bater com o do anexo.
- Não há PUT/PATCH de anexo nem upload de bytes (base64 e multipart são rejeitados). Converter externo → interno é apagar e recriar.
- `externo: false` faz o Tiny **baixar a URL e hospedar** — com duas condições não documentadas que só aparecem em sequência: **máximo 2 MB** e **a URL precisa terminar em extensão de imagem**. URLs de CDN sem extensão: `?x=foto.jpg`, `?.jpg` e `?nome=foto.jpg` falham; **`#foto.jpg` passa** (o validador lê a string inteira; o fragmento é descartado no download; o arquivo chega intacto).

**`GET /estoque/{id}/logs-movimentacao`** existe, mas responde 403 sem o escopo correspondente no cadastro do aplicativo.

**`node:sqlite` não enfileira escritores.** Dois processos gravando no mesmo arquivo dão `database is locked`; sem `busy_timeout` a primeira colisão mata a execução. (Não é do Tiny, mas foi descoberto no mesmo dia e custou o mesmo.)

## Teceo — API de integração da marca

**Cursor de paginação de `/v1/stock`.** Formato: `base64("<createdAt>_<id>")` do último item da página anterior. A primeira página **sem** cursor não é o início da sequência — começar com cursor de época (`1970-01-01T00:00:00.000Z_0000…`). O cursor é inclusivo (repete o último item; deduplicar por id). 1.333 SKUs em 28 páginas e 14 s, contra 966 chamadas individuais em ~2h30.

**Aprovação em duas etapas recria o UUID do pedido** (rascunho → em aprovação → aprovado). Vínculo por UUID quebra; vincular pelo código do pedido.

**Pedidos em rascunho reservam estoque.**

**O endereço de entrega é congelado no pedido** (`conditions[0].deliveryAddress`) no momento da compra; alterar o cadastro do cliente depois não muda o pedido.

**Reverse sync de status: só `{status}` no body basta** (o Swagger lista outros campos como obrigatórios). O import de itens devolve o status como string, valida a disponibilidade contra o **tipo** do pedido (pré-venda só aceita PRE_ORDER/FUTURE_STOCK) e rejeita o **lote inteiro** se um SKU não existir (`skus not found`). `GET /v1/orders/sync/reverse/import/{id}` exige a chave de **escrita** em produção (403 com a de leitura).

**Não existe como trocar o `externalCode` de um pedido já sincronizado.** Report ERROR é recusado ("order is already synchronized"); `PATCH /v1/orders/{id}` só aceita `{archived}`. Contorno: "externalCode efetivo" local por pedido.

**Import de produto é UPSERT.** Re-importar com o mesmo `integrationCode` atualiza (inclusive **preço**, que o `PATCH` de produto não aceita). Com `integrationCode` diferente → "product code already registered". Não há rota para acrescentar um SKU a produto existente — a linha nova é criada na tela e o preço corrigido via re-import. O import é assíncrono (`GET /v1/imports/{id}`).

**Chaves separadas de leitura e escrita** em produção. `/v1/skus` tem `limit` mínimo de 100. `priceTables` é obrigatório por SKU. `collection` aceita só `{code, name}`. CNPJ chega formatado. Só cliente APPROVED aceita sync. A fila de escrita pode levar ~2 min para aplicar um PATCH de estoque — a leitura logo depois mostra o valor antigo, não é regressão. Mensagens de "config inativa" (`integration config is not active`, `service is not allowed on this brand`) significam que o serviço não foi ativado do lado deles, não erro da chamada.

**A base de homologação pode ser resetada sem aviso** (1.013 → 27 SKUs de um dia para o outro). Um ciclo de estoque precisa de uma sentinela para não "sincronizar" um catálogo vazio.

**Swagger completo** está no bundle `swagger-ui-init.js` da UI, mais completo do que o exposto; extraí-lo respondeu sozinho três das cinco perguntas que estavam num ticket para a fornecedora.

## Bling — API v3

**Limites.** 3 req/s e 120 mil/dia **por conta**; bloqueio de IP por 10 min com 300 erros ou 600 requisições em 10 s; 20 acessos a `/oauth/token` em 60 s bloqueiam por 60 min. Fila serializada a 2 req/s por conta; duas filas na mesma conta (script + serviço) estouram o limite.

**Token.** Access token de ~6 h; refresh rotativo com validade estendida a cada uso (~30 dias) — enquanto o serviço sincroniza, nunca precisa reautorizar.

**Situações de pedido de venda** (módulo de vendas): 6 em aberto, 9 atendido, **12 cancelado**, 15 em andamento, 18 venda agenciada, 21 em digitação, 24 verificado.

**NFC-e fica em endpoint separado** (`/nfce`, não `/nfe`). Ler só `/nfe` mostrou 80 notas onde existiam 707. A listagem não traz o modelo: ele sai da **chave de acesso** (posições 20–21: 55 = NF-e, 65 = NFC-e).

**A grade não vem em campo próprio.** A variação está no nome, depois de `Tamanho:`; o "pai" da grade (formato V) é um registro à parte, sempre com saldo zero — nunca deve entrar numa contagem.

**Saldo por depósito** em `GET /estoques/saldos?idsProdutos[]=`; o saldo virtual total já vem na listagem de produtos. NCM só no detalhe do produto (`tributacao.ncm`) — uma chamada por produto.

**`GET /canais-venda`** (não estava na documentação lida) lista loja física e loja virtual (Nuvemshop) com ids; `loja.id = 0` no pedido significa sem canal (digitado ou importação). Pedidos do site trazem `numeroLoja`.

**A API não expõe usuário nem máquina que emitiu a nota.** Para rastrear operador, preencher o vendedor no pedido.

**Datas vêm em horário local sem indicador de fuso** (`2026-08-24 15:14:40`). Converter como se fossem UTC tira 3 h.

## Windows (bônus operacional)

`schtasks /end` não mata o processo Node. `spawn` com `windowsHide: true` esconde o `cmd` que dispara, não a janela da ação da tarefa agendada — a ação precisa ser `wscript.exe arquivo.vbs` (ou um agente) para não abrir console. `Compress-Archive` pode falhar com arquivo recém-copiado (antivírus) — retry resolve. PowerShell come aspas duplas dentro de aspas simples ao passar JSON. O `csc.exe` do .NET Framework, presente em qualquer Windows, compila um agente de bandeja sem baixar nada e sem SmartScreen.

---

## Adendos de setembro (02/09 a 18/09)

### Tiny / Olist v3

**`tipo` e `tipoVariacao` são dois campos.** `tipo`: `V` = "com variações" (é o que faz o ERP recusar o lançamento de estoque da nota), `S` = simples. `tipoVariacao`: `P` = pai da grade, `V` = variação filha, `N` = nenhum. O balde `tipo=V` + `tipoVariacao=N` é o "pai órfão" — 95 no catálogo, invisíveis para qualquer varredura que olhe só os pais. `GET /produtos?limit=100&offset=N&situacao=A` devolve os dois campos: 27 chamadas enumeram 2.656 produtos.

**`POST /notas/{id}/lancar-estoque` solta a reserva E deduz o saldo — no dia da chamada.** Lançar hoje uma nota de junho deduz de um saldo que a contagem física já corrigiu. Mensagem literal ao recusar: "Não é possível lançar estoque de notas fiscais em que foi inserido o produto pai (produto com variações)". A nota é imutável e guarda o **id** do produto da época; migrar o produto não conserta a nota.

**Cancelar o pedido (`PUT /pedidos/{id}/situacao` = 2) solta a reserva sem tocar saldo** — na maioria dos casos. Quando "toca", conferir a autoria antes de concluir que foi o ERP (PM17).

**`PUT /pedidos/{id}` não aceita `itens`** — só datas, observações e pagamento. Não há como trocar o produto de uma linha pela API; na tela, remover a linha e adicionar de novo buscando o mesmo SKU (a busca só oferece produto ativo).

**O pedido guarda o id do produto, não o SKU.** Pedidos criados antes de uma migração de produto continuam apontando para o id velho, e a nota herda.

**O marcador "E" (estoque lançado) não existe na API.** Notas com e sem lançamento têm JSON idêntico (`marcadores` vazio nas duas). Só na tela.

**`dataInicial` da listagem de notas filtra por data de EMISSÃO**, não de alteração. Uma janela de "últimos 45 dias" nunca vê uma nota antiga cancelada hoje.

**Situações de pedido (doc oficial, conferida em 14/09):** 8 dados incompletos · 0 aberta · 3 aprovada · 4 preparando envio · 1 faturada · 7 pronto p/ envio · 5 enviada · 6 entregue · 2 cancelada · 9 não entregue. Na prática a conta usava só 0, 1, 2 e 6 — "em cesto" (4/7) nunca aparece.

**Situações de NOTA diferem entre Tiny e Bling** para o mesmo número. Tiny: 1 pendente · 2 emitida · **3 cancelada** · 4 aguardando recibo · 5 rejeitada · 6 autorizada · 7 DANFE emitida · 8 registrada · 9 aguardando protocolo · 10 denegada. Bling: 1 pendente · **2 cancelada** · 3 aguardando recibo · 4 rejeitada · 5 autorizada · 6 DANFE · 7 registrada · 8 aguardando protocolo · 9 denegada · 10 consulta · 11 bloqueada. Um mapa por família, constante nomeada, nunca número solto.

**Contas a receber nascem da NOTA**, não do pedido: histórico "Ref. a NF nº X (parcela n/m)", `numeroDocumento` "00X/0n" — 100 de 100. `/contas-pagar` existe.

**Uma listagem de produtos filtrada por nome devolve o item duplicado entre páginas** ocasionalmente (visto também no Bling): deduplicar por id.

### Teceo

**Não há histórico de estoque.** `/v1/stock/history`, `/v1/stock/{sku}/history`, `/v1/audit-logs`, `/v1/history`, `/v1/events`, `/v1/stock-movements` — todos 404. Por SKU só existe estado atual + `createdAt` + `updatedAt` (apenas a última alteração). O histórico tem de ser mantido do lado da integração.

**`/v1/catalogs*` existe na spec e responde 403** para as chaves da marca (leitura e escrita), como `accounts-receivable`.

**Mudança de API em 28/08:** `availableForPreOrder`, `controlMaterialStock`, `preOrderStartDate`, `preOrderEndDate` saíram de `GET /v1/skus` e foram para `availabilities` em `GET /v1/skus/{id}`. Ler só `id`/`code` na listagem tornou a mudança invisível.

**Catálogo dinâmico não converte um catálogo existente** — o toggle é desabilitado na edição (`disabled: true` no DOM) e habilitado só na criação. Regras possíveis: disponibilidade, coleção, classificação — **não** categoria. A doc diz que as três são obrigatórias; a tela diz "pelo menos uma".

**Aprovar exige `availableAmount ≥ quantidade`, e o próprio pedido reserva o que acabou de ser lançado** — por isso "destravar" precisa de uma folga de +1 (e de um passe que a desfaça, PM19).

**A plataforma aceita uma sessão de navegador por vez** por usuário. **A API fica indisponível nos fins de semana** e volta segunda de manhã (informação da operação; os ciclos entram em backoff sem dano).

**Exportação de pedido:** CSV com 60 colunas, `;`, uma linha por SKU repetindo o cabeçalho; PDF gerado no servidor, com o nome do produto truncado por linhas × largura — "paisagem + ampliado" foi a única combinação que mostrou todos os nomes inteiros. Um conversor que lê colunas **por nome** (9 obrigatórias) sobrevive a colunas novas.

### Bling v3

**`valorNota` só vem no DETALHE** (`/nfce/{id}`, `/nfe/{id}`); a listagem traz o campo vazio. Desconto dado na nota (não no item) aparece só ali; os itens não têm campo `desconto`. Regra para o espelho: coluna própria + `COALESCE(excluded.valor, nf.valor)` no upsert da listagem, senão a listagem apaga o que o detalhe trouxe. Idem para `notaFiscal.id` do pedido.

**`PUT /pedidos/vendas/{id}`** funciona mandando o detalhe inteiro sem `id`, `total` e `totalProdutos`. Recusa com code 22 ("somatório das parcelas difere do total") quando as parcelas não batem com o total que ELE recalcula: Σ(quantidade × valor − desconto do item em **reais**). O `total` do GET **ignora** esse desconto — um pedido pode chegar já inconsistente, e aí até um PUT idêntico falha.

**A listagem de NFC-e pode devolver o mesmo id em duas páginas** (788 linhas para 787 notas). Deduplicar.

**Múltiplos depósitos:** `/depositos` lista; saldo por depósito em `/estoques/saldos?idsProdutos[]=`; entrada/saída (`E`/`S`) por depósito somam/subtraem no ERP (sem corrida), balanço (`B`) fixa.

### Node / SQLite / Windows (bônus)

`hidden` não sobrevive a `display:` inline — modal com `style="display:flex"` aparece mesmo com `hidden`; precisa de `[hidden]{display:none !important}`. Gerar HTML **depois** de `res.writeHead(200)` transforma erro em 200 vazio. Trocar arquivo de banco com conexão WAL aberta corrompe — sincronizar linha a linha via `ATTACH`. Um `.exe` em uso não pode ser sobrescrito por um atualizador. Arquivos recém-criados por um processo às vezes levam ~1 min para ficarem legíveis por outra ferramenta ("hardlinked").
