# Kibana Kubernetes Deployment and Service Configuration

## Deployment Section (apps/v1)

### API Version and Kind
| Line | Code | Explanation |
|------|------|-------------|
| 1 | `apiVersion: apps/v1` | Specifies the Kubernetes API version for this resource. `apps/v1` is the stable API version for Deployment objects. |
| 2 | `kind: Deployment` | Defines this resource as a Deployment, which manages a set of identical pods with rollout and scaling capabilities. |

### Deployment Metadata
| Line | Code | Explanation |
|------|------|-------------|
| 3 | `metadata:` | Starts the metadata section for the Deployment. |
| 4 | `name: kibana` | Sets the name of this Deployment to "kibana". This name must be unique within the namespace. |
| 5 | `namespace: logging` | Places this Deployment in the "logging" namespace. This allows for logical isolation of logging-related resources from other applications. |

### Deployment Spec
| Line | Code | Explanation |
|------|------|-------------|
| 6 | `spec:` | Starts the specification section that defines the desired state of the Deployment. |
| 7 | `replicas: 1` | Specifies that exactly one pod replica should be running at all times. Kibana typically runs as a single instance, though multiple replicas could be configured for high availability. |
| 8 | `selector:` | Defines how the Deployment identifies which pods it manages. |
| 9 | `matchLabels:` | Uses label matching to select pods. |
| 10 | `app: kibana` | The Deployment will manage pods that have the label `app: kibana`. |

### Pod Template
| Line | Code | Explanation |
|------|------|-------------|
| 11 | `template:` | Starts the pod template section that defines the pods to be created. |
| 12 | `metadata:` | Metadata for the pods that will be created. |
| 13 | `labels:` | Labels applied to the pods. |
| 14 | `app: kibana` | Pods will have the label `app: kibana`, matching the selector above. |
| 15 | `spec:` | Specification for the pod's contents. |

### Container Definition
| Line | Code | Explanation |
|------|------|-------------|
| 16 | `containers:` | Lists the containers to run in the pod. |
| 17 | `- name: kibana` | Defines the first container with name "kibana". The hyphen indicates this is an item in the containers list. |
| 18 | `image: docker.elastic.co/kibana/kibana:7.17.0` | Specifies the official Elastic Kibana image from Elastic's Docker registry. Version 7.17.0 is specified, which is a long-term support (LTS) version. |
| 19 | `env:` | Starts the environment variables section. |
| 20 | `- name: ELASTICSEARCH_HOSTS` | Defines an environment variable named `ELASTICSEARCH_HOSTS`. This tells Kibana where to connect to Elasticsearch. |
| 21 | `value: http://elasticsearch:9200` | Sets the value to `http://elasticsearch:9200`. This uses Kubernetes DNS resolution to connect to a service named "elasticsearch" on port 9200 within the same namespace. |
| 22 | `ports:` | Lists ports to expose from the container. |
| 23 | `- containerPort: 5601` | Exposes port 5601 from the container. This is the default port that Kibana's web interface listens on. |

---

## Service Section (v1)

### API Version and Kind
| Line | Code | Explanation |
|------|------|-------------|
| 25 | `apiVersion: v1` | Specifies the Kubernetes API version for this resource. Services use the core v1 API. |
| 26 | `kind: Service` | Defines this resource as a Service, which provides network access to the Kibana pod(s). |

### Service Metadata
| Line | Code | Explanation |
|------|------|-------------|
| 27 | `metadata:` | Starts the metadata section for the Service. |
| 28 | `name: kibana` | Sets the name of this Service to "kibana". This matches the Deployment name and will be used for DNS resolution within the cluster. |
| 29 | `namespace: logging` | Places this Service in the "logging" namespace, matching the Deployment's namespace. |

### Service Spec
| Line | Code | Explanation |
|------|------|-------------|
| 30 | `spec:` | Starts the specification section for the Service. |
| 31 | `type: NodePort` | Creates a NodePort type service. This exposes the service on a static port (30000-32767) on each node in the cluster, allowing external access through `http://<node-ip>:30601`. This is useful for development, on-premise deployments, or when a cloud load balancer isn't desired. |
| 32 | `selector:` | Defines which pods this service will route traffic to. |
| 33 | `app: kibana` | Selects pods with the label `app: kibana` (matching the Deployment pods). |
| 34 | `ports:` | Lists ports exposed by the service. |
| 35 | `- port: 5601` | The port exposed by the service internally (cluster IP port). Other services in the cluster can access Kibana at `kibana.logging.svc.cluster.local:5601`. |
| 36 | `nodePort: 30601` | The static port exposed on each node's IP address. External users can access Kibana using `http://<any-node-ip>:30601`. This port is within the default NodePort range (30000-32767). |