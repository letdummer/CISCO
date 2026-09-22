# Guia de Simulação Prática de Laboratório no Cisco Packet Tracer

Este guia apresenta o passo a passo completo para montar e configurar um cenário corporativo simulado no Cisco Packet Tracer, englobando a arquitetura física, segmentação em VLANs, roteamento Inter-VLAN (ROAS), serviços de infraestrutura (DHCP Server e Relay), gestão de interfaces virtuais (SVI) e acessos remotos seguros (SSH e Telnet).

---

## 1. Topologia da Rede e Tabela de Endereçamento

### Dispositivos do Laboratório

- **1 Router:** R1-CORE (Cisco 2911 / ISR 4221)
- **1 Switch Principal (Distribuição/Core):** SW-MAIN (Cisco 2960 ou 3560)
- **2 Switches de Acesso:** SW-ACCESS1 e SW-ACCESS2 (Cisco 2960)
- **1 Servidor Dedicado:** SRV-DHCP (Servidor DHCP Central)
- **5 PCs Clientes:** PC1, PC2, PC3, PC4 e PC5

```
                              [ R1-CORE ]
                                   |
                     (Gi0/0/0 - Trunk / ROAS)
                                   |
                             [ SW-MAIN ]
                             /    |    \
                (Trunk Gi0/1)  |   (Trunk Gi0/2)
                          /      |      \
                (Fa0/24) /       |       \
                        /        |        \
              [ SW-ACCESS1 ]     |    [ SW-ACCESS2 ]
                   / \      [SRV-DHCP]      / \
          (Fa0/1) /   \ (Fa0/2)      (Fa0/1)/   \ (Fa0/2)
                [PC1] [PC2]              [PC3] [PC4]
               VLAN10 VLAN20            VLAN10 VLAN20
                                            |
                                       (Fa0/3)
                                         [PC5]
                                      (VLAN 30)
```

### Mapeamento de VLANs e Sub-redes

> Bloco Base `192.168.10.0/24` subdividido com VLSM / CIDR

| VLAN | Nome da VLAN | Sub-rede IP | Máscara / CIDR | Gateway Padrão (R1-CORE) | Dispositivos Associados |
|:---:|:---|:---|:---|:---|:---|
| VLAN 10 | Vendas | 192.168.10.0 | 255.255.255.192 (/26) | 192.168.10.1 | PC1 (Fa0/1), PC3 (Fa0/1) |
| VLAN 20 | Engenharia | 192.168.10.64 | 255.255.255.192 (/26) | 192.168.10.65 | PC2 (Fa0/2), PC4 (Fa0/2) |
| VLAN 30 | Servidores | 192.168.10.128 | 255.255.255.224 (/27) | 192.168.10.129 | SRV-DHCP (.130), PC5 (Fa0/3) |
| VLAN 99 | Gestão | 192.168.10.160 | 255.255.255.224 (/27) | 192.168.10.161 | SVIs: SW-MAIN (.162), SW-ACC1 (.163), SW-ACC2 (.164) |

---

## 2. Passo a Passo da Configuração no Cisco IOS

### TÓPICO 1: Comandos Básicos e Segurança Inicial

#### Explicação Técnica

A configuração inicial garante a identificação única do equipamento na rede (`hostname`), estabelece avisos de acesso não autorizado (`banner motd`), protege o acesso físico via porta de consola e cifra todas as credenciais gravadas na memória RAM em texto limpo (`service password-encryption`).

#### Comandos na CLI (R1-CORE, SW-MAIN, SW-ACCESS1 e SW-ACCESS2)


! Entrar no modo de configuração global
```bash
enable
configure terminal
```


### 1. Alterar o nome do dispositivo
```
hostname R1-CORE
```


### 2. Mensagem legal de aviso ao ligar

```
banner motd #Acesso Restrito! Apenas Pessoal Autorizado.#
```

### 3. Proteção do Modo EXEC Privilegiado
```
enable secret cisco123
```

### 4. Proteção da Porta de Consola

```
 line console 0
 password cisco
 login
 logging synchronous
 exec-timeout 5 0
 exit
```

### 5. Encriptação geral de palavras-passe em texto limpo
```
service password-encryption
```

> Repetir estes comandos básicos ajustando o `hostname` em cada switch: `SW-MAIN`, `SW-ACCESS1`, `SW-ACCESS2`.

---

### TÓPICO 2: Configuração de VLANs e Modos de Interface (Acesso e Tronco)

#### Explicação Técnica

As VLANs segmentam o domínio de broadcast na Camada 2.

- **Portas de Acesso** (`switchport mode access`): Conectam dispositivos finais (PCs, Servidores) e transportam tráfego de apenas uma VLAN específica sem etiquetas (*untagged*).
- **Portas Tronco** (`switchport mode trunk`): Interconectam switches e routers, transportando o tráfego de múltiplas VLANs em simultâneo através da adição da etiqueta IEEE 802.1Q.

#### Comandos na CLI

**1. No Switch Principal (SW-MAIN) — Criar VLANs e Ativar Troncos:**

```bash
SW-MAIN(config)# vlan 10
SW-MAIN(config-vlan)# name Vendas
SW-MAIN(config-vlan)# vlan 20
SW-MAIN(config-vlan)# name Engenharia
SW-MAIN(config-vlan)# vlan 30
SW-MAIN(config-vlan)# name Servidores
SW-MAIN(config-vlan)# vlan 99
SW-MAIN(config-vlan)# name Gestao
SW-MAIN(config-vlan)# exit
```

### Configurar porta ligada ao Router R1-CORE como Trunk
```
SW-MAIN(config)# interface gigabitEthernet 0/0
SW-MAIN(config-if)# switchport mode trunk
SW-MAIN(config-if)# exit
```


### Configurar ligações com SW-ACCESS1 e SW-ACCESS2 como Trunk
```
SW-MAIN(config)# interface range gigabitEthernet 0/1 - 2
SW-MAIN(config-if)# switchport mode trunk
SW-MAIN(config-if)# exit
```

### Configurar porta do Servidor DHCP (SRV-DHCP) na VLAN 30
```
SW-MAIN(config)# interface fastEthernet 0/24
SW-MAIN(config-if)# switchport mode access
SW-MAIN(config-if)# switchport access vlan 30
SW-MAIN(config-if)# exit
```

**2. Nos Switches de Acesso (SW-ACCESS1 e SW-ACCESS2):**

```bash
SW-ACCESS1(config)# vlan 10
SW-ACCESS1(config-vlan)# name Vendas
SW-ACCESS1(config-vlan)# vlan 20
SW-ACCESS1(config-vlan)# name Engenharia
SW-ACCESS1(config-vlan)# vlan 30
SW-ACCESS1(config-vlan)# name Servidores
SW-ACCESS1(config-vlan)# vlan 99
SW-ACCESS1(config-vlan)# name Gestao
SW-ACCESS1(config-vlan)# exit
```

### Porta Tronco para o SW-MAIN
```
SW-ACCESS1(config)# interface gigabitEthernet 0/1
SW-ACCESS1(config-if)# switchport mode trunk
SW-ACCESS1(config-if)# exit
```

### Portas de Acesso para os PCs no SW-ACCESS1
```
SW-ACCESS1(config)# interface fastEthernet 0/1
SW-ACCESS1(config-if)# switchport mode access
SW-ACCESS1(config-if)# switchport access vlan 10
SW-ACCESS1(config-if)# exit
```

```
SW-ACCESS1(config)# interface fastEthernet 0/2
SW-ACCESS1(config-if)# switchport mode access
SW-ACCESS1(config-if)# switchport access vlan 20
SW-ACCESS1(config-if)# exit
```

> No `SW-ACCESS2`: repetir criação de VLANs, meter `Gi0/1` em Trunk, `Fa0/1` na VLAN 10, `Fa0/2` na VLAN 20 e `Fa0/3` na VLAN 30 para o PC5.

---

### TÓPICO 3: Configuração de Interfaces Virtuais de Switch (SVI) e Default Gateway

#### Explicação Técnica

Uma **SVI** (*Switch Virtual Interface*) é uma interface lógica de Camada 3 configurada dentro do switch que permite atribuir um endereço IP para gestão remota. Como o switch opera na Camada 2, ele precisa de um *Default Gateway* para responder a pacotes vindos de sub-redes/VLANs diferentes da sua SVI.

#### Comandos na CLI (SW-ACCESS1 — Exemplo)

```bash
SW-ACCESS1(config)# interface vlan 99
SW-ACCESS1(config-if)# description Interface_Gestao_SW1
SW-ACCESS1(config-if)# ip address 192.168.10.163 255.255.255.224
SW-ACCESS1(config-if)# no shutdown
SW-ACCESS1(config-if)# exit
```

### Apontar o Gateway padrão do Switch para o IP do Router na VLAN 99
```
SW-ACCESS1(config)# ip default-gateway 192.168.10.161
```

> SVI do `SW-MAIN`: `192.168.10.162/27` | SVI do `SW-ACCESS2`: `192.168.10.164/27` | Gateway de ambos: `192.168.10.161`

---

### TÓPICO 4: Roteamento Inter-VLAN via Router-on-a-Stick (ROAS)

#### Explicação Técnica

O tráfego intra-VLAN ocorre diretamente no switch (Camada 2). Para permitir a comunicação inter-VLAN, utiliza-se a técnica **ROAS** (*Router-on-a-Stick*), onde uma única porta física do router (`Gi0/0/0`) é conectada a uma porta tronco do switch. A interface física é dividida em sub-interfaces lógicas, associando cada uma à sua respetiva VLAN através do enquadramento `encapsulation dot1Q`.

#### Comandos na CLI (R1-CORE)

```bash
R1-CORE(config)# interface gigabitEthernet 0/0/0
R1-CORE(config-if)# description Link_Tronco_ROAS_SW-MAIN
R1-CORE(config-if)# no shutdown
R1-CORE(config-if)# exit
```

### Sub-interface VLAN 10 (Vendas)
```
R1-CORE(config)# interface gigabitEthernet 0/0/0.10
R1-CORE(config-subif)# encapsulation dot1Q 10
R1-CORE(config-subif)# ip address 192.168.10.1 255.255.255.192
R1-CORE(config-subif)# exit
```

### Sub-interface VLAN 20 (Engenharia)
```
R1-CORE(config)# interface gigabitEthernet 0/0/0.20
R1-CORE(config-subif)# encapsulation dot1Q 20
R1-CORE(config-subif)# ip address 192.168.10.65 255.255.255.192
R1-CORE(config-subif)# exit
```

### Sub-interface VLAN 30 (Servidores)
```
R1-CORE(config)# interface gigabitEthernet 0/0/0.30
R1-CORE(config-subif)# encapsulation dot1Q 30
R1-CORE(config-subif)# ip address 192.168.10.129 255.255.255.224
R1-CORE(config-subif)# exit
```

### Sub-interface VLAN 99 (Gestão)
```
R1-CORE(config)# interface gigabitEthernet 0/0/0.99
R1-CORE(config-subif)# encapsulation dot1Q 99
R1-CORE(config-subif)# ip address 192.168.10.161 255.255.255.224
R1-CORE(config-subif)# exit
```

---

### TÓPICO 5: Configuração de Serviço DHCP no Servidor Central (SRV-DHCP)

#### Explicação Técnica

O protocolo DHCP automatiza a distribuição de IPs pelo ciclo **DORA** (*Discover, Offer, Request, ACK*). Em redes empresariais, é comum centralizar as *pools* num Servidor DHCP dedicado.

#### Configuração no Servidor Dedicado SRV-DHCP (Packet Tracer GUI)

1. Clique no `SRV-DHCP` → Separador **Desktop** → **IP Configuration**:
   - IP Address: `192.168.10.130`
   - Subnet Mask: `255.255.255.224`
   - Default Gateway: `192.168.10.129`

2. Separador **Services** → **DHCP**:
   - Ativar o serviço: `Service = On`

   **Pool 1 (VLAN 10 - Vendas):**
   - Pool Name: `POOL-VENDAS`
   - Default Gateway: `192.168.10.1`
   - DNS Server: `8.8.8.8`
   - Start IP Address: `192.168.10.10`
   - Subnet Mask: `255.255.255.192` → Clique em **Add**

   **Pool 2 (VLAN 20 - Engenharia):**
   - Pool Name: `POOL-ENGENHARIA`
   - Default Gateway: `192.168.10.65`
   - DNS Server: `8.8.8.8`
   - Start IP Address: `192.168.10.75`
   - Subnet Mask: `255.255.255.192` → Clique em **Add**

---

### TÓPICO 6: Configuração do Agente de Retransmissão DHCP (`ip helper-address`)

#### Explicação Técnica

Como as mensagens `DHCPDISCOVER` são enviadas em *broadcast*, os routers bloqueiam a sua passagem entre sub-redes. O comando `ip helper-address` transforma o *broadcast* recebido na interface dos clientes num pacote *unicast* direcionado ao IP do Servidor DHCP Central (`192.168.10.130`).

#### Comandos na CLI (R1-CORE)

### Aplicar o agente de retransmissão nas sub-interfaces das VLANs clientes
```
R1-CORE(config)# interface gigabitEthernet 0/0/0.10
R1-CORE(config-subif)# ip helper-address 192.168.10.130
R1-CORE(config-subif)# exit
```

```
R1-CORE(config)# interface gigabitEthernet 0/0/0.20
R1-CORE(config-subif)# ip helper-address 192.168.10.130
R1-CORE(config-subif)# exit
```

> A partir deste momento, quando o PC1 ou PC2 solicitarem IP via DHCP, o R1-CORE reencaminhará o pedido para o SRV-DHCP na VLAN 30.

---

### TÓPICO 7: Acesso Remoto Seguro (SSH no Router) e Telnet (nos Switches)

#### Explicação Técnica

- **Telnet** (Porta TCP 23): Transmite credenciais em texto simples sem encriptação (configurado nos switches de acesso para fins de teste).
- **SSH** (Porta TCP 22): Cifra o tráfego de gestão utilizando chaves RSA (configurado no Router R1-CORE).

#### Comandos na CLI

**1. SSH no Router (R1-CORE):**

```bash
R1-CORE(config)# ip domain-name empresa.local
R1-CORE(config)# username admin secret AdminPass123
R1-CORE(config)# crypto key generate rsa 1024
R1-CORE(config)# ip ssh version 2
R1-CORE(config)# line vty 0 4
R1-CORE(config-line)# login local
R1-CORE(config-line)# transport input ssh
R1-CORE(config-line)# exec-timeout 10 0
R1-CORE(config-line)# exit
```

**2. Telnet nos Switches (SW-ACCESS1 — Exemplo):**

```bash
SW-ACCESS1(config)# line vty 0 15
SW-ACCESS1(config-line)# password cisco
SW-ACCESS1(config-line)# login
SW-ACCESS1(config-line)# transport input telnet
SW-ACCESS1(config-line)# exec-timeout 5 0
SW-ACCESS1(config-line)# exit
```

---

## 3. Roteiro de Testes e Validação da Rede

1. **Obtenção Dinâmica de IP (DHCP):**
   - Abra o `PC1` (VLAN 10) → **Desktop** → **IP Configuration** → Selecione `DHCP`.
   - Confirme se o IP atribuído pertence à gama `192.168.10.X/26` com Gateway `192.168.10.1`.

2. **Teste de Comunicação Intra-VLAN:**
   - No `PC1`, abra o Prompt de Comando (CMD) e execute:
     ```bash
     ping 192.168.10.X
     ```
     (IP do PC3, pertencente à mesma VLAN 10)

3. **Teste de Roteamento Inter-VLAN (ROAS):**
   - No `PC1`, execute o ping para o PC2 (VLAN 20):
     ```bash
     ping 192.168.10.75
     ```

4. **Teste de Acesso Remoto Telnet ao Switch:**
   - No `PC1`, aceda ao switch de acesso via Telnet:
     ```bash
     telnet 192.168.10.163
     ```
     (Insira a palavra-passe `cisco`)

5. **Teste de Acesso Remoto Seguro SSH ao Router:**
   - No `PC1`, ligue-se ao Router via SSH:
     ```bash
     ssh -l admin 192.168.10.1
     ```
     (Insira a palavra-passe `AdminPass123`)
