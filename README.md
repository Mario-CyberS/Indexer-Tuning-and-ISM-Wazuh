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
  <img src="https://img.shields.io/badge/-Wazuh_Indexer-0078D4?&style=for-the-badge&logo=Wazuh&logoColor=white" />
  <img src="https://img.shields.io/badge/-OpenSearch-005571?&style=for-the-badge&logo=OpenSearch&logoColor=white" />
</div>

---

## 📝 Configuration Steps

### 1. Set Replica Defaults to Zero
These will ensure you don't run into replica shard issues. Single node systems can't handle replica shards because there isn't another node to assign the replica to, so the replica would become "unassigned" causing an error.
```bash
curl -X PUT "https://localhost:9200/_cluster/settings" \
  -H "Content-Type: application/json" \
  -u "WazuhUser:pass" --insecure \
  -d '{
    "persistent": {
      "opendistro.index_state_management.history.number_of_replicas": 0
    }
  }'
```
```bash
curl -X PUT "https://localhost:9200/_cluster/settings" \
  -H "Content-Type: application/json" \
  -u "WazuhUser:pass" --insecure \
  -d '{
    "persistent": {
      "cluster.default_number_of_replicas": "0"
    }
  }'
```
Confirm with:
```bash
curl -k -u "WazuhUser:pass" -X GET "https://localhost:9200/_cluster/settings?include_defaults=true&pretty" | grep replica
```
Now restart the Wazuh components:
```bash
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-dashboard
```

### 2. Create wazuh-alerts-* Index Template (via WebUI)
Access the Wazuh WebUI and navigate to ☰ → Indexer management → Index Management → Templates.
Click “Create template” for wazuh-alerts-* and add the following configurations:
- Name: wazuh-alerts-changes
- Template Type: Indexes
- Priority: 1
- Simple template
- Number of primary shards: 1
- Number of replicas: 0
Also under “Index mapping”, click on JSON Editor and add the following and save:
```bash
{
  "dynamic_templates": [
    {
      "strings_as_keywords": {
        "match_mapping_type": "string",
        "mapping": {
          "type": "keyword",
          "ignore_above": 256
        }
      }
    }
  ]
}
```
This will make sure that the future indices of wazuh-alerts-* are created with only 1 primary shard (space saver). This will also make sure that all the fields in the mappings of our indices here are using keyword and not text, this will stop future “illegal_argument_exception” errors. We will also want to make a replica template for wazuh-archives-* as well, since these 2 indices generate with 3 primary shards by default too; but no need to add the "Index mapping" section.

### 3. Increase Shard Limit
We only have 1 node running Wazuh and by default each node is allowed 1000, this is not enough for us so we will increase it to 1800. This would be overkill for a single person set up, but most enterprises would need roughly 1800 free shards yearly; depending on the number of agents and log ingestion on the system. 
Up our max shards from 1000 to 1800:
```bash
curl -k -u "WazuhUser:pass" -X PUT "https://localhost:9200/_cluster/settings" \
     -H "Content-Type: application/json" \
     -d '{
       "persistent": {
         "cluster.max_shards_per_node": 1800
       }
     }'
```
Confirm it worked:
```bash
curl -k -u "WazuhUser:pass" -X GET "https://localhost:9200/_cluster/settings?include_defaults=true&pretty" | grep max_shards
```

### 4.Create the ISM Policy (via WebUI)
Navigate to ☰ → Indexer management → State management policies → Create Policy → JSON Editor
click on “Create policy”, then we want to choose JSON editor and add our JSON below:
```bash
{
    "policy": {
        "policy_id": "Wazuh-hot-cold-workflow-ISM",
        "description": "Wazuh ISM policy: 32 days hot, cold on day 33, delete on day 366. All indices use 0 replica shards.",
        "schema_version": 21,
        "error_notification": null,
        "default_state": "hot",
        "states": [
            {
                "name": "hot",
                "actions": [
                    {
                        "retry": {
                            "count": 3,
                            "backoff": "exponential",
                            "delay": "1m"
                        },
                        "replica_count": {
                            "number_of_replicas": 0
                        }
                    }
                ],
                "transitions": [
                    {
                        "state_name": "cold",
                        "conditions": {
                            "min_index_age": "32d"
                        }
                    }
                ]
            },
            {
                "name": "cold",
                "actions": [
                    {
                        "retry": {
                            "count": 3,
                            "backoff": "exponential",
                            "delay": "1m"
                        },
                        "read_only": {}
                    }
                ],
                "transitions": [
                    {
                        "state_name": "delete",
                        "conditions": {
                            "min_index_age": "366d"
                        }
                    }
                ]
            },
            {
                "name": "delete",
                "actions": [
                    {
                        "retry": {
                            "count": 3,
                            "backoff": "exponential",
                            "delay": "1m"
                        },
                        "delete": {}
                    }
                ],
                "transitions": []
            }
        ],
        "ism_template": [
            {
                "index_patterns": [
                    "wazuh-statistics-*",
                    "wazuh-monitoring-*",
                    "wazuh-alerts-*",
                    "wazuh-archives-*"
                ],
                "priority": 1
            }
        ]
    }
}
```
Now that our new ISM policy is made, we want to go ahead and actually apply this new policy to all our old indices (if doing this day of wazuh install then there should be only 3 main indices) this way they roll out with our ISM workflow, otherwise they wont.
First try this command with 1 single index to make sure it works:
```bash
curl -X POST "https://localhost:9200/_plugins/_ism/add/<YOUR-INDEX'S-FULL-NAME-HERE>" \
  -H 'Content-Type: application/json' \
  -u "WazuhUser:pass" --insecure \
  -d '{
    "policy_id": "Wazuh-hot-cold-workflow-ISM"
  }'
```
You can monitor their initializing progress in the “Policy managed indexes” tab in Index Management.

#### Bonus Commands
You can check your Indexer health status with this command via CLI:
```bash
sudo curl -x GET “https://127.0.0.1:9200/_cluster/health?pretty” -k \
--cert /etc/wazuh-indexer/certs/admin.pem \
--key /etc/wazuh-indexer/certs/admin-key.pem
```
You can also run these to check the indexer/filebeat logs for future errors:
```bash
sudo cat /var/log/wazuh-indexer/wazuh-cluster.log | grep -i -E “error|warn” 
```
```bash
sudo cat /var/log/wazuh-indexer/wazuh-cluster.log | grep -i -E “error|warn” 
```

### Wazuh Indexer Heap Space Increase
“500 internal server” errors can be caused by overusing the configured heap space of your Wazuh Indexer.
Run this to see logs if these errors are found on Wazuh Dashboard:
```bash
sudo curl -k -u WazuhUser:pass https://127.0.0.1:9200/_cat/nodes?v
```
Increase the heap space, in the Wazuh box CLI open.
Run this to see the jvm options file and configure it:
```bash
sudo nano /etc/wazuh-indexer/jvm.options
```
You will see something like -Xms3848m (which means 3.8 gb but could be different). These 2 lines are your heap max and min, change them to -Xms8g. We upgraded our RAM to 12 gb but we only want to allow heap space for part of that space, which is why we do 8 g:
```bash
from this:
-Xms3848m
-Xmx3848m
to this:
-Xms8g
-Xmx8g
```
Save and exit, then restart Wazuh Indexer:
```bash
sudo systemctl restart wazuh-indexer
```
Confirm the New Heap Size Is Active:
```bash
ps -ef | grep wazuh-indexer | grep -Eo 'Xmx[0-9]+[mg]'
```
You should see: Xmx8g



















