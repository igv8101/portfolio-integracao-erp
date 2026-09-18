# Como isso vira currículo

Material pronto para copiar. Tudo verificável neste repositório. Ajuste o nome da empresa conforme a decisão de anonimato (aqui está anônimo; no currículo e na entrevista o nome real pode aparecer).

---

## Experiência (versão para currículo, PT)

**Desenvolvedor de integrações e sistemas internos** — grupo de moda (3 empresas) · jul/2026 – atual

- Projetei e implementei, sozinho, a integração entre uma plataforma B2B de atacado (Teceo) e o ERP Tiny/Olist — pedidos, clientes, estoque, notas fiscais, fotos, edições e cancelamentos bidirecionais — do primeiro contrato de API ao go-live em produção em **14 dias**.
- Evoluí a integração para um sistema operacional de **3 empresas e 3 plataformas** (Teceo, Tiny, Bling ×2 contas, Nuvemshop como canal), com bancos fisicamente separados por empresa, identidade unificada e permissões por papel.
- Fiz o sistema rodar em **4 PCs comuns sem servidor**: eleição de host por *leader lease* em Redis (fail-closed), auto-update de código pela rede, réplica dos bancos a cada 2 min e agente de bandeja em C# para operação remota — com a janela de *split-brain* medida por simulação e corrigida.
- Construí uma **guarda de escrita idempotente** para estoque (chave obrigatória na assinatura, livro por operação, retomada automática após queda) e um **harness com ERP falso** que reproduziu 7 erros clássicos de integração — 3 falharam, foram corrigidos, 7/7 passaram.
- Diagnostiquei e reparei incidentes reais de estoque em produção (dedução em dobro em 60 SKUs, reserva presa em 518 SKUs, 113 SKUs negativos por lançamento tardio de nota, saldo oscilando por SKU duplicado, estoque fantasma de cadastro, 95 produtos cadastrados errado escondidos por dois campos da API), com ferramentas de prévia obrigatória, invariante verificada e livro-razão que sobrevive a falha — 1.161 reservas resolvidas e 95 produtos migrados com zero erro; ao fim, ERP sem saldo negativo nem reserva presa.
- Entreguei pedidos, notas fiscais, financeiro (a receber, a pagar, caixa) e relatórios imprimíveis com período personalizado para as 3 empresas a partir de espelhos locais dos ERPs, com edição de pedido gravando no ERP sob prévia, e uma conferência diária automatizada dos relatórios contra os dois ERPs, dia a dia.
- Instrumentei a saúde do sistema para vigiar não só os ciclos mas as 43 telas (verificador de rotas como ciclo de saúde), depois de uma regressão silenciosa; sincronizei o cadastro de usuários linha a linha entre os PCs; escrevi 20 post-mortems, incluindo os erros meus.
- Fiz engenharia reversa de comportamentos não documentados das três APIs (cursor de paginação: 2h30 → 14 s; limite de anexo e exigência de extensão; liberação de reserva por lançamento de nota) e reduzi a dependência da fornecedora de 5 pendências para 1.
- Encontrei e fechei no mesmo dia duas falhas de segurança em auditoria própria (recriação de conta administrativa por migração; rota administrativa sem autenticação).
- Entreguei relatórios que os ERPs não oferecem: ruptura de grade com previsão de perda, estoque parado por nota fiscal, perfil de compra por cliente com sugestões, separação fiscal varejo × feira × importação, faxina de cadastro.
- Stack: Node.js 26 com TypeScript nativo e zero dependências de runtime, `node:sqlite` (WAL), OAuth2, REST, Upstash Redis, C#, PowerShell. 49 testes unitários + harness com ERP falso + 10 simuladores de mecanismo. Desenvolvimento em parceria com IA (Claude) como par de programação.

---

## Experience (résumé version, EN)

**Integration & Internal Systems Developer** — fashion group (3 companies), Brazil · Jul 2026 – present

- Designed and built, single-handedly, the integration between a B2B wholesale platform (Teceo) and the Tiny/Olist ERP — orders, customers, stock, invoices, photos, bidirectional edits and cancellations — from first API call to production go-live in **14 days**.
- Grew it into an operations system for **3 companies across 3 platforms** (Teceo, Tiny, Bling ×2 accounts, Nuvemshop as a channel), with physically separated databases per company, a unified identity store and role-based permissions.
- Ran it on **4 ordinary PCs with no server**: Redis leader lease for host election (fail-closed), LAN code auto-update, database replication every 2 min and a C# tray agent for remote operations — with the split-brain window measured by simulation and fixed.
- Built an **idempotent stock write guard** (mandatory operation key in the function signature, per-operation ledger, automatic resumption after crashes) and a **fake-ERP harness** reproducing 7 classic integration failures — 3 failed, were fixed, 7/7 passed.
- Diagnosed and repaired real production stock incidents (double deduction across 60 SKUs, stuck reservations on 518 SKUs, 113 negative SKUs from late invoice posting, flip-flopping balances from duplicated SKUs, phantom stock from ERP placeholders, 95 mis-typed products hidden behind two similarly named API fields) with preview-first tools, verified invariants and crash-safe ledgers — 1,161 reservations released and 95 products migrated with zero errors; the ERP ended with no negative stock and no stuck reservations.
- Shipped orders, invoices, finance (receivables, payables, cash) and printable reports with custom periods for all 3 companies from local ERP mirrors, with order editing written back to the ERP under mandatory preview, plus an automated daily reconciliation of reports against both ERPs, day by day.
- Instrumented health to watch not only background cycles but all 43 screens (route checker as a health cycle) after a silent regression; row-level user-store sync across PCs; 20 written post-mortems, including my own mistakes.
- Reverse-engineered undocumented behaviour in all three APIs (pagination cursor: 2h30 → 14 s; attachment size/extension rules; reservation release via invoice posting), cutting vendor dependencies from 5 open items to 1.
- Found and fixed, same day, two security flaws in self-audit (admin account recreated by a migration; unauthenticated admin route).
- Delivered reports the ERPs don't offer: size-run gaps with loss forecast, dead stock by invoice, customer purchase profiles with suggestions, retail/fair/import fiscal split, catalogue hygiene.
- Stack: Node.js 26 with native TypeScript and zero runtime dependencies, `node:sqlite` (WAL), OAuth2, REST, Upstash Redis, C#, PowerShell. 49 unit tests + fake-ERP harness + 10 mechanism simulators. Developed in partnership with AI (Claude) as a pair programmer.

---

## Resumo para o LinkedIn ("Sobre")

Engenheiro de software com formação em Ciência da Computação, pós-graduação em Ciência de Dados e em Inteligência Artificial. Nos últimos anos implantei sozinho, em um grupo de três empresas de moda, a loja virtual, dois ERPs e — em cinco semanas de 2026 — a integração completa entre a plataforma B2B e o ERP, que virou o sistema operacional do grupo: quatro PCs sem servidor, eleição de host, replicação, auto-update, guarda idempotente de estoque provada por simulação, e relatórios que os ERPs não entregam. Gosto de problema que só aparece em produção, de ferramenta que se recupera sozinha e de documentar o porquê. Trabalho em parceria com IA e digo isso abertamente. Procurando atuar com engenharia de dados e IA aplicada.

---

## Frases para entrevista (uma por tema, com o número que sustenta)

- **Prazo.** "Do Swagger ao go-live foram 14 dias; da primeira chamada ao sistema completo de três empresas, cinco semanas — enquanto eu operava o estoque."
- **Confiabilidade.** "A regra que ficou: toda escrita externa grava a intenção localmente antes de chamar a API. A guarda de estoque não compila sem uma chave de idempotência. Testei contra um ERP falso com sete cenários de falha; três reprovaram na primeira rodada."
- **Sistemas distribuídos.** "Quatro PCs, um host. Lease em Redis com TTL de 3 minutos. A auditoria achou uma janela de 60 segundos com dois hosts porque a tolerância contava da primeira falha e não da última renovação — provei por simulação, corrigi, provei de novo."
- **Debugging.** "Um SKU pulava entre 16 e 0. Contagem física: 8. Eram dois produtos ativos com o mesmo código no ERP, e cada PC que virava host escolhia um."
- **API.** "O cursor de paginação era base64 de `createdAt_id`, inclusivo, e a primeira página sem cursor não era o início. De 966 chamadas em duas horas e meia para 28 páginas em 14 segundos."
- **Segurança.** "A migração de usuários recriava a conta de dono com a senha antiga sempre que o login era renomeado. Achei numa auditoria minha e fechei no mesmo dia."
- **Produto.** "A régua de inatividade 'de livro' era 60 dias. Para o atacado, é um ano — 60 dias é normal para quem compra por grade. Sem ouvir quem opera, o relatório mentiria."
- **Erro próprio.** "Uma ferramenta de reparo teve quatro erros num dia, dois em produção. O dano foi contido porque ela media antes e depois e anotava o que já tinha feito. Virou regra escrita."
- **Autoria.** "Um cancelamento 'devolveu' estoque e eu desfiz. Estava errado: era a nossa própria regra da cesta. Conferir o resultado não basta — tem que conferir quem fez. Hoje toda compensação consulta os livros-razão das automações antes de agir."
- **Regressão silenciosa.** "Uma cópia velha de um arquivo apagou telas inteiras e nenhum alarme tocou. A saúde vigiava ciclos, não telas. Agora 43 rotas têm expectativa escrita à mão e rodam como ciclo de saúde."
- **IA.** "Usei IA como par de programação e declaro isso. As decisões, os testes em produção, a operação e a responsabilidade foram minhas."

---

## Palavras-chave (para filtros de recrutamento)

Node.js · TypeScript · REST · OAuth2 · SQLite · WAL · integração de ERP · Tiny ERP · Olist · Bling · Teceo · e-commerce B2B · idempotência · leader election · Redis · replicação · failover · rate limiting · engenharia reversa de API · observabilidade · post-mortem · simulação de falhas · C# · PowerShell · Windows · PWA · Chart.js · SQL · análise de vendas · curva ABC · ruptura de grade · NF-e · NFC-e · NCM · segurança de aplicação · scrypt · RBAC
