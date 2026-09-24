# Guia de Simulação — VLANs, Trunk, Router-on-a-Stick e Gestão SSH (Cisco Packet Tracer)

> Este guia parte do enunciado "VLANS" (ex1_vlans, 3 switches Cisco 2960-24TT, sem router) e **acrescenta um router configurado em Router-on-a-Stick (ROAS)** para fazer routing inter-VLAN. Confirma os nomes reais dos teus dispositivos e portas no teu `.pkt` — os nomes usados abaixo (`Switch0`, `Switch1`, `Switch2`, `Router0`, `Fa0/1`...`Fa0/24`, `Gi0/0`) são um exemplo coerente com o diagrama original; ajusta-os se a tua topologia usar outros números.

> ⚠️ **Atenção:** o enunciado original (ponto 8b) espera que o ping **entre VLANs diferentes falhe**, porque a topologia base não tem router. Com o Router-on-a-Stick deste guia, esse teste passa a **ter sucesso**. Se este documento for para entregar como resposta ao enunciado original, confirma com o professor se a adição do router é pretendida.

---

## Índice

1. [Topologia e Endereçamento](#1-topologia-e-endereçamento)
2. [Passo a Passo no Cisco IOS](#2-passo-a-passo-no-cisco-ios)
   - [Tópico 1 — Criar VLANs](#-tópico-1--criar-as-vlans-10-20-e-30)
   - [Tópico 2 — Portas de acesso e trunk](#-tópico-2--portas-de-acesso-e-trunk)
   - [Tópico 3 — Router-on-a-Stick](#-tópico-3--router-on-a-stick-router0)
   - [Tópico 4 — IP e gateway dos PCs](#-tópico-4--atribuir-ip-e-gateway-aos-pcs)
   - [Tópico 5 — Testes de ping](#-tópico-5--testes-de-ping-resultado-atualizado-com-o-router)
   - [Tópico 6 — VLAN 999 e SSH](#-tópico-6--vlan-999-gestão-e-acesso-ssh)
   - [Tópico 7 — PC na Fa0/10](#-tópico-7--pc-na-fa010-com-vlan-999)
   - [Tópico 8 — Fechar portas](#-tópico-8--fechar-portas)
   - [Testar SSH](#-testar-o-acesso-ssh)
3. [Resumo dos resultados esperados](#3-resumo-dos-resultados-esperados-com-router0-instalado)
4. [Checklist rápido](#4-checklist-rápido)

---

# 1. Topologia e Endereçamento

### » Dispositivos

| Dispositivo | Papel |
|---|---|
| `Switch0` | Switch central (distribuição) — liga Switch1, Switch2 e Router0 por trunk |
| `Switch1` | Switch de acesso — liga PC0, PC1, PC2 |
| `Switch2` | Switch de acesso — liga PC3, PC4, PC5 |
| `Router0` | Router-on-a-Stick — faz o routing inter-VLAN |
| `PC0`–`PC5` | Terminais dos utilizadores |

### » Diagrama

```
                                [ Router0 ]
                                Gi0/0 (trunk / ROAS)
                                     |
                     [ Switch0 ]  (central, sem PCs)
              Fa0/1 |      Fa0/2 |      | Fa0/3
          (trunk)   |  (trunk)   |      | (trunk, p/ Router0)
                     |            |
        [ Switch1 ]---            ---[ Switch2 ]
     Fa0/1: PC0 (VLAN10)          Fa0/1: PC3 (VLAN30)
     Fa0/2: PC1 (VLAN20)          Fa0/2: PC4 (VLAN20)
     Fa0/3: PC2 (VLAN30)          Fa0/3: PC5 (VLAN10)
     Fa0/24: trunk → Switch2      Fa0/24: trunk → Switch2
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

| PC | VLAN | IP | Máscara | Gateway |
|---|:---:|---|---|---|
| `PC0` | 10 | `192.168.10.10` | `/24` | `192.168.10.1` |
| `PC5` | 10 | `192.168.10.11` | `/24` | `192.168.10.1` |
| `PC1` | 20 | `192.168.20.10` | `/24` | `192.168.20.1` |
| `PC4` | 20 | `192.168.20.11` | `/24` | `192.168.20.1` |
| `PC2` | 30 | `192.168.30.10` | `/24` | `192.168.30.1` |
| `PC3` | 30 | `192.168.30.11` | `/24` | `192.168.30.1` |

---

# 2. Passo a Passo no Cisco IOS

### ! ALTERAR HOSTNAME DE CADA DISPOSITIVO ! 

```
enable
configure terminal
hostname NAME
```


### » Tópico 1 — Criar as VLANs 10, 20 e 30

Repetir em `Switch0`, `Switch1` e `Switch2`:

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

**Teste:**
`show vlan brief`


---

### » Tópico 2 — Portas de acesso e trunk

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

`REPETIR PARA O SW-2`


---

### » Tópico 3 — Router-on-a-Stick (`Router0`)

> **Explicação técnica:** uma única interface física do router (`Gi0/0`) liga a uma porta trunk do switch e é dividida em **sub-interfaces lógicas**, uma por VLAN, cada uma com `encapsulation dot1Q <vlan>` e o respetivo IP de gateway.

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

**VLAN 20 (Professores)**
```bash
Router0(config)# interface gigabitEthernet 0/0.20
Router0(config-subif)# encapsulation dot1Q 20
Router0(config-subif)# ip address 192.168.20.1 255.255.255.0
Router0(config-subif)# exit
```

**VLAN 30**
```bash
Router0(config)# interface gigabitEthernet 0/0.30
Router0(config-subif)# encapsulation dot1Q 30
Router0(config-subif)# ip address 192.168.30.1 255.255.255.0
Router0(config-subif)# exit
Router0(config-if)# write memory
```

> 📌 A interface física `Gi0/0` **não** recebe IP quando se usam sub-interfaces — só precisa de `no shutdown`. Repara que **não existe** `Gi0/0.999`: a VLAN de gestão fica fora do routing de propósito.

---

### » Tópico 4 — Atribuir IP e gateway aos PCs

Em cada PC: `Desktop → IP Configuration → Static`, com IP, máscara e **Default Gateway** conforme a tabela da secção 1.

---

### » Tópico 5 — Testes de ping (resultado atualizado com o router)

| Teste | PCs | Mesma VLAN? | Antes do router | **Agora** (com `Router0`) |
|:---:|---|:---:|:---:|:---:|
| a | `PC5` ↔ `PC0` | Sim (VLAN10) | ✅ Sucesso | ✅ Sucesso *(sem alteração)* |
| b | `PC1` ↔ `PC5` | Não (VLAN20 ↔ VLAN10) | ❌ Falha | ✅ **Sucesso** *(agora rotado pelo Router0)* |
| c | `PC2` ↔ `PC3` | Sim (VLAN30) | ✅ Sucesso | ✅ Sucesso *(sem alteração)* |

> ✅ **Resumo:** os pings intra-VLAN (a, c) continuam a funcionar por Camada 2. O ping inter-VLAN (b) passa a funcionar porque o `Router0` recebe o pacote em `Gi0/0.20`, consulta a tabela de routing e reencaminha-o por `Gi0/0.10`.

---

### » Tópico 6 — VLAN 999 "gestão" e acesso SSH

Repetir em `Switch0`, `Switch1` e `Switch2`, mudando apenas o IP:

```bash
Switch0(config)# vlan 999
Switch0(config-vlan)# name Gestao
Switch0(config-vlan)# exit

Switch0(config)# interface vlan 999
Switch0(config-if)# ip address 10.0.0.1 255.0.0.0
Switch0(config-if)# no shutdown
Switch0(config-if)# exit
Switch0(config-vlan)# write memory
```

```bash
Switch1(config)# vlan 999
Switch1(config-vlan)# name Gestao
Switch1(config-vlan)# exit

Switch1(config)# interface vlan 999
Switch1(config-if)# ip address 10.0.0.2 255.0.0.0
Switch1(config-if)# no shutdown
Switch1(config-if)# exit
Switch1(config-vlan)# write memory
```
```bash
Switch2(config)# vlan 999
Switch2(config-vlan)# name Gestao
Switch2(config-vlan)# exit

Switch2(config)# interface vlan 999
Switch2(config-if)# ip address 10.0.0.3 255.0.0.0
Switch2(config-if)# no shutdown
Switch2(config-if)# exit
Switch2(config-vlan)# write memory
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

---

### » Tópico 7 — PC na Fa0/10 com VLAN 999

```bash
Switch0(config)# interface fastEthernet 0/10
Switch0(config-if)# switchport mode access
Switch0(config-if)# switchport access vlan 999
Switch0(config-if)# no shutdown
Switch0(config-if)# exit
Switch0(config-if)# write memory

```

| Dispositivo | IP | Máscara |
|---|---|---|
| `PC-Gestao` | `10.0.0.10` | `/8` |

---

### » Tópico 8 — FECHAR PORTAS

**SWITCH 0 (Switch Central):**
- Portas em uso: Fa0/1, Fa0/2, Fa0/3 (trunks).
- Portas sem uso para fechar: Fa0/4 até Fa0/24 + Gi0/1 e Gi0/2.

```
Switch0> enable
Switch0# configure terminal
```


! Seleciona o intervalo de portas FastEthernet de 4 a 24 e as portas Gigabit
```
Switch0(config)# interface range fastEthernet 0/4 - 24 , gigabitEthernet 0/1 - 2
Switch0(config-if-range)# shutdown
Switch0(config-if-range)# exit
```

**SWITCH 1 e 2 (Switches de Acesso):**
- Portas em uso: Fa0/1, Fa0/2, Fa0/3 (PCs) e Fa0/24 (trunk).

- Portas sem uso para fechar: Fa0/4 até Fa0/23 + Gi0/1 e Gi0/2.
(Se tiveres a porta Fa0/10 atribuída à VLAN 999 para o PC-Gestao, lembra-te de não a incluir no range).

```
Switch1> enable
Switch1# configure terminal
```

! Exemplo desativando portas 4 a 9, 11 a 23 e Gigabit:
```
Switch1(config)# interface range fastEthernet 0/4 - 9 , fastEthernet 0/11 - 23 , gigabitEthernet 0/1 - 2
Switch1(config-if-range)# shutdown
Switch1(config-if-range)# exit
```

***Mover portas não utilizadas para uma VLAN sem tráfego (Boa Prática de Segurança)***

`Além de dar shutdown, é uma recomendação oficial de hardening da Cisco mover todas as portas não utilizadas para uma VLAN dedicada sem acesso a nada (ex.: VLAN 99 ou VLAN 666 apelidada de Blackhole ou Unused).`

```
Switch0(config)# vlan 99
Switch0(config-vlan)# name Portas_Inativas
Switch0(config-vlan)# exit

Switch0(config)# interface range fastEthernet 0/4 - 24 , gigabitEthernet 0/1 - 2
Switch0(config-if-range)# switchport mode access
Switch0(config-if-range)# switchport access vlan 99
Switch0(config-if-range)# shutdown
Switch0(config-if-range)# exit
```

***Como verificar se as portas ficaram fechadas?***
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

- [ ] VLANs 10, 20 e 30 criadas em `Switch0`, `Switch1` e `Switch2`
- [ ] VLAN10 renomeada para `Estudantes`, VLAN20 para `Professores`
- [ ] Portas de acesso configuradas (`PC0`–`PC5`) nas VLANs corretas
- [ ] Trunks: `Switch0↔Switch2`, `Switch1↔Switch2`, `Switch2↔Router0`
- [ ] `Router0`: sub-interfaces `Gi0/0.10`, `Gi0/0.20`, `Gi0/0.30` com IP de gateway
- [ ] IP + gateway atribuídos aos 6 PCs
- [ ] Teste de ping 8a realizado (sucesso esperado)
- [ ] Teste de ping 8b realizado (sucesso esperado, com router)
- [ ] Teste de ping 8c realizado (sucesso esperado)
- [ ] VLAN 999 `Gestao` criada e SVI atribuída nos 3 switches
- [ ] SSH configurado (`domain-name`, `username`, RSA 2048 bits, `line vty`) nos 3 switches
- [ ] `PC-Gestao` ligado à `Fa0/10`, VLAN999, IP `10.0.0.10`
- [ ] Teste SSH 12 realizado (sucesso esperado)
- [ ] Teste SSH 13 realizado (falha esperada)
