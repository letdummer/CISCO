# Guia de Configuração — VLANs, Trunk, VLAN Native 40, ROAS e SSH/TELNET (Cisco Packet Tracer)

## Índice

1. [Topologia e Endereçamento](#1-topologia-e-endereçamento)
2. [Passo a Passo no Cisco IOS](#2-passo-a-passo-no-cisco-ios)
   - [Tópico 1 — Criar VLANs e VTP](#-tópico-1--criar-vlans-e-vtp)
   - [Tópico 2 — Portas de acesso, trunk e VLAN nativa](#-tópico-2--portas-de-acesso-trunk-e-vlan-nativa)
   - [Tópico 3 — Router-on-a-Stick e DHCP-Relay](#-tópico-3--router-on-a-stick-e-dhcp-relay)
   - [Tópico 4 — Configuração do DHCP e Pools](#-tópico-4--configuração-do-server-dhcp-e-pools)
   - [Tópico 5 — VLAN 999 e SSH](#-tópico-5--vlan-999-e-ssh)
   - [Tópico 6 — PC na Fa0/10 com VLAN 999 e Porta do Server-DHCP](#-tópico-6--pc-na-fa010-com-vlan-999-e-porta-server-dhcp)
   - [Tópico 7 — Fechar portas](#-tópico-7--fechar-portas)
   - [Testar SSH](#-testar-o-acesso-ssh)
3. [Resumo dos resultados esperados](#3-resumo-dos-resultados-esperados-com-router0-instalado)
4. [Checklist rápido](#4-checklist-rápido)

---

# 1. Topologia e Endereçamento

### » Dispositivos

| Dispositivo | Função |
|---|---|
| `Switch0` | Switch central (distribuição) — liga Switch1, Switch2, Router0, Server-DHCP e PC-Gestao |
| `Switch1` | Switch de acesso — liga PC0, PC1, PC2 |
| `Switch2` | Switch de acesso — liga PC3, PC4, PC5 |
| `Router0` | Router-on-a-Stick — faz o routing inter-VLAN |
| `Server-DHCP` | Servidor DHCP na VLAN 10 para todas sub-redes |
| `PC-Gestao` | PC de gestão na VLAN 999 |
| `PC0`–`PC5` | Terminais dos utilizadores |

### » Diagrama

```
                      [ Router0 ]
                        (Gi0/0)          -(trunk / ROAS)
                          |
                       (Fa0/1) 
                     [ Switch0 ]         -(central)
                /                |                \                 \
            (Fa0/2)           (Fa0/3)           (Fa0/24)          (Fa0/10)
              /                  |                  \           [ Pc-Gestao ] 
             /                   |                   \
      [ Switch1 ]           [ Switch2 ]        [ Server-DHCP ] (Fa0)
   Fa0/1: PC0 (V10)       Fa0/1: PC3 (V30)
   Fa0/2: PC1 (V20)       Fa0/2: PC4 (V20)
   Fa0/3: PC2 (V30)       Fa0/3: PC5 (V10)
```

> 📌 Todos os links trunk (Router0↔Switch0, Switch0↔Switch1, Switch0↔Switch2) usam a **VLAN 40 como VLAN nativa** (tráfego sem tag 802.1Q).

### » VLANs, sub-redes e gateway

| VLAN | Nome | Sub-rede | Máscara | Gateway (`Router0`) |
|:---:|:---|:---|:---|:---|
| `10` | Estudantes | `192.168.10.0` | `/24` | `192.168.10.1` |
| `20` | Professores | `192.168.20.0` | `/24` | `192.168.20.1` |
| `30` | Direcao | `192.168.30.0` | `/24` | `192.168.30.1` |
| `40` | Native | `192.168.40.0` | `/24` | `192.168.40.1` *(sub-interface nativa)* |
| `999` | Gestao | `10.0.0.0` | `/8` | — *(sem gateway, ver nota)* |

> 📌 **Nota (VLAN 40):** a VLAN 40 é a **VLAN nativa** dos trunks. Não tem PCs nem pool DHCP; existe apenas para transportar tráfego sem tag (ex.: CDP/STP). No `Router0` a sub-interface `Gi0/0.40` é criada com `encapsulation dot1Q 40 native`. A VLAN 1 deixa de ser usada como nativa (boa prática de segurança).

> 📌 **Nota:** a VLAN 999 **não tem sub-interface** no router propositadamente, para a rede de gestão ficar isolada das VLANs de utilizador — isto mantém válidos os testes de SSH dos pontos 12 e 13 do enunciado.

### » Endereços IP dos PCs

| PC | VLAN | Método | Endereço IP | Máscara | Gateway |
|---|:---:|---|---|:---|---|
| `PC0` | 10 | `DHCP` | `192.168.10.x` | `/24` | `192.168.10.1` |
| `PC5` | 10 | `DHCP`  | `192.168.10.x` | `/24` | `192.168.10.1` |
| `PC1` | 20 | `DHCP`  | `192.168.20.x` | `/24` | `192.168.20.1` |
| `PC4` | 20 | `DHCP`  | `192.168.20.x` | `/24` | `192.168.20.1` |
| `PC2` | 30 | `DHCP` | `192.168.30.x` | `/24` | `192.168.30.1` |
| `PC3` | 30 | `DHCP`  | `192.168.30.x` | `/24` | `192.168.30.1` |
| `Server-DHCP` | `10` | `DHCP`  | `192.168.10.254` | `/24` | `192.168.10.1` |
| `PC-Gestao` | `999` | `Estático` | `/8` | `sem gateway` |

---

# 2. Passo a Passo no Cisco IOS

## » Tópico 1 — Criar VLANs e VTP


### Opção A: Distribuição Automática via VTP
⚠️ Atenção: Se optares por VTP, siga esta ordem:

   - configurar primeiro os links Trunk entre os switches
         ---> [Configuração Trunk](#-tópico-2--portas-de-acesso-trunk-e-vlan-nativa)
   - configurar o VTP Server (switch0, central)
   - criar as vlans no switch central
   - configurar o VTP Cliente (switch1 e switch2)
  
<details>

**1. Configurar o VTP Server (Switch0):**   
```
Switch0> enable
Switch0# configure terminal
Switch0(config)# vtp domain cinel-domain
Switch0(config)# vtp mode server
Switch0(config)# vtp password cinel
```

> **Criar todas as VLANS, apenas no `Switch0`**

```bash
enable
configure terminal
vlan 10
 name Estudantes
vlan 20
 name Professores
vlan 30
 name Direcao
vlan 40
 name Native
vlan 999
 name Gestao
vlan 99
 name Portas_Inativas
exit
write memory
```

**2. Configurar os VTP Clients (Switch1 e Switch2):**

```
Switch1> enable
Switch1# configure terminal
Switch1(config)# vtp domain cinel-domain
Switch1(config)# vtp mode client
Switch1(config)# vtp password cinel
Switch1(config)# exit
Switch1# write memory
```


### Confirmar funcionamento do VTP

`show vtp status`
`show vlan brief`

</details>


### Opção B: Criação Manual de VLANs

Configurar em `Switch0`, `Switch1` e `Switch2`:

```bash
enable
configure terminal
vlan 10
 name Estudantes
vlan 20
 name Professores
vlan 30
 name Direcao
vlan 40
 name Native
vlan 999
 name Gestao
vlan 99
 name Portas_Inativas
exit
write memory
```

**Teste:**
```
show vlan brief
```


---

## » Tópico 2 — Portas de acesso, trunk e VLAN nativa

**Switch Central (Switch0)**

Trunk para o Router0 (Fa0/1), Switch1 (Fa0/2) e Switch2 (Fa0/3), com **VLAN 40 como nativa**

> ⚠️ A VLAN 40 tem de existir no switch (Tópico 1) e a VLAN nativa tem de ser **igual nas duas pontas** de cada trunk, caso contrário o STP/CDP reporta *native VLAN mismatch*.

```bash
Switch0(config)# interface range fastEthernet 0/1-3
Switch0(config-if-range)# switchport mode trunk
Switch0(config-if-range)# switchport trunk native vlan 40
Switch0(config-if-range)# no shutdown
Switch0(config-if-range)# exit
Switch0# write memory
```


**Switch de Acesso (SW-01)**
<details>

Trunk

```
Switch1(config)# interface fastEthernet 0/24
Switch1(config-if)# switchport mode trunk
Switch1(config-if)# switchport trunk native vlan 40
Switch1(config-if)# no shutdown
Switch1(config-if)# exit
```

Portas de acesso aos PCs

```
Switch1(config)# interface fastEthernet 0/1
Switch1(config-if)# switchport mode access
Switch1(config-if)# switchport access vlan 10
Switch1(config-if)# no shutdown
Switch1(config-if)# exit

Switch1(config)# interface fastEthernet 0/2
Switch1(config-if)# switchport mode access
Switch1(config-if)# switchport access vlan 20
Switch1(config-if)# no shutdown
Switch1(config-if)# exit

Switch1(config)# interface fastEthernet 0/3
Switch1(config-if)# switchport mode access
Switch1(config-if)# switchport access vlan 30
Switch1(config-if)# no shutdown
Switch1(config-if)# exit

Switch1(config)# exit
Switch1# write memory
```
</details>


**Switch de Acesso (SW-02)**
<Details>

Trunk para o SW0

```
Switch2(config)# interface fastEthernet 0/24
Switch2(config-if)# switchport mode trunk
Switch2(config-if)# switchport trunk native vlan 40
Switch2(config-if)# no shutdown
Switch2(config-if)# exit
```

# Portas de acesso aos PCs no Switch2
```
Switch2(config)# interface fastEthernet 0/1
Switch2(config-if)# switchport mode access
Switch2(config-if)# switchport access vlan 30
Switch2(config-if)# no shutdown
Switch2(config-if)# exit
```

```
Switch2(config)# interface fastEthernet 0/2
Switch2(config-if)# switchport mode access
Switch2(config-if)# switchport access vlan 20
Switch2(config-if)# no shutdown
Switch2(config-if)# exit
```

```
Switch2(config)# interface fastEthernet 0/3
Switch2(config-if)# switchport mode access
Switch2(config-if)# switchport access vlan 10
Switch2(config-if)# no shutdown
Switch2(config-if)# exit
```

```
Switch2(config)# exit
Switch2# write memory
```

</Details>

**Confirmação de mudanças:**

```
show interfaces trunk
```

> ✅ Na coluna **Native vlan** deve aparecer `40` em todas as portas trunk (Fa0/1-3 no Switch0 e Fa0/24 no Switch1 e Switch2). A VLAN 40 deve constar em *VLANs allowed and active in management domain*.

<!--
| Port | Mode | Encapsulation | Status | Nativa vlan |
|:----:|:----:|:-------------:|:------:|:-----------:|
| `fa0/2` | `on` | `802.1q` | `/trunking` | `1` |
-->

---

## » Tópico 3 — Router-on-a-Stick e DHCP-Relay

> Mecanismo do DHCP Relay: Como o Server-DHCP se encontra fisicamente na VLAN 10, os pedidos de DHCP (broadcasts) emitidos pelas VLANs 20 e 30 são descartados pelo router por omissão. O comando ip helper-address 192.168.10.254 converte esses broadcasts em mensagens unicast direcionadas diretamente ao IP do servidor.

```bash
Router0> enable
Router0# configure terminal

Router0(config)# interface gigabitEthernet 0/0
Router0(config-if)# no shutdown
Router0(config-if)# exit
```

**VLAN 10 (Estudantes)**
```bash
Router0(config)# interface gigabitEthernet 0/0.10
Router0(config-subif)# encapsulation dot1Q 10

Router0(config-subif)# ip address 192.168.10.1 255.255.255.0

Router0(config-subif)# exit
```

**VLAN 20 (Professores) — com DHCP Relay**
```bash
Router0(config)# interface gigabitEthernet 0/0.20
Router0(config-subif)# encapsulation dot1Q 20

Router0(config-subif)# ip address 192.168.20.1 255.255.255.0

Router0(config-subif)# ip helper-address 192.168.10.254
Router0(config-subif)# exit
```

**VLAN 30 — com DHCP Relay**
```bash
Router0(config)# interface gigabitEthernet 0/0.30
Router0(config-subif)# encapsulation dot1Q 30

Router0(config-subif)# ip address 192.168.30.1 255.255.255.0

Router0(config-subif)# ip helper-address 192.168.10.254
Router0(config-subif)# exit
```

**VLAN 40 (Native) — sem DHCP Relay**
```bash
Router0(config)# interface gigabitEthernet 0/0.40
Router0(config-subif)# encapsulation dot1Q 40 native

Router0(config-subif)# ip address 192.168.40.1 255.255.255.0

Router0(config-subif)# exit
Router0(config)# exit
Router0# write memory
```

> 📌 A palavra-chave `native` na sub-interface indica ao router que o tráfego da VLAN 40 circula **sem tag**, em coerência com `switchport trunk native vlan 40` nos switches. Não é necessário `ip helper-address`, pois não há PCs nesta VLAN.

> 📌 A interface física `Gi0/0` **não** recebe IP quando se usam sub-interfaces — só precisa de `no shutdown`. Repara que a `Gi0/0.40` é a sub-interface **nativa** e que **não existe** `Gi0/0.999`: a VLAN de gestão fica fora do routing de propósito.

---

## » Tópico 4 — Configuração do Server-DHCP e Pools

**Configuração Manual do Server-DHCP | IP Estático**

`Clicar no Server -> Desktop -> Ip Configuration`

| Ip Address | Subnet Mask | Default Gateway |
|:---:|:---:|:---:|
| 192.168.10.254 | 255.255.255.0 | 192.168.10.1 |


`Services -> DHCP` : **ON**

| Pool | Default Gateway | Start IP Address | Subnet Mask |
|:-------:|:------------:|:-------------:|:---:|
| VLAN 10 | 192.168.10.1 | 192.168.10.50 | 255.255.255.0 |
| VLAN 20 | 192.168.20.1 | 192.168.20.50 | 255.255.255.0 |
| VLAN 30 | 192.168.30.1 | 192.168.30.50 | 255.255.255.0 |

**ATRIBUIR IP AOS PCS**

Em cada PC, [ 0 - 5 ]
`Desktop - > IP Configuration`
ativar `DHCP`



---

## » Tópico 5 — VLAN 999 e SSH

> Se não utilizou VTP, configure a VLAN 999:
```bash
Switch0(config)# vlan 999
Switch0(config-vlan)# name Gestao
Switch0(config-vlan)# exit
```

**1. Endereçamento das interfaces virtuais (SVI) nos Switches:**

```
Switch0(config)# interface vlan 999
Switch0(config-if)# ip address 10.0.0.1 255.0.0.0
Switch0(config-if)# no shutdown
Switch0(config-if)# exit
Switch0# write memory
```

> Repetir em `Switch0`, `Switch1` e `Switch2`, mudando apenas o IP (pode acrescentar +1 ao ultimo digito).
> Criar a VLAN 999 em cada switch, se necessário.

<details>
   
```
Switch1(config)# interface vlan 999
Switch1(config-if)# ip address 10.0.0.2 255.0.0.0
Switch1(config-if)# no shutdown
Switch1(config-if)# exit
Switch1# write memory
```

```
Switch2(config)# interface vlan 999
Switch2(config-if)# ip address 10.0.0.3 255.0.0.0
Switch2(config-if)# no shutdown
Switch2(config-if)# exit
Switch2# write memory
```
</details>

### SSH / TELNET (repetir nos três switches e no Router):

```bash
enable
configure terminal

! 1. Domínio e Chave RSA para SSH:

ip domain-name cinel.local
crypto key generate rsa
2048

! 2. Autenticação Local e Modo Privilegiado:

enable secret cinel
username admin privilege 15 secret cinel

! 3. Hardening Geral:

service password-encryption
banner motd # Acesso restrito! #

! 4. Proteção da Porta de Consola:

line console 0
 login local
 exec-timeout 5 0
 logging synchronous
 exit

! 5. Proteção das Linhas de Rede (SSH / Telnet):

line vty 0 4
 login local
 transport input ssh telnet
 exec-timeout 5 0
 logging synchronous
 exit

write memory
```

<details>


**Para testar:**

`telnet 10.0.0.1`

`ssh -l admin 10.0.0.1`



**! Visualizar configurações ativas e resumo de interfaces**
```
Router# show running-config
Router# show ip interface brief

! Guardar alterações da RAM para a NVRAM
Router# copy running-config startup-config

! Atalho equivalente
Router# write memory
```
</details>


---

## » Tópico 6 — PC na Fa0/10 com VLAN 999 e Porta Server-DHCP

# SW-0

**Porta do PC-Gestao (VLAN 999)**
```bash
Switch0(config)# interface fastEthernet 0/10
Switch0(config-if)# switchport mode access
Switch0(config-if)# switchport access vlan 999
Switch0(config-if)# no shutdown
Switch0(config-if)# exit
```


**Porta do Server-DHCP (VLAN 10)**
```
Switch0(config)# interface fastEthernet 0/24
Switch0(config-if)# switchport mode access
Switch0(config-if)# switchport access vlan 10
Switch0(config-if)# no shutdown
Switch0(config-if)# exit
Switch0# exit
Switch0# write memory
```

---

## » Tópico 7 — FECHAR PORTAS

**SWITCH 0 (Switch Central):**
- Portas em uso: `Fa0/1`, `Fa0/2`, `Fa0/3` (trunks). `Fa0/10` (Pc-Gestao) e `Fa0/24` (server-DHCP)
- Portas para fechar: `Fa0/4-9`, `Fa0/11-23`, `Gi0/1-2`.

**(se não foi criada com vtp, criar VLAN 99 para Portas Inativas)**
```
Switch0(config)# vlan 99
Switch0(config-vlan)# name Portas_Inativas
Switch0(config-vlan)# exit
```

**1. No Switch Central (Switch0)**

``` 
Switch0(config)# interface range fastEthernet 0/4 - 9 , fastEthernet 0/11 - 23 , gigabitEthernet 0/1 - 2
Switch0(config-if-range)# switchport mode access
Switch0(config-if-range)# switchport access vlan 99
Switch0(config-if-range)# shutdown
Switch0(config-if-range)# exit
Switch0# write memory
```



**2. Nos Switches de Acesso (Switch1 e Switch2):**
- Portas em uso: `Fa0/1`, `Fa0/2`, `Fa0/3` (PCs) e `Fa0/24` (trunk).
- Portas para fechar: `Fa0/4-23`, `Gi0/1-2`.

**(se não foi criada com vtp, criar VLAN 99 para Portas Inativas)**
```
Switch1(config)# vlan 99
Switch1(config-vlan)# name Portas_Inativas
Switch1(config-vlan)# exit
```

```
Switch1(config)# interface range fastEthernet 0/4-23 , gigabitEthernet 0/1-2
Switch1(config-if-range)# switchport mode access
Switch1(config-if-range)# switchport access vlan 99
Switch1(config-if-range)# shutdown
Switch1(config-if-range)# exit
Switch1# write memory
```

`Repetir exatamente estes mesmos comandos no Switch2`


**Como verificar a VLAN nativa e as sub-interfaces?**

```
Switch0# show interfaces trunk
Router0# show ip interface brief
Router0# show running-config | section Gi0/0.40
```

**Como verificar se as portas ficaram fechadas?**
Usa o comando em modo privilegiado para ver o estado das interfaces:

``` 
show ip interface brief
```





### — Testar o acesso SSH

**Teste 1 — a partir do `PC-Gestao` (VLAN 999):**
```bash
ssh -l admin 10.0.0.1
```
`password: cinel`

> ✅ **Sucesso** — mesma VLAN 999 (propagada pelos trunks) que as SVIs de gestão.

**Teste 2 — a partir de um PC de outra VLAN (ex.: `PC0`, VLAN10):**
```bash
ssh -l admin 10.0.0.1
```

> ❌ **Falha** (`Destination host unreachable`) — mesmo com o `Router0` instalado, porque não existe `Gi0/0.999`; a rede 10.0.0.0/8 continua sem rota a partir das VLANs de utilizador.

---

# 3. Resumo dos resultados esperados (com Router0 instalado)

| Ponto do enunciado | Resultado a observar/reportar |
|---|---|
| 8a. `PC5` ↔ `PC0` | ✅ Sucesso (mesma VLAN10, via trunk) |
| 8b. `PC1` ↔ `PC5` | ✅ **Sucesso** agora que existe o `Router0` *(antes falhava)* |
| 8c. `PC2` ↔ `PC3` | ✅ Sucesso (mesma VLAN30, via trunk) |
| 12. SSH do PC da VLAN999 | ✅ Sucesso — mesmo domínio L2 da gestão |
| 13. SSH de um PC de outra VLAN | ❌ Falha — VLAN 999 sem sub-interface no router |
| Trunks (`show interfaces trunk`) | ✅ Native vlan = `40` em todos os trunks, sem *native VLAN mismatch* |

---

# 4. Checklist rápido

[ ] VLANs: VLANs 10, 20, 30, 40, 99 e 999 criadas em todos os switches.

[ ] Trunks: Links Fa0/1-3 no Switch0 e Fa0/24 nos switches de acesso configurados em modo trunk.

[ ] VLAN Nativa: `switchport trunk native vlan 40` aplicado em todos os trunks (Fa0/1-3 no Switch0, Fa0/24 no Switch1 e Switch2) e confirmado com `show interfaces trunk`.

[ ] Acesso: Portas dos utilizadores associadas às respetivas VLANs.

[ ] Router-on-a-Stick: Sub-interfaces Gi0/0.10, Gi0/0.20, Gi0/0.30 e Gi0/0.40 (`encapsulation dot1Q 40 native`) ativas no Router0.

[ ] DHCP Relay: Comando ip helper-address 192.168.10.254 aplicado em Gi0/0.20 e Gi0/0.30.

[ ] Server-DHCP: IP fixo 192.168.10.254/24 definido e os 3 pools ativos.

[ ] Endereçamento: Todos os PCs a obter IP e Gateway dinamicamente via DHCP.

[ ] Gestão & SSH: SVIs VLAN 999 ativas nos switches e SSH funcional a partir do PC-Gestao.

[ ] Hardening: Passwords encriptadas, login local, banner e portas inativas na VLAN 99 com estado shutdown.
