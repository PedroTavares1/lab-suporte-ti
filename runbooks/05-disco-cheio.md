# Runbook 05: O disco está cheio

**Ambiente:** Windows 11 Enterprise Evaluation (10.0.22000.194), máquina virtual no VirtualBox · **Testado em:** 03/11/2026

## Sintoma
"Não consigo salvar nada." No ensaio, ao criar um arquivo de 5 MB no disco S:, o erro foi: "Erro: Espaço insuficiente no disco." (print 5). No Explorador de Arquivos, a barra do disco DADOS (S:) ficou vermelha, com "2,00 MB livre(s) de 1,98 GB" (print 6).

## Impacto
Uma pessoa sem conseguir salvar arquivos naquele disco. Urgência média. Alta se o disco cheio for o C:, porque o Windows e os programas também usam esse disco para funcionar.

## Antes de começar
Perguntar: qual disco ou pasta? Qual a mensagem exata? Desde quando? Mudou algo (download grande, backup, atualização)? Quem é o dono dos arquivos grandes?

**Só apagar com a autorização do dono dos arquivos, e anotar no chamado quem autorizou e o que foi apagado.** O comando `Remove-Item` apaga direto, sem passar pela Lixeira, então não dá para recuperar.

Num PC de verdade, o disco cheio costuma ser o C:. No ensaio, o defeito foi num segundo disco virtual de 2 GB, para não encher o disco real da máquina virtual. Três termos: **VDI** é o formato do arquivo que o VirtualBox usa como disco virtual. **Volume** é uma parte do disco já formatada, que ganha uma letra (aqui, S:). **NTFS** é o formato de arquivos padrão do Windows.

## Como o defeito foi provocado
**Parte 0: o disco extra.** No VirtualBox (fora da máquina virtual, com ela desligada), em Configurações → Armazenamento, criei um disco virtual dinâmico de 2 GB. O VirtualBox sugeriu o nome `Lab-Win11_1.vdi` (print 1a) e o disco ficou na Porta SATA 1 (print 1b). No Windows, em um PowerShell como administrador, o `Get-Disk` mostrou o disco novo sem formatação, o que o Windows chama de RAW (print 2). Depois inicializei o disco como GPT (a forma moderna de organizar um disco), criei a partição S: e formatei em NTFS com o nome DADOS (print 3):

```
Get-Disk | Where-Object PartitionStyle -eq 'RAW' | Initialize-Disk -PartitionStyle GPT -PassThru | New-Partition -DriveLetter S -UseMaximumSize | Format-Volume -FileSystem NTFS -NewFileSystemLabel "DADOS" -Confirm:$false | Format-Table -AutoSize; Get-Date
```

**O defeito.** Dois arquivos de enchimento: o primeiro com 1 GB e o segundo com o que sobrou, menos 2 MB (print 4):

```
fsutil file createnew S:\enchimento1.bin 1073741824
fsutil file createnew S:\enchimento2.bin ((Get-PSDrive S).Free - 2MB)
Get-PSDrive S | Format-Table Name, Used, Free -AutoSize; Get-Date
```

Tudo isso aconteceu só no disco S:. O disco C: não foi tocado. Ensaio em laboratório, dados fictícios.

## Ponto de partida: antes do defeito
| Print | Comando ou ação | Obtido no ensaio |
| --- | --- | --- |
| 1a | VirtualBox → Configurações → Armazenamento → disco rígido → Criar | Disco `Lab-Win11_1.vdi`, 2,00 GB de tamanho virtual e 2,00 MB de tamanho real (dinâmico), ainda "Not Attached" |
| 1b | Janela principal do VirtualBox, com a máquina desligada | Porta SATA 0: `Lab-Win11.vdi` (64,00 GB). Porta SATA 1: `Lab-Win11_1.vdi` (2,00 GB). Porta SATA 3: disco óptico vazio |
| 2 | `Get-Disk \| Format-Table -AutoSize; Get-Date` | Disco 0: 64 GB, GPT. Disco 1: 2 GB, **RAW**, Online, Healthy (20:04:45) |
| 3 | O comando de preparação, acima | Volume `S`, nome `DADOS`, NTFS, Fixed, Healthy, OK, 1,97 GB livres de 1,98 GB (20:07:32) |

## Diagnóstico: disco cheio
Do sintoma ao arquivo que ocupa o espaço. A coluna "Obtido no ensaio" traz o que apareceu de verdade. O número do print aponta para a seção Evidências.

| Print | Comando ou ação | Esperado (saudável) | Obtido no ensaio | Leitura |
| --- | --- | --- | --- | --- |
| 5 | `fsutil file createnew S:\relatorio.bin 5242880; Get-Date` | Arquivo criado | "Erro: Espaço insuficiente no disco." (20:11:36) | O disco S: não tem espaço para 5 MB |
| 6 | Explorador de Arquivos → Este Computador | Barras azuis | C: com barra azul, "37,2 GB livre(s) de 63,8 GB". S: com barra **vermelha**, "2,00 MB livre(s) de 1,98 GB" | É assim que o usuário vê. Só o S: está cheio |
| 7 | `Get-PSDrive -PSProvider FileSystem \| Format-Table -AutoSize; Get-Date` | Espaço livre em todos | C: 27,00 GB usados e 36,88 GB livres. S: 1,98 GB usados e **0,00 GB livres**. D: (o CD) sem valores (20:13:42) | O disco cheio é o S:, não o C: |
| 8 | `Get-ChildItem S:\ -Recurse -File \| Sort-Object Length -Descending \| Select-Object -First 10 FullName, Length \| Format-Table -AutoSize; Get-Date` | Poucos arquivos pequenos | `S:\enchimento1.bin` com 1.073.741.824 bytes e `S:\enchimento2.bin` com 1.035.259.904 bytes (20:14:25) | Os dois arquivos grandes ocupam quase tudo |

## Causa
O disco S: ficou sem espaço por causa de dois arquivos grandes, `enchimento1.bin` e `enchimento2.bin`, que deixaram só 2.097.152 bytes livres (prints 4, 7 e 8).

## Solução
1. Confirmar com o dono dos arquivos e anotar no chamado (ver "Antes de começar"). No ensaio, o dono fictício autorizou apagar o `enchimento1.bin`.
2. Apagar o arquivo autorizado: `Remove-Item S:\enchimento1.bin`.
3. Conferir o espaço: `Get-PSDrive S | Format-Table Name, Used, Free -AutoSize; Get-Date`. O `Free` subiu para 1.075.838.976 bytes (print 9).

## Verificação
- O `Free` subiu exatamente 1.073.741.824 bytes, que é o tamanho do arquivo apagado. A soma de `Used` e `Free` continuou 2.130.636.800, igual à do print 4. Nada além desse arquivo mudou no S: (print 9, 20:16:14).
- O arquivo que tinha falhado foi salvo: `fsutil file createnew S:\relatorio.bin 5242880` respondeu "Arquivo S:\relatorio.bin criado", e a lista do S: mostrou `enchimento2.bin` e `relatorio.bin` com 5.242.880 bytes, gravado às 20:17:12 (print 10).

## E no disco C: de verdade
Para um C: cheio, a tela para olhar é Configurações → Sistema → Armazenamento. No ensaio ela mostrou o Disco Local (C:) com 63,8 GB, dos quais 26,7 GB usados e 37,1 GB livres. Abaixo, as categorias: "Apps e recursos" com 6,43 GB e "Arquivos temporários" com 274 MB, mais o link "Mostrar mais categorias" (print 11). Na mesma tela, "Sentido de armazenamento" estava desativado, e "Recomendações de limpeza" dizia "Armazenamento otimizado, nenhuma ação é necessária".

A tela "Discos & volumes", dentro de Armazenamento, lista os discos e as partições. No ensaio: o disco 0 com a partição EFI de 100 MB e o C: de 63,8 GB, e o disco 1 com o DADOS (S:) de 1,98 GB (print 11b).

Ações que o laboratório não reproduziu, porque o C: da máquina não estava cheio: esvaziar a Lixeira, usar a Limpeza de Disco (`cleanmgr`), remover arquivos temporários e ligar o Sentido de armazenamento. **Não testadas.**

## Escalonamento
Passar ao N2 se:
- o espaço não voltar depois de apagar o que foi autorizado;
- a soma dos arquivos listados não explicar o espaço usado;
- o disco cheio for o C: e o espaço estiver em arquivos do Windows ou de programas, que não se apagam à mão.

Nenhuma dessas situações aconteceu no ensaio, então não foram testadas.

## Prevenção
- Avisar a pessoa quando a barra do disco ficar vermelha (print 6), antes de o disco encher de vez.
- Mover arquivos grandes, como vídeos e instaladores, para outro disco ou para a nuvem.
- Ligar o Sentido de armazenamento, que estava desativado no ensaio (print 11). Não testei o que ele libera.
- Registrar no chamado quem autorizou apagar e o que foi apagado.

## Notas do ensaio
**Tempo gasto.** A criação do disco extra levou de 19:33 (primeiro print das Configurações, que não está aqui) até 20:00 (relógio do meu PC, no print 1b, que foi recortado e não mostra a hora): 27 minutos, contando uma tentativa de renomear o disco que não deu certo. Do print 2 (20:04:45) ao print 10 (20:17:12) passaram 12 minutos. Com os prints extras 11, 11b e 12, o ensaio terminou às 20:32, ou seja, 59 minutos no total.

**O que não funcionou no começo.**
- Tentei renomear o disco extra e não consegui, porque o VirtualBox não tem um botão para isso. Ele ficou com o nome sugerido, `Lab-Win11_1.vdi`.
- Às 20:15:15 rodei `ls` sem caminho, para espiar o S:. O PowerShell listou a pasta atual, `C:\Windows\system32`, com milhares de linhas. Dei `cls` e segui. Para listar o S:, o certo é `Get-ChildItem S:\`. O histórico (print 12) mostra que cada comando dos passos aparece uma vez só e que o único `Remove-Item` é o do passo de apagar.

**Diferenças em relação ao previsto.**
- O disco se chama `Lab-Win11_1.vdi`, e não `disco-dados`.
- O `Initialize-Disk` funcionou nesta máquina: o disco veio Online e a formatação terminou sem erro (print 3).
- O texto do erro foi "Erro: Espaço insuficiente no disco." (print 5).
- O segundo arquivo deixou exatamente 2.097.152 bytes (2 MB) livres (print 4).
- Acrescentei `| Format-Table ...` antes do `Get-Date` nos comandos que mostram tabela. Sem isso, a hora pode não aparecer no PowerShell 5.1.

## Evidências
**Print 1a.** Seletor de discos do VirtualBox, com o disco `Lab-Win11_1.vdi` recém-criado.
![Print 1a: seletor de discos com o disco de 2 GB criado](../prints/win-05-p01a-seletor-disco-criado.png)

**Print 1b.** Janela principal do VirtualBox, com o disco de 2 GB na Porta SATA 1.
![Print 1b: VirtualBox com o disco de 2 GB na Porta SATA 1](../prints/win-05-p01b-gerenciador-disco-na-porta-sata1.png)

**Print 2.** `Get-Disk`: o disco 1, de 2 GB, está RAW.
![Print 2: Get-Disk, disco de 2 GB RAW](../prints/win-05-p02-get-disk-raw.png)

**Print 3.** Volume S: DADOS, NTFS, Healthy.
![Print 3: volume S DADOS formatado em NTFS](../prints/win-05-p03-volume-s-dados.png)

**Print 4.** Arquivos de enchimento criados e `Free` em 2.097.152 bytes.
![Print 4: enchimento criado, Free em 2 MB](../prints/win-05-p04-enchimento-free-2mb.png)

**Print 5.** "Erro: Espaço insuficiente no disco."
![Print 5: erro de espaço insuficiente](../prints/win-05-p05-erro-espaco-insuficiente.png)

**Print 6.** Explorador de Arquivos: DADOS (S:) com a barra vermelha.
![Print 6: Explorador com o S: em vermelho](../prints/win-05-p06-explorador-barra-vermelha.png)

**Print 7.** `Get-PSDrive`: C: com espaço e S: com 0,00 GB livres.
![Print 7: Get-PSDrive, C com espaço e S sem espaço](../prints/win-05-p07-get-psdrive-c-livre-s-zero.png)

**Print 8.** Os maiores arquivos do S:: os dois de enchimento.
![Print 8: os dois arquivos de enchimento no topo](../prints/win-05-p08-maiores-arquivos-enchimento.png)

**Print 9.** Depois do `Remove-Item`: `Free` em 1.075.838.976 bytes.
![Print 9: Free subiu para cerca de 1 GB](../prints/win-05-p09-remove-enchimento1-free-1gb.png)

**Print 10.** O `relatorio.bin` salvo e listado.
![Print 10: relatorio.bin criado e listado](../prints/win-05-p10-relatorio-salvo.png)

**Print 11.** Configurações → Sistema → Armazenamento, com o disco C: e as categorias.
![Print 11: Armazenamento do disco C](../prints/win-05-p11-armazenamento-categorias-c.png)

**Print 11b.** Configurações → Sistema → Armazenamento → Discos & volumes.
![Print 11b: Discos e volumes](../prints/win-05-p11b-discos-e-volumes.png)

**Print 12.** Histórico do PowerShell, com o `ls` das 20:15:15.
![Print 12: histórico do PowerShell com o ls](../prints/win-05-p12-historico-powershell-ls.png)
