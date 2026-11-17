# SEMANA 25 — Banco de Dados (NoSQL / MongoDB)

**Docente:** Prof. Pedro Yokada  
**Estudante:** Bruno De Lima  
**Turma:** 3° B  
**Componente:** Banco de Dados  

---

## 📋 Atividade da Semana
- **Documento Oficial:** `[SIS]ANO2C4B4S25A2AP.docx`
- **Tema:** *Implementação de sharding no MongoDB para escalabilidade*
- **Arquivo de Resolução:** [`[SIS]ANO2C4B4S25A2AP_Relatorio_Sharding_MongoDB.md`](file:///C:/Users/BrunoDeLima/Desktop/IA/BD/SEMANA%2025/%5BSIS%5DANO2C4B4S25A2AP_Relatorio_Sharding_MongoDB.md)
- **Arquivo de Configuração Docker:** [`docker-compose-sharding.yml`](file:///C:/Users/BrunoDeLima/Desktop/IA/BD/SEMANA%2025/docker-compose-sharding.yml)

---

## 📌 Conteúdos Principais Desenvolvidos
1. **Fundamentos de Sharding:** Diferenciação entre particionamento horizontal (*scale-out*) e replicação (*high availability*).
2. **Cluster Multi-Instância:** Arquitetura com `mongos` (router), Config Server (metadados) e Shards (armazenamento fragmentado).
3. **Estratégia de Shard Key:** Escolha da chave baseada em hash (`clienteId: "hashed"`) para balanceamento simétrico de chunks.
4. **Monitoramento e Operações:** Validação via `sh.status()` e `db.pedidos.getShardDistribution()`, com explicação detalhada sobre falha de nós e ação do Balancer.
