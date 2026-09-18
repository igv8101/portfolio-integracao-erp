# 1. Contexto e problema

## O negócio

Um grupo familiar com três empresas do ramo de moda, no Brasil:

| Empresa | Segmento | Plataformas | Como vende |
|---|---|---|---|
| **A** | Atacado de moda infantil (bebê e criança) | Teceo (B2B) + Tiny/Olist (ERP) | Lojistas compram por grade (RN, P, M, G…); pedidos grandes, ritmo de compra de meses; mercadoria importada com volume limitado por cota de importação distribuída entre os CNPJs |
| **B** | Varejo de confecção adulta | Bling (ERP) + Nuvemshop (loja virtual) | Loja física em shopping, feiras itinerantes e site; ticket pequeno, muitas vendas de balcão |
| **C** | Marca de vestuário esportivo | Bling (ERP) | Marketplace com fulfillment terceirizado |

A pessoa responsável pelo projeto trabalhava no grupo como assistente administrativo e, na prática, cuidava de pedidos dos sites, estoque físico e virtual, fluxo de caixa no ERP, suporte interno de TI e treinamento de equipe. Já havia implantado sozinha, anteriormente, a loja virtual anterior e o ERP anterior (Bling) nas três empresas.

## A migração dupla

Em meados de 2026 a empresa A trocou, ao mesmo tempo, a plataforma de e-commerce (para a Teceo, uma plataforma B2B de atacado) e o ERP (de Bling para Tiny). Produtos e clientes foram migrados primeiro; pedidos, contas a receber, contas a pagar e caixa vieram por transformação de exportações CSV em planilhas de importação.

O que ninguém entregava era o **conector** entre as duas plataformas novas. A Teceo expõe uma API para que a marca integre seu ERP; o Tiny expõe a API v3 da Olist. O trabalho de ligar as duas — previsto como trabalho de equipe — ficou inteiramente com uma pessoa, que continuou com todas as outras funções. A chave de homologação da Teceo chegou no fim de julho de 2026; a de produção, em 10 de agosto.

## O que precisava existir

Na primeira fase, a integração "de livro":

- **Pedidos** aprovados na Teceo → pedidos no Tiny, com o cliente criado ou casado por documento, itens por SKU, endereço de entrega e observações, e o resultado reportado de volta (SUCCESS com o número do ERP, ou ERROR com a causa).
- **Clientes** aprovados na Teceo → contatos no Tiny, sem duplicar quem já existia.
- **Estoque** por SKU entre os dois sistemas, com uma fonte da verdade definida.
- **Notas fiscais** emitidas no Tiny → registradas na Teceo (XML e vínculo com o pedido).
- **Edições e cancelamentos** nos dois sentidos, sem eco (uma mudança espelhada não pode voltar como nova mudança).
- **Fotos** dos produtos, que vivem na Teceo, anexadas nos produtos do Tiny.
- **Cadastro** de produto alterado no Tiny → refletido na Teceo.

Na segunda fase, o que a operação pediu depois de usar:

- Painel de status com saúde de cada ciclo e alerta ativo.
- Contagem de estoque que grave nas duas plataformas de uma vez (na tela, pelo celular ou por planilha).
- Relatórios que respondam às perguntas reais do atacado: qual grade está furada no tamanho do meio (um modelo sem P e M está "em estoque" e comercialmente morto), o que está parado, quem sumiu, o que cada cliente costuma levar.
- As outras duas empresas dentro do mesmo sistema, sem misturar um único dado, com login por pessoa e permissões por empresa.

## As restrições que moldaram tudo

**Não existe servidor.** O sistema roda nos PCs da loja. Eles desligam à noite, no fim de semana, e às vezes ficam fora da rede (notebook numa feira, mudança de sede prevista). Se dois PCs decidirem ao mesmo tempo que são "o sistema", um pedido vira dois no ERP e o estoque é contado duas vezes.

**Cota de API apertada e compartilhada.** O Tiny permite 60 requisições por minuto e castiga o excesso com esperas de até 60 s. Essa cota é a mesma para o ciclo de fundo (que varre 1.300 SKUs) e para a pessoa que está no balcão tentando achar um produto no painel. O Bling permite 3 req/s por conta, com bloqueios de IP por rajada.

**Tokens que morrem.** O refresh token do Tiny expira em 1 dia se não for usado; PC desligado no fim de semana significava reautorizar toda segunda-feira.

**Dados que não são o que parecem.** O ERP tinha produtos cadastrados como "com variações" sem nenhuma variação (e o tipo é imutável depois da criação); produtos duplicados sob o mesmo SKU; SKUs criados já com saldo alto que nunca tinham entrado fisicamente; notas fiscais que o próprio ERP se recusa a lançar no estoque sem avisar por que. Nada disso aparece na documentação — aparece no saldo errado, semanas depois.

**Três empresas, zero mistura.** Regra dita pelo dono do processo, ao pé da letra: "uma empresa é uma coisa, e a outra, é outra. Nada deve-se misturar. Nadinha." A separação precisava ser física e verificável, não uma promessa do código.

**Uma pessoa.** Quem programava era também quem contava peças, emitia nota e atendia lojista. Toda ferramenta que exigisse conferência humana rotineira nasceria morta; toda operação precisava sobreviver a interrupção e se retomar sozinha.

## Como o problema evoluiu

O projeto começou como "integrar dois sistemas" e, em cinco semanas, virou "sistema operacional do grupo": os dados que a integração acumulava (notas fiscais desde o início do ano anterior, catálogo, saldos, movimentações) tornaram possíveis relatórios que os ERPs não ofereciam; a operação em vários PCs exigiu eleição de host, replicação e atualização automática; a entrada das outras empresas exigiu identidade unificada e permissões. A linha do tempo completa está em [`06-linha-do-tempo.md`](06-linha-do-tempo.md).
