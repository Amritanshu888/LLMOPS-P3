# Logstash Kubernetes Configuration - ConfigMap, Deployment, and Service

## ConfigMap Section (v1)

### API Version and Kind
| Line | Code | Explanation |
|------|------|-------------|
| 1 | `apiVersion: v1` | Specifies the Kubernetes API version for this resource. ConfigMaps use the core v1 API. |
| 2 | `kind: ConfigMap` | Defines this resource as a ConfigMap, which stores non-sensitive configuration data as key-value pairs that can be mounted into pods. |

### ConfigMap Metadata
| Line | Code | Explanation |
|------|------|-------------|
| 3 | `metadata:` | Starts the metadata section for the ConfigMap. |
| 4 | `name: logstash-config` | Sets the name of this ConfigMap to "logstash-config". This will be referenced by the Deployment. |
| 5 | `namespace: logging` | Places this ConfigMap in the "logging" namespace, keeping it with other logging-related resources. |

### ConfigMap Data
| Line | Code | Explanation |
|------|------|-------------|
| 6 | `data:` | Starts the data section containing the actual configuration content. |
| 7 | `logstash.conf: \|` | Defines a key named "logstash.conf". The pipe symbol (`\|`) indicates a multi-line string literal that preserves line breaks. |
| 8 | `    input {` | Begins the Logstash input configuration section. |
| 9 | `      beats {` | Configures the Beats input plugin. This receives data from Beats agents (Filebeat, Metricbeat, etc.). |
| 10 | `        port => 5044` | Sets the listening port to 5044, which is the default port for Beats to send data to Logstash. |
| 11 | `      }` | Closes the beats input configuration. |
| 12 | `    }` | Closes the input section. |
| 13 | `    ` | Empty line for readability (ignored in configuration). |
| 14 | `    filter {` | Begins the filter section. Currently empty but can be used for data processing, transformation, and enrichment. |
| 15 | `      ` | Empty filter section with no processing rules. |
| 16 | `    }` | Closes the filter section. |
| 17 | `    ` | Empty line for readability. |
| 18 | `    output {` | Begins the Logstash output configuration section. |
| 19 | `      elasticsearch {` | Configures Elasticsearch as the output destination. |
| 20 | `        hosts => ["http://elasticsearch.logging.svc.cluster.local:9200"]` | Specifies the Elasticsearch endpoint. Uses Kubernetes DNS name format: `service.namespace.svc.cluster.local` pointing to the elasticsearch service in the logging namespace on port 9200. |
| 21 | `        index => "filebeat-%{+YYYY.MM.dd}"` | Defines the index naming pattern. Creates daily indices like `filebeat-2026.03.27`. The `%{+YYYY.MM.dd}` is a Logstash date format pattern. |
| 22 | `      }` | Closes the elasticsearch output configuration. |
| 23 | `    }` | Closes the output section. |

---

## Deployment Section (apps/v1)

### API Version and Kind
| Line | Code | Explanation |
|------|------|-------------|
| 25 | `apiVersion: apps/v1` | Specifies the Kubernetes API version for this resource. `apps/v1` is the stable API version for Deployment objects. |
| 26 | `kind: Deployment` | Defines this resource as a Deployment, which manages Logstash pods with rollout and scaling capabilities. |

### Deployment Metadata
| Line | Code | Explanation |
|------|------|-------------|
| 27 | `metadata:` | Starts the metadata section for the Deployment. |
| 28 | `name: logstash` | Sets the name of this Deployment to "logstash". |
| 29 | `namespace: logging` | Places this Deployment in the "logging" namespace, matching the ConfigMap. |

### Deployment Spec
| Line | Code | Explanation |
|------|------|-------------|
| 30 | `spec:` | Starts the specification section that defines the desired state of the Deployment. |
| 31 | `replicas: 1` | Specifies that exactly one Logstash pod replica should be running. Logstash can be scaled horizontally for higher throughput. |
| 32 | `selector:` | Defines how the Deployment identifies which pods it manages. |
| 33 | `matchLabels:` | Uses label matching to select pods. |
| 34 | `app: logstash` | The Deployment will manage pods that have the label `app: logstash`. |

### Pod Template
| Line | Code | Explanation |
|------|------|-------------|
| 35 | `template:` | Starts the pod template section that defines the pods to be created. |
| 36 | `metadata:` | Metadata for the pods that will be created. |
| 37 | `labels:` | Labels applied to the pods. |
| 38 | `app: logstash` | Pods will have the label `app: logstash`, matching the selector above. |
| 39 | `spec:` | Specification for the pod's contents. |

### Container Definition
| Line | Code | Explanation |
|------|------|-------------|
| 40 | `containers:` | Lists the containers to run in the pod. |
| 41 | `- name: logstash` | Defines the first container with name "logstash". The hyphen indicates this is an item in the containers list. |
| 42 | `image: docker.elastic.co/logstash/logstash:7.17.0` | Specifies the official Elastic Logstash image from Elastic's Docker registry. Version 7.17.0 is used for compatibility with Kibana and Elasticsearch in the previous configurations. |
| 43 | `ports:` | Lists ports to expose from the container. |
| 44 | `- containerPort: 5044` | Exposes port 5044, used by Beats agents to send data to Logstash. |
| 45 | `- containerPort: 9600` | Exposes port 9600, which is Logstash's monitoring API endpoint. This provides metrics and health check information. |
| 46 | `volumeMounts:` | Starts the volume mounts section for mounting configuration files into the container. |
| 47 | `- name: config-volume` | References a volume named "config-volume" (defined in the volumes section). |
| 48 | `mountPath: /usr/share/logstash/pipeline/logstash.conf` | Mounts the file at this path in the container. This is the default location where Logstash expects its pipeline configuration. |
| 49 | `subPath: logstash.conf` | Mounts only the specific file from the volume, not the entire volume. This prevents overwriting other files in the directory. |

### Volume Definition
| Line | Code | Explanation |
|------|------|-------------|
| 50 | `volumes:` | Starts the volumes section for defining storage volumes available to the pod. |
| 51 | `- name: config-volume` | Defines a volume named "config-volume" that will be referenced in volumeMounts. |
| 52 | `configMap:` | Specifies that this volume is backed by a ConfigMap. |
| 53 | `name: logstash-config` | References the ConfigMap named "logstash-config" defined at the beginning of this file. |
| 54 | `items:` | Allows selecting specific keys from the ConfigMap to mount. |
| 55 | `- key: logstash.conf` | Selects the key "logstash.conf" from the ConfigMap. |
| 56 | `path: logstash.conf` | Maps the selected key to a filename "logstash.conf" in the volume. |

---

## Service Section (v1)

### API Version and Kind
| Line | Code | Explanation |
|------|------|-------------|
| 58 | `apiVersion: v1` | Specifies the Kubernetes API version for this resource. Services use the core v1 API. |
| 59 | `kind: Service` | Defines this resource as a Service, which provides network access to the Logstash pod(s). |

### Service Metadata
| Line | Code | Explanation |
|------|------|-------------|
| 60 | `metadata:` | Starts the metadata section for the Service. |
| 61 | `name: logstash` | Sets the name of this Service to "logstash". This will be used for DNS resolution within the cluster. |
| 62 | `namespace: logging` | Places this Service in the "logging" namespace, matching the Deployment and ConfigMap. |

### Service Spec
| Line | Code | Explanation |
|------|------|-------------|
| 63 | `spec:` | Starts the specification section for the Service. |
| 64 | `selector:` | Defines which pods this service will route traffic to. |
| 65 | `app: logstash` | Selects pods with the label `app: logstash` (matching the Deployment pods). |
| 66 | `ports:` | Lists ports exposed by the service. |
| 67 | `- protocol: TCP` | Specifies TCP protocol for the port mapping. |
| 68 | `port: 5044` | The port exposed by the service internally (cluster IP port). Other services can access Logstash at `logstash.logging.svc.cluster.local:5044`. |
| 69 | `targetPort: 5044` | The port on the container to forward traffic to (matches the container port defined on line 44). |