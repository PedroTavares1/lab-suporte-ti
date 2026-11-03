# Runbook 03: Não consigo imprimir

**Ambiente:** Windows 11 Enterprise Evaluation (10.0.22000.194), máquina virtual no VirtualBox · **Testado em:** 03/11/2026

## Sintoma
"Não consigo imprimir." No Bloco de Notas, ao abrir Imprimir, a lista de impressoras vem vazia e o Windows pergunta: "Antes de executar tarefas relacionadas à impressora [...] você precisa instalar uma impressora. Deseja adicionar uma impressora agora?" (print 4)

## Impacto
Uma pessoa sem imprimir. Urgência média; alta se o documento é para entrega no mesmo dia.

## Antes de começar
Perguntar: acontece com todos os documentos ou só com um? Só neste PC ou também em outros que usam a mesma impressora? Desde quando? Mudou algo (atualização, driver)? Não aceitar de cara o aviso "adicionar uma impressora": neste caso as impressoras continuam instaladas (print 6) e o que falhou foi o serviço de impressão.

## Como o defeito foi provocado
No PowerShell como administrador, a partir do snapshot "Limpo": `net stop spooler`. O spooler é o serviço do Windows que recebe os trabalhos de impressão e organiza a fila. Um serviço é um programa que roda em segundo plano, sem janela. A parte da fila presa tem a sua própria seção, mais abaixo. Ensaio em laboratório, dados fictícios.

## Ponto de partida: impressão saudável
Antes do defeito, medi a máquina funcionando (prints 1 e 2).

| Comando ou ação | Obtido no ensaio |
| --- | --- |
| `Get-Service Spooler; Get-Printer \| Format-Table Name, DriverName, PrinterStatus -AutoSize` | Spooler `Running`; três impressoras, todas com status Normal: Microsoft XPS Document Writer, Microsoft Print to PDF e Fax |
| Bloco de Notas → Arquivo → Imprimir | A janela lista Fax, Microsoft Print to PDF e Microsoft XPS Document Writer; a selecionada (Microsoft Print to PDF) mostra o status "Pronta" |

## Diagnóstico: serviço parado
Do sintoma ao serviço, no PowerShell como administrador. A coluna "Obtido no ensaio" traz o que apareceu de verdade. O número do print aponta para a seção Evidências.

| Print | Comando ou ação | Esperado (saudável) | Obtido no ensaio | Leitura |
| --- | --- | --- | --- | --- |
| 3 | `net stop spooler` | Serviço parado | "O serviço de Spooler de Impressão foi finalizado com êxito." | Defeito provocado |
| 4 | Bloco de Notas novo → Arquivo → Imprimir | Lista com as impressoras instaladas | Lista vazia e o aviso para instalar uma impressora, com os botões Sim e Não | O aplicativo não consegue listar as impressoras. Suspeitar do serviço de impressão, não de falta de impressora |
| 5 | `Get-Service Spooler; Get-Printer` | `Running` e a lista de impressoras | `Stopped`; o `Get-Printer` deu erro: "O serviço de spooler não pode ser alcançado. Verifique se o serviço de spooler está em execução." (HRESULT 0x800706ba) | O serviço está parado, e o erro já diz isso. As impressoras não sumiram, como o print 6 mostra |

Atenção ao print 4: o aviso diz para instalar uma impressora, mas não é isso que falta. Não clique em Sim. Clique em Não e vá ao serviço.

## Causa
Serviço Spooler de Impressão parado (print 5).

## Solução
1. Iniciar o serviço: `Start-Service Spooler`.
2. Conferir: `Get-Service Spooler` deve mostrar `Running`, e as impressoras devem voltar (print 6).

## Verificação
- `Get-Service Spooler` mostrou `Running` e as três impressoras de volta, com status Normal (print 6).
- No Bloco de Notas, Arquivo → Imprimir listou as impressoras de novo, e a Microsoft Print to PDF apareceu com status "Pronta" (print 7).
- Imprimi na Microsoft Print to PDF e salvei como `teste-impressao`. O arquivo existe em `C:\Users\Lab-Win11\Documents`: `teste-impressao.pdf`, 38.994 bytes, gravado em 03/11/2026 às 16:09 (print 8).

## Fila presa: limpar a fila
Se o spooler volta mas um trabalho fica preso, o N1 limpa a fila: para o serviço, apaga os arquivos da pasta de spool e liga o serviço de novo. Os arquivos `.SPL` (dados do documento) e `.SHD` (ficha do trabalho) ficam em `C:\Windows\System32\spool\PRINTERS`.

**Como foi provocado.**
1. Pausei a impressora Microsoft Print to PDF pelo PowerShell: `Get-CimInstance Win32_Printer -Filter "Name='Microsoft Print to PDF'" | Invoke-CimMethod -MethodName Pause`. Resultado: `ReturnValue 0` e `PrinterStatus : Paused` (print 9). Não consegui clicar em "Pausar impressão" na janela da fila, por isso usei o PowerShell. As Configurações passaram a mostrar "Pausado", e a janela da fila, "Em Pausa" (print 10).
2. Imprimi do Bloco de Notas na impressora pausada. A janela Imprimir mostrou "Em Pausa" (print 11) e, ao clicar em Imprimir, apareceu a janela de salvar, junto com a caixa "Imprimindo" (print 12).

| Print | Comando | Esperado | Obtido no ensaio | Leitura |
| --- | --- | --- | --- | --- |
| 13 | `Get-Printer ... \| Format-List Name, PrinterStatus; Get-PrintJob ...; Get-ChildItem C:\Windows\System32\spool\PRINTERS` | Impressora pausada, um trabalho listado e arquivos `.SPL` e `.SHD` | `PrinterStatus : Paused`; **nenhum trabalho listado**; dois arquivos na pasta de spool, `FP00001.SHD` e `FP00001.SPL`, com 0 bytes e última gravação às 16:35 | Sobrou resíduo de um trabalho na pasta de spool, sem trabalho listado. É esse resíduo que a limpeza remove |

**Limpeza.**
```
Stop-Service Spooler -Force
Remove-Item C:\Windows\System32\spool\PRINTERS\* -Force -Verbose
Start-Service Spooler
```
O `-Force` do `Stop-Service` também para os serviços que dependem do spooler. As linhas "MODO DETALHADO" do print 14 mostram `FP00001.SHD` e `FP00001.SPL` sendo removidos.

**Verificação.** Depois da limpeza, `Get-PrintJob` e `Get-ChildItem` não devolveram nada: nenhum trabalho e nenhum arquivo na pasta de spool (print 15). Depois, retomei a impressora: `Get-CimInstance Win32_Printer -Filter "Name='Microsoft Print to PDF'" | Invoke-CimMethod -MethodName Resume`, com `ReturnValue 0` (print 16).

**O que este ensaio não mostrou.** Um trabalho listado pelo `Get-PrintJob`, com status de erro, como acontece com uma impressora de rede offline: não testado. O que foi provado é o procedimento de limpeza, com arquivos apagados e pasta vazia.

## Escalonamento
Se o serviço não inicia, ou para de novo logo depois de iniciar, passar ao N2 com a mensagem exata do erro do `Start-Service` e o que mudou antes (atualização, instalação de driver ou de impressora).

## Prevenção
Se repetir no mesmo PC, procurar atualização ou driver de impressora recente e registrar no chamado. Orientar o usuário a avisar o suporte quando a lista de impressoras aparecer vazia.

## Notas do ensaio
**Tempo gasto.** Os prints do ensaio final vão de 15:49 a 16:37, ou seja, 48 minutos no total, com dois intervalos longos sem prints: cerca de 12 minutos entre os prints 6 e 7 e cerca de 15 entre o 8 e o 9. Contando a primeira tentativa, descartada (a partir de 15:36), foram 61 minutos.

**O que não funcionou no começo.**
- O Bloco de Notas não abriu pelo clique no programa. Abri pelo PowerShell, com `Start-Process notepad`.
- Na primeira tentativa, o `Get-Printer` sem formatação saiu como uma lista longa, em 4 telas. Passei a usar `Format-Table Name, DriverName, PrinterStatus -AutoSize`.
- Salvei o primeiro PDF em Downloads, que é a pasta que a janela de salvar sugere (print 7b). Repeti a impressão e salvei em Documentos.
- Na fila presa, o botão "Pausar impressão" da janela da fila não pôde ser clicado. Pausei pelo PowerShell.
- A primeira tentativa de listar a fila e a pasta de spool (16:27) não mostrou trabalho nem arquivos. Os arquivos só apareceram depois que a impressão ficou esperando na janela de salvar (prints 12 e 13).

**Diferenças em relação ao previsto.**
- No print 4, a lista de impressoras saiu vazia em um Bloco de Notas aberto depois de parar o serviço. Na primeira tentativa, num Bloco de Notas que já tinha aberto a janela Imprimir antes de parar o serviço, o aviso apareceu e a lista continuou mostrando as três impressoras (print 4b). Não confirmei a causa.
- A Microsoft Print to PDF é uma impressora virtual. Com ela pausada (prints 10 e 11), a janela de salvar apareceu mesmo assim (print 12), e o `Get-PrintJob` não listou o trabalho (print 13).

## Evidências
**Print 1.** `Get-Service` e `Get-Printer` com o spooler rodando.
![Print 1: spooler Running e as três impressoras](../prints/win-03-p01-spooler-running-impressoras.png)

**Print 2.** Janela Imprimir saudável, com a Microsoft Print to PDF pronta.
![Print 2: janela Imprimir saudável](../prints/win-03-p02-janela-imprimir-saudavel.png)

**Print 3.** `net stop spooler`.
![Print 3: net stop spooler](../prints/win-03-p03-net-stop-spooler.png)

**Print 4.** Aviso para instalar impressora, com a lista vazia.
![Print 4: aviso para instalar impressora, lista vazia](../prints/win-03-p04-aviso-instalar-impressora-lista-vazia.png)

**Print 4b.** O mesmo aviso na primeira tentativa, com a lista ainda cheia.
![Print 4b: aviso na primeira tentativa, lista cheia](../prints/win-03-p04b-aviso-primeira-tentativa-lista-cheia.png)

**Print 5.** Spooler `Stopped` e o erro do `Get-Printer`.
![Print 5: spooler Stopped e erro do Get-Printer](../prints/win-03-p05-spooler-stopped-erro-get-printer.png)

**Print 6.** Spooler `Running` e as impressoras de volta.
![Print 6: spooler Running e as impressoras de volta](../prints/win-03-p06-spooler-running-impressoras-de-volta.png)

**Print 7.** Janela Imprimir com a lista de volta e a impressora pronta.
![Print 7: janela Imprimir com a lista de volta](../prints/win-03-p07-janela-imprimir-pronta.png)

**Print 7b.** Janela de salvar, com a pasta Downloads como padrão.
![Print 7b: janela de salvar abre em Downloads](../prints/win-03-p07b-salvar-pdf-pasta-padrao-downloads.png)

**Print 8.** O PDF gravado em Documentos.
![Print 8: PDF em Documentos](../prints/win-03-p08-pdf-em-documentos.png)

**Print 9.** Impressora pausada pelo PowerShell.
![Print 9: impressora pausada pelo PowerShell](../prints/win-03-p09-pausar-impressora-powershell.png)

**Print 10.** Configurações e janela da fila, com a impressora em pausa.
![Print 10: fila em pausa na janela](../prints/win-03-p10-fila-em-pausa-janela.png)

**Print 11.** Janela Imprimir com o status "Em Pausa".
![Print 11: janela Imprimir com Em Pausa](../prints/win-03-p11-janela-imprimir-em-pausa.png)

**Print 12.** Janela de salvar e caixa "Imprimindo", mesmo com a impressora pausada.
![Print 12: janela de salvar e caixa Imprimindo](../prints/win-03-p12-janela-salvar-e-imprimindo.png)

**Print 13.** Pasta de spool com `FP00001.SHD` e `FP00001.SPL`, sem trabalho listado.
![Print 13: pasta de spool sem trabalho listado](../prints/win-03-p13-spool-sem-trabalho-listado.png)

**Print 14.** Limpeza da fila, com os arquivos removidos.
![Print 14: limpeza da fila](../prints/win-03-p14-limpar-fila.png)

**Print 15.** Fila e pasta de spool vazias.
![Print 15: fila e pasta de spool vazias](../prints/win-03-p15-fila-e-spool-vazios.png)

**Print 16.** Impressora retomada.
![Print 16: impressora retomada](../prints/win-03-p16-retomar-impressora.png)
