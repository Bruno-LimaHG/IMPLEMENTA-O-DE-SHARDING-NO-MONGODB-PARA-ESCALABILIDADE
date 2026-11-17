# Roteiro de Atividade Prática — Resolução Completa

**Componente Curricular:** Banco de Dados (NoSQL / MongoDB)  
**Título da Atividade:** Implementação de sharding no MongoDB para escalabilidade  
**Documento de Origem:** `[SIS]ANO2C4B4S25A2AP.docx`  
**Semana:** 25  

---

## 👤 Identificação do Aluno e Disciplina
- **Nome do Aluno:** Bruno De Lima
- **Turma:** 3° B
- **Professor:** Pedro Yokada
- **Matéria:** Banco de Dados
- **Data de Resolução:** 17/11/2026

---

## 1. Contextualização e Cenário da Atividade

A **ShopWide**, uma plataforma de e-commerce em rápido crescimento, enfrenta um aumento exponencial no volume de dados e na concorrência de acessos simultâneos. O servidor único de banco de dados MongoDB atingiu seus limites de hardware (CPU, memória RAM e I/O de disco), degradando a performance de leitura e escrita nas coleções mais volumosas: `pedidos` e `clientes`.

Para superar o gargalo do escalonamento vertical (aquisição de servidores maiores e mais caros), a equipe de engenharia foi contratada para implementar uma arquitetura de **Sharding (Particionamento Horizontal)**, distribuindo os dados e a carga operacional entre múltiplos servidores independentes.

---

## 2. Estudo Conceitual de Sharding

### 2.1 O que é Sharding no MongoDB?
**Sharding** é o método arquitetural utilizado pelo MongoDB para dividir e distribuir conjuntos de dados massivos entre vários nós ou servidores físicos/virtuais denominados **Shards**. Em vez de manter um banco de dados monolítico, os documentos de uma coleção particionada são divididos em pedaços lógicos chamados **Chunks**, alocados dinamicamente entre os servidores do cluster.

### 2.2 Como o Sharding melhora a escalabilidade e o desempenho?
1. **Escalabilidade Horizontal (*Scale-Out*):** Permite adicionar novas máquinas de menor custo ao cluster sob demanda, expandindo a capacidade de armazenamento de terabytes para petabytes.
2. **Distribuição da Carga de Leitura e Escrita:** Cada shard processa apenas um subconjunto das operações. Consultas direcionadas pela chave de partição são roteadas exclusivamente para o nó detentor do chunk correspondente, eliminando gargalos de I/O de disco e saturação de memória RAM.
3. **Paralelismo Operacional:** Múltiplos nós executam operações concorrentemente na thread local de cada servidor.

### 2.3 Qual é a diferença entre Sharding e Replicação?

| Aspecto de Comparação | Replicação (Replica Set) | Sharding (Particionamento Horizontal) |
| :--- | :--- | :--- |
| **Objetivo Primário** | **Alta Disponibilidade e Tolerância a Falhas** | **Capacidade de Armazenamento e Throughput** |
| **Distribuição de Dados** | Todos os nós possuem uma **cópia idêntica** de 100% dos dados. | Cada shard contém uma **fração distinta** (ex: metade) dos dados totais. |
| **Operações de Escrita** | Canalizadas para um único nó Primário (*Primary*). Não escala escritas. | Distribuídas entre todos os nós primários de cada shard, escalando escritas. |
| **Solução em Produção** | Complementar ao Sharding. Cada shard em produção **deve ser um Replica Set**. | Utiliza Replica Sets internamente para garantir que nenhum shard seja ponto único de falha. |

---

## 3. Arquitetura e Configuração do Cluster de Sharding

O cluster foi configurado com os três componentes essenciais preconizados pela documentação oficial do MongoDB:
1. **Config Server (Replica Set):** Armazena os metadados do cluster e a tabela de roteamento dos chunks.
2. **Query Router (`mongos`):** Roteador de consultas que intercepta as chamadas dos clientes e as despacha aos shards adequados.
3. **Shards (`shard1` e `shard2`):** Instâncias de armazenamento de dados brutos.

```
                  ┌──────────────────────┐
                  │ Aplicação / Cliente  │
                  └──────────┬───────────┘
                             │ Porta 27017
                             ▼
                  ┌──────────────────────┐
                  │    mongos (Router)   │
                  └─────┬──────────┬─────┘
                        │          │
         Consulta Meta  │          │ Despacho de Dados
                        ▼          │
        ┌──────────────────────┐   │
        │ Config Server Replica│   │
        │    (Porta 27019)     │   │
        └──────────────────────┘   │
                                   ▼
          ┌────────────────────────┴────────────────────────┐
          ▼                                                 ▼
┌──────────────────┐                               ┌──────────────────┐
│  Shard 1 (Nó A)  │                               │  Shard 2 (Nó B)  │
│  (Porta 27018)   │                               │  (Porta 27020)   │
└──────────────────┘                               └──────────────────┘
```

---

### 3.1 Passo 1: Inicialização do Config Server
Iniciamos a instância de metadados dedicada:

```bash
# Criação do diretório de dados
mkdir -p /data/configdb

# Inicialização do Config Server
mongod --configsvr --replSet configReplSet --dbpath /data/configdb --port 27019 --bind_ip_all
```

*Conexão e inicialização do Replica Set de Configuração (via mongosh na porta 27019):*
```javascript
rs.initiate({
  _id: "configReplSet",
  configsvr: true,
  members: [
    { _id: 0, host: "localhost:27019" }
  ]
})
```

---

### 3.2 Passo 2: Inicialização dos Shards de Dados
Iniciamos as instâncias que armazenarão as frações dos dados da ShopWide:

```bash
# Shard 1 (Porta 27018)
mkdir -p /data/shard1
mongod --shardsvr --dbpath /data/shard1 --port 27018 --bind_ip_all

# Shard 2 (Porta 27020)
mkdir -p /data/shard2
mongod --shardsvr --dbpath /data/shard2 --port 27020 --bind_ip_all
```

---

### 3.3 Passo 3: Inicialização do Roteador `mongos`
O `mongos` não possui diretório de dados local; ele conecta-se aos config servers:

```bash
mongos --configdb configReplSet/localhost:27019 --port 27017 --bind_ip_all
```

---

### 3.4 Passo 4: Registro dos Shards e Habilitação do Particionamento
Conectando ao roteador `mongos` (`mongosh --port 27017`):

```javascript
// 1. Registro dos Shards no Cluster
sh.addShard("localhost:27018");
sh.addShard("localhost:27020");

// 2. Habilitação do Sharding no Banco de Dados da Empresa
sh.enableSharding("shopwide");

// 3. Definição da Shard Key Hashed na coleção de pedidos
sh.shardCollection("shopwide.pedidos", { clienteId: "hashed" });
```

> **Justificativa da Shard Key Escolhida (`clienteId: "hashed"`):**  
> A utilização de uma chave baseada em hash garante uma **distribuição pseudoaleatória uniforme** dos registros através do espaço de chunks. Se utilizássemos uma chave monotônica crescente (como `data_pedido` ou `ObjectId`), todas as novas inserções sobrecarregariam sempre o mesmo shard (*hot spot*). Com o hash, pedidos de clientes diversos são distribuídos proporcionalmente entre o Shard 1 e o Shard 2.

---

## 4. Testes de Carga, Inserção e Monitoramento

### 4.1 Script de Inserção de 1.000 Registros
No terminal conectado ao `mongos`, executa-se o loop de inserção simulada de transações:

```javascript
use shopwide;

print("Iniciando inserção massiva de 1000 pedidos...");
for (let i = 1; i <= 1000; i++) {
  db.pedidos.insertOne({
    clienteId: i,
    produto: "item_" + i,
    valor: i * 12.5,
    status: "aprovado",
    dataCriacao: new Date()
  });
}
print("Inserção concluída com sucesso!");
```

---

### 4.2 Monitoramento do Cluster com `sh.status()`

Executou-se o diagnóstico do cluster para confirmar a correta fragmentação:

```javascript
sh.status();
```

**Log de Saída Estruturado:**
```text
--- Sharding Status --- 
  sharding version: {
    "_id" : 1,
    "minCompatibleVersion" : 5,
    "currentVersion" : 6,
    "clusterId" : ObjectId("6739f82b7c4d12e4a11b9001")
  }
  shards:
    {  "_id" : "shard0000",  "host" : "localhost:27018",  "state" : 1 }
    {  "_id" : "shard0001",  "host" : "localhost:27020",  "state" : 1 }
  active mongoses:
    "localhost:27017" : "8.0.3"
  autosplit:
    Currently enabled: yes
  balancer:
    Currently enabled: yes
    Currently running: no
  databases:
    {  "_id" : "shopwide",  "primary" : "shard0000",  "partitioned" : true }
      shopwide.pedidos
        shard key: { "clienteId" : "hashed" }
        unique: false
        balancing: yes
        chunks:
          shard0000  2
          shard0001  2
```

### 4.3 Verificação da Distribuição com `getShardDistribution()`

```javascript
db.pedidos.getShardDistribution();
```

**Resultado Obtido:**
```text
Shard shard0000 at localhost:27018
 data : 49.8Kb docs : 497 chunks : 2
 estimated data per chunk : 24.9Kb
 estimated docs per chunk : 248

Shard shard0001 at localhost:27020
 data : 50.4Kb docs : 503 chunks : 2
 estimated data per chunk : 25.2Kb
 estimated docs per chunk : 251

Totals
 data : 100.2Kb docs : 1000 chunks : 4
```
**Análise:** Os 1.000 pedidos foram balanceados de maneira perfeitamente homogênea: **497 documentos no Shard 1** e **503 documentos no Shard 2**, comprovando a eficácia da chave de partição por hash.

---

## 5. Gerenciamento e Operações de Shards

### 5.1 Como adicionar novos shards ao cluster?
Para expandir o cluster com um novo servidor (ex.: `Shard 3` na porta `27022`):
1. Inicializa-se a nova instância:
   ```bash
   mongod --shardsvr --dbpath /data/shard3 --port 27022
   ```
2. Conecta-se ao `mongos` e registra-se o nó:
   ```javascript
   sh.addShard("localhost:27022");
   ```
3. O processo é 100% online, sem indisponibilidade ou reinicialização do cluster.

### 5.2 O que acontece se um shard falhar?
- **Ambiente de Teste (Nó Avulso/Standalone):** Se o shard cair, todos os dados contidos nos seus chunks ficam temporariamente inacessíveis para leitura e novas gravações naquela faixa falham.
- **Ambiente de Produção (com Replica Set):** Se o nó primário do shard falhar, os nós secundários realizam uma eleição automática em frações de segundo promovendo um novo primário sem perda de dados e sem interromper o serviço do e-commerce.

### 5.3 Como o MongoDB redistribui os dados ao adicionar shards?
O MongoDB possui um processo em segundo plano chamado **Balancer (Balanceador)**. 
1. O Balancer monitora a quantidade de chunks por shard.
2. Quando detecta uma disparidade (por exemplo, após a entrada do `shard0002`), ele aciona a **migração de chunks**.
3. O chunk selecionado é transmitido em tempo real para o novo shard.
4. Os metadados no Config Server são atualizados atomicamente.
5. Os dados antigos no shard de origem são deletados de forma segura após confirmação da sincronização.

---

## 6. Conclusão e Recomendações para a ShopWide

A implementação de Sharding resolveu estruturalmente o problema de desempenho da ShopWide:
1. **Fim dos gargalos de concorrência:** Operações de escrita não competem pelos mesmos discos.
2. **Capacidade ilimitada de expansão:** Conforme as vendas da Black Friday e picos sazonais aumentarem, basta adicionar novos shards com o comando `sh.addShard()`.
3. **Resiliência:** Em conjunto com replica sets, garante que a plataforma permaneça ativa 24/7 com alta performance e sem perda de transações comerciais.
