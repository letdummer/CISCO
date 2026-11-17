# Guia Unificado — Cenário Cisco Packet Tracer
**VLANs · Router-on-a-Stick · DHCP Relay · SSH · Port Security · DHCP Snooping · DAI · AAA/802.1X**

> Guia único, organizado **por equipamento e por ordem prática de execução**. Em cada passo indica-se claramente **em que equipamento** a configuração é feita. Quando a mesma configuração se repete noutro equipamento, é apresentada num bloco colapsado pronto a **copiar e colar**.

---

## Índice

1. [Cenário: topologia e endereçamento](#1-cenário)
2. [Ordem de configuração](#2-ordem)
3. [Switch0 (central)](#3-switch0)
4. [Switch1 (acesso)](#4-switch1)
5. [Switch2 (acesso)](#5-switch2)
6. [Router0 (ROAS + DHCP Relay)](#6-router0)
7. [Server-DHCP (GUI)](#7-server)
8. [PCs (DHCP)](#8-pcs)
9. [Port Security](#9-port-security)
10. [DHCP Snooping](#10-dhcp-snooping)
11. [Dynamic ARP Inspection (DAI)](#11-dai)
12. [AAA + 802.1X](#12-aaa-8021x)
13. [Testes finais](#13-testes)
14. [Checklist](#14-checklist)

---



<a id="1-cenário"></a>
## 1. Cenário: topologia e endereçamento

### » Dispositivos

| Dispositivo | Função |
|:---|:---|
| `Switch0` | Switch central (distribuição) — liga Switch1, Switch2, Router0, Server-DHCP e PC-Gestao |
| `Switch1` | Switch de acesso — liga PC0, PC1, PC2 |
| `Switch2` | Switch de acesso — liga PC3, PC4, PC5 |
| `Router0` | Router-on-a-Stick — faz o routing inter-VLAN |
| `Server-DHCP` | Servidor DHCP na **VLAN 10** (`192.168.10.254`) para todas as sub-redes |
| `PC-Gestao` | PC de gestão na **VLAN 40** (rede de gestão) |
| `PC0`–`PC5` | Terminais dos utilizadores (todos por DHCP) |

### » Diagrama

```
                      [ Router0 ]
                        (Gi0/0)        
                          |       -(trunk / ROAS)        
                          |    
                       (Fa0/1) 
                     [ Switch0 ]         -(central)
            (Fa0/2)          (Fa0/3)            (Fa0/24)              (Fa0/10) 
               /               |                  \                    \
              /[trunk]         |[trunk]            \[acesso]            \[acesso]
             /                 |                    \                    \
        (Fa0/24)             (Fa0/24)               (Fa0)                (Fa0)
      [ Switch1 ]           [ Switch2 ]           [ Server-DHCP ]     [ PC-Gestao ] 
   Fa0/1: PC0 (V10)       Fa0/1: PC3 (V10)
   Fa0/2: PC1 (V20)       Fa0/2: PC4 (V20)
   Fa0/3: PC2 (V30)       Fa0/3: PC5 (V30)
```

> 📌 Todos os links trunk (Router0↔Switch0, Switch0↔Switch1, Switch0↔Switch2) usam a **VLAN 40 como VLAN nativa** (tráfego sem tag 802.1Q).

### » VLANs e sub-redes (APENAS estas 4 VLANs)

| VLAN | Nome | Sub-rede | Gateway (`Router0`) |
|:---:|:---|:---|:---|
| `10` | Estudantes | `192.168.10.0`/24 | `192.168.10.1` |
| `20` | Professores | `192.168.20.0`/24 | `192.168.20.1` |
| `30` | Direcao | `192.168.30.0`/24 | `192.168.30.1` |
| `40` | Native / Gestão | `192.168.40.0`/24 | `192.168.40.1` *(sub-if nativa)* |

> 📌 **Nota (VLAN 40):** é a **VLAN nativa** dos trunks (tráfego sem tag, ex.: CDP/STP) e também a **VLAN de gestão** (SVIs dos switches + PC-Gestao). No `Router0`, a sub-interface `Gi0/0.40` é criada com `encapsulation dot1Q 40 native`. A VLAN 1 deixa de ser usada como nativa (boa prática). Não tem PCs de utilizador nem pool DHCP.

### » IPs estáticos e SVIs

| Equipamento | IP | Gateway | Porta / Interface |
|:---|:---|:---|:---|
| `Server-DHCP` | `192.168.10.254`/24 | `192.168.10.1` | Fa0/24 do Switch0 (VLAN 10) |
| `PC-Gestao` | `192.168.40.10`/24 | `192.168.40.1` | Fa0/10 do Switch0 (VLAN 40) |
| SVI `Switch0` | `192.168.40.2`/24 | `192.168.40.1` | `interface vlan 40` |
| SVI `Switch1` | `192.168.40.3`/24 | `192.168.40.1` | `interface vlan 40` |
| SVI `Switch2` | `192.168.40.4`/24 | `192.168.40.1` | `interface vlan 40` |

### » PCs (todos por DHCP)

| PC | Switch | Porta | VLAN | IP obtido |
|:---:|:---:|:---:|:---:|:---|
| `PC0` | Switch1 | Fa0/1 | 10 | `192.168.10.x` |
| `PC1` | Switch1 | Fa0/2 | 20 | `192.168.20.x` |
| `PC2` | Switch1 | Fa0/3 | 30 | `192.168.30.x` |
| `PC3` | Switch2 | Fa0/1 | 10 | `192.168.10.x` |
| `PC4` | Switch2 | Fa0/2 | 20 | `192.168.20.x` |
| `PC5` | Switch2 | Fa0/3 | 30 | `192.168.30.x` |

> ⚠️ **Confirma com o teu enunciado:** os comandos deste guia atribuem Fa0/1→V10, Fa0/2→V20, Fa0/3→V30 **nos dois** switches de acesso (para poderes copiar/colar sem alterações). Se o enunciado pedir VLANs diferentes por porta no Switch2, basta trocar o número no `switchport access vlan X` dessa porta.

---

<a id="2-ordem"></a>
## 2. Ordem de configuração

| Passo | Equipamento | O quê |
|:---:|:---|:---|
| 1 | **Switch0** | Senhas, VLANs, trunks, portas de acesso (PC-Gestao + Server-DHCP), fechar portas, SVI, SSH |
| 2 | **Switch1** | Igual ao Switch0, sem as portas Fa0/10 e Fa0/24; portas de acesso aos PCs |
| 3 | **Switch2** | Igual ao Switch1 — mudam apenas o hostname e o IP da SVI |
| 4 | **Router0** | Sub-interfaces ROAS + DHCP Relay |
| 5 | **Server-DHCP** | IP estático + pools DHCP (interface gráfica) |
| 6 | **PCs** | Ativar DHCP em cada PC |
| 7 | **Testes finais** | Ping, SSH, verificações |
| 8 | **Port Security** | Portas dos PCs: 1 MAC por porta, sticky, violação shutdown (Secção 9) |
| 9 | **DHCP Snooping** | Todos os switches: trunks + porta do servidor trusted (Secção 10) |
| 10 | **DAI** | Depende do DHCP Snooping + PCs já com IP (Secção 11) |
| 11 | **AAA + 802.1X** | Em último lugar: porta bloqueia até autenticar no RADIUS (Secção 12) |

> 📌 **Regra de ouro:** termina sempre cada equipamento com `write memory`.

---

<a id="3-switch0"></a>
## 3. Switch0 (Switch Central)

> Toda a configuração desta secção é feita **no `Switch0`**, na aba **CLI**.

### 3.1 — [SW0] Hostname e senhas

```
enable
configure terminal
hostname Switch0

! -> bloqueia o modo privilegiado (enable)
enable secret cinel

! -> encripta senhas em texto claro no running-config
service password-encryption

! -> aviso apresentado antes de qualquer login
banner motd #Acesso restrito!#

! -> bloqueia o acesso fisico pela porta de consola
line console 0
	password cisco
	login
	exec-timeout 5 0
	logging synchronous
	exit
```

**Verificar:** `show running-config` 
— procura `enable secret 5 ...` e `password 7 ...` (cifradas). 
Depois faz `exit` e volta a entrar: a consola pede a senha `cisco`, e `enable` pede `cinel`.

### 3.2 — [SW0] Criar as VLANs (10, 20, 30 e 40)

```
vlan 10
 name Estudantes
vlan 20
 name Professores
vlan 30
 name Direcao
vlan 40
 name Native
exit
```

**Verificar:** `show vlan brief` — as VLANs 10, 20, 30 e 40 devem aparecer com os nomes corretos.

### # VTP
<details><summary><b>Alternativa: distribuir VLANs por VTP (opcional — clique para abrir)</b></summary>

> ⚠️ Se optares por VTP, a ordem muda: primeiro configura os **trunks** (passo 3.3), só depois o VTP.

**VTP Server (Switch0):**
```
vtp mode server
vtp domain cinel-domain
vtp password cinel
```
Cria as VLANs (bloco acima) **apenas no Switch0**.

**VTP Client (Switch1 e Switch2):**
```
vtp domain cinel-domain
vtp mode client
vtp password cinel
exit
write memory
```

**Verificar:** `show vtp status` · `show vlan brief` (nos clients, as VLANs devem aparecer sem as criares).

</details>

### 3.3 — [SW0] Trunks  com VLAN nativa 40

**SW0 liga a estas portas:**
Fa0/1 → Router0
Fa0/2 → Switch1
Fa0/3 → Switch2

```
interface range fastEthernet 0/1-3
  switchport mode trunk
  switchport trunk allowed vlan 10,20,30,40
  switchport trunk native vlan 40
  switchport nonegotiate
  no shutdown
  exit
```

> ⚠️ A VLAN nativa tem de ser **igual nas duas pontas** de cada trunk, senão o STP/CDP reporta *native VLAN mismatch*.

**Verificar:**
```
show interfaces trunk
```
> ✅ Na coluna **Native vlan** deve aparecer `40` nas portas Fa0/1-3, e `10,20,30,40` em *VLANs allowed and active in management domain*.

### 3.4 — [SW0] Portas de acesso

PC-Gestao: Fa0/10, VLAN 40
Server-DHCP: Fa0/24, VLAN 10

```
interface fastEthernet 0/10
  description PC-Gestao
  switchport mode access
  switchport access vlan 40
  no shutdown
  exit

interface fastEthernet 0/24
  description Server-DHCP
  switchport mode access
  switchport access vlan 10
  no shutdown
  exit
```

**Verificar:** `show vlan brief` 
— Fa0/10 deve constar na VLAN 40 e Fa0/24 na VLAN 10. Se aparecerem na VLAN 1 (*default*), a configuração não foi aplicada.

### 3.5 — [SW0] Fechar as portas não utilizadas

Portas em uso no Switch0: `Fa0/1-3` (trunks), `Fa0/10` (PC-Gestao) e `Fa0/24` (Server-DHCP).

```
interface range fastEthernet 0/4-9, fastEthernet 0/11-23, gigabitEthernet 0/1-2
  switchport mode access
  shutdown
  exit
```

**Verificar:** `show ip interface brief` 
— as portas fechadas devem mostrar `administratively down`.


### 3.6 — [SW0] SVI de gestão (interface vlan 40)

Permite gerir o switch remotamente (SSH/ping) a partir da rede de gestão.

```
interface vlan 40
  ip address 192.168.40.2 255.255.255.0
  no shutdown
  exit

ip default-gateway 192.168.40.1
```

**Verificar:**
```
show ip interface brief | include Vlan40
ping 192.168.40.1
```
> ✅ A `Vlan40` deve estar `up/up` e o ping para o router deve responder.

### 3.7 — [SW0] SSH (acesso remoto seguro)

```
ip domain-name cinel.local
crypto key generate rsa
2048
ip ssh version 2

! -> bloqueia o acesso remoto SSH (linhas VTY)
username admin privilege 15 secret cinel

! ACL: so permite SSH a partir da rede de gestao (VLAN 40)
access-list 10 permit 192.168.40.0 0.0.0.255

line vty 0 4
  login local
  transport input ssh
  access-class 10 in
  exec-timeout 5 0
  logging synchronous
  exit

end
write memory
```

> 📌 Se o exercício exigir Telnet também, muda para `transport input ssh telnet` (Telnet não é encriptado — usar só em laboratório).

**Verificar:** `show ip ssh` 
— deve indicar `SSH version 2` e autenticação ativada.

### 3.8 — [SW0] Bloco completo do Switch0 (copiar e colar de uma vez)

<details><summary><b>Script completo do Switch0 — clique para abrir</b></summary>

```
enable
configure terminal
hostname Switch0
enable secret cinel
service password-encryption
banner motd #Acesso restrito!#
line console 0
 password cisco
 login
 exec-timeout 5 0
 logging synchronous
 exit
vlan 10
 name Estudantes
vlan 20
 name Professores
vlan 30
 name Direcao
vlan 40
 name Native
exit
interface range fastEthernet 0/1-3
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 switchport trunk native vlan 40
 switchport nonegotiate
 no shutdown
 exit
interface fastEthernet 0/10
 description PC-Gestao
 switchport mode access
 switchport access vlan 40
 no shutdown
 exit
interface fastEthernet 0/24
 description Server-DHCP
 switchport mode access
 switchport access vlan 10
 no shutdown
 exit
interface range fastEthernet 0/4-9, fastEthernet 0/11-23, gigabitEthernet 0/1-2
 switchport mode access
 shutdown
 exit
interface vlan 40
 ip address 192.168.40.2 255.255.255.0
 no shutdown
 exit
ip default-gateway 192.168.40.1
ip domain-name cinel.local
crypto key generate rsa
2048
ip ssh version 2
username admin privilege 15 secret cinel
access-list 10 permit 192.168.40.0 0.0.0.255
line vty 0 4
 login local
 transport input ssh
 access-class 10 in
 exec-timeout 5 0
 logging synchronous
 exit
end
write memory
```

</details>

---

<a id="4-switch1"></a>
## 4. Switch1 (Switch de Acesso)

> Toda a configuração desta secção é feita **no `Switch1`**, na aba **CLI**.

### 4.1 — [SW1] Hostname e senhas

```
enable
configure terminal
hostname Switch1

! -> bloqueia o modo privilegiado (enable)
enable secret cinel

! -> encripta senhas em texto claro no running-config
service password-encryption

! -> aviso apresentado antes de qualquer login
banner motd #Acesso restrito!#

! -> bloqueia o acesso fisico pela porta de consola
line console 0
 password cisco
 login
 exec-timeout 5 0
 logging synchronous
 exit
```

**Verificar:** `show running-config` 
— procura `enable secret 5 ...` e `password 7 ...` (cifradas). 
Depois faz `exit` e volta a entrar: a consola pede a senha `cisco`, e `enable` pede `cinel`.

### 4.2 — [SW1] Criar as VLANs (10, 20, 30 e 40)

> Com criação manual, as VLANs têm de existir **em cada switch**. (Se usaste VTP no Switch0, salta este passo — o Switch1 recebe-as como client.)

```
vlan 10
 name Estudantes
vlan 20
 name Professores
vlan 30
 name Direcao
vlan 40
 name Native
exit
```

**Verificar:** `show vlan brief` — as VLANs 10, 20, 30 e 40 devem aparecer com os nomes corretos.

### # VTP
<details><summary><b>Alternativa: receber as VLANs por VTP (opcional — clique para abrir)</b></summary>

> ⚠️ Se optares por VTP, a ordem muda: primeiro configura o **trunk** (passo 4.3), só depois o VTP.

**VTP Client (Switch1):**
```
vtp domain cinel-domain
vtp mode client
vtp password cinel
exit
write memory
```

**Verificar:** `show vtp status` · `show vlan brief` (as VLANs devem aparecer sem as criares).

</details>

### 4.3 — [SW1] Trunk para o Switch0 (Fa0/24) com VLAN nativa 40

**SW1 liga a esta porta:**
Fa0/24 → Switch0

```
interface fastEthernet 0/24
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 switchport trunk native vlan 40
 switchport nonegotiate
 no shutdown
 exit
```

**Verificar:**
```
show interfaces trunk
```
> ✅ Fa0/24 deve aparecer `trunking`, 802.1Q, **Native vlan: 40**.

### 4.4 — [SW1] Portas de acesso aos PCs

PC0: Fa0/1, VLAN 10
PC1: Fa0/2, VLAN 20
PC2: Fa0/3, VLAN 30

```
interface fastEthernet 0/1
 description PC0
 switchport mode access
 switchport access vlan 10
 no shutdown
 exit

interface fastEthernet 0/2
 description PC1
 switchport mode access
 switchport access vlan 20
 no shutdown
 exit

interface fastEthernet 0/3
 description PC2
 switchport mode access
 switchport access vlan 30
 no shutdown
 exit
```

**Verificar:** `show vlan brief` 
— Fa0/1 deve constar na VLAN 10, Fa0/2 na VLAN 20 e Fa0/3 na VLAN 30. Se aparecerem na VLAN 1 (*default*), a configuração não foi aplicada.

### 4.5 — [SW1] Fechar as portas não utilizadas

Portas em uso no Switch1: `Fa0/1-3` (PCs) e `Fa0/24` (trunk).

```
interface range fastEthernet 0/4-23, gigabitEthernet 0/1-2
 switchport mode access
 shutdown
 exit
```

**Verificar:** `show ip interface brief` — portas fechadas em `administratively down`.

### 4.6 — [SW1] SVI de gestão (interface vlan 40)

Permite gerir o switch remotamente (SSH/ping) a partir da rede de gestão.

```
interface vlan 40
 ip address 192.168.40.3 255.255.255.0
 no shutdown
 exit
ip default-gateway 192.168.40.1
```

**Verificar:** `show ip interface brief | include Vlan40` · `ping 192.168.40.1`

### 4.7 — [SW1] SSH (acesso remoto seguro)

```
ip domain-name cinel.local
crypto key generate rsa
2048
ip ssh version 2

! -> bloqueia o acesso remoto SSH (linhas VTY)
username admin privilege 15 secret cinel

! ACL: so permite SSH a partir da rede de gestao (VLAN 40)
access-list 10 permit 192.168.40.0 0.0.0.255

line vty 0 4
 login local
 transport input ssh
 access-class 10 in
 exec-timeout 5 0
 logging synchronous
 exit

end
write memory
```

> 📌 Se o exercício exigir Telnet também, muda para `transport input ssh telnet` (Telnet não é encriptado — usar só em laboratório).

**Verificar:** `show ip ssh` 
— deve indicar `SSH version 2` e autenticação ativada.

### 4.8 — [SW1] Bloco completo do Switch1 (copiar e colar de uma vez)

<details><summary><b>Script completo do Switch1 — clique para abrir</b></summary>

```
enable
configure terminal
hostname Switch1
enable secret cinel
service password-encryption
banner motd #Acesso restrito!#
line console 0
 password cisco
 login
 exec-timeout 5 0
 logging synchronous
 exit
vlan 10
 name Estudantes
vlan 20
 name Professores
vlan 30
 name Direcao
vlan 40
 name Native
exit
interface fastEthernet 0/24
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 switchport trunk native vlan 40
 switchport nonegotiate
 no shutdown
 exit
interface fastEthernet 0/1
 description PC0
 switchport mode access
 switchport access vlan 10
 no shutdown
 exit
interface fastEthernet 0/2
 description PC1
 switchport mode access
 switchport access vlan 20
 no shutdown
 exit
interface fastEthernet 0/3
 description PC2
 switchport mode access
 switchport access vlan 30
 no shutdown
 exit
interface range fastEthernet 0/4-23, gigabitEthernet 0/1-2
 switchport mode access
 shutdown
 exit
interface vlan 40
 ip address 192.168.40.3 255.255.255.0
 no shutdown
 exit
ip default-gateway 192.168.40.1
ip domain-name cinel.local
crypto key generate rsa
2048
ip ssh version 2
username admin privilege 15 secret cinel
access-list 10 permit 192.168.40.0 0.0.0.255
line vty 0 4
 login local
 transport input ssh
 access-class 10 in
 exec-timeout 5 0
 logging synchronous
 exit
end
write memory
```

</details>

---

<a id="5-switch2"></a>
## 5. Switch2 (Switch de Acesso)

> Toda a configuração desta secção é feita **no `Switch2`**, na aba **CLI**. É **idêntica ao Switch1** — mudam apenas o **hostname** e o **IP da SVI** (`.4` em vez de `.3`).

### 5.1 — [SW2] Hostname e senhas

```
enable
configure terminal
hostname Switch2

! -> bloqueia o modo privilegiado (enable)
enable secret cinel

! -> encripta senhas em texto claro no running-config
service password-encryption

! -> aviso apresentado antes de qualquer login
banner motd #Acesso restrito!#

! -> bloqueia o acesso fisico pela porta de consola
line console 0
 password cisco
 login
 exec-timeout 5 0
 logging synchronous
 exit
```

**Verificar:** `show running-config` 
— procura `enable secret 5 ...` e `password 7 ...` (cifradas). 
Depois faz `exit` e volta a entrar: a consola pede a senha `cisco`, e `enable` pede `cinel`.

### 5.2 — [SW2] Criar as VLANs (10, 20, 30 e 40)

> Com criação manual, as VLANs têm de existir **em cada switch**. (Se usaste VTP no Switch0, salta este passo — o Switch2 recebe-as como client.)

```
vlan 10
 name Estudantes
vlan 20
 name Professores
vlan 30
 name Direcao
vlan 40
 name Native
exit
```

**Verificar:** `show vlan brief` — as VLANs 10, 20, 30 e 40 devem aparecer com os nomes corretos.

### # VTP
<details><summary><b>Alternativa: receber as VLANs por VTP (opcional — clique para abrir)</b></summary>

> ⚠️ Se optares por VTP, a ordem muda: primeiro configura o **trunk** (passo 5.3), só depois o VTP.

**VTP Client (Switch2):**
```
vtp domain cinel-domain
vtp mode client
vtp password cinel
exit
write memory
```

**Verificar:** `show vtp status` · `show vlan brief` (as VLANs devem aparecer sem as criares).

</details>

### 5.3 — [SW2] Trunk para o Switch0 (Fa0/24) com VLAN nativa 40

**SW2 liga a esta porta:**
Fa0/24 → Switch0

```
interface fastEthernet 0/24
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 switchport trunk native vlan 40
 switchport nonegotiate
 no shutdown
 exit
```

**Verificar:**
```
show interfaces trunk
```
> ✅ Fa0/24 deve aparecer `trunking`, 802.1Q, **Native vlan: 40**.

### 5.4 — [SW2] Portas de acesso aos PCs

PC3: Fa0/1, VLAN 10
PC4: Fa0/2, VLAN 20
PC5: Fa0/3, VLAN 30

```
interface fastEthernet 0/1
 description PC3
 switchport mode access
 switchport access vlan 10
 no shutdown
 exit

interface fastEthernet 0/2
 description PC4
 switchport mode access
 switchport access vlan 20
 no shutdown
 exit

interface fastEthernet 0/3
 description PC5
 switchport mode access
 switchport access vlan 30
 no shutdown
 exit
```

> ⚠️ **Confirma com o teu enunciado:** se os PCs do Switch2 tiverem de ficar noutras VLANs (ex.: PC3 na VLAN 30), altera o `switchport access vlan X` da porta respetiva **antes** de fechar com `write memory`.

**Verificar:** `show vlan brief` 
— Fa0/1 deve constar na VLAN 10, Fa0/2 na VLAN 20 e Fa0/3 na VLAN 30. Se aparecerem na VLAN 1 (*default*), a configuração não foi aplicada.

### 5.5 — [SW2] Fechar as portas não utilizadas

Portas em uso no Switch2: `Fa0/1-3` (PCs) e `Fa0/24` (trunk).

```
interface range fastEthernet 0/4-23, gigabitEthernet 0/1-2
 switchport mode access
 shutdown
 exit
```

**Verificar:** `show ip interface brief` — portas fechadas em `administratively down`.

### 5.6 — [SW2] SVI de gestão (interface vlan 40)

Permite gerir o switch remotamente (SSH/ping) a partir da rede de gestão.

```
interface vlan 40
 ip address 192.168.40.4 255.255.255.0
 no shutdown
 exit
ip default-gateway 192.168.40.1
```

**Verificar:** `show ip interface brief | include Vlan40` · `ping 192.168.40.1`

### 5.7 — [SW2] SSH (acesso remoto seguro)

```
ip domain-name cinel.local
crypto key generate rsa
2048
ip ssh version 2

! -> bloqueia o acesso remoto SSH (linhas VTY)
username admin privilege 15 secret cinel

! ACL: so permite SSH a partir da rede de gestao (VLAN 40)
access-list 10 permit 192.168.40.0 0.0.0.255

line vty 0 4
 login local
 transport input ssh
 access-class 10 in
 exec-timeout 5 0
 logging synchronous
 exit

end
write memory
```

> 📌 Se o exercício exigir Telnet também, muda para `transport input ssh telnet` (Telnet não é encriptado — usar só em laboratório).

**Verificar:** `show ip ssh` 
— deve indicar `SSH version 2` e autenticação ativada. A partir do `PC-Gestao`: `ssh -l admin 192.168.40.4` (senha `cinel`).

### 5.8 — [SW2] Bloco completo do Switch2 (copiar e colar de uma vez)

<details><summary><b>Script completo do Switch2 — clique para abrir</b></summary>

```
enable
configure terminal
hostname Switch2
enable secret cinel
service password-encryption
banner motd #Acesso restrito!#
line console 0
 password cisco
 login
 exec-timeout 5 0
 logging synchronous
 exit
vlan 10
 name Estudantes
vlan 20
 name Professores
vlan 30
 name Direcao
vlan 40
 name Native
exit
interface fastEthernet 0/24
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
 switchport trunk native vlan 40
 switchport nonegotiate
 no shutdown
 exit
interface fastEthernet 0/1
 description PC3
 switchport mode access
 switchport access vlan 10
 no shutdown
 exit
interface fastEthernet 0/2
 description PC4
 switchport mode access
 switchport access vlan 20
 no shutdown
 exit
interface fastEthernet 0/3
 description PC5
 switchport mode access
 switchport access vlan 30
 no shutdown
 exit
interface range fastEthernet 0/4-23, gigabitEthernet 0/1-2
 switchport mode access
 shutdown
 exit
interface vlan 40
 ip address 192.168.40.4 255.255.255.0
 no shutdown
 exit
ip default-gateway 192.168.40.1
ip domain-name cinel.local
crypto key generate rsa
2048
ip ssh version 2
username admin privilege 15 secret cinel
access-list 10 permit 192.168.40.0 0.0.0.255
line vty 0 4
 login local
 transport input ssh
 access-class 10 in
 exec-timeout 5 0
 logging synchronous
 exit
end
write memory
```

</details>

**Verificação final no Switch2:**
```
show vlan brief
show interfaces trunk
show ip interface brief
```

---

<a id="6-router0"></a>
## 6. Router0 (ROAS + DHCP Relay)

> Toda a configuração desta secção é feita **no `Router0`**, na aba **CLI**.

> 📌 **Porquê o DHCP Relay:** o Server-DHCP (`192.168.10.254`) está fisicamente na VLAN 10. Os pedidos DHCP (broadcasts) das VLANs 20 e 30 são descartados pelo router por omissão. O comando `ip helper-address 192.168.10.254` converte esses broadcasts em **unicast** direcionados ao servidor.

### 6.1 — [R0] Ativar a interface física, hostname e senhas

```
enable
configure terminal
hostname Router0

! -> bloqueia o modo privilegiado (enable)
enable secret cinel

! -> encripta senhas em texto claro no running-config
service password-encryption

! -> aviso apresentado antes de qualquer login
banner motd #Acesso restrito!#

! -> bloqueia o acesso fisico pela porta de consola
line console 0
 password cisco
 login
 exec-timeout 5 0
 logging synchronous
 exit

! -> interface fisica sem IP (o IP fica nas sub-interfaces)
interface gigabitEthernet 0/0
 no shutdown
 exit
```

**Verificar:** `show ip interface brief` — Gi0/0 deve aparecer `up/up` (sem IP address).

### 6.2 — [R0] Sub-interfaces (gateway de cada VLAN)

**VLAN 10 (Estudantes) — rede do servidor DHCP, sem helper:**
```
interface gigabitEthernet 0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 exit
```

**VLAN 20 (Professores) — com DHCP Relay:**
```
interface gigabitEthernet 0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip helper-address 192.168.10.254
 exit
```

**VLAN 30 (Direcao) — com DHCP Relay:**
```
interface gigabitEthernet 0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip helper-address 192.168.10.254
 exit
```

**VLAN 40 (Native/Gestao) — sem DHCP Relay (não há PCs):**
```
interface gigabitEthernet 0/0.40
 encapsulation dot1Q 40 native
 ip address 192.168.40.1 255.255.255.0
 exit
end
write memory
```

> 📌 A palavra-chave `native` na sub-interface indica que o tráfego da VLAN 40 circula **sem tag**, em coerência com `switchport trunk native vlan 40` nos switches. Não leva `ip helper-address` porque não tem PCs.

**Verificar:**
```
show ip interface brief
show running-config | include helper
```
> ✅ Todas as sub-interfaces (Gi0/0, .10, .20, .30, .40) devem estar `up/up`, e o helper deve aparecer nas .20 e .30.

### 6.3 — [R0] SSH no Router0 (opcional — se o exercício pedir)

<details><summary><b>Script SSH do Router0 — clique para abrir</b></summary>

```
configure terminal
ip domain-name cinel.local
crypto key generate rsa
2048
ip ssh version 2
username admin privilege 15 secret cinel
line vty 0 4
 login local
 transport input ssh
 exec-timeout 5 0
 logging synchronous
 exit
end
write memory
```

**Testar (a partir do PC-Gestao):** `ssh -l admin 192.168.40.1` · senha: `cinel`

</details>

---

<a id="7-server"></a>
## 7. Server-DHCP (interface gráfica)

> Configurado **no `Server-DHCP`**, clicando no dispositivo. O cabo liga-o à Fa0/24 do Switch0 (VLAN 10 — já configurada no passo 3.4).

**8.1 — IP estático do servidor:** `Desktop → IP Configuration`

| IP Address | Subnet Mask | Default Gateway |
|:---:|:---:|:---:|
| `192.168.10.254` | `255.255.255.0` | `192.168.10.1` |

**8.2 — Ativar o serviço e criar os pools:** `Services → DHCP` → **ON**

> Cada pool serve a sub-rede da VLAN respetiva; o router encaminha os pedidos das VLANs 20/30 via `ip helper-address`.

| Pool Name | Default Gateway | Start IP Address | Subnet Mask |
|:---:|:---:|:---:|:---:|
| `VLAN10` | `192.168.10.1` | `192.168.10.50` | `255.255.255.0` |
| `VLAN20` | `192.168.20.1` | `192.168.20.50` | `255.255.255.0` |
| `VLAN30` | `192.168.30.1` | `192.168.30.50` | `255.255.255.0` |

> A VLAN 40 **não tem pool** — não existe nela nenhum PC de utilizador.

**Verificar:** nos passos 8 e 13, os PCs devem obter IP da sub-rede correta.

---

<a id="8-pcs"></a>
## 8. PCs (DHCP)

> Em **cada PC** (PC0 a PC5): `Desktop → IP Configuration` → ativar **DHCP**.

> ✅ Cada PC deve obter automaticamente o IP da sub-rede da **sua VLAN** (tabela da Secção 1), gateway incluído. Verifica no painel: o IP deve começar por `192.168.<vlan>.` e o gateway deve ser `192.168.<vlan>.1`.

---

<a id="9-port-security"></a>
## 9. Port Security

> Protege as **portas de acesso**: limita quantos endereços MAC podem transmitir numa porta (por defeito, 1). Impede que um utilizador ligue um switch/hub escondido ou falsifique MACs. Se um MAC não autorizado aparecer, a porta entra em **violação**.
>
> **Configurar em:** `Switch1` e `Switch2` (portas dos PCs — o mesmo bloco). Opcional no `Switch0` (Fa0/10 e Fa0/24).

### 9.1 — [SW1] Port Security nas portas dos PCs

> Requisito: a porta já tem de estar em modo **access** (passo 4.4) — o port-security não funciona em trunks.

```
interface range fastEthernet 0/1-3
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation shutdown
 switchport port-security mac-address sticky
 exit
end
write memory
```

> 📌 **O que cada comando faz:**
> - `switchport port-security` — ativa a segurança de porta (sem isto, nada mais se aplica).
> - `maximum 1` — só 1 MAC é permitido (o valor por defeito).
> - `violation shutdown` — em violação, a porta fica desativada (error-disabled) e o LED apaga.
> - `mac-address sticky` — o MAC aprendido é gravado na running-config (persistente com `write memory`; sem `sticky`, o MAC é só dinâmico e perde-se ao reiniciar).

**Modos de violação (alternativa ao `shutdown`):**

| Modo | Comportamento |
|:---|:---|
| `shutdown` | (Por defeito) porta desativada por erro — LED apaga. Para recuperar: `shutdown` + `no shutdown` |
| `restrict` | Descarta os pacotes do MAC desconhecido e regista a violação (log + LED) |
| `protect` | Descarta os pacotes sem qualquer registo ou notificação |

**Verificar:**
```
show port-security
show port-security interface fa0/1
show port-security address
```
> ✅ Fa0/1-3 devem aparecer `Secure-up` (ou `Secure-shutdown` se houve violação), modo `Shutdown`, e o MAC `sticky` deve constar nas entradas.

**Se uma porta ficou em violação (error-disabled):**
```
interface fastEthernet 0/1
 shutdown
 no shutdown
 exit
```

<details><summary><b>Script do Switch2 — idêntico ao Switch1, clique para abrir (copiar e colar)</b></summary>

```
interface range fastEthernet 0/1-3
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation shutdown
 switchport port-security mac-address sticky
 exit
end
write memory
```

</details>

<details><summary><b>Script do Switch0 — portas Fa0/10 e Fa0/24 (opcional, clique para abrir)</b></summary>

```
interface range fastEthernet 0/10, fastEthernet 0/24
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation shutdown
 switchport port-security mac-address sticky
 exit
end
write memory
```

</details>

---

<a id="10-dhcp-snooping"></a>
## 10. DHCP Snooping

> Filtra as mensagens DHCP nas portas **untrusted** (todas, por defeito). Bloqueia:
> - **DHCP starvation** — atacante inunda o servidor com pedidos DHCP falsos até esgotar o pool (negado de serviço aos restantes PCs);
> - **DHCP poisoning** — um servidor DHCP ilegítimo responde aos clientes e atribui-lhes o gateway falso dele (ataque man-in-the-middle).
>
> **Configurar em TODOS os switches:** `Switch0`, `Switch1` e `Switch2`.
> **Regra:** as portas **uplink/trunk** e a porta do servidor DHCP são **trusted**; as portas dos PCs ficam **untrusted** (por defeito — só se indica o que é trusted).

### 10.1 — [SW0] DHCP Snooping (Switch Central)

```
configure terminal
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40
no ip dhcp snooping information option

! -> trunks (Router0, Switch1, Switch2): passam mensagens DHCP sem inspeção
interface range fastEthernet 0/1-3
 ip dhcp snooping trust
 exit

! -> porta do Server-DHCP (192.168.10.254): tem de ser trusted
interface fastEthernet 0/24
 ip dhcp snooping trust
 exit
end
write memory
```

> 📌 `no ip dhcp snooping information option` — desativa a inserção da Option 82 (compatibilidade com o DHCP Relay do Packet Tracer; sem isto, o helper-address pode falhar).

### 10.2 — [SW1] DHCP Snooping (Switch de Acesso)

```
configure terminal
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40
no ip dhcp snooping information option

! -> trunk para o Switch0: tem de ser trusted
interface fastEthernet 0/24
 ip dhcp snooping trust
 exit
end
write memory
```
> As portas dos PCs (Fa0/1-3) ficam **untrusted** por defeito — é onde o snooping inspeciona as mensagens DHCP dos clientes (e descarta as mensagens de servidor, ex.: um PC a fazer-se passar por servidor DHCP).

<details><summary><b>Script do Switch2 — idêntico ao Switch1, clique para abrir (copiar e colar)</b></summary>

```
configure terminal
ip dhcp snooping
ip dhcp snooping vlan 10,20,30,40
no ip dhcp snooping information option

interface fastEthernet 0/24
 ip dhcp snooping trust
 exit
end
write memory
```

</details>

**Verificar (em todos os switches):**
```
show ip dhcp snooping
show ip dhcp snooping binding
```
> ✅ `show ip dhcp snooping` deve indicar `DHCP snooping is enabled` nas VLANs 10/20/30/40. A **binding table** vai sendo preenchida à medida que os PCs obtêm IP (MAC ↔ IP ↔ porta). Se a tabela aparecer vazia, renova o DHCP num PC (`ipconfig /renew`) e volta a verificar.

**(Opcional) Rate limiting** — limita mensagens DHCP por segundo nas portas untrusted (proteção extra contra starvation):
```
interface range fastEthernet 0/1-3
 ip dhcp snooping limit rate 100
 exit
errdisable recovery cause dhcp-rate-limit
```

---

<a id="11-dai"></a>
## 11. Dynamic ARP Inspection (DAI)

> Filtra as mensagens **ARP** nas portas untrusted, verificando se o par IP↔MAC da mensagem existe na **tabela de bindings do DHCP Snooping**. Bloqueia o **ARP poisoning** (man-in-the-middle): um atacante responde ARP em nome de outro dispositivo para desviar o tráfego.
>
> ⚠️ **Depende do DHCP Snooping** (Secção 10) — configura-o primeiro. A validação usa a binding table, por isso os PCs já têm de ter obtido IP via DHCP.
>
> **Configurar em TODOS os switches:** `Switch0`, `Switch1` e `Switch2`.

### 11.1 — [SW0] DAI (Switch Central)

```
configure terminal
ip arp inspection vlan 10,20,30,40

! -> valida MAC de origem/destino e IP da mensagem ARP (todas as verificações no MESMO comando)
ip arp inspection validate src-mac dst-mac ip

! -> portas ligadas a equipamentos com IP estático ou infraestrutura: trusted
interface range fastEthernet 0/1-3, fastEthernet 0/10, fastEthernet 0/24
 ip arp inspection trust
 exit
end
write memory
```

> 📌 O `PC-Gestao` (Fa0/10) e o `Server-DHCP` (Fa0/24) têm **IP estático** — não constam na binding table. Por isso as suas portas ficam **trusted** (em ambiente real usar-se-iam ARP ACLs). A DAI ativa-se diretamente nas VLANs — ao contrário do DHCP Snooping, que ativa primeiro globalmente e depois por VLAN.

### 11.2 — [SW1] DAI (Switch de Acesso)

```
configure terminal
ip arp inspection vlan 10,20,30,40
ip arp inspection validate src-mac dst-mac ip

! -> trunk para o Switch0: trusted
interface fastEthernet 0/24
 ip arp inspection trust
 exit
end
write memory
```
> As portas dos PCs (Fa0/1-3) ficam **untrusted** — é onde o DAI inspeciona os ARP (o MAC/IP de cada PC tem de constar na binding table do DHCP snooping).

<details><summary><b>Script do Switch2 — idêntico ao Switch1, clique para abrir (copiar e colar)</b></summary>

```
configure terminal
ip arp inspection vlan 10,20,30,40
ip arp inspection validate src-mac dst-mac ip

interface fastEthernet 0/24
 ip arp inspection trust
 exit
end
write memory
```

</details>

**Verificar (em todos os switches):**
```
show ip arp inspection interfaces
show ip arp inspection statistics vlan 10
show ip dhcp snooping binding
```
> ✅ `show ip arp inspection interfaces` deve mostrar `Trusted` nas portas de infraestrutura e `Untrusted` nas portas dos PCs. As estatísticas mostram ARP **forwarded** (válido) e **dropped** (bloqueado).

**(Opcional) Rate limiting ARP** — por defeito, 15 pps nas portas untrusted; para ajustar:
```
interface fastEthernet 0/1
 ip arp inspection limit rate 25 burst interval 2
 exit
```

---

<a id="12-aaa-8021x"></a>
## 12. AAA + 802.1X (autenticação de portas)

> **AAA** = **A**uthentication, **A**uthorization, **A**ccounting — framework que controla **quem entra** (autenticação), **o que pode fazer** (autorização, níveis 0-15) e **o que fez** (contabilização/logs). Até agora o guia usou **AAA local** (utilizadores definidos no próprio equipamento, ex.: `username admin` do SSH). Com o **802.1X**, a autenticação passa a ser feita por um **servidor RADIUS** central.

> **802.1X** = controlo de acesso à porta, com 3 elementos:

| Elemento | Neste cenário | Função |
|:---|:---|:---|
| **Supplicant** | PC0–PC5 | O cliente que pede acesso à rede |
| **Authenticator** | Switch1 / Switch2 | Controla a porta física e reencaminha as credenciais ao servidor |
| **Authentication Server** | Server-DHCP (serviço AAA/RADIUS) | Valida as credenciais e decide se a porta abre |

> ⚠️ **Configurar em ÚLTIMO LUGAR**: enquanto a porta não está autorizada, só passa tráfego EAPOL — o DHCP dos PCs deixa de funcionar até se autenticarem. A porta só liberta tráfego após o RADIUS responder `EAP-Success`.

### 12.1 — [SW1] 802.1X no Switch de Acesso

```
configure terminal
aaa new-model
radius-server host 192.168.10.254 key cinel
aaa authentication dot1x default group radius
dot1x system-auth-control

! -> portas dos PCs: só abrem após autenticação RADIUS
interface range fastEthernet 0/1-3
 authentication port-control auto
 dot1x pae authenticator
 exit
end
write memory
```

> 📌 `aaa new-model` ativa o modelo AAA · `radius-server host ... key cinel` define o servidor e a chave secreta (tem de ser igual à do servidor) · `aaa authentication dot1x default group radius` define o método padrão do 802.1X · `dot1x system-auth-control` ativa o 802.1X globalmente.

**Verificar:**
```
show dot1x
show aaa servers
```
> ✅ `show dot1x` deve indicar as portas `Authorized`/`Unauthorized`. `show aaa servers` confirma o RADIUS configurado.

### 12.2 — Servidor RADIUS (no `Server-DHCP`, interface gráfica)

> Clicar no `Server-DHCP` → `Services → AAA` → **ON**:

| Campo | Valor |
|:---|:---|
| **Clients (autenticadores)** | Name: `Switch1` · IP: `192.168.40.3` · Secret: `cinel` |
| | Name: `Switch2` · IP: `192.168.40.4` · Secret: `cinel` |
| **Users (utilizadores)** | Username: `aluno1` · Password: `cinel` |

> 📌 O IP do client é a **SVI de gestão** do switch — é por aí que o switch fala com o servidor (via Router0). Verifica primeiro: `ping 192.168.10.254` a partir do switch.

### 12.3 — Configurar os PCs (Supplicants)

> Em **cada PC** (PC0 a PC5): `Config → FastEthernet0 → 802.1X Security` → Authentication: **MD5** → username `aluno1` e password `cinel` (os criados no servidor AAA).

> ✅ Depois disto, a porta autentica e o PC volta a ter rede (renova o DHCP se preciso). Se um PC **não** tiver 802.1X ativado, a porta fica bloqueada e o PC não recebe IP — é o comportamento pretendido.

<details><summary><b>Script do Switch2 — idêntico ao Switch1, clique para abrir (copiar e colar)</b></summary>

```
configure terminal
aaa new-model
radius-server host 192.168.10.254 key cinel
aaa authentication dot1x default group radius
dot1x system-auth-control

interface range fastEthernet 0/1-3
 authentication port-control auto
 dot1x pae authenticator
 exit
end
write memory
```

</details>

<details><summary><b>Script do Switch0 (opcional — 802.1X na porta do PC-Gestao, clique para abrir)</b></summary>

```
configure terminal
aaa new-model
radius-server host 192.168.10.254 key cinel
aaa authentication dot1x default group radius
dot1x system-auth-control

interface fastEthernet 0/10
 authentication port-control auto
 dot1x pae authenticator
 exit
end
write memory
```

</details>

---

<a id="13-testes"></a>
## 13. Testes finais

**T1 — DHCP nos PCs:** em qualquer PC, `Desktop → Command Prompt` → `ipconfig` (ou o próprio painel de IP Configuration). Deve mostrar IP, máscara e gateway da VLAN correta.

**T2 — Routing inter-VLAN (ROAS):** a partir do `PC0` (VLAN 10):
```
ping 192.168.20.1
ping 192.168.30.1
```
> ✅ Deve responder — prova que o router encaminha entre VLANs.

**T3 — Acesso ao servidor:** a partir do `PC1` (VLAN 20):
```
ping 192.168.10.254
```
> ✅ Prova que o DHCP Relay funciona para além do próprio DHCP.

**T4 — SSH (bloqueia o acesso remoto com admin/cinel):** a partir do `PC-Gestao` (VLAN 40):
```
ssh -l admin 192.168.40.2
```
Senha: `cinel`
> ✅ Deve entrar no CLI do Switch0. Repete para `192.168.40.3` (Switch1) e `192.168.40.4` (Switch2). A partir de um PC de outra VLAN o SSH deve ser **negado** (ACL `access-class 10 in`).

**T5 — Portas fechadas:** em cada switch:
```
show ip interface brief
```
> ✅ As portas não usadas devem estar `administratively down`.

**T6 — VLAN nativa (vista global):** em cada switch:
```
show interfaces trunk
```
> ✅ Coluna *Native vlan* = `40` em todos os trunks, sem mensagens de *native VLAN mismatch*.

**T7 — Port Security:** em cada switch de acesso:
```
show port-security
show port-security address
```
> ✅ Fa0/1-3 devem aparecer `Secure-up` com o MAC `sticky` de cada PC. **Teste de violação:** liga um segundo PC a uma porta — deve ficar `Secure-shutdown` (reativa com `shutdown` + `no shutdown`).

**T8 — DHCP Snooping:** em cada switch:
```
show ip dhcp snooping
show ip dhcp snooping binding
```
> ✅ Snooping ativo nas VLANs 10/20/30/40 e a binding table com uma entrada por PC (MAC ↔ IP ↔ interface).

**T9 — DAI:** em cada switch:
```
show ip arp inspection interfaces
show ip arp inspection statistics vlan 10
```
> ✅ Portas de infraestrutura `Trusted`, portas dos PCs `Untrusted`; estatísticas com ARP *forwarded* e *dropped*.

**T10 — 802.1X:** a partir de qualquer PC:
```
ping 192.168.10.1
```
> ✅ Com 802.1X ativado no PC (supplicant), responde. Sem supplicant, o PC fica **sem rede** (porta `Unauthorized` — verifica com `show dot1x` no switch).

---

<a id="14-checklist"></a>
## 14. Checklist

- [ ] **Senhas:** `enable secret`, senha de consola e utilizador SSH configurados em Switch0/1/2 e Router0; `service password-encryption` ativo
- [ ] **VLANs:** 10, 20, 30 e 40 criadas em Switch0, Switch1 e Switch2 (`show vlan brief`)
- [ ] **Trunks:** Fa0/1-3 no Switch0 e Fa0/24 nos switches de acesso em modo trunk
- [ ] **VLAN nativa 40:** `switchport trunk native vlan 40` em todos os trunks, confirmado com `show interfaces trunk`
- [ ] **Portas de acesso:** PCs nas respetivas VLANs; Fa0/10 do Switch0 (PC-Gestao, V40) e Fa0/24 do Switch0 (Server-DHCP, V10)
- [ ] **Portas fechadas:** portas não usadas em modo access + `shutdown`
- [ ] **SVIs de gestão:** `interface vlan 40` em cada switch (192.168.40.2/.3/.4) com `ip default-gateway 192.168.40.1`
- [ ] **SSH:** RSA 2048 gerada, `ip ssh version 2`, VTY com `login local` + `transport input ssh` + ACL de gestão
- [ ] **ROAS:** sub-interfaces Gi0/0.10, .20, .30 e .40 (`encapsulation dot1Q 40 native`) ativas no Router0
- [ ] **DHCP Relay:** `ip helper-address 192.168.10.254` em Gi0/0.20 e Gi0/0.30
- [ ] **Server-DHCP:** IP fixo `192.168.10.254/24` e pools das VLANs 10/20/30 ativos
- [ ] **PCs:** todos os PC0–PC5 a obter IP via DHCP
- [ ] **Port Security:** `switchport port-security` + `sticky` + `violation shutdown` nas portas dos PCs (`show port-security address`)
- [ ] **DHCP Snooping:** ativo nas VLANs 10/20/30/40 em todos os switches; trunks + porta do servidor **trusted**; Option 82 desativada (`show ip dhcp snooping binding`)
- [ ] **DAI:** ativo nas VLANs 10/20/30/40; trunks + portas com IP estático **trusted**; `ip arp inspection validate src-mac dst-mac ip` (`show ip arp inspection statistics`)
- [ ] **AAA/802.1X:** `aaa new-model` + RADIUS 192.168.10.254 nos switches; serviço AAA ativo no Server-DHCP (clients + users); PCs configurados como supplicants MD5 (`show dot1x`)
- [ ] **Testes:** ping entre VLANs ✓ · ping ao servidor ✓ · SSH a partir do PC-Gestao ✓ · port-security / snooping / DAI / 802.1X verificados ✓
