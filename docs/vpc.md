# Template de CloudFormation para VPC com Sub-redes Pública, Privada e Isolada

Este template cria uma **VPC (Virtual Private Cloud)** na AWS com três sub-redes: **Pública**, **Privada (Aplicação)** e **Isolada (Banco de Dados)**.

---

## 📋 Sumário
1. [Visão Geral](#-visão-geral)
2. [Estrutura do Template](#-estrutura-do-template)
3. [Recursos Criados](#-recursos-criados)
4. [Personalização](#-personalização)
5. [Como Implantar](#-como-implantar)
6. [FAQ](#-faq)

---

## 🌐 Visão Geral
O template define uma infraestrutura segura e escalável na AWS com:
- **Sub-rede Pública**: Para recursos expostos à internet (ex: servidores web).
- **Sub-rede Privada**: Para recursos com acesso à internet apenas para saída (ex: aplicações internas).
- **Sub-rede Isolada**: Para recursos sem acesso à internet (ex: bancos de dados).

---

## 🏧 Estrutura do Template

### Parâmetros (Parameters)
```yaml
Parameters:
  VpcCidr:
    Type: String
    Default: 10.0.0.0/16
    Description: Bloco CIDR para a VPC (ex: 10.0.0.0/16)
```

- **VpcCidr**: Bloco de IPs da VPC. Valor padrão 10.0.0.0/16 (privado, sem conflitos com redes públicas).
- **PublicSubnetCidr, AppSubnetCidr, DBSubnetCidr**: Blocos de IPs para cada sub-rede.

### Recursos (Resources)

#### 1. VPC
```yaml
VPC:
  Type: AWS::EC2::VPC
  Properties:
    CidrBlock: !Ref VpcCidr
    EnableDnsSupport: true
    EnableDnsHostnames: true
```
**Propósito**: Cria uma rede virtual isolada.

**Configurações**:
- **CidrBlock**: Define o bloco de IPs da VPC.
- **EnableDnsSupport/EnableDnsHostnames**: Habilita resolução de DNS.

#### 2. Sub-redes
```yaml
PublicSubnet:
  Type: AWS::EC2::Subnet
  Properties:
    CidrBlock: !Ref PublicSubnetCidr
    AvailabilityZone: !Select [0, !GetAZs '']
    MapPublicIpOnLaunch: true
```
**Tipos**:
- **Pública**: `MapPublicIpOnLaunch: true` (IP público automático).
- **Privada**: Sem IP público, acesso à internet via NAT Gateway.
- **Isolada**: Sem rotas para internet.

#### 3. Internet Gateway (IGW)
```yaml
InternetGateway:
  Type: AWS::EC2::InternetGateway
```
**Propósito**: Permite acesso à internet para a sub-rede pública.

#### 4. NAT Gateway
```yaml
NATGateway:
  Type: AWS::EC2::NatGateway
  Properties:
    AllocationId: !GetAtt EIP.AllocationId
    SubnetId: !Ref PublicSubnet
```
**Propósito**: Permite acesso à internet para a sub-rede privada.

**Localização**: Deve estar na sub-rede pública (para obter IP público).

#### 5. Tabelas de Rotas (Route Tables)
| Tipo    | Rota        | Gateway          | Sub-redes Associadas       |
|---------|------------|------------------|----------------------------|
| Pública | 0.0.0.0/0  | Internet Gateway | Sub-rede Pública          |
| Privada | 0.0.0.0/0  | NAT Gateway      | Sub-rede de Aplicação      |
| Isolada | Sem rotas  | N/A              | Sub-rede de Banco de Dados |

### Outputs
```yaml
Outputs:
  VPCId:
    Value: !Ref VPC
    Description: ID da VPC criada
```
**Saídas Úteis**: IDs da VPC e sub-redes para uso em outros templates.

---

## 🛠️ Personalização

### 1. Alterar Blocos CIDR
```yaml
Parameters:
  VpcCidr:
    Default: 192.168.0.0/16  # Novo bloco para a VPC
  PublicSubnetCidr:
    Default: 192.168.1.0/24  # Novo bloco para a sub-rede pública
```

### 2. Usar Zonas de Disponibilidade Específicas
```yaml
PublicSubnet:
  Properties:
    AvailabilityZone: us-east-1a  # Altere para a AZ desejada
```

### 3. Adicionar Mais Sub-redes
```yaml
AppSubnet2:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    CidrBlock: 10.0.4.0/24
    AvailabilityZone: us-east-1b
```

---

## 🚀 Como Implantar
1. Salve o template como `vpc-template.yml`.
2. **Console AWS**:
    - Acesse *CloudFormation* > *Create Stack*.
    - Faça upload do template e preencha os parâmetros.
    - Aguarde a conclusão e verifique os recursos na aba *Resources*.

---

## ❓ FAQ

### 1. Por que usar `10.0.0.0/16`?
- **Motivo**: É um bloco de IPs privado (RFC 1918), não roteável na internet pública.
- **Evita conflitos**: Não interfere com a VPC padrão da AWS (`172.31.0.0/16`).

### 2. Posso usar apenas duas sub-redes?
Sim! Remova a seção `DBSubnet` e ajuste as tabelas de rotas.

### 3. Quanto custa o NAT Gateway?
- **Custo**: ~$0.045 por hora + custo de dados processados.

---

## 🔒 Melhores Práticas
- **Security Groups**: Restrinja tráfego entre sub-redes.
- **NACLs**: Adicione regras de rede para controle adicional.
- **Monitoramento**: Use *VPC Flow Logs* para auditoria.