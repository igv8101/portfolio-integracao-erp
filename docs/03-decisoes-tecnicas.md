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

**Atualização (25/09).** Entre 23 e 25/09 uma segunda ferramenta de IA (Codex) fez uma auditoria de segurança a partir de um retrato do código e implantou o protocolo de release assinada (D22) e a maior parte da suíte de testes atual. Vale a mesma regra: está dito aqui, com o que cada uma fez.

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


---

## Semana 9 (19/09 a 28/09)

### D22. Código como *release* assinada com sequência crescente — e publicar só a partir de uma cópia

**Contexto.** PM21: o atualizador por hash copiava "o que o host tem", e um host com código velho levou a frota para trás. Em 23–25/09 uma auditoria de segurança (feita com uma segunda ferramenta de IA, ver D14) redesenhou a distribuição.

**Escolha.** Cada pacote é uma release assinada (Ed25519) com número de sequência sempre crescente; o cliente recusa pacote sem assinatura válida e recusa **downgrade**. As máquinas conversam por HTTPS com identidade própria e CA própria. Publicar tem um protocolo: trabalhar numa **cópia** da instalação ativa, testes e typecheck na cópia, gerar o pacote, armar no host, reiniciar; os standbys puxam pelo canal autenticado. Editar a instalação ao vivo é proibido — o hash diverge do manifesto assinado e os standbys recusam.

**A guarda que veio depois.** Em 25/09 outra ferramenta alterou arquivos direto na produção (uma limpeza de comentários). O gerador de pacote passou a **comparar a produção com a cópia** e recusar se a produção tiver qualquer mudança que a cópia não carregue. A passagem só é liberada com uma lista explícita de arquivos meus e, mesmo assim, recusa se sobrar diferença fora dela.

**Consequências.** Publicar ficou mais lento (minutos em vez de salvar um arquivo) e passou a exigir uma pessoa para reiniciar o host. Em troca, "a versão nova" tem significado verificável, e nenhum PC anda para trás.

---

### D23. Só vira host quem tem o banco em dia

**Contexto.** PM22: o failover perguntava "o host caiu?" e nunca "o meu banco está em dia?".

**Escolha.** Marca de escrita gravada pelo host a cada minuto em cada banco de negócio e na nuvem; candidato a host compara a marca do banco que vai usar com a da nuvem (tolerância de 10 min) e fica em standby se estiver atrás. Adoção de réplica só quando a réplica não é mais velha que o local. Saída de emergência explícita (um arquivo com nome inequívoco, uso único) e aviso no Windows quando um PC se recusa a assumir.

**Consequências.** Pode haver minutos sem host se o único PC ligado estiver com banco velho — escolha consciente: minutos sem integração custam menos que um dia de dados apagados. O banco de identidade continua com regra própria (D16), porque ali a pergunta é "tem gente dentro?", não "é recente?".

---

### D24. Tokens seguem o cadeado, não a réplica

**Contexto.** 21/09, troca de host: o ERP de varejo respondeu `invalid_grant` nas duas contas. O refresh do Bling é de **uso único**; o novo host tinha herdado pela réplica uma cópia que o host anterior já tinha queimado.

**Escolha.** Um cofre na mesma nuvem do cadeado, uma chave por conta. Toda gravação de token (autorização ou renovação) vai também para o cofre, com data e máquina. Antes de renovar — ou antes de desistir por falta de token — o PC olha o cofre e adota o mais novo; ao assumir o host, sincroniza os três antes de ligar qualquer ciclo. Sem configuração de nuvem, desliga; nuvem fora, segue com o local e avisa.

**Consequências.** Quem renovou por último ganha, em qualquer máquina. Efeito colateral previsto e documentado: se alguém autorizar com o usuário errado, o cofre espalha o usuário errado — por isso a reautorização mostra na tela o que foi concedido.

---

### D25. Trocar o host por bilhete de preferência, nunca "arrancando" o cadeado

**Contexto.** Pedido do dia a dia: "traga o host para este PC, e deixe um botão para isso".

**Opções.** (a) O painel apaga o cadeado e o PC escolhido pega; (b) o painel escreve uma preferência e a troca acontece pelo próprio protocolo.

**Escolha.** (b). O botão grava um bilhete; o host atual o vê na renovação seguinte (≤ 15 s), solta o cadeado e reinicia como standby; os outros PCs **não disputam** enquanto o bilhete apontar para outro; o escolhido verifica a cada 20 s e assume em ~1 min. O bilhete só manda por 3 min — depois vira sugestão e quem estiver de pé pode assumir (senão, um PC escolhido e desligado deixaria a frota sem host) — e expira em 15 min.

**Consequências.** Nunca há dois hosts, porque ninguém tira o cadeado de ninguém: o dono solta. O mesmo bilhete virou parte do protocolo de publicação (D22), para o host certo reassumir depois do reinício.

---

### D26. Webhook atrás de uma portaria mínima — e o polling continua

**Contexto.** D2 escolheu polling porque a loja não tem endereço público. Com a nova versão da plataforma B2B (20/09), os webhooks passaram a ter assinatura HMAC.

**Escolha.** Um processo separado, a **portaria**, é a única coisa exposta pelo túnel: só `POST` na rota do webhook passa, todo o resto é 404; há um ping para teste e um diário de entregas. O painel valida a assinatura e o timestamp. O polling **não saiu**: o webhook adianta, o polling garante.

**Consequências.** A superfície exposta é uma rota. Pendências registradas: o túnel atual tem endereço temporário (endereço fixo depende de domínio próprio) e a portaria encaminha para o PC local — precisa passar a seguir o cadeado.

---

### D27. Classificar os erros que **provam** o desfecho

**Contexto.** PM23: um diário de pedidos que tratava todo erro como "não sei se criou".

**Escolha.** Duas listas explícitas. Recusa definitiva (400, 401, 403, 404, 422, 429): o ERP não criou — volta para a fila. Ambíguo (timeout, 5xx, 409): pode ter criado — fica em reconciliação, com revisão humana num botão que exige escrever o que foi conferido.

**Consequências.** Nenhum pedido duplicado e nenhum pedido esquecido por excesso de cautela. A lista é código, com teste próprio, não uma interpretação espalhada pelos `catch`.

---

### D28. Renovar o token pela nuvem — fora da empresa e com três freios

**Contexto.** PM24. O token do ERP precisa de uma renovação a cada 24 h, inclusive quando todos os PCs estão desligados.

**Opções.** (a) Um PC ligado no fim de semana; (b) lembrete para reautorizar; (c) uma função em nuvem com plano gratuito que exige cartão; (d) uma tarefa agendada no GitHub Actions, gratuita e sem cartão.

**Escolha.** (d), num repositório **privado** na conta pessoal do autor, **sem código da empresa** — só um script de renovação que lê e grava o mesmo cofre de tokens (D24), com as credenciais em segredos do repositório. Roda a cada 6 h com três freios, nesta ordem:
1. **interruptor** no cofre: desligado → não faz nada;
2. token renovado há menos de 12 h → não faz nada (os PCs estão cuidando);
3. sistema sem host há mais de **5 dias** → não faz nada (a tarefa existe para fim de semana e feriado, não para manter viva uma integração abandonada).

**Consequências.** Na segunda, o host adota pelo cofre o token que a nuvem manteve. E a tarefa foi desenhada para **o autor poder ir embora**: ela para sozinha quando a empresa deixa de usar o sistema; o interruptor pode ser acionado pela empresa (botão no painel, previsto) ou pelo dono do repositório; e, sendo gratuita e sem cartão, o pior caso de estourar o limite é ela parar. Uma integração que depende da conta pessoal de alguém precisa de um jeito limpo de deixar de depender.

---

### D29. Registrar cada ato por computador — e escolher o que **não** registrar

**Contexto.** Quatro PCs, cinco pessoas, papéis por empresa. A auditoria de estoque já dizia "quem"; faltava "de qual máquina" para tudo que muda estado.

**Escolha.** Todo POST/PUT/PATCH/DELETE do painel, mais as rotas de autorização OAuth (que são GET), gravado quando a resposta termina: hora, IP, computador (IP fixo → cadastro de máquinas), usuário, empresa (pela rota), método, rota, parâmetros com segredos mascarados (`code`, `state`, tokens), status e duração. Não entra: o **corpo** (pode ter senha), consultas GET, e o tráfego máquina-com-máquina (agente, réplica, atualização, webhook). Só o host grava, no banco de identidade — replicado, sobrevive à troca de host —, com retenção de 180 dias e tela só para o dono do sistema.

**Consequências.** O login não tinha usuário na requisição (a sessão ainda não existia); o registro passou a ler a sessão criada na **resposta**. Detalhe pequeno que só apareceu com o primeiro ato real — a mesma lição do PM25.
