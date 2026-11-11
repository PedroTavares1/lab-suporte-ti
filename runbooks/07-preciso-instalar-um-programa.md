# Runbook 07: Preciso instalar um programa

**Ambiente:** Windows 11 Enterprise Evaluation, máquina virtual no VirtualBox, a partir do snapshot "Limpo" · **Testado em:** 11/11/2026

## Sintoma
"Preciso instalar um programa e não consigo." No ensaio, o usuário comum `joao` abriu o instalador do WinRAR (programa que abre e cria arquivos compactados, como `.rar` e `.zip`) e clicou em Instalar. O Windows pediu o nome e a senha de um administrador (print 3).

## Impacto
Uma pessoa que precisa de um programa para trabalhar e não consegue instalá-lo. Urgência baixa ou média. Sobe se o programa for necessário para uma tarefa com prazo. Há risco também do outro lado: instalar sem aprovação pode trazer problema de licença ou de segurança.

## Antes de começar
Perguntar: qual programa e qual versão? Para que a pessoa precisa dele? De onde veio o instalador? Quem aprova a instalação? O programa tem licença para uso na empresa? Que mensagem apareceu quando a pessoa tentou instalar?

**Instalar só com aprovação, e anotar no chamado quem aprovou, qual programa e versão, e a que horas.** O laboratório não faz isso: não há quem aprove, então esse registro não foi testado.

**Não passar a senha de administrador ao usuário e não dar poder de administrador a ele para resolver.** Dar administrador resolve este pedido, mas deixa a pessoa instalar qualquer programa daí em diante. Isso é decisão de política, e não correção técnica. No ensaio o `joao` não recebeu esse poder em nenhum momento, e não digitei nada no UAC (print 3).

Seis termos. **Instalador** é o arquivo que instala o programa. No ensaio, `winrar-x64-723br.exe`. **UAC** (do inglês "User Account Control", Controle de Conta de Usuário) é a janela "Deseja permitir que este aplicativo faça alterações no seu dispositivo?". Para um usuário comum, ela pede o nome e a senha de um administrador (print 3). **Usuário padrão** (ou comum) é a conta sem poder de administrador, como o `joao`. **Administrador** é a conta que pode instalar programas e mudar o sistema, como a `Lab-Win11`. **winget** é o instalador de programas por linha de comando do Windows, por exemplo `winget install <programa>`. Nesta máquina ele não existe (print 5). **Instalação silenciosa** é a que roda sem janelas nem perguntas, por meio de um parâmetro. Aqui, o `/S`.

## Como o defeito foi provocado
Partindo do snapshot "Limpo", com o PowerShell aberto como administrador (`cls` antes de cada bloco).

**Parte 0: o instalador.** Na conta de administrador (`Lab-Win11`), baixei o instalador do WinRAR, versão 7.23 de 64 bits, na página de download do site oficial (`win-rar.com/start.html`). O arquivo `winrar-x64-723br` ficou na pasta Downloads, com 5.952 KB (print 1a). Depois copiei o instalador para a pasta pública, `C:\Users\Public\Downloads`, que todos os usuários do PC conseguem abrir. É de lá que o `joao` vai abri-lo (print 1):

```
Copy-Item "$HOME\Downloads\winrar*.exe" C:\Users\Public\Downloads\winrar-instalador.exe
Get-Item C:\Users\Public\Downloads\winrar-instalador.exe; Get-Date
```

O que ele faz:
- A primeira linha copia o instalador da pasta Downloads da minha conta para a pasta pública, com o nome `winrar-instalador.exe`. O `$HOME` é a pasta do usuário, `C:\Users\Lab-Win11`. O `winrar*.exe` pega o arquivo que começa com `winrar` e termina com `.exe`.
- A segunda linha mostra o arquivo copiado, com tamanho e data, e a hora.

**Parte 1: o usuário comum.** Criei o usuário `joao`, sem poder de administrador, e listei quem é administrador (print 2):

```
net user joao Senha@123 /add
net localgroup Administradores
Get-Date
```

O `net user joao Senha@123 /add` cria a conta `joao` com a senha `Senha@123`. É uma senha de laboratório, fictícia, que só existe nesta máquina virtual descartável. **Num PC de verdade, não usar senha combinada assim.** O `net localgroup Administradores` lista os membros do grupo dos administradores.

Tudo isso aconteceu só dentro da máquina virtual. Ensaio em laboratório, dados fictícios.

## Ponto de partida: antes do defeito
| Print | Comando ou ação | Obtido no ensaio |
| --- | --- | --- |
| 1a | Explorador de Arquivos → Downloads, na conta `Lab-Win11` | 1 item: `winrar-x64-723br`, Aplicativo, 5.952 KB, modificado em 11/11/2026 às 11:58 (relógio da barra: 12:00) |
| 1 | O comando da Parte 0 | `winrar-instalador.exe` na pasta pública: 6094752 bytes, gravado em 11/11/2026 às 11:58 (12:01:10) |
| 2 | O comando da Parte 1 | "Comando concluído com êxito." para o usuário. Administradores: `Administrador` e `Lab-Win11`, sem o `joao` (12:01:59) |

A hora dos prints do PowerShell vem do `Get-Date`. A dos prints de janela vem do relógio da barra de tarefas. O print 3 (UAC) não mostra relógio, porque o UAC cobre a tela: foi enviado por volta de 12:28.

## Diagnóstico: por que não instala
Do sintoma à causa. A coluna "Obtido no ensaio" traz o que apareceu de verdade. O número do print aponta para a seção Evidências. Entrei como `joao` pela tela de entrada (sair da conta `Lab-Win11` e escolher `joao`) e abri o instalador pelo Explorador de Arquivos.

| Print | Comando ou ação | Esperado | Obtido no ensaio | Leitura |
| --- | --- | --- | --- | --- |
| 3a | Conta `joao` → Explorador de Arquivos → `C:\Users\Public\Downloads` → dois cliques em `winrar-instalador` | O UAC pedindo senha de administrador logo ao abrir (o plano foi escrito para o 7-Zip) | Abriu a janela do instalador "WinRAR 7.23", com "Pasta de destino" `C:\Program Files\WinRAR` e o aviso de que é uma versão de avaliação, com 40 dias de teste. Nenhum UAC ao abrir (12:25) | Abrir o instalador não bastou para o UAC aparecer |
| 3 | Clicar em **Instalar** | O UAC | "Deseja permitir que este aplicativo faça alterações no seu dispositivo?", WinRAR, "Fornecedor verificado: win.rar GmbH", "Origem do arquivo: Disco rígido deste computador". "Para continuar, digite um nome de usuário e uma senha de administrador." A conta oferecida é a `Lab-Win11` (`DESKTOP-OFJ9UI7\Lab-Win11`), com o campo Senha vazio. Botões Sim e Não (por volta de 12:28) | O usuário comum não instala sozinho. Clicar em **Não**, sem digitar nada |
| 4 | PowerShell comum (sem administrador), na conta `joao` (comando abaixo) | O `joao` fora da lista de administradores | `desktop-ofj9ui7\joao`. Administradores: `Administrador` e `Lab-Win11` (12:31:48) | O `joao` é usuário padrão. Quem pode digitar uma senha de administrador no UAC é quem está nessa lista |
| 5 | PowerShell como administrador, na conta `Lab-Win11` (comando abaixo) | A versão do `winget` ou "não reconhecido" | "O termo 'winget' não é reconhecido como nome de cmdlet, função, arquivo de script ou programa operável." `CommandNotFoundException` (12:35:15) | Não há `winget` nesta máquina. A instalação vai pelo instalador baixado |

O comando do print 4:

```
whoami; net localgroup Administradores; Get-Date
```

O comando do print 5:

```
winget --version; Get-Date
```

## Causa
O `joao` é usuário padrão (print 4). O instalador pediu elevação, ou seja, permissão de administrador (print 3), e o UAC exige o nome e a senha de um administrador, que o `joao` não tem e não deve ter. É o comportamento esperado do Windows (ver Fontes), e não um defeito: a pessoa só não tem permissão. O instalador propõe instalar em `C:\Program Files\WinRAR` (print 3a). Não investiguei por que, no WinRAR, o UAC só apareceu ao clicar em Instalar, e não ao abrir o instalador.

## Solução
O técnico instala com a conta de administrador dele. A pessoa não recebe a senha nem o poder de administrador.
1. Entrar na conta de administrador (`Lab-Win11`) e abrir o PowerShell **como administrador**.
2. Ver se o `winget` existe (`winget --version`, print 5). Se existir, instalar por ele, por exemplo `winget install --id <id do programa> -e --accept-package-agreements --accept-source-agreements`. **Não testado**: nesta máquina o `winget` não existe.
3. Sem `winget`, fazer a instalação silenciosa com o instalador que está na pasta pública (print 6):

```
Start-Process C:\Users\Public\Downloads\winrar-instalador.exe -ArgumentList '/S' -Wait; Get-Date
```

O `Start-Process` roda o instalador. O `-ArgumentList '/S'` passa o parâmetro da instalação silenciosa. O `-Wait` faz o PowerShell esperar o instalador terminar antes de devolver o prompt. O comando voltou sem erro (12:36:27).

4. Conferir se instalou (print 7):

```
Test-Path "C:\Program Files\WinRAR\WinRAR.exe"
(Get-Item "C:\Program Files\WinRAR\WinRAR.exe").VersionInfo.ProductVersion
Get-Date
```

O primeiro comando responde `True` se o arquivo do programa existe. O segundo mostra a versão.

5. Abrir o programa pelo Iniciar, primeiro na conta de administrador e depois na do usuário (prints 8 e 9).

## Verificação
- Instalou: `True` e a versão `7.23.0` (print 7, 12:37:08).
- Abre na conta de administrador: Iniciar → WinRAR abriu, na janela "Desktop (cópia de avaliação)", em `C:\Users\Lab-Win11\Desktop` (print 8, relógio 12:43).
- Abre na conta do usuário: saí da conta, entrei como `joao` e abri o WinRAR pelo Iniciar. Abriu sem pedir senha de administrador, na janela "system32 (cópia de avaliação)", em `C:\Windows\system32` (print 9, relógio 12:45). Esse é o teste que importa: o programa é do computador, e não de uma conta.
- Resumo: o usuário comum não conseguiu instalar (UAC, print 3), o administrador instalou de forma silenciosa (prints 6 e 7) e o usuário comum usa o programa (print 9). O `joao` não recebeu poder de administrador em nenhum momento.

## E num PC de verdade
- **Aprovação registrada.** Quem pediu, quem aprovou, qual programa e versão, e quando. O laboratório não fez isso, porque não há aprovador. **Não testado.**
- **Licença.** O instalador do WinRAR se apresenta como versão de avaliação, com 40 dias de teste (print 3a), e o programa abriu como "cópia de avaliação" (prints 8 e 9). Numa empresa, conferir a licença antes de instalar. **Não testado:** não vi o que acontece depois dos 40 dias.
- **Origem do instalador.** Baixar do site do fabricante. O UAC mostrou "Fornecedor verificado: win.rar GmbH" (print 3). **Não testei** o que o UAC mostra com um instalador de origem desconhecida ou sem assinatura.
- **O instalador na pasta pública.** Para o `joao` conseguir abrir o arquivo, deixei o instalador em `C:\Users\Public\Downloads`, que todos os usuários do PC enxergam. No ensaio ele ficou lá. Apagar o instalador depois **não foi feito**.
- **Instalação por ferramenta de gerenciamento.** Em empresas, o programa costuma ser instalado por uma ferramenta de gerenciamento de software, sem o técnico passar no PC. **Não testado.**
- **Conta de administrador sem senha.** No laboratório, a conta `Lab-Win11` não tem senha, por praticidade (o UAC do print 3 mostra o campo Senha vazio). Combinei de tratar, neste runbook, as duas contas como se tivessem senha. É uma suposição: os prints mostram o que realmente apareceu. Num PC de verdade, a conta de administrador tem senha. **Não testei** se o UAC aceita senha vazia.

## Escalonamento
Passar ao N2 (o segundo nível do suporte) se:
- o pedido não tem aprovação, ou o programa não está na lista de programas aprovados;
- o programa é pago e não há licença;
- o instalador falha mesmo com a conta de administrador, ou o programa não abre depois de instalado;
- o UAC não aparece, ou mostra um erro diferente do pedido de credenciais;
- a pessoa precisa de poder de administrador de forma permanente (decisão de política, de quem responde pela área).

Nenhuma dessas situações aconteceu no ensaio, então não foram testadas.

## Prevenção
- Manter o usuário como padrão. A Microsoft recomenda que a conta principal da pessoa seja de usuário padrão (ver Fontes).
- Ter uma lista de programas aprovados e saber quem aprova cada pedido. Não testada.
- Registrar no chamado quem aprovou, o programa e a versão.
- Conferir a licença antes de instalar. O instalador do ensaio era uma avaliação de 40 dias (print 3a).
- Baixar só do site do fabricante e olhar o "Fornecedor" no UAC (print 3).
- Não passar a senha de administrador ao usuário. No ensaio, nada foi digitado no UAC (print 3).
- Apagar o instalador da pasta pública depois de instalar. Não feito no ensaio.

## Notas do ensaio
**Tempo gasto.** Do primeiro comando (print 1, 12:01:10) ao último print (print 9, 12:45 no relógio da barra) passaram cerca de 44 minutos. Contando o download do instalador (arquivo gravado às 11:58), cerca de 47 minutos. Os prints de janela não têm segundos, então o tempo é aproximado. A tentativa anterior, antes de restaurar o "Limpo", não tem print nem hora.

**O que não funcionou.**
- Antes do ensaio, instalei o WinRAR direto, sem seguir o roteiro (sem print). Como o teste precisa de um computador sem o programa, restaurei o snapshot "Limpo" e baixei o instalador de novo.
- Ao abrir o instalador na conta `joao`, o UAC não apareceu (print 3a). Ele apareceu depois que cliquei em Instalar. O primeiro UAC eu não fotografei: o print 3 é de uma segunda vez.
- Na conta `Lab-Win11`, ao procurar "WinRAR" no Iniciar, abri o instalador por engano (print 8a). A busca também lista os arquivos `.exe` baixados, `winrar-x64-723br.exe` e `winrar-instalador.exe`, junto com o programa (print 8b). Fechei o instalador sem instalar e abri o programa pelo primeiro resultado, "WinRAR", em "Melhor correspondência" (print 8). Que a busca causou o engano é a explicação mais provável, mas não confirmei.

**Diferenças em relação ao previsto.**
- Programa: o plano usava o 7-Zip, e usei o WinRAR por escolha minha. Troquei o nome do instalador para `winrar-instalador.exe` e o arquivo da verificação para `C:\Program Files\WinRAR\WinRAR.exe` (o plano verificava o `7zFM.exe`).
- UAC: o plano previa o UAC logo ao abrir o instalador. No WinRAR ele só apareceu ao clicar em Instalar.
- Sem `winget`: o passo 10 do plano (listar as atualizações com `winget upgrade`) só valia se o `winget` existisse, e não foi feito.
- O `Get-Date` do print 1 saiu em formato de lista (DisplayHint, Date, Day...), porque o comando anterior (`Get-Item`) não tinha `| Format-Table -AutoSize` (ver o runbook 06). A hora está na linha DateTime (12:01:10).
- Os prints 1a, 3a, 8a e 8b não estavam previstos. Acrescentei.

**Prints que não publiquei.** Três: a tela de entrada com a conta `joao` (12:12), um segundo print do UAC (por volta de 12:31, a mesma janela do print 3, ainda aberta) e a busca do WinRAR na conta `joao` (12:44), que listou só o aplicativo, sem os instaladores. O primeiro só mostra o nome da conta, o segundo repete o print 3, e o terceiro não era preciso para explicar o engano.

**O que não conferi.**
- O parâmetro `/S` na documentação oficial do WinRAR. Usei fontes de terceiros (ver Fontes). No ensaio funcionou: o comando voltou sem erro e o programa passou a existir (prints 6 e 7).
- Por que o UAC só apareceu ao clicar em Instalar, e por que o `winget` não existe nesta máquina.
- Se o UAC aceitaria uma senha vazia ou errada: não digitei nada nele.
- A versão do Windows. O canto do print 3 mostra "Windows 11 Enterprise Evaluation" e "22000.co_release.210604-1628" (o texto está cortado), mas não conferi o número completo da versão (o README diz 10.0.22000.194).

## Fontes
- [How User Account Control works (Microsoft Learn)](https://learn.microsoft.com/windows/security/identity-protection/user-account-control/how-user-account-control-works): quando um usuário padrão tenta uma tarefa administrativa, o UAC pede credenciais, e elas precisam ser de uma conta de administrador local. O pedido aparece na área de trabalho segura. A Microsoft recomenda que a conta principal do usuário seja padrão.
- [Use the winget tool to install and manage applications (Microsoft Learn)](https://learn.microsoft.com/windows/package-manager/winget): o `winget` é uma ferramenta de linha de comando para instalar e gerenciar programas, e vem no App Installer, que se obtém na Microsoft Store. Não investiguei por que ele não existe na máquina do ensaio (print 5).
- [WinRAR Silent Install (Silent Install HQ)](https://silentinstallhq.com/winrar-silent-install-how-to-guide/): site de terceiros, não oficial. Indica o `/S` para a instalação silenciosa do instalador de 64 bits. O [wingetly.io](https://wingetly.io/apps/rarlab/win-rar/silent-install), também de terceiros, lista o `-s1`. No ensaio só testei o `/S`.

## Evidências
**Print 1a.** Explorador de Arquivos: a pasta Downloads da conta `Lab-Win11`, com o instalador baixado.
![Print 1a: Explorador de Arquivos com o instalador winrar-x64-723br na pasta Downloads](../prints/win-07-p01a-downloads-instalador-baixado.png)

**Print 1.** O instalador copiado para a pasta pública, com tamanho e data (12:01:10).
![Print 1: PowerShell com o winrar-instalador.exe na pasta pública](../prints/win-07-p01-copia-instalador-pasta-publica.png)

**Print 2.** O usuário `joao` criado e a lista de administradores, sem ele (12:01:59).
![Print 2: PowerShell com o usuário joao criado e os administradores Administrador e Lab-Win11](../prints/win-07-p02-usuario-joao-criado-e-administradores.png)

**Print 3a.** Conta `joao`: o instalador do WinRAR abriu sem o UAC.
![Print 3a: janela do instalador do WinRAR aberta na conta joao, antes do UAC](../prints/win-07-p03a-joao-instalador-abre-antes-do-uac.png)

**Print 3.** O UAC pedindo nome de usuário e senha de administrador, depois de clicar em Instalar.
![Print 3: UAC do WinRAR pedindo senha de administrador](../prints/win-07-p03-uac-pede-senha-de-administrador.png)

**Print 4.** Conta `joao`: `whoami` mostra o `joao`, e a lista de administradores não o inclui (12:31:48).
![Print 4: PowerShell na conta joao com whoami e a lista de administradores](../prints/win-07-p04-joao-whoami-e-administradores.png)

**Print 5.** Conta `Lab-Win11`: o `winget` não é reconhecido (12:35:15).
![Print 5: PowerShell como administrador com o erro de comando winget não reconhecido](../prints/win-07-p05-winget-nao-reconhecido.png)

**Print 6.** Instalação silenciosa: o comando voltou sem erro (12:36:27).
![Print 6: PowerShell como administrador com a instalação silenciosa concluída](../prints/win-07-p06-instalacao-silenciosa-concluida.png)

**Print 7.** `Test-Path` com `True` e a versão `7.23.0` (12:37:08).
![Print 7: PowerShell com True e a versão 7.23.0 do WinRAR](../prints/win-07-p07-winrar-instalado-true-7-23-0.png)

**Print 8a.** Conta `Lab-Win11`: o instalador aberto por engano, que fechei sem instalar.
![Print 8a: janela do instalador do WinRAR aberta por engano na conta Lab-Win11](../prints/win-07-p08a-instalador-aberto-por-engano-no-lab.png)

**Print 8b.** A busca "WinRAR" no Iniciar: o aplicativo e, abaixo, os dois instaladores.
![Print 8b: busca do Iniciar com o aplicativo WinRAR e os dois arquivos de instalador](../prints/win-07-p08b-busca-winrar-lista-instaladores.png)

**Print 8.** O WinRAR aberto na conta `Lab-Win11`, como cópia de avaliação.
![Print 8: WinRAR aberto na conta Lab-Win11](../prints/win-07-p08-winrar-aberto-no-lab.png)

**Print 9.** O WinRAR aberto na conta `joao`, sem pedir senha de administrador.
![Print 9: WinRAR aberto na conta joao](../prints/win-07-p09-winrar-aberto-no-joao.png)
