# Guia de Simulação Prática de Laboratório no Cisco Packet Tracer

Este guia apresenta o passo a passo completo para montar e configurar um cenário corporativo simulado no Cisco Packet Tracer, englobando a arquitetura física, segmentação em VLANs, roteamento Inter-VLAN (ROAS), serviços de infraestrutura (DHCP Server e Relay), gestão de interfaces virtuais (SVI) e acessos remotos seguros (SSH e Telnet).

---

## 1. Topologia da Rede e Tabela de Endereçamento

### Dispositivos do Laboratório

- **1 Router:** Router-01 (Cisco 2911 / ISR 4221)
- **1 Switch Principal (Distribuição/Core):** SW-00 (Cisco 2960 ou 3560)
- **2 Switches de Acesso:** SW-01 e SW-02 (Cisco 2960)
- **1 Servidor Dedicado:** SRV-DHCP (Servidor DHCP Central)
- **5 PCs Clientes:** PC1, PC2, PC3, PC4 e PC5

```
                              [ Router-01 ]
                                   |
                     (Gi0/0/0 - Trunk / ROAS)
                                   |
                             [ SW-00 ]
                             /    |    \
                (Trunk Gi0/1)  |   (Trunk Gi0/2)
                          /      |      \
                (Fa0/24) /       |       \
                        /        |        \
              [ SW-01 ]     |    [ SW-02 ]
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

| VLAN | Nome da VLAN | Sub-rede IP | Máscara / CIDR | Gateway Padrão (Router-01) | Dispositivos Associados |
|:---:|:---|:---|:---|:---|:---|
| VLAN 10 | Estudantes | 192.168.10.0 | 255.255.255.192 (/26) | 192.168.10.1 | PC1 (Fa0/1), PC3 (Fa0/1) |
| VLAN 20 | Professores | 192.168.10.64 | 255.255.255.192 (/26) | 192.168.10.65 | PC2 (Fa0/2), PC4 (Fa0/2) |
| VLAN 30 | Servidores | 192.168.10.128 | 255.255.255.224 (/27) | 192.168.10.129 | SRV-DHCP (.130), PC5 (Fa0/3) |
| VLAN 99 | Gestão | 192.168.10.160 | 255.255.255.224 (/27) | 192.168.10.161 | SVIs: SW-00 (.162), SW-ACC1 (.163), SW-ACC2 (.164) |

---

## 2. Passo a Passo da Configuração no Cisco IOS

### TÓPICO 1: Comandos Básicos e Segurança Inicial

#### Explicação Técnica

A configuração inicial garante a identificação única do equipamento na rede (`hostname`), estabelece avisos de acesso não autorizado (`banner motd`), protege o acesso físico via porta de consola e cifra todas as credenciais gravadas na memória RAM em texto limpo (`service password-encryption`).

#### Comandos na CLI (Router-01, SW-00, SW-01 e SW-02)
## -> Para cada dispositivo, alterar apenas o `hostname`

! Entrar no modo de configuração global
```bash
enable
configure terminal
```


### 1. Alterar o nome do dispositivo
```
hostname Router-01
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

> Repetir estes comandos básicos ajustando o `hostname` em cada switch: `SW-00`, `SW-01`, `SW-02`.

---

### TÓPICO 2: Configuração de VLANs e Modos de Interface (Acesso e Tronco)

#### Explicação Técnica

As VLANs segmentam o domínio de broadcast na Camada 2.

- **Portas de Acesso** (`switchport mode access`): Conectam dispositivos finais (PCs, Servidores) e transportam tráfego de apenas uma VLAN específica sem etiquetas (*untagged*).
- **Portas Tronco** (`switchport mode trunk`): Interconectam switches e routers, transportando o tráfego de múltiplas VLANs em simultâneo através da adição da etiqueta IEEE 802.1Q.

#### Comandos na CLI

**1. No Switch Principal (SW-00) — Criar VLANs e Ativar Troncos:**

```bash
SW-00(config)# vlan 10
SW-00(config-vlan)# name Estudantes
SW-00(config-vlan)# vlan 20
SW-00(config-vlan)# name Professores
SW-00(config-vlan)# vlan 30
SW-00(config-vlan)# name Servidores
SW-00(config-vlan)# vlan 99
SW-00(config-vlan)# name Gestao
SW-00(config-vlan)# exit
```

**Configurar porta ligada ao Router Router-01 como Trunk**
```
SW-00(config)# interface gigabitEthernet 0/1
SW-00(config-if)# switchport mode trunk
SW-00(config-if)# exit
```


**Configurar ligações com SW-01 e SW-02 como Trunk**
```
SW-00(config)# interface range gigabitEthernet 0/1 - 2
SW-00(config-if)# switchport mode trunk
SW-00(config-if)# exit
```

**Configurar porta do Servidor DHCP (SRV-DHCP) na VLAN 30**
```
SW-00(config)# interface fastEthernet 0/2
SW-00(config-if)# switchport mode access
SW-00(config-if)# switchport access vlan 30
SW-00(config-if)# exit
```

**2. Nos Switches de Acesso (SW-01 e SW-02):**

```bash
SW-01(config)# vlan 10
SW-01(config-vlan)# name Estudantes
SW-01(config-vlan)# vlan 20
SW-01(config-vlan)# name Professores
SW-01(config-vlan)# vlan 30
SW-01(config-vlan)# name Servidores
SW-01(config-vlan)# vlan 99
SW-01(config-vlan)# name Gestao
SW-01(config-vlan)# exit
```

**Configurar Trunk para o Router (Gig0/0 do Router <-> Gig0/1 do SW-00)**
```
SW-00(config)# interface gigabitEthernet 0/1
SW-00(config-if)# switchport mode trunk
SW-00(config-if)# no shutdown
SW-00(config-if)# exit
```

**PConfigurar Trunk para os Switches SW-01 (Fa0/3) e SW-02 (Fa0/4)**
```
SW-00(config)# interface range fastEthernet 0/3 - 4
SW-00(config-if)# switchport mode trunk
SW-00(config-if)# no shutdown
SW-00(config-if)# exit
```

**Configurar a porta do Servidor DHCP (Fa0/2) na VLAN 30**
```
SW-00(config)# interface fastEthernet 0/2
SW-00(config-if)# switchport mode access
SW-00(config-if)# switchport access vlan 30
SW-00(config-if)# no shutdown
SW-00(config-if)# exit
```


**Configuração Requerida nos Switches de Acesso (SW-01 e SW-02)**
Para a comunicação funcionar em ambos os sentidos, não se esqueça de ativar o Trunk nas portas Fa0/1 de cada um dos switches de acesso:

# SW-01:
```
SW-01(config)# interface fastEthernet 0/1
SW-01(config-if)# switchport mode trunk
SW-01(config-if)# no shutdown
SW-01(config-if)# exit
```

# SW-02:
```
SW-02(config)# interface fastEthernet 0/1
SW-02(config-if)# switchport mode trunk
SW-02(config-if)# no shutdown
SW-02(config-if)# exit
```

---

### TÓPICO 3: Configuração de Interfaces Virtuais de Switch (SVI) e Default Gateway

#### Explicação Técnica

Uma **SVI** (*Switch Virtual Interface*) é uma interface lógica de Camada 3 configurada dentro do switch que permite atribuir um endereço IP para gestão remota. Como o switch opera na Camada 2, ele precisa de um *Default Gateway* para responder a pacotes vindos de sub-redes/VLANs diferentes da sua SVI.

#### Comandos na CLI (SW-01 — Exemplo)

```bash
SW-01(config)# interface vlan 99
SW-01(config-if)# description Interface_Gestao_SW1
SW-01(config-if)# ip address 192.168.10.163 255.255.255.224
SW-01(config-if)# no shutdown
SW-01(config-if)# exit
```

### Apontar o Gateway padrão do Switch para o IP do Router na VLAN 99
```
SW-01(config)# ip default-gateway 192.168.10.161
```

> SVI do `SW-00`: `192.168.10.162/27` | SVI do `SW-02`: `192.168.10.164/27` | Gateway de ambos: `192.168.10.161`

---

### TÓPICO 4: Roteamento Inter-VLAN via Router-on-a-Stick (ROAS)

#### Explicação Técnica

O tráfego intra-VLAN ocorre diretamente no switch (Camada 2). Para permitir a comunicação inter-VLAN, utiliza-se a técnica **ROAS** (*Router-on-a-Stick*), onde uma única porta física do router (`Gi0/0/0`) é conectada a uma porta tronco do switch. A interface física é dividida em sub-interfaces lógicas, associando cada uma à sua respetiva VLAN através do enquadramento `encapsulation dot1Q`.

#### Comandos na CLI (Router-01)

```bash
Router-01(config)# interface gigabitEthernet 0/0/0
Router-01(config-if)# description Link_Tronco_ROAS_SW-00
Router-01(config-if)# no shutdown
Router-01(config-if)# exit
```

### Sub-interface VLAN 10 (Estudantes)
```
Router-01(config)# interface gigabitEthernet 0/0/0.10
Router-01(config-subif)# encapsulation dot1Q 10
Router-01(config-subif)# ip address 192.168.10.1 255.255.255.192
Router-01(config-subif)# exit
```

### Sub-interface VLAN 20 (Professores)
```
Router-01(config)# interface gigabitEthernet 0/0/0.20
Router-01(config-subif)# encapsulation dot1Q 20
Router-01(config-subif)# ip address 192.168.10.65 255.255.255.192
Router-01(config-subif)# exit
```

### Sub-interface VLAN 30 (Servidores)
```
Router-01(config)# interface gigabitEthernet 0/0/0.30
Router-01(config-subif)# encapsulation dot1Q 30
Router-01(config-subif)# ip address 192.168.10.129 255.255.255.224
Router-01(config-subif)# exit
```

### Sub-interface VLAN 99 (Gestão)
```
Router-01(config)# interface gigabitEthernet 0/0/0.99
Router-01(config-subif)# encapsulation dot1Q 99
Router-01(config-subif)# ip address 192.168.10.161 255.255.255.224
Router-01(config-subif)# exit
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

   **Pool 1 (VLAN 10 - Estudantes):**
   - Pool Name: `POOL-Estudantes`
   - Default Gateway: `192.168.10.1`
   - DNS Server: `8.8.8.8`
   - Start IP Address: `192.168.10.10`
   - Subnet Mask: `255.255.255.192` → Clique em **Add**

   **Pool 2 (VLAN 20 - Professores):**
   - Pool Name: `POOL-Professores`
   - Default Gateway: `192.168.10.65`
   - DNS Server: `8.8.8.8`
   - Start IP Address: `192.168.10.75`
   - Subnet Mask: `255.255.255.192` → Clique em **Add**

---

### TÓPICO 6: Configuração do Agente de Retransmissão DHCP (`ip helper-address`)

#### Explicação Técnica

Como as mensagens `DHCPDISCOVER` são enviadas em *broadcast*, os routers bloqueiam a sua passagem entre sub-redes. O comando `ip helper-address` transforma o *broadcast* recebido na interface dos clientes num pacote *unicast* direcionado ao IP do Servidor DHCP Central (`192.168.10.130`).

#### Comandos na CLI (Router-01)

### Aplicar o agente de retransmissão nas sub-interfaces das VLANs clientes
```
Router-01(config)# interface gigabitEthernet 0/0/0.10
Router-01(config-subif)# ip helper-address 192.168.10.130
Router-01(config-subif)# exit
```

```
Router-01(config)# interface gigabitEthernet 0/0/0.20
Router-01(config-subif)# ip helper-address 192.168.10.130
Router-01(config-subif)# exit
```

> A partir deste momento, quando o PC1 ou PC2 solicitarem IP via DHCP, o Router-01 reencaminhará o pedido para o SRV-DHCP na VLAN 30.

---

### TÓPICO 7: Acesso Remoto Seguro (SSH no Router) e Telnet (nos Switches)

#### Explicação Técnica

- **Telnet** (Porta TCP 23): Transmite credenciais em texto simples sem encriptação (configurado nos switches de acesso para fins de teste).
- **SSH** (Porta TCP 22): Cifra o tráfego de gestão utilizando chaves RSA (configurado no Router Router-01).

#### Comandos na CLI

**1. SSH no Router (Router-01):**

```bash
Router-01(config)# ip domain-name empresa.local
Router-01(config)# username admin secret AdminPass123
Router-01(config)# crypto key generate rsa 1024
Router-01(config)# ip ssh version 2
Router-01(config)# line vty 0 4
Router-01(config-line)# login local
Router-01(config-line)# transport input ssh
Router-01(config-line)# exec-timeout 10 0
Router-01(config-line)# exit
```

**2. Telnet nos Switches (SW-01 — Exemplo):**

```bash
SW-01(config)# line vty 0 15
SW-01(config-line)# password cisco
SW-01(config-line)# login
SW-01(config-line)# transport input telnet
SW-01(config-line)# exec-timeout 5 0
SW-01(config-line)# exit
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
