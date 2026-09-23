# Guia de Simulação Prática de Laboratório no Cisco Packet Tracer (Versão Corrigida)

> **Nota importante:** este documento corrige os erros de cópia (prompts trocados, IPs duplicados, portas inconsistentes) encontrados na versão original. Antes de colar os comandos, confirma os **nomes reais dos dispositivos e das portas** no teu ficheiro `.pkt`, pois a topologia que montaste usa nomes ligeiramente diferentes (ex.: `PC0-PC4`, `Switch0/Switch1/Switch2`, `Server0`). Os nomes usados abaixo (`Router-01`, `SW-00`, `PC1..PC5`) são os do enunciado original — substitui-os pelos teus conforme necessário.

Este guia apresenta o passo a passo completo para montar e configurar um cenário corporativo simulado no Cisco Packet Tracer, englobando a arquitetura física, segmentação em VLANs, roteamento Inter-VLAN (ROAS), serviços de infraestrutura (DHCP Server e Relay), gestão de interfaces virtuais (SVI) e acessos remotos seguros (SSH e Telnet).

---

## 1. Topologia da Rede e Tabela de Endereçamento

### Dispositivos do Laboratório

- **1 Router:** Router-01 (Cisco 2911) — *usar sempre a notação `Gi0/0` fixa, coerente com os comandos do Tópico 4*
- **1 Switch Principal (Distribuição/Core):** SW-00
- **2 Switches de Acesso:** SW-01 e SW-02
- **1 Servidor Dedicado:** SRV-DHCP
- **5 PCs Clientes:** PC1, PC2, PC3, PC4 e PC5

```
                              [ Router-01 ]
                                   |
                          Gi0/0 (Trunk / ROAS)
                                   |
                             [ SW-00 ]
                        Fa0/1 (trunk, Router)
                        Fa0/2 (access, SRV-DHCP)
                        Fa0/3 (trunk, SW-01)
                        Fa0/4 (trunk, SW-02)
                             /          \
                       [ SW-01 ]      [ SW-02 ]
                       Fa0/1: trunk    Fa0/1: trunk
                       Fa0/2: PC1 (V10) Fa0/2: PC3 (V10)
                       Fa0/3: PC2 (V20) Fa0/3: PC4 (V20)
                                        Fa0/4: PC5 (V30)
```

**Correção aplicada:** o desenho original misturava duas numerações de porta diferentes para o SW-00 (`Gi0/1`/`Gi0/2`/`Fa0/24` no diagrama vs. `Fa0/3`/`Fa0/4`/`Fa0/2` nos comandos). Ficou fixada a numeração usada nos comandos, que é a que realmente se aplica no IOS.

### Mapeamento de VLANs e Sub-redes

> Bloco Base `192.168.10.0/24` subdividido com VLSM / CIDR

| VLAN | Nome da VLAN | Sub-rede IP | Máscara / CIDR | Gateway Padrão (Router-01) | Dispositivos Associados |
|:---:|:---|:---|:---|:---|:---|
| VLAN 10 | Estudantes | 192.168.10.0 | 255.255.255.192 (/26) | 192.168.10.1 | PC1 (Fa0/1), PC3 (Fa0/1) |
| VLAN 20 | Professores | 192.168.10.64 | 255.255.255.192 (/26) | 192.168.10.65 | PC2 (Fa0/2), PC4 (Fa0/2) |
| VLAN 30 | Servidores | 192.168.10.128 | 255.255.255.224 (/27) | 192.168.10.129 | SRV-DHCP (.130), PC5 (Fa0/3) |
| VLAN 99 | Gestão | 192.168.10.160 | 255.255.255.224 (/27) | 192.168.10.161 | SVIs: SW-00 (.162), SW-01 (.163), SW-02 (.164) |

---

## 2. Passo a Passo da Configuração no Cisco IOS

### TÓPICO 1: Comandos Básicos e Segurança Inicial

#### Explicação Técnica

A configuração inicial garante a identificação única do equipamento na rede (`hostname`), estabelece avisos de acesso não autorizado (`banner motd`), protege o acesso físico via porta de consola e cifra todas as credenciais gravadas na memória em texto limpo (`service password-encryption`).

#### Comandos na CLI (Router-01, SW-00, SW-01 e SW-02)
*Para cada dispositivo, alterar apenas o `hostname` conforme o equipamento.*

```bash
enable
configure terminal
```

**1. Alterar o nome do dispositivo**
```
hostname Router-01
```

**2. Mensagem legal de aviso ao ligar**
```
banner motd #Acesso Restrito! Apenas Pessoal Autorizado.#
```

**3. Proteção do Modo EXEC Privilegiado**
```
enable secret cisco123
```

**4. Proteção da Porta de Consola**
```
line console 0
password cisco
login
logging synchronous
exec-timeout 5 0
exit
```

**5. Encriptação geral de palavras-passe em texto limpo**
```
service password-encryption
```

> Repetir estes comandos básicos ajustando apenas o `hostname` em cada switch: `SW-00`, `SW-01`, `SW-02`.

---

### TÓPICO 2: Configuração de VLANs e Modos de Interface (Acesso e Tronco)

#### Explicação Técnica

As VLANs segmentam o domínio de broadcast na Camada 2.

- **Portas de Acesso** (`switchport mode access`): conectam dispositivos finais (PCs, Servidores) e transportam tráfego de apenas uma VLAN, sem etiquetas (*untagged*).
- **Portas Tronco** (`switchport mode trunk`): interconectam switches e routers, transportando o tráfego de múltiplas VLANs através da etiqueta IEEE 802.1Q.

#### 1. No Switch Principal (SW-00) — Criar VLANs e Ativar Troncos

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

**Configurar a porta ligada ao Router-01 como Trunk**
```
SW-00(config)# interface fastEthernet 0/1
SW-00(config-if)# switchport mode trunk
SW-00(config-if)# no shutdown
SW-00(config-if)# exit
```

**Configurar a porta do Servidor DHCP (SRV-DHCP) na VLAN 30**
```
SW-00(config)# interface fastEthernet 0/2
SW-00(config-if)# switchport mode access
SW-00(config-if)# switchport access vlan 30
SW-00(config-if)# no shutdown
SW-00(config-if)# exit
```

**Configurar as ligações com SW-01 e SW-02 como Trunk**
```
SW-00(config)# interface range fastEthernet 0/3 - 4
SW-00(config-if)# switchport mode trunk
SW-00(config-if)# no shutdown
SW-00(config-if)# exit
```

#### 2. No Switch de Acesso SW-01

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

**Trunk na porta Fa0/1 (ligada ao SW-00)**
```
SW-01(config)# interface fastEthernet 0/1
SW-01(config-if)# switchport mode trunk
SW-01(config-if)# no shutdown
SW-01(config-if)# exit
```

**Porta do PC1 (VLAN 10)**
```
SW-01(config)# interface fastEthernet 0/2
SW-01(config-if)# switchport mode access
SW-01(config-if)# switchport access vlan 10
SW-01(config-if)# no shutdown
SW-01(config-if)# exit
```

**Porta do PC2 (VLAN 20)**
```
SW-01(config)# interface fastEthernet 0/3
SW-01(config-if)# switchport mode access
SW-01(config-if)# switchport access vlan 20
SW-01(config-if)# no shutdown
SW-01(config-if)# exit
```

#### 3. No Switch de Acesso SW-02

> **Correção:** na versão original, todo este bloco (VLANs e trunk) usava por engano o prompt `SW-01(...)`, copiado do switch anterior. Abaixo está corrigido para `SW-02(...)`.

```bash
SW-02(config)# vlan 10
SW-02(config-vlan)# name Estudantes
SW-02(config-vlan)# vlan 20
SW-02(config-vlan)# name Professores
SW-02(config-vlan)# vlan 30
SW-02(config-vlan)# name Servidores
SW-02(config-vlan)# vlan 99
SW-02(config-vlan)# name Gestao
SW-02(config-vlan)# exit
```

**Trunk na porta Fa0/1 (ligada ao SW-00)**
```
SW-02(config)# interface fastEthernet 0/1
SW-02(config-if)# switchport mode trunk
SW-02(config-if)# no shutdown
SW-02(config-if)# exit
```

**Porta do PC3 (VLAN 10)**
```
SW-02(config)# interface fastEthernet 0/2
SW-02(config-if)# switchport mode access
SW-02(config-if)# switchport access vlan 10
SW-02(config-if)# no shutdown
SW-02(config-if)# exit
```

**Porta do PC4 (VLAN 20)**
```
SW-02(config)# interface fastEthernet 0/3
SW-02(config-if)# switchport mode access
SW-02(config-if)# switchport access vlan 20
SW-02(config-if)# no shutdown
SW-02(config-if)# exit
```

**Porta do PC5 (VLAN 30)**
```
SW-02(config)# interface fastEthernet 0/4
SW-02(config-if)# switchport mode access
SW-02(config-if)# switchport access vlan 30
SW-02(config-if)# no shutdown
SW-02(config-if)# exit
```

---

### TÓPICO 3: Configuração de Interfaces Virtuais de Switch (SVI) e Default Gateway

#### Explicação Técnica

Uma **SVI** (*Switch Virtual Interface*) é uma interface lógica de Camada 3 configurada dentro do switch que permite atribuir um endereço IP para gestão remota. Como o switch opera na Camada 2, precisa de um *Default Gateway* para responder a pacotes vindos de sub-redes/VLANs diferentes da sua SVI.

> **Correção crítica:** na versão original, os três switches (SW-00, SW-01, SW-02) usavam o mesmo prompt (`SW-00`) e o mesmo IP (`192.168.10.162`), o que causaria **conflito de IP** na VLAN 99. Cada switch tem agora o seu próprio prompt e IP, conforme a tabela do Tópico 1.

**SW-00**
```bash
SW-00(config)# interface vlan 99
SW-00(config-if)# description Interface_Gestao_SW0
SW-00(config-if)# ip address 192.168.10.162 255.255.255.224
SW-00(config-if)# no shutdown
SW-00(config-if)# exit
SW-00(config)# ip default-gateway 192.168.10.161
```

**SW-01**
```bash
SW-01(config)# interface vlan 99
SW-01(config-if)# description Interface_Gestao_SW1
SW-01(config-if)# ip address 192.168.10.163 255.255.255.224
SW-01(config-if)# no shutdown
SW-01(config-if)# exit
SW-01(config)# ip default-gateway 192.168.10.161
```

**SW-02**
```bash
SW-02(config)# interface vlan 99
SW-02(config-if)# description Interface_Gestao_SW2
SW-02(config-if)# ip address 192.168.10.164 255.255.255.224
SW-02(config-if)# no shutdown
SW-02(config-if)# exit
SW-02(config)# ip default-gateway 192.168.10.161
```

> SVI do `SW-00`: `192.168.10.162/27` | SVI do `SW-01`: `192.168.10.163/27` | SVI do `SW-02`: `192.168.10.164/27` | Gateway comum: `192.168.10.161`

---

### TÓPICO 4: Roteamento Inter-VLAN via Router-on-a-Stick (ROAS)

#### Explicação Técnica

O tráfego intra-VLAN ocorre diretamente no switch (Camada 2). Para permitir a comunicação inter-VLAN, utiliza-se a técnica **ROAS** (*Router-on-a-Stick*), onde uma única porta física do router (`Gi0/0`) é conectada a uma porta tronco do switch. A interface física é dividida em sub-interfaces lógicas, cada uma associada à sua VLAN através do enquadramento `encapsulation dot1Q`.

```bash
Router-01(config)# interface gigabitEthernet 0/0
Router-01(config-if)# description Link_Tronco_ROAS_SW-00
Router-01(config-if)# no shutdown
Router-01(config-if)# exit
```

**Sub-interface VLAN 10 (Estudantes)**
```
Router-01(config)# interface gigabitEthernet 0/0.10
Router-01(config-subif)# encapsulation dot1Q 10
Router-01(config-subif)# ip address 192.168.10.1 255.255.255.192
Router-01(config-subif)# exit
```

**Sub-interface VLAN 20 (Professores)**
```
Router-01(config)# interface gigabitEthernet 0/0.20
Router-01(config-subif)# encapsulation dot1Q 20
Router-01(config-subif)# ip address 192.168.10.65 255.255.255.192
Router-01(config-subif)# exit
```

**Sub-interface VLAN 30 (Servidores)**
```
Router-01(config)# interface gigabitEthernet 0/0.30
Router-01(config-subif)# encapsulation dot1Q 30
Router-01(config-subif)# ip address 192.168.10.129 255.255.255.224
Router-01(config-subif)# exit
```

**Sub-interface VLAN 99 (Gestão)**
```
Router-01(config)# interface gigabitEthernet 0/0.99
Router-01(config-subif)# encapsulation dot1Q 99
Router-01(config-subif)# ip address 192.168.10.161 255.255.255.224
Router-01(config-subif)# exit
```

---

### TÓPICO 5: Configuração de Serviço DHCP no Servidor Central (SRV-DHCP)

#### Explicação Técnica

O protocolo DHCP automatiza a distribuição de IPs pelo ciclo **DORA** (*Discover, Offer, Request, ACK*). Em redes empresariais é comum centralizar as *pools* num Servidor DHCP dedicado.

#### Configuração no Servidor Dedicado SRV-DHCP (Packet Tracer GUI)

1. Clique no `SRV-DHCP` → separador **Desktop** → **IP Configuration**:
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
   - Subnet Mask: `255.255.255.192` → clique em **Add**

   **Pool 2 (VLAN 20 - Professores):**
   - Pool Name: `POOL-Professores`
   - Default Gateway: `192.168.10.65`
   - DNS Server: `8.8.8.8`
   - Start IP Address: `192.168.10.75`
   - Subnet Mask: `255.255.255.192` → clique em **Add**

---

### TÓPICO 6: Configuração do Agente de Retransmissão DHCP (`ip helper-address`)

#### Explicação Técnica

Como as mensagens `DHCPDISCOVER` são enviadas em *broadcast*, os routers bloqueiam a sua passagem entre sub-redes. O comando `ip helper-address` transforma o *broadcast* recebido na interface dos clientes num pacote *unicast* direcionado ao IP do Servidor DHCP Central (`192.168.10.130`).

**Aplicar o agente de retransmissão nas sub-interfaces das VLANs clientes**
```
Router-01(config)# interface gigabitEthernet 0/0.10
Router-01(config-subif)# ip helper-address 192.168.10.130
Router-01(config-subif)# exit
```

```
Router-01(config)# interface gigabitEthernet 0/0.20
Router-01(config-subif)# ip helper-address 192.168.10.130
Router-01(config-subif)# exit
```

> A partir deste momento, quando o PC1 ou PC2 solicitarem IP via DHCP, o Router-01 reencaminhará o pedido para o SRV-DHCP na VLAN 30.

---

### TÓPICO 7: Acesso Remoto Seguro (SSH no Router) e Telnet (nos Switches)

#### Explicação Técnica

- **Telnet** (Porta TCP 23): transmite credenciais em texto simples, sem encriptação (configurado nos switches de acesso apenas para fins de teste).
- **SSH** (Porta TCP 22): cifra o tráfego de gestão utilizando chaves RSA (configurado no Router-01).

**1. SSH no Router (Router-01)**
```bash
Router-01(config)# ip domain-name empresa.local
Router-01(config)# username admin secret AdminPass123
Router-01(config)# crypto key generate rsa
% How many bits in the modulus [512]: 1024
Router-01(config)# ip ssh version 2
Router-01(config)# line vty 0 4
Router-01(config-line)# login local
Router-01(config-line)# transport input ssh
Router-01(config-line)# exec-timeout 10 0
Router-01(config-line)# exit
```

**2. Telnet nos Switches (exemplo no SW-01 — repetir no SW-02)**
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

1. **Obtenção Dinâmica de IP (DHCP)**
   - Abra o `PC1` (VLAN 10) → **Desktop** → **IP Configuration** → selecione `DHCP`.
   - Confirme se o IP atribuído pertence à gama `192.168.10.10-.62` com Gateway `192.168.10.1`.

2. **Teste de Comunicação Intra-VLAN**
   - No `PC1`, abra o Prompt de Comando (CMD) e execute:
     ```bash
     ping 192.168.10.X
     ```
     (IP do PC3, pertencente à mesma VLAN 10)

3. **Teste de Roteamento Inter-VLAN (ROAS)**
   - No `PC1`, execute o ping para o PC2 (VLAN 20):
     ```bash
     ping 192.168.10.75
     ```

4. **Teste de Acesso Remoto Telnet ao Switch**
   - No `PC1`, aceda ao switch de acesso via Telnet:
     ```bash
     telnet 192.168.10.163
     ```
     (Insira a palavra-passe `cisco`)

5. **Teste de Acesso Remoto Seguro SSH ao Router**
   - No `PC1`, ligue-se ao Router via SSH:
     ```bash
     ssh -l admin 192.168.10.1
     ```
     (Insira a palavra-passe `AdminPass123`)

---

## 4. Resumo das Correções Feitas

| Local | Problema na versão original | Correção |
|:---:|:---:|:---:|
| Tópico 2, SW-02 | Prompts usavam `SW-01(...)` em vez de `SW-02(...)` | Prompts corrigidos para `SW-02` |
| Tópico 3, SVIs | SW-01 e SW-02 repetiam o prompt e o IP do SW-00 (`.162`) | Cada switch com prompt e IP próprios: `.162`, `.163`, `.164` |
| Seção 1, diagrama | Portas do SW-00 inconsistentes com os comandos (`Gi0/1`/`Gi0/2`/`Fa0/24` vs. `Fa0/1`–`Fa0/4`) | Diagrama alinhado com as portas realmente usadas nos comandos |
| Seção 1, modelo do Router | Mistura de notação modular (`Gi0/0/0`) com notação fixa (`Gi0/0`) | Fixado o modelo Cisco 2911 e a notação `Gi0/0` |
| Tópico 7, SSH | `crypto key generate rsa 1024` (sintaxe inválida no IOS) | Separado o comando do prompt de nº de bits (`% How many bits...`) |
