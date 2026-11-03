# Runbook 01: Computador sem internet

**Ambiente:** Windows 11 Enterprise Evaluation (10.0.22000.194), máquina virtual no VirtualBox, rede NAT · **Testado em:** 03/11/2026

## Sintoma
"Estou sem internet." O navegador não abre nenhum site.

## Impacto
Uma pessoa sem e-mail e sem sistemas na nuvem. Urgência média; alta se ela atende clientes.

## Antes de começar
Confirmar o nome do usuário e da máquina. Perguntar: mais alguém está sem internet? Desde quando? Mudou algo (cabo, Wi-Fi, atualização)?

## Como os defeitos foram provocados
O ensaio tem dois defeitos, um depois do outro, a partir do snapshot "Limpo":

- **A, cabo desconectado:** com a máquina ligada, a placa de rede virtual foi desconectada no VirtualBox (Dispositivos → Rede → desmarcar "Conectar Placa de Rede").
- **B, DNS errado:** com a rede já reconectada, o servidor DNS foi trocado por um endereço que não existe na rede: `Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "10.255.255.1"`, seguido de `ipconfig /flushdns`.

Ensaio em laboratório, dados fictícios.

## Ponto de partida: rede saudável
Antes de provocar qualquer defeito, medi a máquina funcionando (prints 1 e 2). Esses valores são a referência.

| Comando | Obtido no ensaio |
| --- | --- |
| `ipconfig /all` | Adaptador Intel PRO/1000 MT, DHCP habilitado, IPv4 10.0.2.15, gateway 10.0.2.2, servidores DNS 8.8.8.8 e 8.8.4.4 |
| `ping 8.8.8.8` | 4 respostas, de 49 a 51 ms, 0% de perda |
| `nslookup google.com` | Servidor dns.google (8.8.8.8); google.com resolvido para 142.250.78.110 e um endereço IPv6 |

## Diagnóstico A: cabo desconectado
Do mais perto ao mais longe, no PowerShell como administrador. A coluna "Obtido no ensaio" traz o que apareceu de verdade. O número do print aponta para a seção Evidências.

| Print | Comando | Esperado (rede saudável) | Obtido no ensaio | Leitura |
| --- | --- | --- | --- | --- |
| 4 | `ipconfig /all` | Adaptador com IPv4 e gateway | "Estado da mídia: mídia desconectada" no adaptador Intel PRO/1000 MT; nenhum IPv4, gateway ou DNS listado | O Windows não detecta cabo. O defeito está no cabo ou na placa, antes de qualquer configuração |
| 5 | `Get-NetAdapter` | Status "Up" | Status "Disconnected", velocidade 0 bps | Confirma: a placa está sem link |
| 5 | `ping 127.0.0.1` | Respostas | 4 respostas, menos de 1 ms, 0% de perda | Só o software de rede do próprio Windows funciona. Não testa cabo nem placa: respondeu com o cabo desconectado |
| 6 | `ping 10.0.2.2` (gateway) | Respostas do 10.0.2.2 | 4 vezes "Resposta de 10.0.2.15: Host de destino inacessível." | Quem respondeu foi o próprio PC (10.0.2.15): ele não achou caminho até o gateway. Defeito local |
| 6 | `ping 8.8.8.8` | Respostas do 8.8.8.8 | Igual ao gateway: 4 vezes "Host de destino inacessível", vindas de 10.0.2.15 | Sem caminho para sair. O defeito é local, não da internet |
| 7 | `nslookup google.com` | Devolve endereços | "Os servidores padrão não estão disponíveis", "Servidor: UnKnown", "Address: 127.0.0.1", "No response from server" | Sem rede, o PC ficou sem servidor DNS. Consequência da falta de rede, não defeito do DNS |

Atenção ao print 6: nos dois pings o resumo mostrou "Recebidos = 4, Perdidos = 0 (0% de perda)", mas nenhuma resposta veio do destino. O Windows conta o aviso "inacessível" como pacote recebido. Leia as linhas, não só o resumo.

## Diagnóstico B: DNS errado
Aqui a rede funciona e só o catálogo de nomes está errado. O usuário diz "sem internet", mas por número ainda funciona.

| Print | Comando | Esperado (rede saudável) | Obtido no ensaio | Leitura |
| --- | --- | --- | --- | --- |
| 12 | `Get-DnsClientServerAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4` | DNS 8.8.8.8 e 8.8.4.4 | 10.255.255.1 | O PC aponta para um servidor DNS errado |
| 12 | `ping 8.8.8.8` | Respostas | 4 respostas, de 49 a 51 ms, 0% de perda | A rede e a saída para a internet funcionam, por número |
| 12 | `ping google.com` | Respostas | "A solicitação ping não pôde encontrar o host google.com. Verifique o nome e tente novamente." | O nome não vira endereço. Aponta para o DNS, não para a rede |
| 13 | `nslookup google.com` | Devolve endereços | "DNS request timed out" cinco vezes; "Servidor: UnKnown", "Address: 10.255.255.1"; "O tempo limite da solicitação para UnKnown expirou" | O servidor DNS configurado não responde |

**Como separar os dois defeitos:** se o `ping 8.8.8.8` responde e o `ping google.com` falha, o problema é o DNS (defeito B). Se a mídia está desconectada e nem o gateway responde, o problema é o cabo ou a placa (defeito A).

## Causa
- **A:** placa de rede virtual desconectada (cabo virtual removido). O print 4 mostrou "mídia desconectada".
- **B:** servidor DNS configurado com um endereço que não responde (10.255.255.1). O print 12 mostrou o endereço errado, e o print 13 mostrou o tempo esgotado.

## Solução
- **A:** reconectar a placa de rede virtual: VirtualBox → Dispositivos → Rede → Conectar Placa de Rede (print 8).
- **B:** devolver o DNS ao automático e limpar o cache de nomes:
  1. `Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses`
  2. `ipconfig /flushdns`

## Verificação
- **A:** `Get-NetAdapter` mostrou "Up" a 1 Gbps e o IPv4 10.0.2.15 voltou (print 9). O `ping google.com` resolveu o nome para 172.217.162.238 e teve 4 respostas, de 50 a 51 ms, 0% de perda (print 10). O site google.com abriu no navegador da máquina (print 11).
- **B:** o DNS voltou para 8.8.8.8 e 8.8.4.4. O `nslookup` devolveu endereços do google.com, e o `ping google.com` resolveu o nome para 142.250.78.110 e teve 4 respostas, de 49 a 52 ms, 0% de perda (print 14).

## Escalonamento
Se a mídia aparece conectada e o gateway (10.0.2.2) não responde, passar ao N2 (quem cuida da rede ou da operadora), levando os resultados dos diagnósticos e a hora em que o problema começou. Se o DNS está no automático e o nome continua sem resolver, levar também os prints do `nslookup`.

## Prevenção
Etiquetar cabos e registrar na ficha da máquina o gateway e o DNS esperados.

## Notas do ensaio
**Tempo gasto.** Os 14 prints do ensaio final levaram 11 minutos (de 11:26 a 11:36). Contando a primeira tentativa, descartada, e os ajustes do ambiente, foram cerca de 1 h 20 min no total (do primeiro ao último print, de 10:16 a 11:36).

**O que não funcionou no começo.**
- Na primeira tentativa, parte dos prints saiu sem o comando visível, porque a saída rolou para cima. Refiz o chamado inteiro a partir do snapshot "Limpo".
- Copiar e colar entre o PC e a máquina virtual só passou a funcionar depois de instalar as Adições de Convidado, os drivers do VirtualBox. O snapshot "Limpo" original não tinha esses drivers, então criei um novo "Limpo" com eles instalados.
- A resolução da máquina ficou fixa em 1024 × 768 (o campo estava bloqueado nas configurações do Windows). Para os comandos mais longos caberem na tela, usei escala de 100% e a janela do PowerShell maximizada.

**Diferenças em relação ao previsto.**
- O DNS automático da máquina é 8.8.8.8 e 8.8.4.4, e não o 10.0.2.3 que eu esperava. Depois do `-ResetServerAddresses` ele voltou aos mesmos dois endereços, o que indica que vêm do DHCP.
- Nos blocos que juntam `Get-NetAdapter` ou `Get-DnsClientServerAddress` com `Get-Date`, o PowerShell mostra a data em lista. A hora está na linha DateTime (prints 5, 9, 12 e 14).
- O `nslookup` e o `ping` mostraram endereços diferentes para google.com no print 14 (172.217.162.238 e 142.250.78.110). Isso é esperado: o Google responde com vários endereços.

## Evidências
**Print 1.** `ipconfig /all` com a rede saudável.
![Print 1: ipconfig /all com a rede saudável](../prints/win-01-p01-ipconfig-rede-saudavel.png)

**Print 2.** `ping 8.8.8.8` e `nslookup` com a internet saudável.
![Print 2: ping 8.8.8.8 e nslookup com a internet saudável](../prints/win-01-p02-ping-e-nslookup-saudaveis.png)

**Print 3.** Menu do VirtualBox com a placa de rede desmarcada (cabo virtual removido).
![Print 3: menu do VirtualBox com a placa de rede desmarcada](../prints/win-01-p03-menu-placa-de-rede-desmarcada.png)

**Print 4.** `ipconfig /all` com a mídia desconectada.
![Print 4: ipconfig /all com a mídia desconectada](../prints/win-01-p04-ipconfig-midia-desconectada.png)

**Print 5.** `Get-NetAdapter` com status Disconnected e `ping 127.0.0.1` respondendo.
![Print 5: Get-NetAdapter Disconnected e ping 127.0.0.1](../prints/win-01-p05-get-netadapter-disconnected-e-loopback.png)

**Print 6.** `ping` ao gateway e ao 8.8.8.8 com "Host de destino inacessível".
![Print 6: ping ao gateway e ao 8.8.8.8 inacessíveis](../prints/win-01-p06-ping-gateway-e-internet-inacessiveis.png)

**Print 7.** `nslookup` sem resposta.
![Print 7: nslookup sem resposta](../prints/win-01-p07-nslookup-sem-resposta.png)

**Print 8.** Menu do VirtualBox com a placa de rede marcada de novo.
![Print 8: menu do VirtualBox com a placa de rede marcada](../prints/win-01-p08-menu-placa-de-rede-marcada.png)

**Print 9.** `Get-NetAdapter` com status Up e o IPv4 de volta.
![Print 9: Get-NetAdapter Up e IPv4 de volta](../prints/win-01-p09-get-netadapter-up-ipv4-de-volta.png)

**Print 10.** `ping google.com` por nome, com respostas.
![Print 10: ping google.com por nome](../prints/win-01-p10-ping-google-por-nome.png)

**Print 11.** O site abrindo no navegador da máquina virtual.
![Print 11: site abrindo no navegador](../prints/win-01-p11-site-abre-no-navegador.png)

**Print 12.** DNS errado: o ping por número responde e o ping por nome falha.
![Print 12: DNS errado, ping por número responde e por nome falha](../prints/win-01-p12-dns-errado-ping-por-nome-falha.png)

**Print 13.** `nslookup` com tempo esgotado no DNS errado.
![Print 13: nslookup com tempo esgotado no DNS errado](../prints/win-01-p13-nslookup-timeout-dns-errado.png)

**Print 14.** DNS automático restaurado e o nome resolvendo.
![Print 14: DNS automático restaurado e nome resolvendo](../prints/win-01-p14-dns-automatico-restaurado.png)
