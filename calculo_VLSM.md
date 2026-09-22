# Cálculo de VLSM (Variable Length Subnet Mask)

O cálculo de VLSM para qualquer bloco de IP fornecido segue uma metodologia padronizada. A regra fundamental do VLSM é **ordenar sempre os requisitos das sub-redes/VLANs da maior necessidade de hosts para a menor** antes de atribuir os endereços.

## Roteiro Genérico em 4 Passos

1. **Ordenar as VLANs:** Organize as sub-redes da maior quantidade de hosts necessários para a menor.

2. **Encontrar os bits de host (**$h$**):** Utilize a fórmula:
   

 `
   2^h - 2 >= Hosts Necessários
 `

   
   *(Subtraem-se 2 endereços para o IP de Rede e o IP de Broadcast)*.

3. **Calcular a Máscara e o Salto:**

   * **Prefixo CIDR:** $32 - h$

   * **Salto (Tamanho do Bloco):** $2^h$

   * **Máscara Decimal:** Converte-se o prefixo CIDR para formato decimal de 4 octetos.

4. **Mapear a Tabela em Sequência:**

   * O IP da Rede da primeira VLAN é o IP inicial do bloco fornecido.

   * O IP da Rede da VLAN seguinte é igual ao IP da Rede anterior + o Salto ($2^h$).

## Exemplo Prático: Bloco Base 172.16.10.0/24 para 4 VLANs

Suponha que recebe o bloco `172.16.10.0/24` e precisa de criar 4 VLANs com os seguintes requisitos:

* **VLAN 10 (Vendas):** 50 hosts

* **VLAN 20 (Engenharia):** 20 hosts

* **VLAN 30 (Gestão):** 10 hosts

* **VLAN 40 (Link de Roteamento):** 2 hosts

### Passo 1 & 2: Ordenar e Calcular Bits ($h$), CIDR e Salto

#### 1ª VLAN — VLAN 10 (50 hosts)

* **Fórmula:** 2^h - 2 >= 50 -> h = 6 bits (2^6 - 2 = 62 hosts válidos)

* **CIDR:** 32 - 6 = /26

* **Máscara:** `255.255.255.192`

* **Salto:** 2^6 = /64

#### 2ª VLAN — VLAN 20 (20 hosts)

* **Fórmula:** 2^h - 2 >=20 -> h = 5 bits (2^5 - 2 = 30\hosts válidos)

* **CIDR:** 32 - 5 = /27

* **Máscara:** `255.255.255.224`

* **Salto:** $2^5 = 32

#### 3ª VLAN — VLAN 30 (10 hosts)

* **Fórmula:** 2^h - 2 >= 10 -> h = 4 bits (2^4 - 2 = 14 hosts válidos)

* **CIDR:** 32 - 4 = /28

* **Máscara:** `255.255.255.240`

* **Salto:** 2^4 = 16

#### 4ª VLAN — VLAN 40 (2 hosts)

* **Fórmula:** 2^h - 2 >= 2 -> h = 2 bits (2^2 - 2 = 2  hosts válidos)

* **CIDR:** 32 - 2 = /30

* **Máscara:** `255.255.255.252`

* **Salto:** 2^2 = 4

### Passo 3 & 4: Mapeamento Completo de Endereços

Começando do IP base `172.16.10.0` e somando os saltos:

| **VLAN**    | **IP de Rede**  | **1º IP Válido** | **Último IP Válido** | **IP de Broadcast** | **Máscara / CIDR**        | 
|:-------- |:--------:|:--------:|:--------:|:--------:| --------:|
| **VLAN 10** | `172.16.10.0`   | `172.16.10.1`    | `172.16.10.62`       | `172.16.10.63`      | `255.255.255.192` (`/26`) | 
| **VLAN 20** | `172.16.10.64`  | `172.16.10.65`   | `172.16.10.94`       | `172.16.10.95`      | `255.255.255.224` (`/27`) | 
| **VLAN 30** | `172.16.10.96`  | `172.16.10.97`   | `172.16.10.110`      | `172.16.10.111`     | `255.255.255.240` (`/28`) | 
| **VLAN 40** | `172.16.10.112` | `172.16.10.113`  | `172.16.10.114`      | `172.16.10.115`     | `255.255.255.252` (`/30`) | 
