# Guia de Configuração — VLANs, Trunk, ROAS e SSH/TELNET (Cisco Packet Tracer)

## Índice

1. [Topologia e Endereçamento](#1-topologia-e-endereçamento)
2. [Passo a Passo no Cisco IOS](#2-passo-a-passo-no-cisco-ios)
   - [Tópico 1 — Criar VLANs](#-tópico-1--criar-vlans)
   - [Tópico 2 — Portas de acesso e trunk](#-tópico-2--portas-de-acesso-e-trunk)
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

### » VLANs, sub-redes e gateway

| VLAN | Nome | Sub-rede | Máscara | Gateway (`Router0`) |
|:---:|:---|:---|:---|:---|
| `10` | Estudantes | `192.168.10.0` | `/24` | `192.168.10.1` |
| `20` | Professores | `192.168.20.0` | `/24` | `192.168.20.1` |
| `30` | Direcao | `192.168.30.0` | `/24` | `192.168.30.1` |
| `999` | Gestao | `10.0.0.0` | `/8` | — *(sem gateway, ver nota)* |

> 📌 **Nota:** a VLAN 999 **não** tem sub-interface no router propositadamente, para a rede de gestão ficar isolada das VLANs de utilizador — isto mantém válidos os testes de SSH dos pontos 12 e 13 do enunciado.

### » Endereços IP dos PCs

| PC | VLAN | Método | Máscara | Gateway |
|---|:---:|---|---|---|
| `PC0` | 10 | `DHCP` | `/24` | `192.168.10.1` |
| `PC5` | 10 | `DHCP`  | `/24` | `192.168.10.1` |
| `PC1` | 20 | `DHCP`  | `/24` | `192.168.20.1` |
| `PC4` | 20 | `DHCP`  | `/24` | `192.168.20.1` |
| `PC2` | 30 | `DHCP` | `/24` | `192.168.30.1` |
| `PC3` | 30 | `DHCP`  | `/24` | `192.168.30.1` |
| `Server-DHCP` | `10` | `DHCP`  | `/24` | `192.168.10.1` |
| `PC-Gestao` | `999` | `Estático` | `/8` | `sem gateway` |

---

# 2. Passo a Passo no Cisco IOS

## » Tópico 1 — Criar VLANs


### Criar VLANs com VTP : 
Caso escolha utilizar VTP, deve `configurar o TRUNK` primeiro para que os switchs recebam a tabela de VLANs.

---> [Configuração Trunk](#-tópico-2--portas-de-acesso-e-trunk)
  
<details>
   
```
Switch0> enable
Switch0# configure terminal
Switch0(config)# vTP domain cinel-domain
Switch0(config)# vtp mode server
   
! (Opcional, mas boa prática):
Switch0(config)# vtp password cinel

```

> **Criar todas as VLANS apenas no `Switch0`**

```bash
enable
configure terminal
vlan 10
 name Estudantes
vlan 20
 name Professores
vlan 30
 name Direcao
vlan 999
 name Gestao
vlan 99
 name Portas_Inativas
exit
write memory
```

### Configurar Cliente VTP no SW1 e SW2:

```
Switch1> enable
Switch1# configure terminal
Switch1(config)# vtp domain cinel-domain
Switch1(config)# vtp mode client
! Se definiste password no Server:
Switch1(config)# vtp password cinel
Switch1(config)# exit
Switch1# write memory
```


### Confirmar funcionamento do VTP

`show vtp status`

</details>


### Criar VLANs sem vtp:

```bash
enable
configure terminal
vlan 10
 name Estudantes
vlan 20
 name Professores
vlan 30
 name Direcao
exit
write memory
```

Repetir em `Switch0`, `Switch1` e `Switch2`:

<details>

```
enable
configure terminal
vlan 10
name Estudantes
vlan 20
name Professores
vlan 30
name Direcao
exit
exit
write memory
```
</details>

**Teste:**
```
show vlan brief
```


---

## » Tópico 2 — Portas de acesso e trunk

<summary><strong>Switch0</strong> — Central (Trunks para Switch1, Switch2 e Router0)</summary>

```bash
Switch0(config)# interface fastEthernet 0/1
Switch0(config-if)# switchport mode trunk
Switch0(config-if)# no shutdown
Switch0(config-if)# exit

Switch0(config)# interface fastEthernet 0/2
Switch0(config-if)# switchport mode trunk
Switch0(config-if)# no shutdown
Switch0(config-if)# exit

Switch0(config)# interface fastEthernet 0/3
Switch0(config-if)# switchport mode trunk
Switch0(config-if)# no shutdown
Switch0(config-if)# exit

Switch0(config)# exit
Switch0# write memory
```

**SW-01**
<details>

**1. Trunk**
```
Switch1(config)# interface fastEthernet 0/24
Switch1(config-if)# switchport mode trunk
Switch1(config-if)# no shutdown
Switch1(config-if)# exit
```

**2. Portas de acesso aos PCs**
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


**SW-02**
<Details>

```
Switch2(config)# interface fastEthernet 0/24
Switch2(config-if)# switchport mode trunk
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

<!--
| Port | Mode | Encapsulation | Status | Nativa vlan |
|:----:|:----:|:-------------:|:------:|:-----------:|
| `fa0/2` | `on` | `802.1q` | `/trunking` | `1` |
-->

---

## » Tópico 3 — Router-on-a-Stick e DHCP-Relay

> **Explicação técnica:** Uma única interface física do router (Gi0/0) liga ao switch em trunk e divide-se em sub-interfaces lógicas. Como o Server-DHCP se encontra na VLAN 10, adicionamos o comando `ip helper-address 192.168.10.254` nas sub-interfaces das VLANs 20 e 30. Este comando transforma os broadcasts DHCP das VLANs 20 e 30 em pacotes Unicast reencaminhados diretamente ao servidor DHCP.

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
Router0(config-if)# write memory
```

> 📌 A interface física `Gi0/0` **não** recebe IP quando se usam sub-interfaces — só precisa de `no shutdown`. Repara que **não existe** `Gi0/0.999`: a VLAN de gestão fica fora do routing de propósito.

---

## » Tópico 4 — Configuração do Server-DHCP e Pools

**Configuração Manual do Server-DHCP**

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
`DHCP`



---

## » Tópico 5 — VLAN 999 e SSH

Repetir em `Switch0`, `Switch1` e `Switch2`, mudando apenas o IP:

```bash
Switch0(config)# vlan 999
Switch0(config-vlan)# name Gestao
Switch0(config-vlan)# exit

Switch0(config)# interface vlan 999
Switch0(config-if)# ip address 10.0.0.1 255.0.0.0
Switch0(config-if)# no shutdown
Switch0(config-if)# exit
Switch0# write memory
```

```bash
Switch1(config)# vlan 999
Switch1(config-vlan)# name Gestao
Switch1(config-vlan)# exit

Switch1(config)# interface vlan 999
Switch1(config-if)# ip address 10.0.0.2 255.0.0.0
Switch1(config-if)# no shutdown
Switch1(config-if)# exit
Switch1# write memory
```


```bash
Switch2(config)# vlan 999
Switch2(config-vlan)# name Gestao
Switch2(config-vlan)# exit

Switch2(config)# interface vlan 999
Switch2(config-if)# ip address 10.0.0.3 255.0.0.0
Switch2(config-if)# no shutdown
Switch2(config-if)# exit
Switch2# write memory
```

**SSH** (repetir nos três switches):
```bash
Switch0(config)# ip domain-name cinel.wan
Switch0(config)# username admin secret cinel
Switch0(config)# crypto key generate rsa
% How many bits in the modulus [512]: 2048
Switch0(config)# ip ssh version 2
Switch0(config)# line vty 0 4
Switch0(config-line)# login local
Switch0(config-line)# transport input ssh
Switch0(config-line)# exit
```

<details>
```
ip domain-name cinel.wan
username admin secret cinel
crypto key generate rsa
2048
ip ssh version 2
line vty 0 4
login local
transport input ssh
exit
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

``` 
Switch0(config)# interface range fastEthernet 0/4 - 9 , fastEthernet 0/11 - 23 , gigabitEthernet 0/1 - 2
Switch0(config-if-range)# switchport mode access
Switch0(config-if-range)# switchport access vlan 99
Switch0(config-if-range)# shutdown
Switch0(config-if-range)# exit
Switch0# write memory
```



**SWITCH 1 e 2 (Switches de Acesso):**
- Portas em uso: `Fa0/1`, `Fa0/2`, `Fa0/3` (PCs) e `Fa0/24` (trunk).
- Portas para fechar: `Fa0/4-23`, `Gi0/1-2`.

**(se não foi criada com vtp, criar VLAN 99 para Portas Inativas)**
```
Switch1(config)# vlan 99
Switch1(config-vlan)# name Portas_Inativas
Switch1(config-vlan)# exit
```

```
Switch1(config)# interface range fastEthernet 0/4 - 23 , gigabitEthernet 0/1 - 2
Switch1(config-if-range)# switchport mode access
Switch1(config-if-range)# switchport access vlan 99
Switch1(config-if-range)# shutdown
Switch1(config-if-range)# exit
Switch1# write memory
```

**`Repetir exatamente estes mesmos comandos no Switch2`**


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

---

# 4. Checklist rápido

- [ ] VLANs 10, 20 e 30 criadas em Switch0, Switch1 e Switch2

- [ ] Portas de acesso aos PCs associadas às VLANs corretas (PC0 a PC6)

- [ ] Trunks configurados nas interligações entre switches e com o Router0

- [ ] Router0: sub-interfaces Gi0/0.10, Gi0/0.20, Gi0/0.30 configuradas com gateway

- [ ] Router0: ip helper-address 192.168.10.254 adicionado em Gi0/0.20 e Gi0/0.30

- [ ] Server-DHCP: IP fixo 192.168.10.254/24 e as 3 Pools ativas (serverPool, VLAN20, VLAN30)

- [ ] Os 6 PCs configurados em modo DHCP obtendo os seus respetivos IPs

- [ ] Testes de ping 8a, 8b e 8c realizados com sucesso

- [ ] VLAN 999 Gestao criada e SVIs configuradas nos 3 switches

- [ ] SSH configurado nos 3 switches e testado a partir do PC-Gestao (sucesso) e de um PC de acesso (falha)

- [ ] Portas inativas fechadas com a VLAN 99 em todos os switches
