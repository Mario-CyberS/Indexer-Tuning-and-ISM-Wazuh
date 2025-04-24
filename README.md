# Wazuh Indexer Tuning and ISM Configuration  
This project focuses on optimizing the Wazuh Indexer for a **single-node setup** by tuning shard counts, replica settings, index lifecycle policies (ISM), and increasing heap memory usage to avoid cluster overload and log ingestion issues.

---

## 🎯 Objective  
To improve performance and stability of a Wazuh Indexer running on a single node by:
- Eliminating unnecessary replica shards
- Configuring ISM policies to manage index lifecycle
- Reducing primary shards to conserve space
- Increasing allowed shards per node
- Expanding JVM heap size for indexer performance

---

## 🔍 Why This Matters  
By default, Wazuh creates indices with multiple primary and replica shards — suitable for large, multi-node clusters. On a single node, this:
- Wastes storage
- Hits shard limits
- Causes “500 Internal Server Error” due to heap exhaustion

Tuning index templates and system settings is crucial for efficient logging, compliance retention (e.g., CJIS 1-year), and overall SIEM stability.

---

## 📚 Skills Learned  
- Index template editing (primary shards, replicas, & keyword mapping)  
- Cluster setting tuning using cURL  
- Creating & applying ISM policies in Wazuh  
- JVM heap space configuration  
- Troubleshooting Wazuh Indexer and Filebeat logs

---

## 🛠️ Tools Used  
<div>
  <img src="https://img.shields.io/badge/-Wazuh-0078D4?&style=for-the-badge&logo=Wazuh&logoColor=white" />
  <img src="https://img.shields.io/badge/-OpenSearch-005571?&style=for-the-badge&logo=OpenSearch&logoColor=white" />
</div>

---

## 📝 Configuration Steps

### 1. Set Replica Defaults to Zero  
```bash
curl -X PUT "https://localhost:9200/_cluster/settings" \
  -H "Content-Type: application/json" \
  -u "mtagaras:pass" --insecure \
  -d '{
    "persistent": {
      "opendistro.index_state_management.history.number_of_replicas": 0
    }
  }'

curl -X PUT "https://localhost:9200/_cluster/settings" \
  -H "Content-Type: application/json" \
  -u "mtagaras:pass" --insecure \
  -d '{
    "persistent": {
      "cluster.default_number_of_replicas": "0"
    }
  }'
