# Roteiro de Laboratório - Roteamento OSPFv3 em uma rede IPv6

> [!NOTE]
> **Licença e atribuição:** este roteiro é uma obra adaptada do livro *Laboratório de IPv6: aprenda na prática usando um emulador de redes*, da **Equipe IPv6.br**, publicado pela Novatec Editora em 2015. O material original pode ser obtido em [lab.ipv6.br](http://lab.ipv6.br). Em conformidade com a licença do livro, esta adaptação é disponibilizada sob a licença [Creative Commons Atribuição-NãoComercial-CompartilhaIgual 4.0 Internacional (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.pt-br). Seu uso e redistribuição devem preservar a atribuição, limitar-se a fins não comerciais e manter esta mesma licença em eventuais obras derivadas.

Neste laboratório vamos configurar e analisar o protocolo de roteamento dinâmico **OSPFv3** em uma rede IPv6. O cenário utiliza três roteadores em uma única área OSPF e dois hosts conectados a sub-redes diferentes. Ao longo das atividades, vamos observar o estado inicial da rede, estabelecer adjacências entre os roteadores, validar as rotas aprendidas e analisar a reconvergência após a falha de um enlace.

O cenário utilizado neste laboratório é ilustrado na figura abaixo:

![Topologia do laboratório OSPFv3](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab-ospf-ipv6/images/topologia.png)

Os roteadores **n1Backbone**, **n2Backbone** e **n3Backbone** formam a área de backbone do OSPF, identificada por `0.0.0.0`. O host **n4HostA** utiliza o endereço `2001:db8:3::20/64`, enquanto o host **n5HostB** utiliza o endereço `2001:db8:4::20/64`.

> [!NOTE]
> Este roteiro foi adaptado da experiência "OSPFv3: configuração de uma única área", elaborada pela equipe do projeto IPv6.br/NIC.br. Os nomes dos nós, endereços e imagens foram mantidos para facilitar a comparação com o material original.

## Objetivos

Ao final deste laboratório, você deverá ser capaz de:

- explicar a função do OSPFv3 em uma rede IPv6;
- configurar uma única área OSPFv3;
- verificar o estado do processo OSPFv3 e suas adjacências;
- identificar rotas aprendidas dinamicamente;
- analisar a reconvergência da rede após a indisponibilidade de um enlace.

## Introdução teórica

O **OSPF** (_Open Shortest Path First_) é um protocolo de roteamento interno, ou IGP (_Interior Gateway Protocol_). Por meio dele, os roteadores trocam informações sobre as redes conhecidas e o estado de seus enlaces. Com essas informações, cada roteador constrói uma visão da topologia e utiliza o algoritmo de Dijkstra para calcular os caminhos de menor custo.

O OSPF permite organizar uma rede de forma hierárquica por meio de **áreas**. Cada área possui um identificador de 32 bits, chamado **Area ID**. A área `0.0.0.0` é a área de backbone e, em uma rede sem outras divisões, é a única área necessária.

O **OSPFv3** foi criado para operar com IPv6. No cenário deste laboratório, ele será usado para que os roteadores aprendam dinamicamente os prefixos IPv6 que não estão diretamente conectados a eles.

## Atividade 1 - Verificação da topologia e dos endereços

1. Inicie a simulação com a topologia apresentada no início deste roteiro.

2. Abra o terminal de cada nó e verifique os endereços IPv6 configurados:

```console
ip -6 address show
```

3. Confirme a presença dos seguintes nós:

- `n1Backbone`, `n2Backbone` e `n3Backbone`: roteadores que formarão a área OSPF;
- `n4HostA`: host da rede `2001:db8:3::/64`;
- `n5HostB`: host da rede `2001:db8:4::/64`.

> [!IMPORTANT]
> Registre os endereços IPv6 e as interfaces identificadas em cada nó.
>
> <textarea name="resposta_enderecos" rows="8" cols="80" placeholder="Liste os nós, as interfaces e os respectivos endereços IPv6."></textarea>

## Atividade 2 - Testes iniciais de conectividade

Antes de configurar o OSPFv3, vamos verificar quais destinos o `n4HostA` consegue alcançar.

### 2.1 Teste com um roteador diretamente conectado

No terminal do `n4HostA`, teste a conectividade com o endereço `2001:db8:3::1`, pertencente ao roteador `n2Backbone`:

```console
ping6 -c 4 2001:db8:3::1
```

O teste deve ser bem-sucedido, pois os dois nós estão na mesma sub-rede.

![Teste de conectividade entre n4HostA e n2Backbone](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab-ospf-ipv6/images/ping-host-a-roteador.png)

### 2.2 Teste entre hosts de sub-redes diferentes

Ainda no `n4HostA`, teste a conectividade com o `n5HostB`:

```console
ping6 -c 4 2001:db8:4::20
```

Neste momento, o teste deve falhar. Embora os roteadores estejam fisicamente interligados, eles ainda conhecem apenas suas redes diretamente conectadas.

![Falha no teste de conectividade entre n4HostA e n5HostB](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab-ospf-ipv6/images/ping-host-a-host-b-falha.png)

### 2.3 Inspeção da tabela de roteamento

No terminal do `n2Backbone`, exiba as rotas IPv6 conhecidas pelo sistema operacional:

```console
ip -6 route show
```

![Tabela de rotas inicial do n2Backbone](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab-ospf-ipv6/images/rotas-iniciais-r2.png)

Repita o comando em `n1Backbone` e `n3Backbone`. Observe que cada roteador conhece somente os prefixos diretamente conectados a ele. Sem uma rota para `2001:db8:4::/64`, o `n2Backbone` não consegue encaminhar o tráfego do `n4HostA` ao `n5HostB`.

> [!IMPORTANT]
> Por que o primeiro teste de ping funciona, enquanto o segundo falha?
>
> <input type="radio" name="resposta_conectividade_inicial" id="conectividade-inicial-1" value="ospf-bloqueia-icmp" /> <label for="conectividade-inicial-1">O OSPFv3 bloqueia pacotes ICMPv6 enquanto ainda não existem adjacências entre todos os roteadores.</label><br>
> <input type="radio" name="resposta_conectividade_inicial" id="conectividade-inicial-2" value="destino-diretamente-conectado" /> <label for="conectividade-inicial-2">O primeiro destino está em uma rede diretamente conectada, enquanto a rota para a rede do segundo destino ainda não é conhecida.</label><br>
> <input type="radio" name="resposta_conectividade_inicial" id="conectividade-inicial-3" value="ipv6-somente-local" /> <label for="conectividade-inicial-3">Endereços IPv6 só podem ser usados para comunicação entre nós da mesma sub-rede.</label><br>
> <input type="radio" name="resposta_conectividade_inicial" id="conectividade-inicial-4" value="host-b-desligado" /> <label for="conectividade-inicial-4">O segundo teste falha necessariamente porque o n5HostB está desligado.</label><br>

## Atividade 3 - Configuração do OSPFv3

Nesta atividade vamos ativar o OSPFv3 nos três roteadores, associar suas interfaces à área `0.0.0.0` e anunciar as redes diretamente conectadas.

> [!NOTE]
> O comando `vtysh` abre a interface de configuração da suíte de roteamento Quagga. Dependendo da imagem usada no laboratório, a implementação pode ser o FRRouting, que oferece uma interface de comandos compatível para esta atividade.

### 3.1 Configuração do n1Backbone

No terminal do `n1Backbone`, execute:

```console
vtysh
configure terminal
router ospf6
router-id 1.1.1.1
interface eth0 area 0.0.0.0
interface eth1 area 0.0.0.0
redistribute connected
exit
exit
exit
```

![Configuração OSPFv3 do n1Backbone](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab-ospf-ipv6/images/configuracao-ospfv3-r1.png)

### 3.2 Configuração do n2Backbone

No terminal do `n2Backbone`, execute:

```console
vtysh
configure terminal
router ospf6
router-id 2.2.2.2
interface eth0 area 0.0.0.0
interface eth1 area 0.0.0.0
redistribute connected
exit
exit
exit
```

![Configuração OSPFv3 do n2Backbone](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab-ospf-ipv6/images/configuracao-ospfv3-r2.png)

### 3.3 Configuração do n3Backbone

No terminal do `n3Backbone`, execute:

```console
vtysh
configure terminal
router ospf6
router-id 3.3.3.3
interface eth0 area 0.0.0.0
interface eth1 area 0.0.0.0
redistribute connected
exit
exit
exit
```

![Configuração OSPFv3 do n3Backbone](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab-ospf-ipv6/images/configuracao-ospfv3-r3.png)

Os principais comandos utilizados têm as seguintes funções:

| Comando | Função |
| --- | --- |
| `router ospf6` | Inicia ou acessa o processo OSPFv3. |
| `router-id A.B.C.D` | Define um identificador de 32 bits para o roteador. O formato se parece com um endereço IPv4, mas sua função aqui é somente identificar o roteador. |
| `interface ethX area 0.0.0.0` | Associa a interface indicada à área de backbone. |
| `redistribute connected` | Redistribui no OSPFv3 as rotas diretamente conectadas. |

> [!IMPORTANT]
> Por que cada roteador precisa ter um Router ID diferente, mesmo que a rede encaminhe tráfego IPv6?
>
> <input type="radio" name="resposta_router_id" id="router-id-1" value="endereco-proximo-salto" /> <label for="router-id-1">Porque o Router ID será utilizado como o endereço IPv6 do próximo salto.</label><br>
> <input type="radio" name="resposta_router_id" id="router-id-2" value="identificacao-unica" /> <label for="router-id-2">Porque o Router ID identifica unicamente cada roteador no domínio OSPF e evita ambiguidades nas informações do protocolo.</label><br>
> <input type="radio" name="resposta_router_id" id="router-id-3" value="area-diferente" /> <label for="router-id-3">Porque cada roteador precisa pertencer a uma área OSPF diferente.</label><br>
> <input type="radio" name="resposta_router_id" id="router-id-4" value="substitui-ipv6" /> <label for="router-id-4">Porque o Router ID substitui os endereços IPv6 configurados nas interfaces.</label><br>

## Atividade 4 - Validação das adjacências e das rotas

No terminal do `n2Backbone`, entre novamente na interface de gerenciamento e execute:

```console
vtysh
show ipv6 ospf6
show ipv6 ospf6 neighbor
show ipv6 route
exit
```

![Estado do OSPFv3, vizinhos e rotas no n2Backbone](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab-ospf-ipv6/images/status-ospfv3-r2.png)

Analise as informações apresentadas:

- `show ipv6 ospf6` exibe um resumo do processo OSPFv3, incluindo o Router ID, as áreas e as interfaces participantes;
- `show ipv6 ospf6 neighbor` lista os vizinhos OSPFv3 e o estado de cada adjacência;
- `show ipv6 route` exibe a tabela de roteamento mantida pela suíte de roteamento. As rotas indicadas com `O` foram aprendidas por OSPF.

Em seguida, consulte a tabela de roteamento instalada no sistema operacional:

```console
ip -6 route show
```

![Tabela de rotas IPv6 após a configuração do OSPFv3](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab-ospf-ipv6/images/rotas-ospfv3-r2.png)

Repita os comandos em `n1Backbone` e `n3Backbone`. Os resultados devem ser semelhantes, com diferenças nos Router IDs, vizinhos, interfaces de saída e próximos saltos.

> [!IMPORTANT]
> Quantos vizinhos OSPFv3 aparecem no `n2Backbone`? Em qual estado estão as adjacências?
>
> <input type="radio" name="resposta_vizinhos" id="vizinhos-1" value="um-init" /> <label for="vizinhos-1">Um vizinho, no estado Init.</label><br>
> <input type="radio" name="resposta_vizinhos" id="vizinhos-2" value="dois-full" /> <label for="vizinhos-2">Dois vizinhos, com as adjacências no estado Full.</label><br>
> <input type="radio" name="resposta_vizinhos" id="vizinhos-3" value="tres-exstart" /> <label for="vizinhos-3">Três vizinhos, no estado ExStart.</label><br>
> <input type="radio" name="resposta_vizinhos" id="vizinhos-4" value="nenhum" /> <label for="vizinhos-4">Nenhum vizinho, pois o OSPFv3 não forma adjacências usando IPv6.</label><br>

> [!IMPORTANT]
> Identifique na tabela do `n2Backbone` a rota para `2001:db8:4::/64`. Informe o protocolo, o próximo salto e a interface de saída.
>
> <textarea name="resposta_rota_host_b" rows="4" cols="80" placeholder="Descreva a rota encontrada."></textarea>

## Atividade 5 - Teste de conectividade após a configuração

No terminal do `n4HostA`, repita o teste com o `n5HostB`:

```console
ping6 -c 4 2001:db8:4::20
```

Agora o teste deve ser bem-sucedido, pois os roteadores conhecem os prefixos da topologia e conseguem encaminhar os pacotes até a rede de destino.

![Teste de conectividade bem-sucedido entre n4HostA e n5HostB](https://raw.githubusercontent.com/hackinsdn/labs/refs/heads/main/lab-ospf-ipv6/images/ping-host-a-host-b-sucesso.png)

Registre também o caminho utilizado:

```console
traceroute6 2001:db8:4::20
```

> [!IMPORTANT]
> Compare este resultado com o teste inicial. Qual informação aprendida pelo OSPFv3 tornou possível a comunicação?
>
> <input type="radio" name="resposta_conectividade_final" id="conectividade-final-1" value="endereco-mac" /> <label for="conectividade-final-1">O endereço MAC do n5HostB, anunciado diretamente a todos os hosts.</label><br>
> <input type="radio" name="resposta_conectividade_final" id="conectividade-final-2" value="rota-prefixo-destino" /> <label for="conectividade-final-2">Uma rota para o prefixo `2001:db8:4::/64`, incluindo próximo salto e interface de saída.</label><br>
> <input type="radio" name="resposta_conectividade_final" id="conectividade-final-3" value="rota-ipv4" /> <label for="conectividade-final-3">Uma rota IPv4 padrão usada para encapsular automaticamente os pacotes IPv6.</label><br>
> <input type="radio" name="resposta_conectividade_final" id="conectividade-final-4" value="servidor-dns" /> <label for="conectividade-final-4">O endereço de um servidor DNS capaz de resolver o nome do n5HostB.</label><br>

## Atividade 6 - Falha de enlace e reconvergência

Nesta atividade vamos verificar como o OSPFv3 reage à indisponibilidade de um enlace.

1. No `n4HostA`, inicie um teste contínuo de conectividade:

```console
ping6 2001:db8:4::20
```

2. No terminal do `n2Backbone`, desative temporariamente a interface `eth0`:

```console
ip link set dev eth0 down
```

3. Observe o ping em execução. Aguarde a atualização das adjacências e das rotas.

4. Consulte novamente os vizinhos, as rotas e o caminho até o destino:

```console
vtysh -c "show ipv6 ospf6 neighbor"
vtysh -c "show ipv6 route"
ip -6 route show
```

No `n4HostA`, execute:

```console
traceroute6 2001:db8:4::20
```

5. Ao finalizar o teste, reative a interface:

```console
ip link set dev eth0 up
```

> [!WARNING]
> Confirme, antes de desativar a interface, que `eth0` é realmente o enlace entre roteadores indicado no cenário. O nome da interface pode variar em uma implementação diferente da topologia original.

> [!IMPORTANT]
> A conectividade foi mantida durante a falha? Houve perda de pacotes? Compare o caminho anterior com o novo caminho e explique como o OSPFv3 reagiu.
>
> <textarea name="resposta_reconvergencia" rows="7" cols="80" placeholder="Registre suas observações sobre a falha, a convergência e o caminho alternativo."></textarea>

## Conclusão

Neste laboratório configuramos o OSPFv3 em uma única área e verificamos a formação de adjacências, a aprendizagem dinâmica de prefixos IPv6 e a instalação das rotas no sistema operacional. Também observamos como o protocolo recalcula os caminhos quando ocorre uma mudança na topologia.

> [!IMPORTANT]
> Qual alternativa descreve melhor a principal vantagem do roteamento dinâmico observada neste laboratório em comparação com rotas estáticas?
>
> <input type="radio" name="resposta_conclusao" id="conclusao-1" value="dispensa-enderecos" /> <label for="conclusao-1">Ele elimina a necessidade de configurar endereços IPv6 nas interfaces.</label><br>
> <input type="radio" name="resposta_conclusao" id="conclusao-2" value="aprendizado-reconvergencia" /> <label for="conclusao-2">Ele permite aprender e atualizar rotas automaticamente, adaptando os caminhos quando a topologia muda.</label><br>
> <input type="radio" name="resposta_conclusao" id="conclusao-3" value="impede-falhas" /> <label for="conclusao-3">Ele impede fisicamente que enlaces e roteadores apresentem falhas.</label><br>
> <input type="radio" name="resposta_conclusao" id="conclusao-4" value="sempre-menor-latencia" /> <label for="conclusao-4">Ele garante latência zero e escolhe sempre o caminho com menos roteadores, independentemente do custo.</label><br>

## Referências

- EQUIPE DO PROJETO IPv6.BR; NIC.br. *Laboratório de IPv6: aprenda na prática usando um emulador de redes*. Capítulo 5, Experiência 5.1 - OSPFv3: configuração de uma única área.
- COLTUN, R. et al. [RFC 5340 - OSPF for IPv6](https://www.rfc-editor.org/rfc/rfc5340).
