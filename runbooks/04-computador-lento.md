# Runbook 04: O computador está lento

**Ambiente:** Windows 11 Enterprise Evaluation (snapshot "Limpo", o mesmo dos runbooks 01, 02, 03 e 05), máquina virtual no VirtualBox, com 2 processadores virtuais e 4 GB de memória · **Testado em:** 04/11/2026

## Sintoma
"O computador está lento." É o chamado mais vago do service desk: a causa pode estar na CPU (processador), na memória, no disco ou na rede. No ensaio, o sintoma foi medido: a CPU ficou em 100% nas 4 medidas (print 3) e o Windows PowerShell apareceu no topo do Gerenciador de Tarefas, com 92,4% (print 4).

## Impacto
Uma pessoa trabalhando mais devagar, com travadas ao abrir programas e ao digitar. Urgência média. Alta se for um PC de atendimento ao público ou se a lentidão atingir várias pessoas ao mesmo tempo.

## Antes de começar
Perguntar: desde quando? Só neste PC? Mudou algo (programa novo, atualização)? Quantos programas e abas estão abertos? Dá para reiniciar agora, ou há trabalho aberto sem salvar?

**Antes de encerrar qualquer processo, anotar o nome e o PID** (o número que identifica o processo). Só finalizar um programa que a pessoa abriu e que dá para reconhecer. Processo do sistema ou do antivírus (`svchost`, `lsass`, `MsMpEng`) não se finaliza à mão: passar ao N2.

Seis termos. **CPU** (processador) é o cérebro do PC, e a **carga** é quanto dele está ocupado, em %: com 100%, não sobra nada para abrir um programa novo. **Processo** é um programa em execução (um navegador abre vários). **Job** é uma tarefa que o PowerShell roda em segundo plano. **Processador virtual** é a fatia de processador que o VirtualBox entrega à máquina virtual: esta tem 2. **Baseline** é a medida de referência, tirada antes do defeito, para comparar com o depois.

## Como o defeito foi provocado
**Parte 0: o Windows Update parado (ajuste de laboratório).** No primeiro ensaio, a máquina virtual já estava lenta sem defeito nenhum, porque o Windows instalava atualizações (prints 11 a 13). Assim não dá para provar antes e depois. Então, só nesta máquina virtual descartável, bloqueei o Windows Update antes de provocar o defeito. **Num PC de verdade não se faz isso:** lá, o certo é deixar a atualização terminar e definir o Horário ativo, para ela não atrapalhar o trabalho.

O script, em um PowerShell como administrador (o resultado está no print 1a):

```
$p = 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU'
if (-not (Test-Path $p)) { New-Item -Path $p -Force | Out-Null }
Set-ItemProperty -Path $p -Name NoAutoUpdate -Value 1 -Type DWord
foreach ($s in 'wuauserv','UsoSvc','BITS') { Stop-Service $s -Force -WarningAction SilentlyContinue -ErrorAction SilentlyContinue; Set-Service $s -StartupType Disabled -ErrorAction SilentlyContinue }
Stop-Service TrustedInstaller -Force -WarningAction SilentlyContinue -ErrorAction SilentlyContinue; Stop-Process -Name TiWorker -Force -ErrorAction SilentlyContinue
Get-Service wuauserv,UsoSvc,BITS,TrustedInstaller,WaaSMedicSvc | Select-Object Name,Status,StartType | Format-Table -AutoSize; Get-Date
```

O que ele faz:
- `NoAutoUpdate = 1` é o valor de registro que desativa as atualizações automáticas, o mesmo que a política de grupo "Configurar Atualizações Automáticas" grava (ver Fontes).
- `wuauserv` (Windows Update) decide o que instalar, `UsoSvc` (Update Orchestrator) agenda e `BITS` baixa em segundo plano. O script para os três e os desabilita.
- `TrustedInstaller` é o serviço que executa a instalação. O processo dele é o `TiWorker.exe`, que o Gerenciador de Tarefas chama de "Windows Modules Installer Worker". O script só **para** esse serviço. **Quem o desabilitou fui eu, na mão**, antes do print 1a (não anotei a hora nem o método).
- `WaaSMedicSvc` (Windows Update Medic) existe para proteger e consertar os componentes do Windows Update. O script não mexe nele: ficou em Manual (print 1a). Conferi os serviços de novo no fim (print 10).

Restaurar o snapshot Limpo também desfaz o ajuste. Reverter na mão, removendo o `NoAutoUpdate` e voltando os serviços para Manual, **não foi testado**.

**Parte 1: o defeito.** Dois jobs, cada um com um laço infinito que não faz nada além de ocupar um processador inteiro:

```
1..2 | ForEach-Object { Start-Job { while ($true) { } } } | Format-Table -AutoSize; Get-Date
```

Com 2 processadores virtuais, dois jobs enchem a CPU. O print 2 mostra os dois em Running (14:13:46). Os Id 1 e 3 são normais: o PowerShell cria um job pai para cada um (Id 1 e 3) e um job filho (Id 2 e 4), e o `Get-Job` lista só os pais.

Tudo isso aconteceu só dentro da máquina virtual. Ensaio em laboratório, dados fictícios.

## Ponto de partida: antes do defeito
Medida de referência: a carga da CPU 12 vezes, uma a cada 10 segundos (2 minutos):

```
1..12 | ForEach-Object { (Get-CimInstance Win32_Processor | Measure-Object LoadPercentage -Average).Average; Start-Sleep 10 }; Get-Date
```

A regra que adotei: só provocar o defeito se pelo menos 9 das 12 medidas ficarem abaixo de 20. E tirar o print **antes** de limpar a tela com `cls`.

| Print | Comando ou ação | Obtido no ensaio |
| --- | --- | --- |
| 1a | O script da Parte 0 (a última linha mostra os serviços) | BITS, UsoSvc, wuauserv e TrustedInstaller: **Stopped** e **Disabled**. WaaSMedicSvc: Stopped e Manual (14:04:06) |
| 1b | As 12 medidas, acima | 34, 26, 9 e nove zeros. Média 5,75. 10 das 12 abaixo de 20, então a regra foi atendida (14:08:58) |

As duas primeiras medidas (34 e 26) provavelmente são sobra de atividade logo depois do bloqueio. Não medi quem era, então isso é hipótese.

## Diagnóstico: CPU alta
Do sintoma ao processo culpado. A coluna "Obtido no ensaio" traz o que apareceu de verdade. O número do print aponta para a seção Evidências.

| Print | Comando ou ação | Esperado (saudável) | Obtido no ensaio | Leitura |
| --- | --- | --- | --- | --- |
| 2 | O comando do defeito (Parte 1) | Nenhum job | Job1 (Id 1) e Job3 (Id 3), ambos **Running** (14:13:46) | Há dois jobs rodando em segundo plano |
| 3 | 4 medidas de CPU, uma a cada 3 s: `1..4 \| ForEach-Object { (Get-CimInstance Win32_Processor \| Measure-Object LoadPercentage -Average).Average; Start-Sleep 3 }; Get-Date` | Abaixo de 20 | 100, 100, 100 e 100 (14:18:25) | A CPU está saturada |
| 4 | Gerenciador de Tarefas (Ctrl+Shift+Esc) → aba Processos → clicar no cabeçalho CPU | CPU total baixa e nenhum processo acima de uns 10% | CPU total **100%**. Windows PowerShell (3) com **92,4%** (14:19) | O grupo do PowerShell usa quase tudo. O grupo estava recolhido, então os 3 processos não aparecem um a um |
| 4b | Gerenciador de Tarefas → aba Desempenho | Gráfico baixo | CPU 100%, 2 processadores virtuais, memória em 40% (1,6 de 4,0 GB) (14:20) | Os 2 processadores estão cheios. A memória está normal |
| 5 | Variação de CPU em 5 s (comando abaixo) | Valores baixos e espalhados | `powershell` 6388 = **6,2**. `powershell` 4140 = **6,1**. `dwm` = 0,3. `conhost` = 0,1. Dois `svchost` = 0 (14:22:12) | Dois processos `powershell`, um por processador, e nada mais pesa |
| 6 | Itens de inicialização (comando abaixo) | Só programas conhecidos | 5 itens: OneDriveSetup (2 vezes), OneDrive, SecurityHealth e VBoxTray (13:44:29) | Nada suspeito. Não é problema de inicialização |

O comando do print 5:

```
$a=@{}; Get-Process | ForEach-Object { if ($null -ne $_.CPU) { $a[$_.Id]=$_.CPU } }; Start-Sleep 5; Get-Process | ForEach-Object { if ($null -ne $_.CPU -and $a.ContainsKey($_.Id)) { [pscustomobject]@{Nome=$_.Name; Id=$_.Id; CPU_5s=[math]::Round([double]($_.CPU - $a[$_.Id]),1)} } } | Sort-Object CPU_5s -Descending | Select-Object -First 6 | Format-Table -AutoSize; Get-Date
```

Como ele funciona: lê o tempo de CPU de cada processo duas vezes, com 5 segundos entre as leituras, e mostra a diferença, que são os segundos de processador que cada um usou nesse intervalo. Com 2 processadores, o máximo seria uns 10. Cada job deu um pouco mais de 6 porque, com a CPU em 100%, listar os processos demora, e o intervalo real passou de 5 segundos. Um processo com um laço só não usa mais de 1 segundo de CPU por segundo, então o intervalo real foi de pelo menos 6,2 s. **Regra de bolso: nenhum valor acima de uns 10. Se aparecer, o comando está errado.**

O comando do print 6:

```
Get-CimInstance Win32_StartupCommand | Select-Object Name, Command, Location | Format-Table -AutoSize; Get-Date
```

No print 6, cobri com uma tarja preta os valores da coluna Location, porque mostravam o identificador (SID) da máquina virtual. O resto do print não foi alterado. Esse print é da primeira tentativa (13:44), antes da rodada final, e a máquina virtual é a mesma.

## Causa
Dois jobs do PowerShell (Job1 e Job3), cada um com um laço infinito, ocuparam um processador virtual inteiro cada um e levaram a CPU a 100% (prints 2 a 5). A variação de CPU apontou dois processos `powershell` (PIDs 6388 e 4140) e nada mais com carga relevante.

A ligação entre esses dois PIDs e os dois jobs é dedução: eram dois jobs, havia dois processos `powershell` no topo, e a CPU caiu logo depois do `Remove-Job`. Não tirei um print depois só para provar que esses dois PIDs sumiram.

## Solução
1. Anotar o nome e o PID do culpado (print 5: `powershell` 6388 e 4140).
2. Parar e remover os jobs: `Get-Job | Remove-Job -Force; Get-Job; Get-Date`. O `-Force` para também os jobs que ainda estão rodando. O `Get-Job` seguinte não listou nada (print 7, 14:26:24).
3. Conferir com novas medidas (ver "Verificação").

Num PC de verdade, o culpado costuma ser um programa, e não um job. Aí, no Gerenciador de Tarefas, botão direito no processo → **Finalizar tarefa**, ou `Stop-Process -Id <PID>`. **Não testado.**

## Verificação
- As 12 medidas depois do conserto: nove 0, um 7 e dois 0. Média 0,6 (print 8, 14:28:55). A primeira medida saiu uns 30 segundos depois do `Remove-Job` e já deu 0.
- Gerenciador de Tarefas às 14:30: CPU total 4%. No topo: Gerenciador de Janelas da Área de Trabalho (`dwm`) com 2,3%, o próprio Gerenciador com 1,4% e Interrupções do sistema com 0,5%. Nenhum PowerShell no topo (print 9). Na aba Desempenho, CPU em 4% (print 9b).
- Os serviços do Windows Update continuaram Stopped e Disabled (print 10, 14:31:26). A queda da CPU não veio do Windows Update.
- Resumo: **antes 5,75 (média de 12 medidas), com o defeito 100 (4 de 4), depois 0,6 (média de 12)**.

## O que apareceu antes do defeito: Windows Update (a causa natural)
Antes de provocar qualquer defeito, a máquina virtual já estava lenta por conta própria (04/11, sem job nenhum rodando). Essa é uma causa comum de "está lento" num PC de verdade: o Windows instalando atualização.
- 11:52, Configurações → Windows Update: "Atualizações disponíveis". A atualização cumulativa de outubro de 2024 (KB5044280) estava em "Instalando - 4%", com outras três na fila (print 11).
- 13:10, Gerenciador de Tarefas: o Windows Modules Installer Worker usava 38,4% de CPU, de 48% no total (cerca de 80% da carga), e 53,9 MB/s de disco (print 12).
- 13:12:50, as 12 medidas: 16, 26, 15, 51, 56, 81, 48, 37 e quatro zeros no fim (print 13). Os zeros vieram depois de eu finalizar tarefas durante a medição (pelo que lembro: Windows Modules Installer Worker, Copilot e Edge). Não anotei qual encerrei em cada momento, então não dá para atribuir a queda a um processo só.

A conduta de suporte num PC de verdade é deixar a atualização terminar (e reiniciar quando o Windows pedir) ou agendar pelo Horário ativo (Configurações → Windows Update → Opções avançadas). Não finalizar o `TiWorker`. Finalizar alivia por alguns minutos e não resolve: às 12:46 a CPU ficou calma (6 medidas abaixo de 20) e às 12:52 já estava em 74% de novo (sem print publicado). O Horário ativo e o agendamento do reinício **não foram testados**.

## E num PC de verdade
Ações do dia a dia que o laboratório não reproduziu: tirar programas da inicialização (Gerenciador de Tarefas → aba Inicializar), fechar abas e programas em excesso, reiniciar, checar se há atualização pendente e se o antivírus está fazendo uma varredura. **Não testadas.**

Lentidão com a CPU baixa: olhar também a memória e o disco (Gerenciador de Tarefas → Desempenho). O ensaio cobriu só a CPU. No print 9 o disco estava em 30% (33% no print 9b), contra 0% no print 4. Não investiguei isso, e a CPU estava normal.

## Escalonamento
Passar ao N2 se:
- a CPU continuar alta depois de encerrar o culpado, ou voltar logo em seguida;
- o culpado for um processo do sistema ou do antivírus (`svchost`, `lsass`, `MsMpEng`) que não baixa sozinho;
- o processo for desconhecido ou suspeito (possível vírus);
- a lentidão não vier da CPU (memória cheia, disco ocupado, rede) e os passos acima não resolverem.

No ensaio o culpado era conhecido e a CPU voltou ao normal, então nenhuma dessas situações foi testada.

## Prevenção
- Anotar nome e PID antes de encerrar qualquer processo.
- Tirar uma medida de referência (baseline) quando o PC está bom. Sem ela, "melhorou" é só impressão.
- Olhar os programas da inicialização de vez em quando (print 6: 5 itens, nenhum suspeito).
- Num PC de verdade, definir o Horário ativo do Windows Update, para a atualização não pegar o expediente. Não testado.
- Tirar o print antes de limpar a tela com `cls`.

## Notas do ensaio
**Tempo gasto.** Da criação do defeito (14:13:46) ao `Remove-Job` (14:26:24) passaram 12 min 38 s. Do `Remove-Job` ao último print (14:31:26), conserto e verificação levaram 5 min 02 s. A rodada final, a dos prints 1a a 10, durou 28 minutos (de 14:03 a 14:31). O chamado inteiro levou muito mais: duas sessões (a partir das 21:00 de 03/11 e das 09:22 de 04/11), cerca de 7 horas de relógio, com uma pausa de 1h30 no meio. A maior parte foi gasta com a lentidão natural do Windows Update, e não com o defeito.

**O que não funcionou.**
- Na primeira tentativa, limpei a tela com `cls` antes do print e perdi a medida de antes do defeito. Refiz a rodada do zero, com os prints 1a e 1b tirados antes de limpar a tela.
- O primeiro comando de variação de CPU estava errado: mostrou o `dwm` com 36, um valor impossível (com 2 processadores, o máximo em 5 s é uns 10). Quando a primeira leitura vinha vazia, o comando mostrava o tempo acumulado. Corrigi para só calcular quando a primeira leitura existe e refiz (print 5). Lição: desconfiar de número que não cabe na conta.
- Finalizei tarefas (Windows Modules Installer Worker, Copilot e Edge, pelo que lembro) sem anotar nome nem PID. A CPU caiu, mas não sei dizer qual processo fez a diferença.
- O primeiro bloqueio do Windows Update não valeu: às 13:30 o serviço `UsoSvc` apareceu rodando. Não sei dizer se o bloqueio se perdeu na restauração do Limpo ou se o Windows religou o serviço. Rodei o script outra vez e conferi com `Get-Service` (print 1a). Lição: conferir o bloqueio depois de qualquer restauração.
- Deixei o Gerenciador de Tarefas aberto, minimizado, durante as 12 medidas do print 8: ele não aparece na tela, mas o ícone dele continua na barra de tarefas. O previsto era fechar. Mesmo assim, 11 das 12 medidas deram 0, então ele não pesou.
- Não testei qual parte do bloqueio (a política ou os serviços) foi a que valeu. O conjunto bastou para a CPU ficar calma nas duas medidas de 12 (prints 1b e 8).

**Diferenças em relação ao previsto.**
- Referência de 12 medidas a cada 10 s, e não de 4 a cada 3 s. As 4 medidas oscilaram demais: 75, 92, 11 e 17 logo depois de ligar a máquina, e 0, 33, 27 e 93 três minutos depois (03/11, sem print publicado).
- Diagnóstico por variação de CPU em 5 s, e não por `Get-Process | Sort-Object CPU -Descending`. Esse comando mostra o tempo de CPU acumulado desde que o processo nasceu e mistura o histórico da máquina: em 03/11 o topo era o antivírus (`MsMpEng`, com 2.185 s), sem defeito nenhum (sem print publicado).
- O `| Format-Table -AutoSize` vem antes do `Get-Date` nos comandos que mostram tabela. Sem isso, a hora pode não aparecer no PowerShell 5.1.
- Os prints 1a, 3, 4b, 9b, 10 e 11 a 13 não estavam previstos. Acrescentei.
- O ajuste de laboratório (Windows Update bloqueado) não estava previsto.
- O plano do laboratório deixava em aberto se dois jobs levam a CPU a quase 100%. Com 2 processadores virtuais, levaram (print 3).

## Fontes
- [Configure Automatic Updates using Registry Editor (Microsoft)](https://technet.microsoft.com/en-us/library/dd939844.aspx): o valor `NoAutoUpdate`, com 0 para ativar e 1 para desativar as atualizações automáticas.

## Evidências
**Print 1a.** Serviços do Windows Update depois do bloqueio: Stopped e Disabled (WaaSMedicSvc em Manual).
![Print 1a: serviços do Windows Update parados e desabilitados](../prints/win-04-p01a-servicos-update-bloqueado.png)

**Print 1b.** Medida de referência: 12 medidas (34, 26, 9 e nove zeros).
![Print 1b: 12 medidas de CPU antes do defeito](../prints/win-04-p01b-baseline-12-amostras.png)

**Print 2.** Os dois jobs, Job1 e Job3, em Running.
![Print 2: tabela com Job1 e Job3 em Running](../prints/win-04-p02-defeito-2-jobs-running.png)

**Print 3.** Quatro medidas com o defeito: 100, 100, 100 e 100.
![Print 3: 4 medidas de CPU, todas em 100](../prints/win-04-p03-4-amostras-cpu-100.png)

**Print 4.** Gerenciador de Tarefas: CPU total 100% e Windows PowerShell (3) com 92,4%.
![Print 4: Gerenciador de Tarefas com o PowerShell em 92,4%](../prints/win-04-p04-gerenciador-powershell-92pct.png)

**Print 4b.** Aba Desempenho com a CPU em 100% e 2 processadores virtuais.
![Print 4b: Desempenho com a CPU em 100%](../prints/win-04-p04b-desempenho-cpu-100.png)

**Print 5.** Variação de CPU em 5 s: dois `powershell` no topo, com 6,2 e 6,1.
![Print 5: dois processos powershell no topo da variação de CPU](../prints/win-04-p05-delta-cpu-2-powershell-no-topo.png)

**Print 6.** Itens de inicialização: 5 itens, nenhum suspeito. A coluna Location está coberta com uma tarja preta.
![Print 6: itens de inicialização, com a coluna Location coberta](../prints/win-04-p06-itens-de-inicializacao.png)

**Print 7.** Depois do `Remove-Job`: o `Get-Job` não lista mais nada.
![Print 7: Remove-Job e nenhum job listado](../prints/win-04-p07-remove-job-sem-jobs.png)

**Print 8.** As 12 medidas depois do conserto: nove 0, um 7 e dois 0.
![Print 8: 12 medidas de CPU depois do conserto](../prints/win-04-p08-12-amostras-depois.png)

**Print 9.** Gerenciador de Tarefas depois do conserto: CPU total 4% e nenhum PowerShell no topo.
![Print 9: Gerenciador de Tarefas com a CPU em 4%](../prints/win-04-p09-gerenciador-depois-cpu-4pct.png)

**Print 9b.** Aba Desempenho depois do conserto: CPU em 4%, gráfico baixo.
![Print 9b: Desempenho com a CPU em 4%](../prints/win-04-p09b-desempenho-depois-cpu-4pct.png)

**Print 10.** Os serviços do Windows Update continuam Stopped e Disabled.
![Print 10: serviços do Windows Update ainda parados](../prints/win-04-p10-servicos-depois.png)

**Print 11.** Configurações → Windows Update (11:52): a atualização KB5044280 em "Instalando - 4%" e três na fila.
![Print 11: Windows Update instalando a KB5044280](../prints/win-04-p11-windows-update-instalando.png)

**Print 12.** Gerenciador de Tarefas (13:10): Windows Modules Installer Worker com 38,4% de CPU, de 48% no total.
![Print 12: Windows Modules Installer Worker no topo](../prints/win-04-p12-gerenciador-tiworker-38pct.png)

**Print 13.** 12 medidas (13:12:50): 16, 26, 15, 51, 56, 81, 48, 37 e quatro zeros no fim.
![Print 13: 12 medidas oscilando e zeros no fim](../prints/win-04-p13-12-amostras-oscilando-zeros-no-fim.png)
