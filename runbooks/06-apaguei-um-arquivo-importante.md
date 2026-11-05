# Runbook 06: Apaguei um arquivo importante

**Ambiente:** Windows 11 Enterprise Evaluation, máquina virtual no VirtualBox, a partir do snapshot "Limpo" · **Testado em:** 05/11/2026

## Sintoma
"Apaguei um arquivo importante e preciso dele de volta." No ensaio foram dois acidentes. O `contrato3` foi apagado e ficou na Lixeira (prints 2a e 3). O `contrato5` foi apagado e a Lixeira foi esvaziada: ele sumiu da pasta e da Lixeira (prints 4c e 5).

## Impacto
Uma pessoa sem um arquivo de que precisa para trabalhar. Urgência média. Alta se for o único exemplar, se tiver prazo (um contrato, uma prestação de contas) ou se for de um cliente.

## Antes de começar
Perguntar: qual o nome do arquivo? Em que pasta ele estava? Quando sumiu? A Lixeira já foi esvaziada? Existe backup (disco externo, pasta de rede, nuvem)?

**Restaurar só a pedido do dono do arquivo, e anotar no chamado o que foi restaurado, de onde e a que horas.**

**Não rodar o backup de novo antes de achar o arquivo.** Um backup que espelha a pasta (`/MIR`) apaga, no backup, o que não existe mais na pasta original. Depois do acidente, isso apagaria a única cópia (prints 6 e 7).

Seis termos. **Lixeira** é a pasta especial onde o Windows guarda o que a pessoa apaga pelo Explorador de Arquivos, até alguém esvaziá-la. No ensaio, o `contrato3` ficou lá, com a pasta de origem e a hora da exclusão (print 3). **Backup** é uma cópia feita antes do acidente, em outro lugar. No ensaio, `C:\Backup\Cliente` guarda uma cópia dos 5 contratos (print 2). **Robocopy** (do inglês "Robust File Copy") é o comando do Windows que copia pastas: `robocopy origem destino`. **`/L`** é a opção do robocopy que só lista o que ele faria, sem copiar nem apagar nada. **`/MIR`** (do inglês "mirror", espelho) é a opção que deixa o destino igual à origem, o que inclui apagar no destino o que não existe mais na origem. **Hash** é a impressão digital de um arquivo: dois arquivos com o mesmo conteúdo dão o mesmo valor, e trocar uma letra muda o valor inteiro. O algoritmo usado aqui é o SHA-256 (do inglês "Secure Hash Algorithm", de 256 bits).

## Como o defeito foi provocado
**Parte 0: os arquivos e o backup.** Em um PowerShell como administrador, criei a pasta `Cliente` dentro de Documentos, com 5 arquivos de texto: `contrato1.txt` a `contrato5.txt`. Cada um tem uma linha, "Contrato de teste" e o número, e 21 bytes (print 1). Conferi no Explorador de Arquivos (prints 1a e 1b).

```
New-Item -ItemType Directory -Path C:\Users\Lab-Win11\Documents\Cliente -Force | Out-Null
1..5 | ForEach-Object { "Contrato de teste $_" | Set-Content "C:\Users\Lab-Win11\Documents\Cliente\contrato$_.txt" }
Get-ChildItem C:\Users\Lab-Win11\Documents\Cliente | Format-Table -AutoSize; Get-Date
```

O que ele faz:
- A primeira linha cria a pasta `Cliente` em Documentos. O `-Force` não reclama se ela já existir, e o `Out-Null` esconde a resposta.
- A segunda linha repete o bloco para 1, 2, 3, 4 e 5. O `$_` é o número da vez: cada volta grava "Contrato de teste" e o número em `contrato<número>.txt`.
- A terceira linha lista a pasta e mostra a hora.

Depois, o backup, com o robocopy (print 2):

```
robocopy C:\Users\Lab-Win11\Documents\Cliente C:\Backup\Cliente /E /R:1 /W:1
Get-Date
```

O `/E` copia também as subpastas. O `/R:1 /W:1` manda tentar de novo só uma vez, esperando 1 segundo, se um arquivo falhar. O backup ficou no mesmo disco, em `C:\Backup\Cliente`, só porque é um ensaio. **Num PC de verdade, o backup fica em outro disco, na rede ou na nuvem:** com as duas cópias no mesmo disco, se o disco quebrar, as duas vão juntas.

**Parte 1: o primeiro acidente (caso A).** Apaguei o `contrato3` no Explorador de Arquivos (sem print da ação). A pasta ficou com 4 itens (print 2a) e o arquivo foi para a Lixeira (print 3).

**Parte 2: o segundo acidente (caso B).** Com o `contrato3` já restaurado (ver "Solução", caso A), a pasta tinha 5 itens de novo. Antes de apagar o `contrato5`, calculei o hash dele, para comparar no fim (print 4b):

```
Get-FileHash C:\Users\Lab-Win11\Documents\Cliente\contrato5.txt | Format-List Algorithm, Hash, Path
Get-Date
```

Depois apaguei o `contrato5` no Explorador (4 itens, print 4c) e esvaziei a Lixeira (print 5).

Tudo isso aconteceu só dentro da máquina virtual. Ensaio em laboratório, dados fictícios.

## Ponto de partida: antes do defeito
| Print | Comando ou ação | Obtido no ensaio |
| --- | --- | --- |
| 1 | O comando da Parte 0 (a última linha lista a pasta) | 5 arquivos, `contrato1.txt` a `contrato5.txt`, de 21 bytes, gravados em 05/11/2026 às 16:21 (16:21:35) |
| 1a | Explorador de Arquivos → Documentos → Cliente | 5 itens, de `contrato1` a `contrato5`, com 1 KB cada (16:22) |
| 1b | Explorador de Arquivos → Documentos | 1 item: a pasta Cliente (16:23) |
| 2 | O comando do backup (Parte 0) | "Novo Dir" e 5 "Novo Arquivo". Copiada: 5 arquivos e 105 bytes. FALHA: 0 (16:27:38) |
| 4b | `Get-FileHash` do `contrato5.txt` original (Parte 2) | SHA256 `7F50B235FBAE048D4768762775A00320527BF68004B78759A700DFECD019716A` (16:50:14) |

A hora dos prints do PowerShell vem do `Get-Date`. A dos prints do Explorador vem do relógio da barra de tarefas.

O Explorador mostra `contrato1`, sem o `.txt`: ele esconde a extensão dos tipos de arquivo que conhece. O PowerShell mostra o nome completo. O print 1 foi tirado às 16:24 (relógio da barra de tarefas), depois dos prints 1a e 1b, mas o comando rodou às 16:21:35. O print 4b é de depois do primeiro acidente e antes do segundo.

## Diagnóstico: onde está o arquivo
Do sintoma ao lugar onde o arquivo ainda existe. A coluna "Obtido no ensaio" traz o que apareceu de verdade. O número do print aponta para a seção Evidências. A ordem é fixa: **olhar primeiro a Lixeira, depois o backup.**

### Caso A: o arquivo está na Lixeira (`contrato3`)
| Print | Comando ou ação | Esperado | Obtido no ensaio | Leitura |
| --- | --- | --- | --- | --- |
| 2a | Explorador de Arquivos → Documentos → Cliente | 5 itens (nada apagado) | 4 itens: `contrato1`, `contrato2`, `contrato4` e `contrato5` (16:36) | O `contrato3` saiu da pasta |
| 3 | Lixeira: `explorer shell:RecycleBinFolder` | O arquivo apagado aparece aqui | 1 item: `contrato3`. Local Original: `C:\Users\Lab-Win11\Documents\Cliente`. Data de Exclusão: 05/11/2026 16:36 (16:37) | O arquivo está na Lixeira, e o Local Original diz para onde ele volta |

### Caso B: a Lixeira não tem o arquivo (`contrato5`)
| Print | Comando ou ação | Esperado | Obtido no ensaio | Leitura |
| --- | --- | --- | --- | --- |
| 4c | Explorador de Arquivos → Documentos → Cliente | 5 itens (nada apagado) | 4 itens: `contrato1` a `contrato4` (16:55) | O `contrato5` saiu da pasta |
| 5 | Lixeira | O arquivo apagado aparece aqui | "Esta pasta está vazia.", 0 itens (16:56) | O arquivo não está na Lixeira. Por aqui não dá para restaurar |
| 6 | A pasta do cliente e o backup, um depois do outro (comando abaixo) | Os mesmos arquivos nos dois | Cliente: `contrato1.txt` a `contrato4.txt`. Backup: `contrato1.txt` a `contrato5.txt`. Todos com 21 bytes (16:59:10) | O backup tem o arquivo que falta |
| 7 | `robocopy ... /MIR /L` (comando abaixo) | Nenhum arquivo EXTRA | `*Arquivo EXTRA`, 21 bytes, `contrato5.txt`. No resumo, Arquivos: Copiada 0 e Extras 1, com 21 bytes (17:01:56) | Para o `/MIR`, o `contrato5.txt` do backup é sobra e seria apagado. Com o `/L`, só listou |

O comando do print 6:

```
Get-ChildItem C:\Users\Lab-Win11\Documents\Cliente | Format-Table -AutoSize
Get-ChildItem C:\Backup\Cliente | Format-Table -AutoSize
Get-Date
```

O comando do print 7:

```
robocopy C:\Users\Lab-Win11\Documents\Cliente C:\Backup\Cliente /MIR /L
Get-Date
```

É um comando de backup com `/MIR`, o que alguém rodaria para "atualizar o backup" depois do acidente. A linha de opções do print mostra `/PURGE /MIR`: o `/MIR` traz junto o `/PURGE`, que apaga no destino o que não existe na origem. O `/L` fez dele só um ensaio, e nada foi apagado: o `contrato5.txt` continuou no backup, e foi dele que saiu a cópia nos prints 8 e 9. **Não tirar o `/L` deste comando.** O apagamento de verdade, sem o `/L`, **não foi testado**, de propósito.

## Causa
Dois apagamentos no Explorador de Arquivos. O `contrato3` ficou na Lixeira (prints 2a e 3). O `contrato5` foi apagado e a Lixeira foi esvaziada em seguida (prints 4c e 5). Dos lugares que verifiquei (a pasta do cliente, a Lixeira e o backup), o único em que o `contrato5` ainda existia era o backup (print 6). Não olhei OneDrive, pasta de rede nem "Versões anteriores".

## Solução
**Caso A: o arquivo está na Lixeira.**
1. Abrir a Lixeira (`explorer shell:RecycleBinFolder`), achar o arquivo pelo nome e anotar o Local Original (print 3).
2. Selecionar o arquivo, botão direito → **Restaurar**. Ele volta para o Local Original.
3. Conferir na pasta: o arquivo voltou e a pasta tem 5 itens (print 4). A Lixeira ficou vazia (print 4a).

**Caso B: a Lixeira não tem o arquivo, mas há backup.**
1. **Não rodar o backup de novo.** Listar a pasta do cliente e o backup e confirmar que o arquivo só existe no backup (print 6).
2. Ensaiar a cópia, com `/L` (print 8).
3. Copiar de verdade, o mesmo comando sem o `/L` (print 9).
4. Conferir a pasta e o hash (prints 9a e 10, ver "Verificação").

Comandos dos passos 2 e 3. Aqui a origem é o backup e o destino é a pasta do cliente, o contrário do backup. O último item, `contrato5.txt`, limita a cópia a esse arquivo. Primeiro, com `/L`:

```
robocopy C:\Backup\Cliente C:\Users\Lab-Win11\Documents\Cliente contrato5.txt /L
Get-Date
```

O print 8 mostrou só "Novo Arquivo", `contrato5.txt`, 21 bytes: só ele seria copiado. Depois, de verdade:

```
robocopy C:\Backup\Cliente C:\Users\Lab-Win11\Documents\Cliente contrato5.txt
Get-Date
```

Comandos do passo 4: listar a pasta (print 9a) e comparar o hash do backup com o do arquivo restaurado (print 10):

```
Get-ChildItem C:\Users\Lab-Win11\Documents\Cliente | Format-Table -AutoSize; Get-Date
```

```
Get-FileHash C:\Backup\Cliente\contrato5.txt, C:\Users\Lab-Win11\Documents\Cliente\contrato5.txt | Format-List Algorithm, Hash, Path
Get-Date
```

## Verificação
- Caso A: a pasta voltou a ter 5 itens e o `contrato3` manteve a data de modificação original, 05/11/2026 16:21 (print 4). A Lixeira ficou vazia (print 4a, 16:44).
- Caso B, a cópia: "Novo Arquivo", 21 bytes, a 100%. No resumo, Arquivos: 1 copiado e FALHA 0 (print 9, 17:06:03).
- Caso B, a pasta: 5 arquivos, `contrato1.txt` a `contrato5.txt`, de 21 bytes. O `contrato5.txt` restaurado manteve a data do original, 05/11/2026 16:21, porque o robocopy copia as datas por padrão (`/COPY:DAT`, ver Fontes) (print 9a, 17:06:19).
- Caso B, o conteúdo: o hash do `contrato5.txt` no backup e na pasta do cliente é o mesmo, `7F50B235FBAE048D4768762775A00320527BF68004B78759A700DFECD019716A` (print 10, 17:07:01). É também o hash do original, calculado antes de apagar (print 4b, 16:50:14). O arquivo restaurado é idêntico ao original.
- Conferência extra, fora da máquina virtual: o SHA-256 do conteúdo esperado do arquivo, "Contrato de teste 5" mais a quebra de linha do Windows (21 bytes no total), dá o mesmo valor.
- Resumo: dois arquivos apagados e dois recuperados. O `contrato3` voltou da Lixeira. O `contrato5` voltou do backup, com o mesmo hash do original.

## E num PC de verdade
- **Backup fora da máquina:** disco externo, pasta de rede ou nuvem. A regra 3-2-1: três cópias dos dados, em dois tipos de mídia, uma delas fora do local. **Não testada.** No ensaio houve uma só cópia extra, no mesmo disco.
- **Outros lugares onde procurar**, que o laboratório não verificou: o OneDrive (tem uma lixeira própria), a pasta de rede (o servidor pode ter cópia) e as "Versões anteriores" do Windows, se estiverem ligadas. **Não testados.**
- **Nem tudo passa pela Lixeira.** Arquivo apagado com Shift+Delete, ou apagado de um pen drive ou de uma pasta de rede, em geral não vai para a Lixeira. Nesses casos, ir direto ao backup. **Não testado.**
- **Tentativas do robocopy.** Sem `/R` e `/W`, ele usa `/R:1000000 /W:30` (prints 7 a 9): com um arquivo travado, tentaria um milhão de vezes, esperando 30 segundos entre as tentativas. No backup (print 2) usei `/R:1 /W:1`. Na restauração não usei, e o padrão apareceu no print 9. Num PC de verdade, acrescentar `/R:1 /W:1`. **Não testado com arquivo travado.**
- **Sobrescrever.** Se já existir um arquivo com o mesmo nome no destino, o robocopy pode substituí-lo. No ensaio isso não aconteceu, porque o `/L` mostrou só "Novo Arquivo". Por isso o ensaio com `/L` vem antes da cópia. **A substituição não foi testada.**

## Escalonamento
Passar ao N2 (o segundo nível do suporte) se:
- o arquivo não está na Lixeira e não há backup;
- o arquivo estava em pasta de rede ou na nuvem, onde quem cuida do servidor ou da conta pode ter outra cópia;
- o backup tem o arquivo, mas a cópia não abre ou o hash dela é diferente do original (quando existe um hash de referência);
- o robocopy mostra FALHA diferente de 0, ou o disco dá erro ao ler ou copiar;
- o arquivo é sigiloso e quem pediu não é o dono.

Nenhuma dessas situações aconteceu no ensaio, então não foram testadas.

## Prevenção
- Ter backup fora da máquina, seguindo a regra 3-2-1 (três cópias, em dois tipos de mídia, uma fora do local). Não testada.
- Testar a restauração, e não só o backup. No ensaio, o `contrato5` voltou do backup com o mesmo hash do original (prints 4b e 10).
- Depois de um acidente, não rodar o backup com `/MIR`. Para olhar o que ele faria, usar o `/L` e ler as linhas EXTRA (print 7).
- Não esvaziar a Lixeira por hábito. Ela foi o jeito mais simples de recuperar o `contrato3` (prints 3 e 4).
- Registrar no chamado o que foi restaurado, de onde e a que horas.

## Notas do ensaio
**Tempo gasto.** Do primeiro comando (print 1, 16:21:35) ao último (print 10, 17:07:01) passaram 45 min 26 s. Contando a primeira tentativa e as pausas, do primeiro print da sessão (14:25, não publicado) ao último (17:07), passaram 2 h 42 min.

**O que não funcionou.**
- Na primeira tentativa (antes das 14:55), criei os 5 arquivos em `C:\Users\Public\Documents\Cliente`, a pasta Documentos pública, e não os achei no Explorador. Ele abre a pasta Documentos do meu usuário, `C:\Users\Lab-Win11\Documents` (print 4). Refiz tudo em `C:\Users\Lab-Win11\Documents\Cliente`. Não tirei print da primeira tentativa.
- Antes de criar os arquivos, apaguei por precaução as pastas `Cliente` da pasta pública e de `C:\Backup`, para partir do zero. Não tirei print dessa limpeza.
- O primeiro print da pasta depois da Lixeira (16:37) ainda mostrava 4 itens, porque o `contrato3` ainda não tinha sido restaurado. Restaurei pela Lixeira (botão direito → Restaurar) e refiz o print às 16:41 (print 4). O de 16:37 não está publicado.
- Um print do menu do botão direito (Restaurar) saiu só da tela da máquina, sem a barra do VirtualBox. Não está publicado.

**Diferenças em relação ao previsto.**
- A pasta de teste ficou em `C:\Users\Lab-Win11\Documents\Cliente`, e não em `C:\Users\Public\Documents\Cliente`, como estava no plano.
- O robocopy mostra os rótulos em português: "Novo Dir", "Novo Arquivo" e "*Arquivo EXTRA", e as colunas Total, Copiada, Ignorada, Incompatibilidade, FALHA e Extras. O plano deixava em aberto os rótulos exatos com `/L`.
- Com `/L`, o resumo do print 8 mostra "Copiada 1" (21 bytes), mas nada foi copiado: na cópia de verdade, às 17:06:03 (print 9), o arquivo ainda saiu como "Novo Arquivo".
- A restauração virou dois blocos (prints 9 e 9a): em um só, a saída do robocopy empurraria o comando para fora da tela.
- Os prints 1a, 1b, 2a, 4a, 4b, 4c e 9a não estavam previstos. Acrescentei.
- O `| Format-Table -AutoSize` vem antes do `Get-Date` nos comandos que mostram tabela. Sem isso, a hora pode não aparecer no PowerShell 5.1.

**Prints que não publiquei.** Cinco: o Explorador com 5 itens antes de apagar o `contrato3` (16:36), o Explorador com 4 itens antes de restaurar (16:37), o menu do botão direito (só a tela da máquina), um segundo Explorador com 5 itens depois de restaurar (16:44) e o Explorador final com 5 itens (17:07). Quatro repetem o que outros prints já mostram, e o do menu não mostra a janela inteira.

**O que não conferi.** A versão do Windows, nesta rodada.

## Fontes
- [robocopy (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy): o `/MIR` equivale a `/E` mais `/PURGE`. O `/PURGE` apaga no destino o que não existe na origem. O `/L` só lista, sem copiar, apagar nem alterar datas. Os padrões são `/R:1000000`, `/W:30` e `/COPY:DAT` (dados, atributos e datas).
- [Get-FileHash (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash?view=powershell-7.6): calcula o hash de um arquivo, e o algoritmo padrão é o SHA256. A página é do PowerShell 7. No Windows PowerShell do ensaio, o print 4b mostrou `Algorithm : SHA256` sem eu ter pedido o algoritmo.

## Evidências
**Print 1.** `Get-ChildItem`: os 5 contratos, de 21 bytes (16:21:35).
![Print 1: PowerShell com a lista dos 5 contratos](../prints/win-06-p01-listagem-5-contratos.png)

**Print 1a.** Explorador de Arquivos: a pasta Cliente, com 5 itens.
![Print 1a: Explorador com a pasta Cliente e 5 itens](../prints/win-06-p01a-explorer-cliente-5-itens.png)

**Print 1b.** Explorador de Arquivos: a pasta Documentos, com a pasta Cliente.
![Print 1b: Explorador com a pasta Documentos e a pasta Cliente](../prints/win-06-p01b-explorer-documentos-pasta-cliente.png)

**Print 2.** Backup com o robocopy: 5 arquivos copiados e FALHA 0.
![Print 2: robocopy copiando os 5 arquivos para o backup](../prints/win-06-p02-backup-robocopy-5-copiados.png)

**Print 2a.** Explorador: a pasta Cliente com 4 itens, sem o `contrato3`.
![Print 2a: pasta Cliente com 4 itens, sem o contrato3](../prints/win-06-p02a-explorer-cliente-4-itens-contrato3-apagado.png)

**Print 3.** Lixeira com o `contrato3`, o Local Original e a Data de Exclusão.
![Print 3: Lixeira com o contrato3](../prints/win-06-p03-lixeira-com-contrato3.png)

**Print 4.** Explorador: a pasta Cliente com 5 itens, com o `contrato3` restaurado.
![Print 4: pasta Cliente com 5 itens depois de restaurar o contrato3](../prints/win-06-p04-explorer-cliente-5-itens-restaurado.png)

**Print 4a.** Lixeira vazia depois de restaurar.
![Print 4a: Lixeira vazia, 0 itens](../prints/win-06-p04a-lixeira-vazia-apos-restaurar.png)

**Print 4b.** Hash SHA-256 do `contrato5.txt` original, antes de apagar.
![Print 4b: hash SHA256 do contrato5 original](../prints/win-06-p04b-hash-original-contrato5.png)

**Print 4c.** Explorador: a pasta Cliente com 4 itens, sem o `contrato5`.
![Print 4c: pasta Cliente com 4 itens, sem o contrato5](../prints/win-06-p04c-explorer-cliente-4-itens-contrato5-apagado.png)

**Print 5.** Lixeira vazia depois de esvaziar.
![Print 5: Lixeira vazia depois de esvaziar](../prints/win-06-p05-lixeira-vazia-apos-esvaziar.png)

**Print 6.** A pasta do cliente com 4 arquivos e o backup com 5.
![Print 6: pasta do cliente com 4 arquivos e backup com 5](../prints/win-06-p06-diagnostico-4-no-cliente-5-no-backup.png)

**Print 7.** `robocopy /MIR /L`: o `contrato5.txt` aparece como EXTRA.
![Print 7: robocopy com MIR e L listando o contrato5 como EXTRA](../prints/win-06-p07-robocopy-mir-l-contrato5-extra.png)

**Print 8.** Ensaio da restauração, com `/L`: só o `contrato5.txt`, como Novo Arquivo.
![Print 8: robocopy com L listando o contrato5 como Novo Arquivo](../prints/win-06-p08-robocopy-ensaio-l-contrato5-novo.png)

**Print 9.** Restauração do `contrato5.txt`: 1 arquivo copiado e FALHA 0.
![Print 9: robocopy copiando o contrato5 do backup](../prints/win-06-p09-robocopy-restaura-contrato5.png)

**Print 9a.** A pasta do cliente com os 5 arquivos de novo.
![Print 9a: pasta do cliente com 5 arquivos depois da restauração](../prints/win-06-p09a-listagem-5-arquivos-restaurado.png)

**Print 10.** Hash do `contrato5.txt` no backup e na pasta do cliente: iguais.
![Print 10: hashes do backup e da pasta do cliente iguais](../prints/win-06-p10-hash-backup-igual-restaurado.png)
