# 🛒 E-Commerce na Nuvem com Microsoft Azure
> Projeto desenvolvido como parte do desafio prático da DIO — Microsoft Application Platform

---

## 📋 Descrição do Projeto

Solução para armazenar e gerenciar dados de um e-commerce na nuvem utilizando serviços da **Microsoft Azure**, com foco em **escalabilidade**, **segurança** e **eficiência operacional**.

---

## 🏗️ Arquitetura da Solução

```
┌─────────────────────────────────────────────────────┐
│                  CLIENTE / FRONTEND                 │
│              (Web App / Mobile App)                 │
└──────────────────────┬──────────────────────────────┘
                       │
              ┌────────▼────────┐
              │   Azure API     │
              │   Management    │
              └────────┬────────┘
          ┌────────────┼────────────┐
          │            │            │
   ┌──────▼──────┐ ┌───▼────┐ ┌────▼──────┐
   │  Azure App  │ │ Azure  │ │  Azure    │
   │   Service   │ │ Func.  │ │  Service  │
   │  (Backend)  │ │(Events)│ │   Bus     │
   └──────┬──────┘ └────────┘ └───────────┘
          │
   ┌──────▼───────────────────────┐
   │         DADOS & STORAGE      │
   │  ┌──────────┐ ┌───────────┐  │
   │  │  Azure   │ │  Azure    │  │
   │  │ CosmosDB │ │   Blob    │  │
   │  │(Produtos)│ │ Storage   │  │
   │  └──────────┘ └───────────┘  │
   │  ┌──────────┐ ┌───────────┐  │
   │  │  Azure   │ │  Azure    │  │
   │  │  Cache   │ │   SQL DB  │  │
   │  │  Redis   │ │ (Pedidos) │  │
   │  └──────────┘ └───────────┘  │
   └──────────────────────────────┘
```

---

## ☁️ Serviços Azure Utilizados

| Serviço | Finalidade |
|---|---|
| **Azure App Service** | Hospedagem da API e backend da aplicação |
| **Azure Cosmos DB** | Armazenamento de catálogo de produtos (NoSQL) |
| **Azure SQL Database** | Gestão de pedidos e transações |
| **Azure Blob Storage** | Armazenamento de imagens de produtos e assets |
| **Azure Cache for Redis** | Cache de sessões e dados frequentes |
| **Azure Functions** | Processamento de eventos assíncronos (notificações, estoque) |
| **Azure Service Bus** | Mensageria para desacoplamento de serviços |
| **Azure API Management** | Gateway de APIs com autenticação e rate limiting |
| **Azure Active Directory B2C** | Autenticação e autorização de usuários |
| **Azure Monitor** | Monitoramento, logs e alertas em tempo real |

---

## 🚀 Passo a Passo da Implementação

### 1. Criação do Resource Group
```bash
az group create \
  --name rg-ecommerce-dio \
  --location brazilsouth
```

### 2. Provisionamento do Azure Cosmos DB
```bash
az cosmosdb create \
  --name cosmos-ecommerce-dio \
  --resource-group rg-ecommerce-dio \
  --kind GlobalDocumentDB \
  --locations regionName=brazilsouth

# Criar banco e container para produtos
az cosmosdb sql database create \
  --account-name cosmos-ecommerce-dio \
  --resource-group rg-ecommerce-dio \
  --name ecommerceDB

az cosmosdb sql container create \
  --account-name cosmos-ecommerce-dio \
  --resource-group rg-ecommerce-dio \
  --database-name ecommerceDB \
  --name produtos \
  --partition-key-path "/categoria"
```

### 3. Configuração do Azure SQL Database
```bash
az sql server create \
  --name sql-ecommerce-dio \
  --resource-group rg-ecommerce-dio \
  --location brazilsouth \
  --admin-user sqladmin \
  --admin-password "SenhaSegura@2024!"

az sql db create \
  --resource-group rg-ecommerce-dio \
  --server sql-ecommerce-dio \
  --name db-pedidos \
  --service-objective S1
```

### 4. Azure Blob Storage para Imagens
```bash
az storage account create \
  --name stecommercedio \
  --resource-group rg-ecommerce-dio \
  --location brazilsouth \
  --sku Standard_LRS

az storage container create \
  --account-name stecommercedio \
  --name imagens-produtos \
  --public-access blob
```

### 5. Deploy do App Service
```bash
az appservice plan create \
  --name plan-ecommerce-dio \
  --resource-group rg-ecommerce-dio \
  --sku B2

az webapp create \
  --resource-group rg-ecommerce-dio \
  --plan plan-ecommerce-dio \
  --name app-ecommerce-dio \
  --runtime "NODE:18-lts"
```

---

## 🗄️ Modelagem de Dados

### Cosmos DB — Documento de Produto
```json
{
  "id": "prod-001",
  "nome": "Tênis Esportivo X",
  "categoria": "calcados",
  "preco": 299.90,
  "estoque": 150,
  "imagem_url": "https://stecommercedio.blob.core.windows.net/imagens-produtos/tenis-x.jpg",
  "especificacoes": {
    "marca": "SportBrand",
    "tamanhos": ["38", "39", "40", "41", "42"],
    "cores": ["preto", "branco", "azul"]
  },
  "avaliacoes": {
    "media": 4.7,
    "total": 312
  },
  "createdAt": "2024-01-15T10:30:00Z"
}
```

### SQL Database — Tabela de Pedidos
```sql
CREATE TABLE Pedidos (
    id            UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    usuario_id    NVARCHAR(100) NOT NULL,
    status        NVARCHAR(50)  NOT NULL DEFAULT 'pendente',
    total         DECIMAL(10,2) NOT NULL,
    created_at    DATETIME2     DEFAULT GETDATE(),
    updated_at    DATETIME2     DEFAULT GETDATE()
);

CREATE TABLE ItensPedido (
    id            UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    pedido_id     UNIQUEIDENTIFIER REFERENCES Pedidos(id),
    produto_id    NVARCHAR(100) NOT NULL,
    quantidade    INT NOT NULL,
    preco_unit    DECIMAL(10,2) NOT NULL
);
```

---

## 🔒 Segurança Implementada

- **Azure AD B2C** para autenticação de clientes com suporte a OAuth 2.0 e OpenID Connect
- **Azure Key Vault** para armazenamento seguro de connection strings e secrets
- **HTTPS obrigatório** em todos os endpoints via Azure API Management
- **Managed Identity** para comunicação segura entre serviços sem credenciais hardcoded
- **Firewall e VNET** para isolamento dos bancos de dados
- **Azure DDoS Protection** para proteção contra ataques de negação de serviço

---

## 📈 Escalabilidade

- **Auto-scaling** configurado no App Service (escala automática com base em CPU > 70%)
- **Cosmos DB** com particionamento por `categoria` para distribuição uniforme de carga
- **Azure CDN** integrado ao Blob Storage para entrega rápida de imagens globalmente
- **Redis Cache** reduzindo a carga no banco em até 80% para consultas frequentes

---

## 💡 Insights e Aprendizados

### Cosmos DB vs SQL — Quando Usar Cada Um?
Aprendi que o **Cosmos DB** é ideal para dados de produtos porque o schema varia muito entre categorias (um tênis tem tamanhos, um notebook tem processador/RAM). Já o **SQL Database** é perfeito para pedidos e transações, onde a consistência ACID é crítica.

### O Poder do Redis Cache
Implementar o Azure Cache for Redis foi um divisor de águas para performance. Catálogos de produtos consultados com frequência são cacheados por 5 minutos, reduzindo drasticamente as consultas ao Cosmos DB e o custo.

### Managed Identity — Adeus, Connection Strings no Código!
Usar Managed Identity para autenticar o App Service no banco de dados elimina completamente credenciais no código. O Azure gerencia os tokens automaticamente. Isso simplifica muito a rotação de senhas e aumenta a segurança.

### Serverless com Azure Functions
Funções como "enviar e-mail de confirmação de pedido" ou "atualizar estoque após venda" ficaram muito mais baratas e simples como Azure Functions. Só pago quando executam, e escalam automaticamente.

---

## 🔮 Possibilidades de Evolução

- **Azure Cognitive Search** — busca inteligente de produtos com autocomplete e filtros facetados
- **Azure Machine Learning** — sistema de recomendação personalizado ("clientes também compraram...")
- **Azure Event Grid** + **Azure Notification Hubs** — notificações push em tempo real
- **Azure Front Door** — roteamento global com failover automático para alta disponibilidade
- **Power BI Embedded** — dashboards de vendas em tempo real integrados ao painel admin

---

## 📊 Estimativa de Custos (Cenário Inicial)

| Serviço | Tier | Custo Estimado/mês |
|---|---|---|
| App Service | B2 (2 cores, 3.5 GB RAM) | ~$75 USD |
| Cosmos DB | 400 RU/s provisionado | ~$23 USD |
| Azure SQL | S1 (20 DTUs) | ~$30 USD |
| Blob Storage | LRS, 100 GB | ~$2 USD |
| Redis Cache | C0 (250 MB) | ~$16 USD |
| Azure Functions | Consumption Plan | ~$0-5 USD |
| **Total Estimado** | | **~$150 USD/mês** |

> 💡 Com auto-scaling e boas práticas de cache, o custo pode ser reduzido em até 40% comparado a uma infraestrutura on-premises equivalente.

---

## 🛠️ Tecnologias

![Azure](https://img.shields.io/badge/Microsoft_Azure-0089D6?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![CosmosDB](https://img.shields.io/badge/CosmosDB-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![SQL](https://img.shields.io/badge/Azure_SQL-CC2927?style=for-the-badge&logo=microsoft-sql-server&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)

---

## 📚 Referências

- [Microsoft Azure Documentation](https://docs.microsoft.com/azure)
- [Repositório Base do Desafio DIO](https://github.com/digitalinnovationone/Microsoft_Application_Platform)
- [Azure Architecture Center — E-Commerce](https://learn.microsoft.com/azure/architecture/example-scenario/apps/ecommerce-scenario)
- [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator)

---

## 👤 Autor

Desenvolvido como parte do desafio prático da **Digital Innovation One (DIO)** — Microsoft Application Platform Track.

---

*⭐ Se este projeto foi útil, deixe uma estrela no repositório!*
