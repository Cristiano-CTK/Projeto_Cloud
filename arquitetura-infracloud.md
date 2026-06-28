# Arquitetura de Infraestrutura Cloud — E-commerce de Pneus (Multi-Cloud Azure + OCI + Infra On-Premises)

## 1. Contexto

A plataforma de e-commerce responde por mais de 80% da receita da empresa, portanto **disponibilidade e resiliência são críticas**. Cerca de 80% da infraestrutura já está sendo disponibilizada na estrutura do fornecedor **Microsoft Azure**, ainda há serviços **on-premises**, e há um início de movimentação estratégica para **multi-cloud com a Oracle Cloud Infrastructure (OCI)**. As recomendações abaixo sustentam o ambiente atual e habilitam a evolução futura. Devo salientar que a maioria das ações aqui propostas são consideradas a serem tomadas/executadas em curto e médio prazo; para que se possa propor ações de longo prazo — que tendem a trazer resultados interessantes principalmente na parte de FinOps — devo me aprofundar mais no cotidiano da empresa para entender se é viável utilizar ações como spot instances para certas workloads, o que neste exemplo traz uma redução profunda de alguns custos.

**Princípios balizadores do projeto:** Azure first (nuvem primária), topologia Hub-Spoke, Zero Trust, Infraestrutura como Código (IaC), FinOps, observabilidade unificada entre nuvens públicas e ambiente on-premises com workloads não críticas.

---

## 2. Como cada ponto obrigatório é atendido

### 2.1 Arquitetura de Rede
- **Topologia Hub-Spoke** sobre **Azure Virtual WAN**. O **Hub VNet** centraliza conectividade: Azure Firewall Premium, ExpressRoute Gateway, VPN Gateway, Bastion host, Private DNS e API Management.
- **Spokes** segregam responsabilidades: *Spoke de Aplicação* (frontend, APIs, containers) e *Spoke de Dados* (banco, cache, storage), com **NSGs por subnet** e **Private Endpoints** (sem exposição pública dos dados).
- Conectividade híbrida com o ambiente on-premises via **ExpressRoute + VPN Site-to-Site (ativo/ativo)**. O ExpressRoute é privado, mas não criptografa o tráfego por padrão; por isso aplicamos IPSec sobre ExpressRoute. A VPN Site-to-Site já é criptografada nativamente por IPSec. A camada de SD-WAN complementa, garantindo criptografia ponta a ponta e otimização do tráfego ao disponibilizar utilização ativo/ativo de ambas as conexões.
- Entrada de tráfego pela borda com **Azure Front Door + WAF + DDoS + CDN**.

### 2.2 Resiliência da Plataforma
- **Multi-região** (Brazil South primária + região secundária de DR) com **Zonas de Disponibilidade**.
- **Azure Front Door** faz **failover global automático** entre regiões.
- **Autoscaling** em todas as camadas: AKS (Cluster Autoscaler/HPA), VMSS e Container Apps.
- Estratégia DR **Active/Warm-Standby** com **Azure Backup + Site Recovery** e **geo-replicação de dados**; runbooks e testes de DR periódicos automatizados (RTO/RPO baixos).

### 2.3 Segurança
- Gestão de identidade centralizada via **Microsoft Entra ID**: MFA, Conditional Access e **Zero Trust**.
- **Microsoft Defender for Cloud** (CSPM/CWP) e **Microsoft Sentinel** (SIEM/SOAR). Parto do princípio de que o ambiente ainda não possui camadas proativas (SIEM/SOC/SOAR), por isso proponho Sentinel + Defender.
- **Azure Firewall Premium (IDPS) + WAF + DDoS Protection Standard**.
- **Key Vault** para segredos/chaves/certificados, criptografia em repouso e trânsito.
- Acesso administrativo exclusivo via **Bastion host**; VMs **sem IP público**; comunicação via **Private Endpoints**.

### 2.4 Observabilidade
- **Azure Monitor + Log Analytics** como workspace central.
- **Application Insights** para APM e tracing distribuído.
- **Azure Managed Grafana + Managed Prometheus** para dashboards e métricas.
- Alertas, painéis de saúde/SLA e **coleta unificada de Azure + OCI + on-premises**.

### 2.5 Banco de Dados
- Sugestão de migração do **MongoDB em VMs** para **Azure Cosmos DB for MongoDB (vCore)** — serviço gerenciado, multi-AZ, backup contínuo e **Point-in-Time Restore**. Medida aplicada apenas aos databases críticos. Devemos, em fase 2, analisar quais medidas tomar para os databases considerados não críticos.
- **Geo-replicação** para a região de DR com failover automático.
- **Azure Cache for Redis** para sessão e performance.

### 2.6 Governança
- **Management Groups + Subscriptions** organizados por **Azure Landing Zones**.
- **Azure Policy** para guardrails e compliance; **RBAC + PIM** (acesso mínimo, just-in-time).
- Padronização de **Tags + Blueprints**; provisionamento por **IaC Terraform**, já adaptando para futura conexão multi-cloud.

### 2.7 Multi-Cloud (OCI)
- **Oracle Interconnect for Azure**: **ExpressRoute (Azure) + FastConnect (OCI)** — conexão privada, baixa latência (~2 ms) e sem taxa de transferência.
- Cargas selecionadas em OCI em modelo **split-stack** (ex.: ambiente de Data Lake da empresa sendo consumido via OCI com custo menor).
- **SSO federado OCI ↔ Entra ID** para identidade unificada.
- **Balanceamento de cargas Azure × OCI** — à medida que a maturidade do ambiente atinge os indicadores propostos no planejamento, a equipe deve redirecionar workloads para a OCI region contratada, levando em consideração disponibilidade, SLA e custo. Medida adotada como plano de contingência para um cenário de indisponibilidade total do Azure.

### 2.8 Gestão de Custos
- **Azure Cost Management + Billing** com **budgets**, alertas e detecção de anomalias.
- **Reservations + Savings Plans** para cargas estáveis; **VM Scale Sets (VMSS)** para cargas elásticas, com padrões bem definidos evitando workloads com overprovisioning.
- **Right-sizing** via Azure Advisor e cultura **FinOps** com rateio por Tags/área.
- **Comitê de Custos Cloud** — criação de um comitê com reunião mensal para apresentação de contas e análises sintéticas, direcionando os gastos em cloud. O comitê inicia com faturas das clouds + análises em Excel (fase 1); na próxima fase, passa a consumir APIs de billing para disponibilizar uma visão *Single Pane of Glass* dos custos de Tecnologia da empresa. Uma vez atingido o último nível de maturidade estabelecido, inicia-se a validação de soluções de sistema de gestão financeira de cloud (CFM).

### 2.9 Microsoft 365
- Integrado à mesma identidade (**Entra ID**), garantindo SSO e Conditional Access a todos os sistemas da empresa. Por meio do Entra ID podemos publicar de forma segura, via **Microsoft Entra Private Access / Global Secure Access (GSA)**, os ambientes da empresa.
- **Microsoft Purview** (DLP/compliance) e **Defender for Office 365** para proteção de colaboração (Teams, Exchange, SharePoint, OneDrive).

---

## 3. Fluxo Principal de Requisição

1. Clientes acessam via Internet -> **Front Door (WAF/DDoS/CDN)**.
2. Front Door roteia para a região saudável -> **Application Gateway + WAF v2**. O Application Gateway encaminha para o **AKS** (frontend, APIs, containers auxiliares). Para este objeto devemos implementar análise de rate-limit, controle de solicitações das aplicações e gestão de tráfego, pois será por meio do Application Gateway que as requisições terão contato direto com os objetos ou filas necessárias para o andamento do fluxo do sistema. Estratégia conhecida como defesa em profundidade, pois teremos o controlador de entrada no ambiente e o controlador de entrega do ambiente.
3. AKS consome dados via **Private Endpoint** no **Cosmos DB for MongoDB** (e Redis/Storage).
4. Falha regional -> Front Door redireciona automaticamente para a região de DR. Ao utilizarmos um serviço global, temos condição de redirecionar o tráfego para outra região em caso de falha. Importante colocar que podemos considerar a utilização de outro serviço de CDN, como Akamai ou Cloudflare, que traz novos prós e contras para o ambiente a serem considerados.
5. Usuários corporativos e M365 autenticam no **Entra ID**; on-premises sincroniza via **Entra Connect**; OCI conecta-se pelo **Interconnect** privado.

---

## 4. Melhorias e Adaptações Propostas

- Migrar os **File Servers tradicionais** para **Azure Files / Azure File Sync**, reduzindo o legado on-premises e entregando acesso granular controlado aos usuários da empresa.
- Uma vez que a empresa já utiliza esteiras de CI/CD, podemos analisar como estão as branches para utilizar o framework de **GitOps/CI-CD** no deploy dos clusters AKS e nos pipelines de IaC, bem como nas validações de testes do ambiente antes da subida em produção.
- Evoluir o on-premises para um modelo de **testes e validações em ambientes não críticos** quando viável. Levando em conta que a análise de custo do ambiente on-premises é mais estática, estruturamos para que somente workloads não críticas se mantenham rodando nesse ambiente.
- Utilização de IaC no ambiente on-premises: uma vez que a empresa já utiliza esteiras de CI/CD e IaC para os ambientes cloud, devemos desenvolver a integração das esteiras com este ambiente local, a fim de que ele entre também na gestão de Tags para controle de ambiente, custos e otimizações como um todo.
- Implementar **Chaos Engineering** para validar continuamente os mecanismos de resiliência. Sugestão de implementação por fases (de 1 a 5): a fase 1 começa com a ingestão de falhas menores no ambiente e, à medida que os KPIs dessas métricas forem atendidos, avançamos para a próxima fase, com ingestão de falhas mais críticas, até atingirmos a última fase — que pode chegar a uma engenharia de caos completa, com a parada total da zona de produção, validando assim todas as rotinas de H.A., D.R. e demais medidas necessárias.
- Elaboração de um **plano anti lock-in**: tendo em vista que a empresa não pode estar totalmente vinculada a um único vendor, devemos elaborar um plano em que qualquer tecnologia utilizada não traga lock-in de ambiente (ou traga o menor lock-in possível).

---

## 5. Entregáveis

- 'arquitetura-infracloud.drawio' — diagrama editável (draw.io).
- 'arquitetura-infracloud.png' — imagem.
- 'arquitetura-infracloud.md' — descrição.
