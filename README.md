# Guia Unificado — Cenário Cisco Packet Tracer
**VLANs 10/20/30 · VLAN Nativa 40 · Router-on-a-Stick · DHCP Relay · SSH**

> Guia único, organizado **por equipamento e por ordem prática de execução**. Em cada passo indica-se claramente **em que equipamento** a configuração é feita. Quando a mesma configuração se repete noutro equipamento, é apresentada num bloco colapsado pronto a **copiar e colar**.

---

## Índice

0. [Lembretes IOS](#0-lembretes-ios)
1. [Cenário: topologia e endereçamento](#1-cenário)
2. [Ordem de configuração](#2-ordem)
3. [Switch0 (central)](#3-switch0)
4. [Switch1 (acesso)](#4-switch1)
5. [Switch2 (acesso)](#5-switch2)
6. [Router0 (ROAS + DHCP Relay)](#6-router0)
7. [Server-DHCP (GUI)](#7-server)
8. [PCs (DHCP)](#8-pcs)
9. [Testes finais](#9-testes)
10. [Checklist](#10-checklist)

---

<a id="0-lembretes-ios"></a>
## 0. Lembretes IOS

<details><summary><b>Modos, navegação e gravação (clique para abrir)</b></summary>

| Modo | Prompt | Como entrar |
|:---|:---|:---|
| EXEC Utilizador | `Switch>` | (é o modo inicial) |
| EXEC Privilegiado | `Switch#` | `enable` |
| Config Global | `Switch(config)#` | `configure terminal` |
| Config de Interface | `(config-if)#` | `interface ...` |

- Recuar um nível: `exit` · Voltar direto ao modo privilegiado: `end` (ou `Ctrl+Z`)
- **RAM** = `running-config` (perde-se ao desligar) · **NVRAM** = `startup-config` (persistente)
- **Guardar sempre no fim de cada equipamento:** `write memory` (equivale a `copy running-config startup-config`)

</details>

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

> 📌 **Regra de ouro:** termina sempre cada equipamento com `write memory`.

---

<a id="3-switch0"></a>
## 3. Switch0 (Switch Central)

> Toda a configuração desta secção é feita **no `Switch0`**, na aba **CLI**.

### 3.1 — Hostname e senhas

```
enable
configure terminal
hostname Switch0
!
! ===== SENHAS =====
! "cinel" -> bloqueia o modo privilegiado (enable)
enable secret cinel
! encripta senhas em texto claro no running-config
service password-encryption
! aviso apresentado antes de qualquer login
banner motd #Acesso restrito!#
!
! "cisco" -> bloqueia o acesso fisico pela porta de consola
line console 0
 password cisco
 login
 exec-timeout 5 0
 logging synchronous
 exit
```

**Verificar:** `show running-config` — procura `enable secret 5 ...` e `password 7 ...` (cifradas). Depois faz `exit` e volta a entrar: a consola pede a senha `cisco`, e `enable` pede `cinel`.

### 3.2 — Criar as VLANs (10, 20, 30 e 40)

```
! ===== VLANS (apenas 10, 20, 30 e 40) =====
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

### 3.3 — Trunks (Fa0/1 → Router0, Fa0/2 → Switch1, Fa0/3 → Switch2) com VLAN nativa 40

```
! ===== TRUNKS =====
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

### 3.4 — Portas de acesso: PC-Gestao (Fa0/10, VLAN 40) e Server-DHCP (Fa0/24, VLAN 10)

```
! ===== PORTAS DE ACESSO =====
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

**Verificar:** `show vlan brief` — Fa0/10 deve constar na VLAN 40 e Fa0/24 na VLAN 10. Se aparecerem na VLAN 1 (*default*), a configuração não foi aplicada.

### 3.5 — Fechar as portas não utilizadas

Portas em uso no Switch0: `Fa0/1-3` (trunks), `Fa0/10` (PC-Gestao) e `Fa0/24` (Server-DHCP).

```
! ===== FECHAR PORTAS NAO USADAS =====
interface range fastEthernet 0/4-9, fastEthernet 0/11-23, gigabitEthernet 0/1-2
 switchport mode access
 shutdown
 exit
```

**Verificar:** `show ip interface brief` — as portas fechadas devem mostrar `administratively down`.

### 3.6 — SVI de gestão (interface vlan 40)

Permite gerir o switch remotamente (SSH/ping) a partir da rede de gestão.

```
! ===== SVI DE GESTAO =====
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

### 3.7 — SSH (acesso remoto seguro)

```
! ===== SSH =====
! "admin"/"cinel" -> bloqueia o acesso remoto SSH (linhas VTY)
ip domain-name cinel.local
crypto key generate rsa
2048
ip ssh version 2
username admin privilege 15 secret cinel
!
! ACL: so permite SSH a partir da rede de gestao (VLAN 40)
access-list 10 permit 192.168.40.0 0.0.0.255
line vty 0 4
 login local
 transport input ssh
 access-class 10 in
 exec-timeout 5 0
 logging synchronous
 exit
!
end
write memory
```

> 📌 Se o exercício exigir Telnet também, muda para `transport input ssh telnet` (Telnet não é encriptado — usar só em laboratório).

**Verificar:** `show ip ssh` — deve indicar `SSH version 2` e autenticação ativada.

### 3.8 — Bloco completo do Switch0 (copiar e colar de uma vez)

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

### 4.1 — Hostname e senhas

Igual ao Switch0.

```
enable
configure terminal
hostname Switch1
!
! ===== SENHAS =====
! "cinel" -> bloqueia o modo privilegiado (enable)
enable secret cinel
service password-encryption
banner motd #Acesso restrito!#
!
! "cisco" -> bloqueia o acesso fisico pela porta de consola
line console 0
 password cisco
 login
 exec-timeout 5 0
 logging synchronous
 exit
```

### 4.2 — Criar as VLANs (10, 20, 30 e 40)

> Com criação manual, as VLANs têm de existir **em cada switch**. (Se usaste VTP no Switch0, salta este passo — o Switch1 recebe-as como client.)

```
! ===== VLANS =====
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

**Verificar:** `show vlan brief`

### 4.3 — Trunk para o Switch0 (Fa0/24) com VLAN nativa 40

```
! ===== TRUNK =====
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

### 4.4 — Portas de acesso aos PCs (Fa0/1→V10, Fa0/2→V20, Fa0/3→V30)

```
! ===== PORTAS DE ACESSO AOS PCS =====
interface fastEthernet 0/1
 switchport mode access
 switchport access vlan 10
 no shutdown
 exit
interface fastEthernet 0/2
 switchport mode access
 switchport access vlan 20
 no shutdown
 exit
interface fastEthernet 0/3
 switchport mode access
 switchport access vlan 30
 no shutdown
 exit
```

**Verificar:** `show vlan brief` — Fa0/1 na VLAN 10, Fa0/2 na 20, Fa0/3 na 30.

### 4.5 — Fechar as portas não utilizadas

Portas em uso no Switch1: `Fa0/1-3` (PCs) e `Fa0/24` (trunk).

```
! ===== FECHAR PORTAS NAO USADAS =====
interface range fastEthernet 0/4-23, gigabitEthernet 0/1-2
 switchport mode access
 shutdown
 exit
```

**Verificar:** `show ip interface brief` — portas fechadas em `administratively down`.

### 4.6 — SVI de gestão (interface vlan 40)

```
! ===== SVI DE GESTAO =====
interface vlan 40
 ip address 192.168.40.3 255.255.255.0
 no shutdown
 exit
ip default-gateway 192.168.40.1
```

**Verificar:** `show ip interface brief | include Vlan40` · `ping 192.168.40.1`

### 4.7 — SSH

```
! ===== SSH =====
! "admin"/"cinel" -> bloqueia o acesso remoto SSH (linhas VTY)
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
!
end
write memory
```

**Verificar:** `show ip ssh`

### 4.8 — Bloco completo do Switch1 (copiar e colar de uma vez)

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
 switchport mode access
 switchport access vlan 10
 no shutdown
 exit
interface fastEthernet 0/2
 switchport mode access
 switchport access vlan 20
 no shutdown
 exit
interface fastEthernet 0/3
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

### 5.1 — Hostname e senhas

```
enable
configure terminal
hostname Switch2
!
! ===== SENHAS =====
! "cinel" -> bloqueia o modo privilegiado (enable)
enable secret cinel
service password-encryption
banner motd #Acesso restrito!#
!
! "cisco" -> bloqueia o acesso fisico pela porta de consola
line console 0
 password cisco
 login
 exec-timeout 5 0
 logging synchronous
 exit
```

**Verificar:** `show running-config` — procura `enable secret 5 <hash>` e as senhas cifradas. Depois faz `exit` e volta a entrar: a consola pede `cisco` e o `enable` pede `cinel`.

### 5.2 — Criar as VLANs (10, 20, 30 e 40)

> Com criação manual, as VLANs têm de existir **em cada switch**. (Se usaste VTP no Switch0, salta este passo — o Switch2 recebe-as como client.)

```
! ===== VLANS =====
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

### 5.3 — Trunk para o Switch0 (Fa0/24) com VLAN nativa 40

```
! ===== TRUNK =====
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

### 5.4 — Portas de acesso aos PCs (Fa0/1→V10, Fa0/2→V20, Fa0/3→V30)

```
! ===== PORTAS DE ACESSO AOS PCS =====
interface fastEthernet 0/1
 switchport mode access
 switchport access vlan 10
 no shutdown
 exit
interface fastEthernet 0/2
 switchport mode access
 switchport access vlan 20
 no shutdown
 exit
interface fastEthernet 0/3
 switchport mode access
 switchport access vlan 30
 no shutdown
 exit
```

> ⚠️ **Confirma com o teu enunciado:** se os PCs do Switch2 tiverem de ficar noutras VLANs (ex.: PC3 na VLAN 30), altera o `switchport access vlan X` da porta respetiva **antes** de fechar com `write memory`.

**Verificar:** `show vlan brief` — Fa0/1 na VLAN 10, Fa0/2 na 20, Fa0/3 na 30.

### 5.5 — Fechar as portas não utilizadas

Portas em uso no Switch2: `Fa0/1-3` (PCs) e `Fa0/24` (trunk).

```
! ===== FECHAR PORTAS NAO USADAS =====
interface range fastEthernet 0/4-23, gigabitEthernet 0/1-2
 switchport mode access
 shutdown
 exit
```

**Verificar:** `show ip interface brief` — portas fechadas em `administratively down`.

### 5.6 — SVI de gestão (interface vlan 40)

```
! ===== SVI DE GESTAO =====
interface vlan 40
 ip address 192.168.40.4 255.255.255.0
 no shutdown
 exit
ip default-gateway 192.168.40.1
```

**Verificar:** `show ip interface brief | include Vlan40` · `ping 192.168.40.1`

### 5.7 — SSH

```
! ===== SSH =====
! "admin"/"cinel" -> bloqueia o acesso remoto SSH (linhas VTY)
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
!
end
write memory
```

**Verificar:** `show ip ssh` — deve indicar `SSH version 2`. A partir do `PC-Gestao`: `ssh -l admin 192.168.40.4` (senha `cinel`).

### 5.8 — Bloco completo do Switch2 (copiar e colar de uma vez)

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
 switchport mode access
 switchport access vlan 10
 no shutdown
 exit
interface fastEthernet 0/2
 switchport mode access
 switchport access vlan 20
 no shutdown
 exit
interface fastEthernet 0/3
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

### 6.1 — Ativar a interface física (sem IP na porta física)

```
enable
configure terminal
hostname Router0
!
! ===== SENHAS (bloqueiam enable e consola do router) =====
enable secret cinel
service password-encryption
banner motd #Acesso restrito!#
line console 0
 password cisco
 login
 exec-timeout 5 0
 logging synchronous
 exit
!
interface gigabitEthernet 0/0
 no shutdown
 exit
```

### 6.2 — Sub-interfaces (gateway de cada VLAN)

```
! VLAN 10 (Estudantes) - rede do servidor DHCP, sem helper
interface gigabitEthernet 0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 exit
!
! VLAN 20 (Professores) - com DHCP Relay
interface gigabitEthernet 0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip helper-address 192.168.10.254
 exit
!
! VLAN 30 (Direcao) - com DHCP Relay
interface gigabitEthernet 0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip helper-address 192.168.10.254
 exit
!
! VLAN 40 (Native/Gestao) - sem DHCP Relay (nao ha PCs)
interface gigabitEthernet 0/0.40
 encapsulation dot1Q 40 native
 ip address 192.168.40.1 255.255.255.0
 exit
!
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

### 6.3 — SSH no Router0 (opcional — se o exercício pedir)

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

**Verificar:** nos passos 9/10, os PCs devem obter IP da sub-rede correta.

---

<a id="8-pcs"></a>
## 8. PCs (DHCP)

> Em **cada PC** (PC0 a PC5): `Desktop → IP Configuration` → ativar **DHCP**.

> ✅ Cada PC deve obter automaticamente o IP da sub-rede da **sua VLAN** (tabela da Secção 1), gateway incluído. Verifica no painel: o IP deve começar por `192.168.<vlan>.` e o gateway deve ser `192.168.<vlan>.1`.

---

<a id="9-testes"></a>
## 9. Testes finais

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

---

<a id="10-checklist"></a>
## 10. Checklist

- [ ] **Senhas:** `enable secret`, senha de consola e utilizador SSH configurados em Switch0/1/2 e Router0; service password-encryption` ativo
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
- [ ] **Testes:** ping entre VLANs ✓ · ping ao servidor ✓ · SSH a partir do PC-Gestao ✓
