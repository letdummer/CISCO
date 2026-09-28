# Guia de Estudo Prático: Cisco Packet Tracer e IOS

## 📑 Índice

1. [Comandos Básicos e Navegação no Cisco IOS](#1-comandos-básicos-e-navegação-no-cisco-ios)
2. [Configuração de Acesso Remoto via Telnet](#2-configuração-de-acesso-remoto-via-telnet)
3. [Configuração de Acesso Remoto Seguro via SSH](#3-configuração-de-acesso-remoto-seguro-via-ssh)
4. [Configuração de VLANs (Portas de Acesso e Tronco / Trunk)](#4-configuração-de-vlans-portas-de-acesso-e-tronco--trunk)
5. [Configuração de Interfaces Virtuais de Switch (SVI)](#5-configuração-de-interfaces-virtuais-de-switch-svi)
6. [Roteamento Inter-VLAN via Router-on-a-Stick (ROAS)](#6-roteamento-inter-vlan-via-router-on-a-stick-roas)
7. [Configuração de Serviço DHCP no Servidor / Router Cisco](#7-configuração-de-serviço-dhcp-no-servidor--router-cisco)
8. [Configuração de Agente de Retransmissão DHCP (Relay Agent)](#8-configuração-de-agente-de-retransmissão-dhcp-relay-agent)

---

## 1. Comandos Básicos e Navegação no Cisco IOS

### Explicação Conceitual

O sistema operativo **Cisco IOS (Internetwork Operating System)** utiliza uma interface por linha de comandos (CLI) estruturada de forma hierárquica. A navegação divide-se em modos de execução com diferentes níveis de privilégio e acesso:

- **Modo EXEC Utilizador (`Router>`)** — Permite apenas visualização e comandos básicos de monitorização.
- **Modo EXEC Privilegiado (`Router#`)** — Concede acesso total a diagnósticos, testes e comandos de visualização do sistema (`show`).
- **Modo de Configuração Global (`Router(config)#`)** — Permite alterar parâmetros globais que afetam todo o dispositivo.
- **Submodos de Configuração (`(config-if)#`, `(config-line)#`)** — Permitem alterar configurações específicas de interfaces físicas/virtuais ou linhas de comunicação.

A memória do equipamento divide-se em **RAM** (onde reside a `running-config`, volátil e apagada ao desligar) e **NVRAM** (onde é guardada a `startup-config`, mantida após o reinício).

### Passo a Passo de Configuração

**1. Acesso aos Modos de Configuração**

```
Router> enable
Router# configure terminal
Router(config)#
```

> Nota: para recuar um nível usa-se `exit`; para voltar diretamente ao modo Privilegiado usa-se `end` ou `Ctrl+Z`.

**2. Alteração do Nome do Equipamento (Hostname)**

```
Router(config)# hostname R1-Matriz
R1-Branch(config)#
```

> Regras: começar por letra, sem espaços e com menos de 64 caracteres.

**3. Proteção do Acesso à Porta Consola**

```
R1-Branch(config)# line console 0
R1-Branch(config-line)# password cisco
R1-Branch(config-line)# login
R1-Branch(config-line)# logging synchronous
R1-Branch(config-line)# exec-timeout 5 0
R1-Branch(config-line)# exit
```

> `logging synchronous` impede que alertas do sistema interrompam os comandos digitados; `exec-timeout 5 0` encerra sessões inativas após 5 minutos.

**4. Proteção do Modo EXEC Privilegiado**

```
R1-Branch(config)# enable secret cisco123
```

> `enable secret` aplica uma encriptação forte no ficheiro de configuração.

**5. Encriptação de Senhas em Texto Limpo e Mensagem Legal (MOTD)**

```
R1-Branch(config)# service password-encryption
R1-Branch(config)# banner motd #Acesso Restrito e Monitorizado!#
```

> `service password-encryption` cifra as senhas simples configuradas em linhas de consola e VTY; o caractere `#` delimita o início e o fim da mensagem do banner.

**6. Salvamento e Verificação de Configuração**

```
! Visualizar configurações ativas e resumo de interfaces
R1-Branch# show running-config
R1-Branch# show ip interface brief

! Guardar alterações da RAM para a NVRAM
R1-Branch# copy running-config startup-config

! Atalho equivalente
R1-Branch# write memory
```

---

## 2. Configuração de Acesso Remoto via Telnet

### Explicação Conceitual

O **Telnet** é um protocolo tradicional de gestão remota operante na porta **TCP 23**. Apresenta a fragilidade de **não utilizar encriptação**, transmitindo todos os dados (incluindo nomes de utilizador e palavras-passe) em texto limpo através da rede. Deve ser utilizado prioritariamente em ambientes de simulação/laboratório ou isolados. É configurado nas linhas virtuais de terminal (**VTY**) do equipamento.

### Passo a Passo de Configuração

```
SW1# configure terminal

! 1. Entrar no intervalo das linhas virtuais (0 a 15 no switch)
SW1(config)# line vty 0 15

! 2. Definir a palavra-passe de acesso e exigir autenticação
SW1(config-line)# password cisco
SW1(config-line)# login

! 3. Permitir o protocolo Telnet para conexões de entrada
SW1(config-line)# transport input telnet

! 4. Definir tempo limite de inatividade (ex.: 5 minutos)
SW1(config-line)# exec-timeout 5 0
SW1(config-line)# exit
```

---

## 3. Configuração de Acesso Remoto Seguro via SSH

### Explicação Conceitual

O **SSH (Secure Shell)** é o padrão recomendado para gestão remota segura na porta **TCP 22**. Ao contrário do Telnet, todas as comunicações e comandos são cifrados através de algoritmos de criptografia assimétrica (chaves **RSA**). Para funcionar, exige que o dispositivo possua um **hostname** definido, um **nome de domínio IP**, uma **chave RSA** gerada e uma conta de **utilizador local**.

### Passo a Passo de Configuração

```
R1# configure terminal

! 1. Definir Hostname e Nome de Domínio
R1(config)# hostname R1-Branch
R1-Branch(config)# ip domain-name empresa.local

! 2. Criar utilizador local com palavra-passe encriptada
R1-Branch(config)# username admin secret AdminPass123

! 3. Gerar as chaves criptográficas RSA (recomendado mínimo 1024 bits)
R1-Branch(config)# crypto key generate rsa
How many bits in the modulus : 1024

! 4. Forçar a utilização do protocolo SSH versão 2
R1-Branch(config)# ip ssh version 2

! 5. Restringir as linhas VTY para aceitar EXCLUSIVAMENTE SSH e autenticação local
R1-Branch(config)# line vty 0 4
R1-Branch(config-line)# login local
R1-Branch(config-line)# transport input ssh
R1-Branch(config-line)# exec-timeout 10 0
R1-Branch(config-line)# exit
```

---

## 4. Configuração de VLANs (Portas de Acesso e Tronco / Trunk)

### Explicação Conceitual

Uma **VLAN (Virtual Local Area Network)** divide uma rede física de Camada 2 em múltiplos domínios de broadcast lógicos independentes. O tráfego de uma VLAN é isolado das restantes.

- **Porta de Acesso (Access)** — Porta ligada a um dispositivo final (PC, Impressora). Pertence a uma única VLAN e transmite quadros Ethernet normais (sem tag).
- **Porta Tronco (Trunk)** — Link de interconexão entre dois switches (ou switch e router) capaz de transportar tráfego de múltiplas VLANs em simultâneo através da adição de uma etiqueta no quadro sob a norma **IEEE 802.1Q**.

### Passo a Passo de Configuração

**1. Criar e Nomear as VLANs no Switch**

```
SW1# configure terminal
SW1(config)# vlan 10
SW1(config-vlan)# name Vendas
SW1(config-vlan)# exit

SW1(config)# vlan 20
SW1(config-vlan)# name Engenharia
SW1(config-vlan)# exit
```

**2. Atribuir Portas em Modo de Acesso (Access Port)**
Usada para configurar terminais, impressoras etc.

**! Associar a interface FastEthernet 0/1 à VLAN 10**
```
SW1(config)# interface fastEthernet 0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# exit
```

**! Associar a interface FastEthernet 0/2 à VLAN 20**
```
SW1(config)# interface fastEthernet 0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
SW1(config-if)# exit
```

**3. Configurar Porta em Modo Tronco (Trunk Port)**


**! Configurar a porta GigabitEthernet 0/1 (ligada a outro switch ou router)**
```
SW1(config)# interface gigabitEthernet 0/1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk native vlan 40 => modifica a VLAN nativa da VLAN 1 para outra vlan desejada.
SW1(config-if)# switchport trunk allowed vlan 10,20,30,99  => especifica quais vlans estão autorizadas para esta conexão
SW1(config-if)# exit
```

> Verificação: `show vlan brief`
> `show interfaces vlan 20`
> `show interfaces trunk`
> `show vlan name student`
> `show vlan summary`

---

## 5. Configuração de Interfaces Virtuais de Switch (SVI)

### Explicação Conceitual

Os switches de Camada 2 operam apenas no encaminhamento de quadros por endereço MAC e não possuem IP nas portas físicas. Para permitir a gestão remota do switch via Telnet/SSH ou testes ICMP (ping), configura-se uma **SVI (Switch Virtual Interface)**. A SVI é uma interface lógica associada a uma VLAN (geralmente a VLAN 1 por omissão). Para ser gerido a partir de redes externas, o switch necessita também da definição de um **Default Gateway** apontado para o router.

### Passo a Passo de Configuração

**! 1. Entrar na interface virtual da VLAN de gestão**
```
SW1(config)# interface vlan 99 (ou 40)
```

**! 2. Atribuir o endereço IP e a máscara de sub-rede**
```
SW1(config-if)# ip address 192.168.10.2 255.255.255.0
```

**! 3. Ativar a interface virtual**
```
SW1(config-if)# no shutdown
SW1(config-if)# exit
```

**! 4. Configurar o Gateway Padrão para acesso a partir de outras sub-redes**
```
SW1(config)# ip default-gateway 192.168.10.1
SW1(config)# exit
```

---

## 6. Roteamento Inter-VLAN via Router-on-a-Stick (ROAS)

### Explicação Conceitual

Como cada VLAN é um domínio de broadcast isolado de Camada 2, os computadores pertencentes a VLANs diferentes não conseguem comunicar diretamente sem um dispositivo de Camada 3 (Router). O método **Router-on-a-Stick (ROAS)** utiliza uma **única interface física** no router conectada a uma porta **Trunk** do switch.

A interface física do router é dividida em **sub-interfaces lógicas**, onde cada sub-interface atua como o Gateway Padrão de uma VLAN específica, utilizando o protocolo **dot1Q (IEEE 802.1Q)** para etiquetar e desempacotar os pacotes das respetivas VLANs.

```
                     [ Router R1 ]
                           |  (Gi0/0/0 - Link Tronco)
                           |  Gi0/0/0.10 (Gateway VLAN 10)
                           |  Gi0/0/0.20 (Gateway VLAN 20)
                     [ Switch SW1 ]
                      /          \
  (Modo Access VLAN 10)          (Modo Access VLAN 20)
           /                                \
     [ PC Vendas ]                    [ PC Engenharia ]
```

### Passo a Passo de Configuração

**1. No Switch: Configurar a porta conectada ao Router como Tronco (Trunk)**

```
SW1(config)# interface gigabitEthernet 0/1
SW1(config-if)# switchport mode trunk
SW1(config-if)# exit
```

**2. No Router: Configurar as Sub-interfaces e Encapsulamento dot1Q**


**! Ativar a interface física principal (não atribuir IP direto à porta física)**
```
R1(config)# interface gigabitEthernet 0/0/0
R1(config-if)# no shutdown
R1(config-if)# exit
```

> Sintaxe:
> número da interface física (ponto) número da subinterface

**! --- Sub-interface para a VLAN 10 ---**
```
R1(config)# interface gigabitEthernet 0/0/0.10
R1(config-subif)# description Gateway_VLAN_Vendas
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# ip address 192.168.10.1 255.255.255.0
R1(config-subif)# exit
```

**! --- Sub-interface para a VLAN 20 ---**
```
R1(config)# interface gigabitEthernet 0/0/0.20
R1(config-subif)# description Gateway_VLAN_Engenharia
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip address 192.168.20.1 255.255.255.0
R1(config-subif)# exit
```

---

## 7. Configuração de Serviço DHCP no Servidor / Router Cisco

### Explicação Conceitual

O **DHCP (Dynamic Host Configuration Protocol)** automatiza a atribuição de endereços IPv4 e parâmetros de rede aos dispositivos. A negociação cliente-servidor segue o ciclo **DORA**:

- **Discover** — O cliente envia uma mensagem em *broadcast* à procura de um servidor DHCP.
- **Offer** — O servidor responde com uma proposta de IP.
- **Request** — O cliente solicita o IP oferecido.
- **ACK** — O servidor confirma a atribuição e o período de empréstimo (*lease*).

No Cisco IOS, deve-se primeiro **excluir os endereços IP estáticos** (usados em routers, switches e impressoras) antes de criar a *pool* de endereços.

### Passo a Passo de Configuração


**1. Excluir os endereços estáticos reservados para infraestrutura**
```
R1(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.10
R1(config)# ip dhcp excluded-address 192.168.20.1 192.168.20.10
```

**! 2. Criar a Pool DHCP para a VLAN 10**
```
R1(config)# ip dhcp pool POOL-VENDAS
R1(dhcp-config)# network 192.168.10.0 255.255.255.0
R1(dhcp-config)# default-router 192.168.10.1
R1(dhcp-config)# dns-server 8.8.8.8
R1(dhcp-config)# domain-name empresa.local
R1(dhcp-config)# exit
```

**OBS.:** Se for configurar no Router em vez de no Servidor, basta não configurar a linha "domain-name". 

**! 3. Criar a Pool DHCP para a VLAN 20**
```
R1(config)# ip dhcp pool POOL-ENGENHARIA
R1(dhcp-config)# network 192.168.20.0 255.255.255.0
R1(dhcp-config)# default-router 192.168.20.1
R1(dhcp-config)# dns-server 8.8.8.8
R1(dhcp-config)# exit
```

> Comandos de verificação:
> `show ip dhcp binding`
> `show ip dhcp server statistics`
> `show running-config | section dhcp`

---

## 8. Configuração de Agente de Retransmissão DHCP (Relay Agent)

### Explicação Conceitual

Os pedidos DHCPDISCOVER são enviados como mensagens de *broadcast* de Camada 2/3. Como os routers **bloqueiam mensagens de broadcast por padrão**, um cliente situado numa sub-rede onde não exista servidor DHCP local não consegue obter endereço dinâmico se o servidor estiver noutra sub-rede.

O **Agente de Retransmissão (DHCP Relay Agent)** resolve este problema: configurado na interface do router voltada para a sub-rede dos clientes, intercepta as solicitações em broadcast e reencaminha-as como mensagens **unicast** diretamente para o endereço IP do Servidor DHCP central.

```
  [ PC Cliente ] ---> Broadcast (DHCPDISCOVER) ---> [ Router R1 (Interface Gi0/0/0) ]
                                                           |
                                               ip helper-address 10.1.1.2
                                                           |
                                                           v
                                               Encaminhamento Unicast
                                                           |
                                                           v
                                               [ Servidor DHCP Central (10.1.1.2) ]
```

### Passo a Passo de Configuração


**! Entrar na `interface/sub-interface` ligada à LAN de clientes que precisam de DHCP**
```
R1(config)# interface gigabitEthernet 0/0/0.10
```

**! Configurar o `IP do Servidor DHCP` Central para onde os pedidos devem ser redirecionados**
```
R1(config-subif)# ip helper-address 10.1.1.2
R1(config-subif)# exit
```

**! `Repetir o processo para outras interfaces` que necessitem de retransmissão**
```
R1(config)# interface gigabitEthernet 0/0/0.20
R1(config-subif)# ip helper-address 10.1.1.2
R1(config-subif)# exit
```
