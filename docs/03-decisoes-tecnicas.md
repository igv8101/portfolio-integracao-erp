# 3. Decisões técnicas

Cada decisão no formato **contexto → opções → escolha → consequências**. Datas são de 2026.

---

### D1. Node 26 com TypeScript nativo e zero dependências de runtime

**Contexto.** O sistema roda em PCs de loja, instalado por `.cmd` a partir de um zip, atualizado pela rede arquivo a arquivo. Cada dependência é um arquivo a mais para distribuir, uma versão a mais para quebrar e uma superfície a mais para o antivírus.

**Opções.** (a) Stack tradicional com build (`tsc`/`esbuild`) e `node_modules`; (b) Node 26 executando `.ts` diretamente, `node:sqlite` embutido, `fetch` nativo, HTTP padrão.

**Escolha.** (b). O único "vendor" é o Chart.js servido localmente para os gráficos.

**Consequências.** Instalação = descompactar + `node --env-file=.env src/index.ts`. Auto-update é diff de arquivos-fonte. O custo é abrir mão de bibliotecas de conveniência (ORM, framework HTTP, validação) — o código cresce em utilitários próprios, mas todos são pequenos e legíveis. `tsc --noEmit` roda como *typecheck* em CI local; o Node não checa tipos ao executar.

---

### D2. Polling em vez de webhooks

**Contexto.** A Teceo suporta webhook, mas exige endereço público (a loja não tem IP fixo nem servidor). O Tiny oferece webhooks, porém a configuração é global da conta e depende do plano contratado.

**Escolha.** Polling com intervalos por ciclo (60 s para pedidos, 30 min para edições, 6 h para varredura de estoque), com `changeStartDate` e janelas de 60 dias para não varrer o histórico.

**Consequências.** Latência de até 60 s para pedido novo (aceitável — o gargalo humano é maior). Cota de API consumida mesmo sem novidade; mitigado com deltas, caches e o limitador local. Webhooks do Tiny e um túnel (cloudflared) ficam no roadmap.

---

### D3. Tiny é a fonte da verdade do estoque

**Contexto.** Dois sistemas com saldo por SKU. Alguém precisa mandar.

**Opções.** Teceo (onde o lojista compra) ou Tiny (onde a nota é emitida e a contagem física é lançada).

**Escolha.** Tiny, com sincronização Tiny → Teceo por delta e força manual por SKU no painel. Decisão do dono do processo em 03/08, com ressalvas registradas.

**Consequências.** A Teceo sempre pode ser reconstruída a partir do Tiny. Toda contagem grava nos dois. O sync manda o **saldo físico** (não o "disponível" = saldo − reservado) — descoberto na pele: mandar `disponivel` fazia peça reservada sumir da loja como se vendida (ver post-mortem 5). A Teceo faz a própria reserva.

---

### D4. Um arquivo SQLite por empresa, mais um só de identidade

**Contexto.** Regra do negócio: nada de uma empresa se mistura com a outra. E "quem é a pessoa" precisa ser único para existir uma tela que responda "quem tem acesso a quê".

**Opções.** (a) Um banco com coluna `empresa_id`; (b) um banco por empresa com usuário em cada um; (c) um banco por empresa + um banco de acesso.

**Escolha.** (c). O SQLite não faz JOIN entre arquivos: a separação é física, verificada por script (zero tabelas de negócio em comum). O banco de acesso guarda só pessoas, hashes, sessões e permissões.

**Consequências.** Nenhuma consulta cruza empresas por acidente. Backup e réplica tratam cada arquivo. O preço: código que precisa ser "fábrica por empresa" (cliente de API, espelho, contagem) em vez de singleton — o que acabou sendo uma boa forma de compartilhar sem copiar.

---

### D5. Leader lease em Redis (Upstash, REST) em vez de descoberta na rede local

**Contexto.** Ver [`02-arquitetura.md`](02-arquitetura.md). Rede local deixava dois hosts nascerem quando um PC saía da rede.

**Opções.** (a) Trava por rede local mais rígida; (b) algo na nuvem que só precise de "estar online".

**Escolha.** (b), com `SET NX PX` (atômico, sem empate), TTL 3 min, renovação 60 s, fail-closed sem Redis. Plano gratuito cobre ~30× o uso.

**Consequências.** Failover em até 5 min. Dependência nova (Redis) — se cair, o sistema fica sem host até voltar, o que é o comportamento desejado. Descobriu-se depois que a tolerância à perda do Redis estava contada do ponto errado (post-mortem 3).

---

### D6. Réplica do banco por `VACUUM INTO`, não por cópia do arquivo

**Contexto.** Quando um standby assume, precisa dos livros de idempotência do host anterior, senão deduz nota de novo. Copiar o `.db` cru de um banco em WAL dá imagem incompleta.

**Escolha.** `VACUUM INTO` gera cópia consistente lendo o WAL. O mesmo mecanismo serve backup diário e réplica a cada 2 min. Validado escrevendo um marcador no WAL e conferindo que aparece na cópia com `integrity_check = ok`.

**Consequências.** Migrar o banco principal para WAL (depois da corrupção — post-mortem 8) não exigiu mudar backup nem réplica.

---

### D7. Toda escrita de estoque passa por uma guarda com chave obrigatória na assinatura

**Contexto.** Post-mortem 1 (baixa em dobro). Exigência do dono: "em código, para que nunca mais dispare, mesmo por meios semelhantes".

**Opções.** (a) Marcar "feito" em tabelas por caso de uso; (b) uma função única de escrita, que não compila sem uma chave de idempotência.

**Escolha.** (b). `tiny.lancarEstoque(idProduto, payload, chave)` — a chave identifica a operação de negócio (`nf:<id>:<sku>`, `cesta:<pedido>:<sku>`, …). A guarda lê o saldo antes, grava a intenção como pendente, escreve, lê o saldo depois, confirma. Chave já confirmada não chama a API.

**Consequências.** Duas leituras extras por escrita (aceito). Nenhum script, tela ou sync escreve estoque por fora — o compilador impede. A tabela `estoque_op` passou a ser também o livro para a retomada automática.

---

### D8. Dedução relativa em vez de balanço absoluto

**Contexto.** A baixa por nota lia o saldo, subtraía e gravava o valor absoluto (balanço). Uma venda entrando no Tiny entre a leitura e a gravação era apagada (*lost update*). Achado pelo harness de simulação (cenário S3).

**Escolha.** Enviar **saída relativa** (`tipo: 'S'`, quantidade) — o Tiny subtrai do saldo que ele tem naquele instante. A Teceo recebe o saldo lido depois da saída.

**Consequências.** Balanço continua sendo usado quando o *objetivo* é fixar um número (contagem física). Devolução de saldo em reparos usa **entrada** relativa — porque o balanço é recusado quando o alvo é negativo (post-mortem 2, erro 2).

---

### D9. Prioridade interativa via `AsyncLocalStorage`

**Contexto.** Uma cota de 60 req/min para o ciclo de fundo e para a pessoa no balcão.

**Escolha.** Tudo que roda dentro de uma requisição HTTP é marcado como interativo via `AsyncLocalStorage`; chamadas de fundo cedem a vez por 45 s após o último uso do painel; um limitador local de janela deslizante evita chegar ao 429. Catálogo local responde primeiro.

**Consequências.** Busca de 30 s–3 min para 0,02 s. O ciclo de fundo fica mais lento quando alguém usa o painel — correto por definição.

---

### D10. Relatórios: "peça vendida" é peça que saiu em nota fiscal

**Contexto.** Pedido aprovado, separado ou em aberto não tira peça do estoque. E o histórico útil está nas notas — inclusive as importadas do ERP anterior, que mantiveram o mesmo SKU de propósito.

**Escolha.** Cache local de notas fiscais (retomável, 250 por rodada) desde o início do ano anterior; todos os relatórios de giro, grade, cliente e perdas leem notas, nunca pedidos. Na empresa de varejo, a decisão foi a oposta (pedidos como fonte, NF cobre pouco) — e está escrita na tela.

**Consequências.** Descoberta ao implementar: `idNotaFiscal` no pedido vem preenchido até em pedido cancelado; a prova de faturamento é `dataFaturamento` ou a nota em situação autorizada. Ver [`08-comportamentos-nao-documentados-das-apis.md`](08-comportamentos-nao-documentados-das-apis.md).

---

### D11. Ferramentas de reparo com prévia obrigatória e registro por operação

**Contexto.** Post-mortem 2: quatro erros num dia numa ferramenta que escrevia estoque em produção.

**Escolha.** Regras fixas para todo script que escreve fora: (1) modo prévia é o padrão, `--aplicar` é explícito; (2) fotografa antes e depois e só confirma quando a variação bate exatamente; (3) registro local por operação, gravado **imediatamente**, com `busy_timeout` e fallback em arquivo se o banco estiver travado; (4) reexecutável sem repetir efeito.

**Consequências.** Os reparos seguintes (110 peças em 57 SKUs) rodaram com zero problemas.

---

### D12. Senha mestre separada do login, redefinível só por e-mail fixo no código

**Contexto.** O agente permite rodar PowerShell em qualquer PC da empresa. Estar logado como dono não deveria bastar — um programador futuro com acesso ao painel teria esse poder.

**Escolha.** Hash separado no banco de acesso, semeado da senha do dono na primeira vez e independente dali em diante. Sem botão de troca. Redefinição só por token enviado a um e-mail pessoal fixo no código (não no banco: quem mexe no banco não redireciona). Verificação autoritativa no host — os standbys perguntam.

**Consequências.** A primeira versão verificava no banco local de cada standby, que é vazio até o failover: manutenção remota impossível de destravar. Corrigido no dia seguinte (a manutenção remota "trancada demais" é um erro de desenho tão real quanto "aberta demais").

---

### D13. Documentação e comentários datados como ativo

**Contexto.** Uma pessoa, muitas decisões por dia, muitas delas reversíveis só numa direção.

**Escolha.** Cada dia de trabalho gera um documento com o que foi feito, por quê, o que foi testado e o que ficou pendente. Comentários no código registram decisão e data. Mensagem de commit explica o porquê.

**Consequências.** Este repositório foi escrito a partir de 63 desses documentos.

---

### D14. Declarar o uso de IA

**Contexto.** O desenvolvimento foi feito em parceria com um assistente de IA como par de programação.

**Escolha.** Dizer isso no README, sem maquiar. Decisão do autor: "sinceridade sempre em primeiro lugar".

**Consequências.** Quem lê sabe exatamente o que está avaliando: capacidade de definir o problema, decidir, testar em produção, operar e assumir a responsabilidade — com a IA como ferramenta.

---

### D15. Checkpoint por SKU na varredura, sem tocar no atalho que a faz caber na cota

**Contexto.** PM15. A varredura de 1.355 SKUs leva ~1 h pela cota; qualquer reinício no meio recomeçava do zero.

**Escolha.** Tabela `espelho_varredura(passe, sku, quando)`; passe aberto com menos de 12 h é retomado; todo caminho de saída do laço — inclusive erro — marca o SKU, para que um SKU que falha sempre não prenda o passe. O atalho `lastSynced(sku) === saldo → pula` continua (é o que faz a varredura caber), e a brecha que ele abre (PM19) foi fechada por um livro paralelo, não mexendo nele.

**Consequências.** Três reinícios manuais no meio de um passe custaram zero SKUs. Um bug de precisão de minuto no id do passe foi pego por teste (dois passes no mesmo minuto herdavam linhas um do outro).

---

### D16. Sincronizar cadastro linha a linha, e nunca trocar arquivo com conexão aberta

**Contexto.** PM16. A réplica de arquivo do banco de acesso só entrava em uso no failover com marcador; um host que assumiu sem o marcador serviu um cadastro vazio.

**Opções.** (a) Trocar o arquivo do banco de acesso nos standbys a cada réplica; (b) `ATTACH` da réplica e upsert numa transação, no banco vivo.

**Escolha.** (b). Trocar arquivo com conexão WAL aberta já corrompeu banco em 01/09. Import tardio do módulo de banco dentro da função, porque o módulo de réplica é carregado antes de qualquer banco abrir. Sessões e log de acesso ficam fora (token é por máquina; copiar seria furo). Nunca apaga usuário que só exista no standby.

**Consequências.** Duas políticas distintas: adoção (troca de arquivo) recusa réplica menor; sincronização aceita qualquer réplica legível com gente dentro — porque, como nunca apaga, nada se perde, e exigir "≥ local" faria uma remoção legítima no host parar a sincronização para sempre em silêncio.

---

### D17. Verificar as telas como mais um ciclo de saúde, não como um alarme paralelo

**Contexto.** PM18. Havia vigia de ciclos desde 21/08 e nada olhava rotas.

**Escolha.** Lista de rotas esperadas escrita à mão (é a expectativa; se a rota sumir do servidor, a lista continua cobrando), cada uma com o que precisa devolver, rodando dentro do mesmo sistema de saúde que já tinha semáforo, alerta nativo e anti-spam. Sessão de dono temporária, encerrada no fim.

**Consequências.** Tela fora do ar vira faixa vermelha e notificação nas três empresas, com o nome da tela e o motivo. Criar tela nova exige acrescentar a rota na lista — um único lugar.

---

### D18. Antes de compensar um movimento inesperado, conferir a autoria

**Contexto.** PM17. Uma trava que tratava qualquer movimento de saldo como erro do ERP desfez uma devolução correta feita por uma automação nossa.

**Escolha.** Toda compensação consulta os livros-razão das automações próprias (`cesta_devolucao`, `estoque_op` com chave por origem) antes de agir; subida registrada fica, só subida de origem desconhecida é compensada. E compensação desfaz **só o delta** que a nossa ação causou — nunca puxa para um número absoluto (outra pessoa pode ter lançado no meio).

**Consequências.** A regra "cancelar não toca saldo" virou "cancelar pode devolver — sempre conferir e desfazer o que não tem dono".

---

### D19. Corrigir sempre no host

**Contexto.** PM16. Uma blindagem escrita num standby foi sobrescrita no ciclo seguinte pelo código do host, que era a versão antiga.

**Escolha.** Regra operacional: código anda do host para os standbys, nunca ao contrário. Correção urgente feita num standby só vale se ele assumir antes do próximo ciclo — senão, aplicar no host.

---

### D20. Auditar por uma porta diferente da varredura

**Contexto.** PM20. Uma auditoria que usa o mesmo endpoint da varredura herda a mesma cegueira.

**Escolha.** Bater por pelo menos um caminho que a varredura não usa: filtro explícito de situação, janela maior, tipo de documento ignorado. O número pode estar certo e o buraco ser real ao mesmo tempo.

---

### D21. Não fazer um ERP

**Contexto.** Pergunta do dono do projeto em 14/09: continuar a integração e torná-la produto, ou começar um ERP próprio do zero?

**Leitura.** O código pertence à empresa (Lei 9.609/98, art. 4º; o precedente mais parecido, TST 2025, foi contra o empregado mesmo sem função de programador). Um ERP completo assume o fiscal brasileiro em plena reforma tributária, contra concorrentes de centenas de pessoas, com nicho de moda já ocupado. O diferencial construído aqui não é o cadastro de NF — é a camada de inteligência sobre os ERPs (grade furada, reservas presas, saldo fantasma, conciliação).

**Escolha.** Nem um nem outro: fechar bem o que é da empresa; formalizar por escrito a autorização do case e a não-oposição a um produto independente; começar pequeno, *clean-room*, no PC pessoal, uma função só, multi-tenant desde o primeiro dia, meta de 3 clientes pagando antes de expandir. Está aqui porque decidir o que **não** construir também é engenharia.
