# 4. Problemas difíceis — post-mortems

Formato: sintoma → investigação → causa → correção → o que ficou de lição. Datas de 2026. Quantidades reais; sem valores financeiros, sem nomes.

---

## PM1 · A baixa por nota fiscal deduzia o mesmo item várias vezes (31/08)

**Sintoma.** Um cliente pediu uma peça, ela existia, foi separada — e as variações pedidas apareceram zeradas. O histórico do SKU mostrava, para uma única nota com quantidade 1, **cinco deduções** em horários diferentes na mesma noite (5→4→3→2→1→0).

**Investigação.** O log de contagens da família do produto, cruzado com os itens da nota, mostrou o padrão em três notas "travadas" (notas que o ERP se recusa a lançar no estoque, ver PM2). Uma delas: item de quantidade 2 deduzido em seis passadas — 12 peças. Total medido: **60 SKUs, 115 peças a mais**.

**Causa.** O módulo de baixa deduzia item a item e só **no fim** marcava a nota como feita. Não existia registro por item. Qualquer interrupção no meio — erro da API, `database is locked` ao marcar, ou o serviço reiniciando (naquele dia reiniciou várias vezes) — fazia a nota voltar inteira no ciclo seguinte (10 min) e deduzir tudo de novo, em cima do saldo já reduzido.

**Correção.**
1. Tabela `nf_baixa_item (nf_id, sku, quantidade, quando)`: cada item é anotado **na hora**; o laço pula item já anotado.
2. Gravação insistente (até 8 tentativas em `database is locked`) e `busy_timeout` na conexão.
3. A nota só entra em `nf_baixadas` quando **todos** os itens terminaram; se um falhou, ela fica aberta e volta no próximo ciclo — só o que faltou é reprocessado.
4. Guarda de escrita com chave `nf:<id>:<sku>` (ver [`05-confiabilidade.md`](05-confiabilidade.md)).
5. Script de reparo que reconstrói do log quanto cada (nota, SKU) deduziu, compara com a nota e devolve a diferença por entrada relativa — com uma trava: SKU que teve contagem manual **depois** da dedução errada fica de fora (a contagem já colocou o saldo no físico; devolver por cima criaria peça inexistente). Resultado: 57 SKUs, 110 peças devolvidas, 0 problemas.

**Lição.** Toda escrita externa precisa de registro local **por operação**, gravado **imediatamente**. Marcar "feito" só no fim de um lote é receita para duplicar quando o lote é interrompido — e em produção o lote sempre acaba interrompido alguma vez. O código estava em produção havia duas semanas.

---

## PM2 · A reserva fantasma no Tiny (28/08)

**Sintoma.** O ERP dizia que não havia peça de SKUs que estavam na prateleira: `disponível = saldo − reservado` muito negativo. Numa variação, o Tiny mostrava reservado 29; a plataforma B2B, 2; pedidos realmente em aberto, 6 peças.

**Investigação.** Nos 288 SKUs com reserva > 0, em **124** o reservado batia exatamente com a soma dos pedidos **faturados** — só 17 batiam com "em aberto". Hipótese: a reserva de pedidos já faturados nunca era liberada. Experimentos num único pedido: marcar como *Entregue* não soltou nada (reservado 29→29); `POST /pedidos/{id}/lancar-estoque` respondeu 400 "não é possível lançar estoque de pedidos com nota fiscal gerada"; `POST /notas/{id}/lancar-estoque` → **204**, e a reserva caiu exatamente a quantidade da nota.

**Causa.** Quem solta a reserva no Tiny é o **lançamento de estoque da nota fiscal**, não a situação do pedido. E o Tiny se recusa a lançar estoque de qualquer nota que contenha um produto cadastrado como "com variações" **sem nenhuma variação** ("pai órfão") — uma peça dessas trava a nota inteira, sem mensagem. A empresa tinha 19 desses, e o tipo do produto é imutável depois da criação (a API ignora o campo; a tela bloqueia). Cadeia: nota trava → reserva nunca consumida → acumula em cada pedido faturado → ERP nega peça que existe.

**Correção.**
- Receita de reparo: lançar a nota (solta a reserva) e devolver o que o lançamento tirou (as notas antigas já tinham sido baixadas por outro caminho). 75 notas, rate limit de ~50 s entre blocos.
- Conserto definitivo: recriar os 19 pais órfãos como produtos simples mantendo o SKU (prévia → OK → aplicar, um a um), excluir os antigos via `DELETE /produtos/{id}` (a API ignora `situacao` no PUT; DELETE funciona mesmo com nota emitida) e adicionar à "faxina de cadastro" um detector de pai órfão — que na primeira rodada achou mais 5.

**A ferramenta de reparo teve quatro erros num dia:**

| # | Erro | Onde foi pego |
|---|---|---|
| 1 | Devolvia `saldo_atual + q` — somaria se o Tiny demorasse a refletir | antes de rodar |
| 2 | Devolvia por **balanço** (valor absoluto), que o Tiny recusa quando o alvo é negativo | **em produção** — 15 SKUs ficaram faltando |
| 3 | Lia itens do **pedido** em vez da **nota** (5 de 75 divergiam) | antes de causar dano |
| 4 | Sem `busy_timeout`; colisão com o serviço matou a execução **depois** de devolver e **antes** de anotar | **em produção** — 1 SKU quase devolvido em dobro |

Os dois que chegaram em produção vieram da mesma raiz: assumir o caminho feliz. O dano foi contido porque a ferramenta media antes e depois e registrava o que já tinha feito.

**Lição.** Virou regra escrita (D11): prévia obrigatória, foto antes/depois com conferência exata, registro imediato que sobrevive a falha, reexecutável sem repetir efeito. E: três heurísticas locais para prever quais notas seriam recusadas erraram todas — não vale insistir em prever o que se descobre com uma chamada.

---

## PM3 · Split-brain no leader lease (01/09)

**Sintoma.** Nenhum, ainda. Achado numa auditoria de "erros incomuns" pedida pelo dono do processo.

**Investigação.** O host renova o lease a cada 60 s (`RENOVA_MS`) e a chave expira em 180 s (`LEASE_MS`). Quando o host perde acesso ao Redis, ele tolera a falha por um tempo antes de parar de agir como host. A tolerância estava contada a partir da **primeira falha** (que acontece em t ≈ 60 s) e durava `LEASE_MS` — o host parava em t = 240 s. Mas a chave expira em t = 180 s, e outro PC pode pegá-la nesse instante. Janela de **~60 s com dois hosts** escrevendo nos ERPs, num cenário de partição parcial (host alcança o Tiny mas não o Redis).

**Correção.** Rastrear `ultimaRenovacaoOk` e parar em `LEASE_MS − RENOVA_MS` (120 s) a partir dela — o host velho para 60 s **antes** de outro poder assumir. Simulação (`sim-splitbrain.ts`): antes parava em 240 s (sobreposição de 60 s); depois em 120 s (folga de 60 s).

**Lição.** Em lease, a conta é sempre "desde a última confirmação de que ainda sou dono", nunca "desde que comecei a falhar". Sem fencing token nas APIs de destino, essa margem é a única proteção — e precisa ser provada, não presumida.

---

## PM4 · Pedidos duplicados por UUID recriado (10/08, dia do go-live)

**Sintoma.** Dois pedidos legados vinculados à mão apareceram duplicados no ERP (#98 cópia do #96, #99 cópia do #24).

**Causa.** A Teceo tem aprovação em duas etapas (rascunho → em aprovação → aprovado) e **recria o UUID do pedido** na reaprovação. O pré-vínculo local era por UUID; o ciclo viu "pedido novo" e importou de novo.

**Correção.** Antes de importar, o processamento procura SUCCESS pelo **código** do pedido (que não muda) e herda o vínculo — só reporta, nunca recria. Além disso: pedido criado no Tiny é SUCCESS local **sempre**; se o reporte à Teceo falhar, os ciclos seguintes só re-reportam.

**Lição.** Identificador "único" da outra plataforma pode não ser estável. Casar por algo que o negócio reconhece (código do pedido) e tratar o reporte como etapa separada da criação.

---

## PM5 · O padrão do reservado — `disponivel` em vez de `saldo` (14/08)

**Sintoma.** Grade de um modelo com diferença Teceo × Tiny **exatamente** igual ao `reservado` do Tiny, linha por linha. Um incidente de agosto: peça reservada sumia da loja como se vendida; 242 SKUs sobrescritos numa rodada.

**Causa.** O sync mandava `estoque.disponivel` (saldo − reservado) como saldo total. A Teceo faz a própria reserva; recebia um número já descontado e descontava de novo.

**Correção.** Todos os pontos passaram a ler `estoque.saldo`. Ciclo religado com prova de vida no primeiro tick (saldos enviados bateram com os alinhados à mão).

**Lição.** Quando dois sistemas fazem reserva, a interface entre eles carrega o físico. Uma diferença que bate exatamente com um campo é uma pista, não uma coincidência.

---

## PM6 · Saldo pulando 16 ↔ 0 — SKU duplicado + failover (01/09)

**Sintoma.** Um SKU com histórico "impossível": 16 → 0 → 16 → 0 em dias diferentes, vindo de PCs diferentes. Contagem física: 8. Nem o 16 nem o 0 estavam certos. O dono procurava o segundo produto na tela do ERP e não achava.

**Causa.** Dois produtos **ativos** no Tiny com o mesmo SKU: um legítimo (variação numa grade, saldo 16) e um fantasma (produto simples avulso, criado um mês depois com nome/SKU/preço idênticos, saldo −6). O `Map sku→id` do catálogo ficava com um ou com outro conforme a ordem, e como o papel de host passou entre PCs, cada host escolhia um. Na tela do ERP as variações ficam agrupadas dentro da grade — o fantasma se confundia com a variação já exibida.

**Correção.** A sincronia detecta SKU com produto duplicado, **congela** no último valor bom e avisa no log. O fantasma foi renomeado para `<sku>-DUP-INATIVO` via PUT (a API ignora `situacao`; inativar só pela tela), o catálogo local alinhado, e o saldo do legítimo acertado para 8 nos dois sistemas. Um segundo SKU no mesmo estado foi localizado pela varredura.

**Lição.** Unicidade de SKU não é garantida pelo ERP. E um bug que "só aparece com failover" precisa que a correção chegue a **todos** os PCs — qualquer um pode ser host.

---

## PM7 · Duas brechas de segurança em auditoria própria (25/08 e 28/08)

**Brecha 1 — a conta fantasma.** A migração de usuários do banco antigo para o banco de acesso decidia "já migrei esta pessoa?" verificando se existia alguém com aquele login. Rodava a cada requisição. Quando o dono trocou o próprio login de um apelido curto para o e-mail, o login antigo sumiu — e a migração **recriou a conta na requisição seguinte, com o hash de senha antigo e poder de dono**. A senha velha voltava a valer sozinha. Correção: tabela `migracao_feita` (origem + login de origem), que não muda com renomeação. Conferido rodando a migração 5 vezes.

**Brecha 2 — a rota administrativa aberta.** `POST /api/acesso/usuario` (criar usuário, trocar senha, dar permissão) checava `if (euUsuario && !euUsuario.dono) → 403`. **Sem sessão nenhuma**, `euUsuario` era nulo, o `if` dava falso e a requisição passava. A rota não estava atrás de nenhum portão (o portão da empresa principal estava desligado por decisão operacional). Qualquer máquina da rede interna podia criar um usuário dono. Correção: exige sessão **e** dono; sem sessão, 401. No mesmo dia: um segundo sistema de login antigo, ainda vivo com sessão válida até dezembro, foi desativado.

**Lição.** "Existe alguém com esse nome?" não é o mesmo que "isso já foi feito". E checagem de permissão precisa ser positiva ("tem sessão e é dono"), nunca negativa ("não é o caso de negar").

**Brecha 3 (bônus, 01/09) — dois portões de login.** Havia dois pontos de verificação de login no servidor; a lista de rotas de máquina liberadas (update, réplica, agente) foi corrigida em um deles e não no outro. Os standbys pediam `/update/manifest` e recebiam a tela de login em HTML; o `JSON.parse` falhava e o log dizia "host não respondeu". Correção: uma única função `rotaLiberadaSemLogin()` usada por todos os portões.

---

## PM8 · Banco corrompido por encerramento forçado (01/09)

**Sintoma.** `integrity_check` do banco principal: "database disk image is malformed". Doze cópias estáticas falharam — corrupção real, não concorrência.

**Causa.** O banco principal ainda estava em `journal_mode=delete` (os outros três já eram WAL). Durante o diagnóstico dos standbys, o processo foi encerrado com `Stop-Process -Force` no meio de escritas.

**Correção.** Os 14 backups diários estavam íntegros; o mais recente limpo foi preservado com nome próprio. Migração do banco principal para WAL (`synchronous=NORMAL`, `busy_timeout` 15 s). Backup e réplica **não precisaram mudar** — ambos usam `VACUUM INTO`, que lê o WAL (validado com um marcador escrito no WAL e conferido na cópia). A adoção de réplica passou a apagar `-wal`/`-shm` órfãos antes de trocar o arquivo. Bônus: em WAL dá para ler o banco vivo de fora sem o falso "malformed" que o modo delete causava.

**Lição.** Um banco que só o serviço escreve, mas que o serviço pode largar no meio, precisa de WAL. E cada ferramenta que copia banco precisa ser conferida contra o modo de journal.

---

## PM9 · Estoque fantasma que nasceu no cadastro (01/09)

**Sintoma.** Clientes compraram peças que não existiam. Suspeita natural: a integração "inventa" estoque.

**Investigação.** Sincronia equilibrada (217 subidas × 336 descidas — bug de "somar em vez de substituir" só subiria). Nenhum caminho de escrita Teceo → Tiny tocou as peças reclamadas. No log, cada peça fantasma apareceu pela **primeira vez** já com saldo alto (`null → 36..40`), todas na mesma tarde, num lote de 128 SKUs novos, 27 deles com 20–99 unidades e **nenhuma** nota de entrada. Placeholder de cadastro no ERP, nunca recebido de verdade.

**Correção.** "Guarda de produto novo": SKU visto pela primeira vez pela sincronia com saldo ≥ 15 é registrado como retido e publicado na plataforma B2B com `availability=false` (número visível, não vendável) até um clique de liberação no painel. Produto já conhecido (restoque) não é afetado; força manual não retém. 12 testes de lógica.

**Lição.** "A automação não inventa estoque" precisa ser provado com dados, não afirmado. E o sistema de integração é o único lugar que enxerga "este SKU nasceu com saldo e sem entrada" — então é dele a responsabilidade de segurar.

---

## PM10 · Cota de 60 req/min saturada pelo próprio serviço (18/08)

**Sintoma.** Busca por descrição no painel levava de 30 s a 3 min; a grade de um produto, minutos. 356 esperas de 429 numa hora.

**Causa.** A varredura completa de estoque (1.353 SKUs × 2 chamadas) rodava no boot a cada reinício e saturava a cota; a busca do painel esperava o Tiny mesmo tendo a resposta no catálogo local; e o filtro `nome=` do Tiny casa só o **prefixo** da descrição ("smock" no meio do nome devolvia zero).

**Correção.** Catálogo local primeiro (todas as palavras, sem acento, qualquer ordem — 0,02 s); prioridade interativa via `AsyncLocalStorage`; limitador local de janela deslizante; varredura nunca no boot e cedendo a vez. Grade de um modelo tamanho a tamanho: 3,1 s.

---

## PM11 · Cache de vendas parado por 9 dias (20/08)

**Sintoma.** Relatórios sem dados novos. Ninguém percebeu por nove dias.

**Causa.** `vendas_sync_status` ficou com `rodando: true` de uma execução que morreu num restart. O próximo ciclo via "já está rodando" e desistia.

**Correção.** Critério de morte: início anterior ao boot do processo ou há mais de 1 h → marca órfã, com WARN. A mesma proteção foi aplicada no cache de notas fiscais. Depois, uma página de saúde de todos os ciclos com alerta ativo (notificação do Windows) — o roadmap registra: "hoje descobrimos por acaso um cache parado havia 9 dias".

---

## PM12 · Horário no futuro nos logs de auditoria (24/08)

**Sintoma.** Tela de contagem mostrando "18:01" quando eram 15:01.

**Causa.** Registros gravados em ISO/UTC (correto) e a tela fatiando a string (`slice(11,16)`) — mostrava UTC como se fosse local. Usar `toLocaleString` sem fuso funcionaria por acaso (a máquina está em −03) e quebraria em qualquer PC com outro fuso. E as datas do Bling vêm em horário local **sem** indicador — converter essas tiraria 3 h indevidamente.

**Correção.** Um único módulo decide data/hora: ISO com Z/offset → converte para `America/Sao_Paulo`; formato do ERP → só reformata. As telas mostram "agora são HH:MM" no rodapé — se o fuso quebrar de novo, salta aos olhos.

**Lição.** Horário errado em log de auditoria é pior do que não ter log.

---

## PM13 · O lançamento tardio de nota antiga come o estoque (08/09)

**Sintoma.** Uma grade inteira negativa no ERP (seis tamanhos somando −33, com 45 peças reservadas). Varredura: **113 SKUs negativos, −384 peças**.

**Investigação.** Para o modelo que abriu a porta: saldo logo após a contagem física de 12/08 era 7; hoje, −33; logo saíram 40 peças. Deduzidas pela integração: **0**. Peças em notas posteriores à contagem: 2. Peças em notas **anteriores** à contagem (que a contagem já tinha visto na prateleira vazia): 74. Nos 113 SKUs, **100 não tinham nenhuma dedução nossa** — o ERP deduziu sozinho.

**Causa.** `POST /notas/{id}/lancar-estoque` — a única alavanca para soltar reserva presa (PM2) — **também deduz o saldo**, no dia em que é chamado. Lançar hoje uma nota de junho deduz de um saldo que a contagem física já tinha corrigido: a mesma venda é contada duas vezes. O lote de 28/08 previa devolver o que a nota tirou, mas só devolvia onde a queda batia exatamente; onde não batia, marcava "conferir" e não devolvia.

**Correção.** Lote `soltar-reservas-v2` com **invariante**: foto ao vivo de cada SKU → lança → relê → devolve por entrada exatamente a queda → relê → **exige saldo final = saldo inicial**; qualquer divergência para a nota e o lote. Livro-razão por nota e por item, retomável. Provado numa nota de 12 SKUs / 32 peças, inclusive os negativos. Ao longo de 14–18/09: **91 notas tratadas, 739 reservas soltas com saldo idêntico**; 25 notas recusadas por "produto pai" (a nota é imutável e guarda o id do produto antigo) resolvidas cancelando o pedido — 12 pedidos, 348 reservas. Por fim, os 114 SKUs negativos (−375) zerados por balanço, por decisão do dono, com a ressalva registrada de que o físico decide na próxima contagem.

**Bug no meio do caminho.** O lote lia o token OAuth uma vez no início; o access token dura ~4 h, então a partir de certa nota tudo voltou 401 e 14 notas foram anotadas "recusada" sem ser. Correção: token pedido a cada tentativa; 401 vira "adiada", nunca "recusada"; anotações falsas apagadas.

**Lição.** A decisão de 02/09 de **não** aplicar a baixa em massa das notas antigas (porque a contagem física já tinha absorvido essas vendas) ganhou prova empírica pelo caminho inverso. E: uma ferramenta que só devolve "onde bate" deixa buraco onde não bate — a invariante precisa ser "saldo final igual ao inicial", verificada, ou o lote para.

---

## PM14 · Os 95 pais órfãos escondidos por dois campos com o mesmo nome (08/09)

**Sintoma.** Depois de migrar 19 "pais órfãos" (PM2), uma NF nova ainda travou.

**Investigação.** Primeiro achado: a nota travada apontava para produtos **excluídos** — o pedido nasceu antes da migração e guarda o **id** do produto, não o SKU; a nota herda o id. Só 2 pedidos antigos ainda podiam travar. Mas ao varrer o catálogo apareceram órfãos que nunca tinham sido vistos.

**Causa.** A API devolve dois campos que eram tratados como um: `tipo` (`V` = "com variações", o que faz o ERP recusar a nota; `S` = simples) e `tipoVariacao` (`P` = pai da grade, `V` = variação filha, `N` = nenhum). O balde `tipo=V, tipoVariacao=N` — "com variações" mas nem pai nem filho — tinha **95 produtos**; todas as varreduras anteriores olhavam só os 439 pais legítimos. Para piorar, o catálogo local usava as mesmas letras com outro sentido (`V` = variação, `P` = pai), e a primeira varredura do dia checou 2.097 variações à toa. O mesmo engano produziu, no mesmo dia, uma planilha de estoque com 553 linhas todas zeradas (só os pais).

**Correção.** Enumeração barata (`GET /produtos?limit=100&offset=N` traz os dois campos: 27 chamadas para o catálogo inteiro). Fila por risco (0 com pedido aberto, 5 com saldo, 88 zerados). `migrar-lote.ts` com livro-razão retomável, re-verificação na hora, desfaz o rename se a criação falhar e para o lote se o saldo final não bater. **95 de 95 migrados, zero erro**, em 5 lotes de 10–20 min.

**Bug descoberto de carona.** O script de migração de 18/08 chamava a escrita de estoque **sem a chave** que a guarda passou a exigir em 31/08 — quebrou no meio de um SKU (renomeou o velho, criou o novo, não moveu o saldo). Fechado à mão e corrigido. Quando a assinatura de uma função muda, os scripts que ninguém rodou desde então quebram na hora errada.

**Lição.** Dois campos com nomes parecidos e domínios sobrepostos pedem uma tabela de verdade explícita no código e no doc. E "varredura limpa" só vale para o balde que foi varrido.

---

## PM15 · Estoque parado três dias: a varredura que nunca terminava (11–14/09)

**Sintoma.** "O estoque dos pedidos parou desde 3 dias atrás." Último passe completo da varredura Tiny → Teceo: 08/09.

**Investigação.** Primeira hipótese (falta de retomada + cadeado frágil) — parcialmente errada como causa principal, e registrado como tal. Medição de 10/09, dia em que o processo **não** reiniciou nenhuma vez: 20.382 linhas "rate limit; aguardando 58 s" (98% do log) e **0** linhas de saldo enviado. Não caía: ficava pendurado.

**Causas (três, somadas).**
1. **Fome artificial.** Qualquer rota `/api/` marcava "painel em uso", com uma lista de exceções que só crescia. A tela da fila de impressão consultava uma rota a cada poucos segundos — rota que nem toca no ERP — e a varredura, que cedia a vez a cada SKU, cedia para sempre: **0/1355 em 70 min**.
2. **Efeito manada no 429, sem teto.** Todas as chamadas pendentes liam o mesmo `x-ratelimit-reset`, dormiam o mesmo tanto, acordavam juntas e tomavam 429 juntas; a tentativa era infinita, então uma chamada podia pendurar o ciclo o dia inteiro.
3. **Cedência a cada SKU** em horário comercial: ~1,1 SKU/min.

**Correções.** A marcação de uso interativo saiu do servidor HTTP e foi para dentro do cliente do ERP — quem marca é a chamada que **realmente gasta cota**. Um 429 bloqueia todas as chamadas até a cota virar, com espera sorteada (±25%) e teto de 6 tentativas. Cedência de 10 em 10 SKUs, máximo 30 s. Checkpoint de varredura (`espelho_varredura`) com retomada — em 14/09, três reinícios no meio do passe custaram zero SKUs. Renovação do lease tentada a cada 15 s (dois timeouts seguidos estouravam a margem anti-split-brain e o host se derrubava); a conta da margem não mudou. Caixa-preta para `uncaughtException`.

**Segundo erro, meu (14/09).** O teto adaptativo de cota era uma catraca de mão única: caía 25% por recusa e só subia após 60 s sem nenhuma — como as recusas vinham a cada ~30 s, descia até o piso e ficava. E a premissa estava errada: as recusas não eram nossas — vinham dos **outros PCs em código antigo** (com o 429 infinito), hipótese do dono, confirmada quando eles ligaram e o ritmo despencou de 24 para 2,8 SKU/min. Corrigido para punição suave, recuperação por tempo; 25 SKU/min.

**Por que os standbys não atualizavam.** A "versão" incluía o `.env`, que difere por máquina de propósito (`PEERS`) — então nunca batia e a tela de computadores dizia "todos diferentes" sem significar nada (e eu concluí "nenhum tem as correções" a partir desse sinal: conclusão precipitada, retirada). O `.exe` do agente em uso não pode ser sobrescrito. Backups `.bak` entravam no manifesto. E `aplicarAtualizacao` devolvia −1 em silêncio. Correções: versão = hash de tudo menos `.env` e o `.exe`; extensões de backup ignoradas; log do −1; rota de manifesto só leitura + script de diff host × standby.

**Lição.** Medir 20 min antes de prometer hora. E o único sinal confiável de "mesmo código" é o hash do código — não do ambiente.

---

## PM16 · Login parou: o host do dia tinha o banco de acesso vazio (15/09 → 18/09)

**Sintoma.** "Minha senha no painel não está mais funcionando." Cadastro intacto; nenhuma alteração no log; a trava de 5 erros dura 1 min.

**Causa.** Às 09:29 um PC que ligou primeiro pegou o lease e virou host **sem adotar a réplica** — e o banco de acesso dele tinha **zero usuários** (criado do zero por `CREATE TABLE IF NOT EXISTS`). Quem abria o painel servido por ele não encontrava conta nenhuma. Provado sem palpite: a réplica que este PC baixava daquele host, aberta só para leitura, tinha 0 usuários contra 5 no local (comparação por impressão digital dos hashes, sem imprimir senha); as tentativas de login do dia estavam no banco do host, nenhuma no local.

**O risco pior.** Se qualquer PC assumisse com o marcador de adoção, substituiria os bancos locais pelas réplicas — e o vazio se espalharia pela frota.

**Correção em duas etapas.**
1. (15/09) Cópia limpa extra do banco bom; blindagem na adoção: o banco de acesso só é adotado se a réplica **não fizer minguar** o número de usuários. E o pedido do dono de "tirar a senha do painel por enquanto" foi recusado: não resolveria (o código anda do host para os standbys, e o host era o PC vazio), e o painel tem financeiro e escrita de estoque. O caminho oferecido — entrar por `127.0.0.1` e derrubar o host vazio — resolveu em minutos.
2. (18/09) Ao implementar a sincronização, descoberta grave: **a blindagem de 15/09 tinha sumido do código**. Foi escrita num standby; no ciclo seguinte ele puxou o código do host (o PC vazio, com a versão antiga) e a correção foi sobrescrita. Reposta. Depois: sincronização do cadastro **linha a linha** no banco vivo dos standbys a cada 2 min (`ATTACH` + upsert numa transação — nunca troca de arquivo com conexão WAL aberta, que já corrompeu banco em 01/09), espelhando usuários, permissões, senha mestre e rótulos; sessões e log ficam de fora de propósito; `ultimo_acesso` só anda para frente; nunca apaga usuário que só exista no standby. Duas políticas distintas e justificadas: adoção (troca de arquivo) recusa réplica menor que a local; sincronização (linha a linha) aceita qualquer réplica legível com gente dentro — exigir "≥ local" faria com que apagar uma pessoa no host parasse a sincronização para sempre, em silêncio. Quatro testes em banco descartável; o quarto pegou um defeito real (a conta de diferença olhava só um sentido, então uma permissão removida não disparava a sincronização).

**Lição.** Correção feita num standby vive só até o próximo ciclo de atualização — o código anda do host para os standbys, nunca ao contrário. Corrigir sempre no host. E: um serviço que aceita virar host com cadastro vazio precisa falhar barulhento (pendência registrada).

---

## PM17 · Cancelar pedido "devolveu" estoque — era a nossa própria regra, e a compensação estava errada (18/09)

**Sintoma.** Ao cancelar o 8º de 12 pedidos com nota "produto pai" (PM13), 11 dos 24 SKUs subiram exatamente +1. Nos 11 cancelamentos anteriores, cancelar nunca tinha mexido em saldo. A trava de conferência item a item parou o lote.

**O que foi feito primeiro (e estava errado).** Diagnóstico: "o ERP devolveu peça; saldo ficou 1 acima do físico". Com autorização do dono baseada nesse diagnóstico, foi lançada **saída** de 1 em cada um — desfazendo apenas o delta que o cancelamento causou, nunca puxando para um número absoluto (dois SKUs tinham recebido outra entrada de outra pessoa nesse meio tempo; forçar o valor antigo apagaria o trabalho dela). Isso, ao menos, estava certo como método.

**Causa real.** Quem devolveu foi **o nosso próprio sistema**, cumprindo a regra da cesta de 20/08: a contagem física conta só a cesta (peças soltas), então quando um pedido vinculado é cancelado, as peças voltam à cesta e o sistema devolve ao saldo — **só** para SKUs que tiveram contagem manual depois da integração do pedido. A prova: dos 11 que subiram, 11 tinham contagem posterior; dos 13 que não mexeram, 12 não tinham. O livro `estoque_op` tinha, para cada um dos 11, uma operação `cesta:<pedido>:<sku>` de +1 daquele minuto. Hipóteses descartadas no caminho estão registradas para não refazer.

**Correção.** A compensação apagou uma devolução correta: 6 SKUs ficaram com 1 peça a menos (pendente de decisão do dono; os outros 5 estavam negativos e foram zerados depois). No código: o cancelamento consulta `cesta_devolucao` antes de compensar — subida registrada lá **fica**; só compensa subida de origem desconhecida. Sem isso, todo cancelamento futuro apagaria peça real da prateleira.

**Lição.** A trava "saldo não pode mexer ao cancelar" tratava qualquer movimento como erro sem perguntar **quem** o fez. Num sistema com automações próprias, conferir o resultado não basta — é preciso conferir a **autoria**. O livro-razão por automação, com chave por origem, foi o que respondeu em dez minutos. E: dano contido é dano — o erro está escrito aqui com o nome que tem.

---

## PM18 · Uma cópia velha do `server.ts` apagou telas inteiras — e nada acusou (18/09)

**Sintoma.** Todas as folhas de impressão respondendo 404. Ao olhar: `server.ts` era uma cópia anterior a 14/09 com os ajustes do dia aplicados por cima. Fora do ar sem ninguém perceber: relatórios das outras duas empresas (uma servia a tela antiga, a outra nem tinha), 47 botões de impressão, o período personalizado da API (a tela sempre mostrava 90 dias, ignorando o filtro), o ajuste de estoque de uma empresa e, no `index.ts`, o espelho automático da terceira empresa — que já tinha parado uma vez em 09/09 pelo mesmo motivo.

**Causa.** Edição a partir de uma cópia *staged* dias antes, durante a troca dos logos. No mesmo dia, o mesmo acidente já tinha apagado o gráfico de faturamento por dia feito de manhã.

**Correção.** `server.ts` reconstruído juntando a versão de 14/09 com o trabalho do dia (10 blocos reaplicados um a um, typecheck limpo). E o **verificador de rotas**: `rotasEsperadas.ts`, uma lista escrita à mão das 43 rotas que o painel precisa ter (a lista é a expectativa; se a rota sumir, a lista cobra), cada uma podendo exigir status, trechos no corpo, tamanho mínimo, tipo de conteúdo e uma conferência em código (a da API de relatórios pede uma janela de 3 dias e exige 3 dias de volta com as datas certas). Roda como mais um ciclo da aba Saúde — 90 s após o boot e a cada 6 h — com sessão de dono temporária, e falha para a mesma notificação nativa que qualquer ciclo parado. 43/43 em 7,4 s; teste do alarme apontando para porta errada acusou as 43.

**Lição.** Arquivo grande só se edita a partir de uma cópia re-obtida na hora. Quando um recurso "some", conferir primeiro se o arquivo ainda tem a fiação, antes de procurar bug na lógica. E a saúde de um sistema não é só "os ciclos rodam" — é "as telas existem".

---

## PM19 · A folga do "destravar pedido" que podia ficar para sempre (18/09)

**Sintoma.** Pergunta do dono: "o +1 do destravar fica depois do pedido? Acrescer 1 em todos os desbloqueios me parece que bagunça o estoque."

**Contexto.** Destravar um pedido escreve na plataforma B2B `total = reservado + exigido + folga(1)`, porque ela só deixa aprovar quando `disponível ≥ quantidade` e o próprio pedido reserva o que acabou de ser lançado. Só mexe na plataforma; o ERP (a verdade) não é tocado.

**Auditoria.** 77 desbloqueios, 9 pedidos, 57 SKUs, +172 peças somadas na plataforma (como a função *define* o total em vez de somar, o pulo pode ser grande: um SKU foi de 8 para 17). Conferência **ao vivo** em 17 SKUs: todos iguais ao ERP, zero resíduo — o espelhamento sobrescreve. (Uma passada anterior apontou 6 "inflados" comparando com o espelho local, que estava defasado; ao vivo estavam certos. Não usar espelho local como régua enquanto não realinhar.)

**A brecha real.** O espelhamento compara o saldo do ERP com **o que ele próprio gravou por último**, não com o que a plataforma tem. Escrita por outro caminho é invisível. Se o saldo de um SKU no ERP nunca mais mudar depois de um desbloqueio, o número inflado fica na plataforma indefinidamente, em silêncio. Não mordeu nos 57 porque todos tiveram movimento depois — risco latente.

**Correção.** Sem tocar no laço quente do espelhamento (aquele atalho é o que faz 1.355 SKUs caberem na cota). Livro-razão `destrava_folga` de todo desbloqueio que infla; ciclo próprio a cada 20 min no host: carência de 15 min; folga fica de pé enquanto o pedido estiver em aprovação; saiu de aprovação → devolve ao saldo real do ERP; prazo máximo de 24 h mesmo com pedido travado; **nunca levanta estoque**; respeita trava manual e produto retido; grava também em `stock_state` (exatamente o atalho que deixava o inflado invisível); teto de 40 SKUs por rodada. Varredura do passado: 55 de 56 alinhados; o único divergente era atraso do espelho, não resíduo.

**Lição.** "Não fica" só vale enquanto o acaso ajuda. A garantia de que nunca vai ficar precisa de livro-razão e de um passe que não dependa de o ERP se mexer.

---

## PM20 · Três armadilhas de "só vem no detalhe" e uma janela cega (14/09)

Quatro achados menores do mesmo dia, todos da mesma família: a listagem de uma API não traz o que o detalhe traz, e o cache apaga sem perceber.

- **Notas canceladas invisíveis.** O cache de notas, depois de completar o histórico, passava a listar só os últimos 45 dias — mas o `dataInicial` do ERP filtra por **data de emissão**, não de alteração. Uma nota emitida em janeiro e cancelada em agosto ficaria "autorizada" para sempre no cache. Não tinha mordido só porque a única cancelada foi cancelada dentro da própria janela. Achado porque o dono desconfiou do número ("tenho quase certeza que tem mais de 1 cancelada") — e a auditoria foi feita por **três portas diferentes** de propósito (filtro explícito de situação, sem filtro de data desde 2020, tipo de nota que a varredura ignora), porque bater pela mesma porta da varredura herdaria a mesma cegueira. O número estava certo (1 nota; o que estava na cabeça eram 43 *pedidos* cancelados); o buraco era real. Correção: listar sempre o histórico inteiro (7 chamadas para 648 notas) e baixar detalhe só de quem mudou.
- **Desconto na nota.** O faturamento de um dia divergia do ERP de varejo. A listagem de notas não traz `valorNota` — só o detalhe; e o upsert da listagem fazia `valor = excluded.valor` (nulo) a cada ciclo, apagando o que o detalhe tinha trazido. Duas notas com desconto dado na nota (não no item) explicavam a diferença. Correção: coluna própria + `COALESCE(excluded.valor, nf.valor)`; backfill único. A mesma armadilha tinha aparecido de manhã com o id da nota no pedido.
- **PUT que não mudava nada era recusado.** No ERP de varejo, `PUT /pedidos/vendas/{id}` recalcula o total como Σ(quantidade × valor − desconto do item em reais) e recusa se as parcelas não baterem — mas o `total` que o GET devolve ignora esse desconto. Um pedido chegava com parcela ≠ total dos itens e qualquer PUT, mesmo idêntico, falhava. Correção: quando a edição muda o total, as parcelas acompanham na proporção; quando o pedido já chega inconsistente, a tela bloqueia e explica (mexer na parcela mudaria quanto o cliente deve — não é decisão nossa).
- **200 com corpo vazio.** Gerar o HTML depois do `writeHead(200)` transforma um erro em resposta vazia com status de sucesso. Foi assim que a primeira versão de uma tela "funcionou" sem mostrar nada (a causa raiz era uma coluna que ainda não existia naquele banco). HTML agora é montado antes do cabeçalho.
