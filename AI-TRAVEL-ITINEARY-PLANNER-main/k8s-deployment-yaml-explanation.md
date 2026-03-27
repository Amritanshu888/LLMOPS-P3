# Kubernetes Deployment and Service Configuration

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
| 4 | `name: streamlit-app` | Sets the name of this Deployment to "streamlit-app". This name must be unique within the namespace. |
| 5 | `labels:` | Begins the labels section for organizing and selecting resources. |
| 6 | `app: streamlit` | Adds a label `app: streamlit` to the Deployment for identification and selection. |

### Deployment Spec
| Line | Code | Explanation |
|------|------|-------------|
| 7 | `spec:` | Starts the specification section that defines the desired state of the Deployment. |
| 8 | `replicas: 1` | Specifies that exactly one pod replica should be running at all times. |
| 9 | `selector:` | Defines how the Deployment identifies which pods it manages. |
| 10 | `matchLabels:` | Uses label matching to select pods. |
| 11 | `app: streamlit` | The Deployment will manage pods that have the label `app: streamlit`. |

### Pod Template
| Line | Code | Explanation |
|------|------|-------------|
| 12 | `template:` | Starts the pod template section that defines the pods to be created. |
| 13 | `metadata:` | Metadata for the pods that will be created. |
| 14 | `labels:` | Labels applied to the pods. |
| 15 | `app: streamlit` | Pods will have the label `app: streamlit`, matching the selector above. |
| 16 | `spec:` | Specification for the pod's contents. |

### Container Definition
| Line | Code | Explanation |
|------|------|-------------|
| 17 | `containers:` | Lists the containers to run in the pod. |
| 18 | `- name: streamlit-container` | Defines the first container with name "streamlit-container". The hyphen indicates this is an item in the containers list. |
| 19 | `image: streamlit-app:latest` | Specifies the container image to use. Uses the "latest" tag of "streamlit-app" image. |
| 20 | `imagePullPolicy: IfNotPresent` | Kubernetes will only pull the image if it doesn't exist locally. Skips pulling if already present on the node. |
| 21 | `ports:` | Lists ports to expose from the container. |
| 22 | `- containerPort: 8501` | Exposes port 8501 from the container. This is the default port that Streamlit applications listen on. |
| 23 | `envFrom:` | Indicates that environment variables will be injected from external sources. |
| 24 | `- secretRef:` | References a Secret resource for environment variables. |
| 25 | `name: llmops-secrets` | Specifies the name of the Secret resource to import environment variables from. |

---

## Service Section (v1)

### API Version and Kind
| Line | Code | Explanation |
|------|------|-------------|
| 27 | `apiVersion: v1` | Specifies the Kubernetes API version for this resource. Services use the core v1 API. |
| 28 | `kind: Service` | Defines this resource as a Service, which provides network access to pods. |

### Service Metadata
| Line | Code | Explanation |
|------|------|-------------|
| 29 | `metadata:` | Starts the metadata section for the Service. |
| 30 | `name: streamlit-service` | Sets the name of this Service to "streamlit-service". |

### Service Spec
| Line | Code | Explanation |
|------|------|-------------|
| 31 | `spec:` | Starts the specification section for the Service. |
| 32 | `type: LoadBalancer` | Creates a LoadBalancer type service. On cloud providers, this provisions an external load balancer with a public IP address. For local clusters like minikube, it may create a service accessible via localhost. |
| 33 | `selector:` | Defines which pods this service will route traffic to. |
| 34 | `app: streamlit` | Selects pods with the label `app: streamlit` (matching the Deployment pods). |
| 35 | `ports:` | Lists ports exposed by the service. |
| 36 | `- protocol: TCP` | Specifies TCP protocol for the port mapping. |
| 37 | `port: 80` | The port exposed by the service (external port). Clients will connect to this port. |
| 38 | `targetPort: 8501` | The port on the container to forward traffic to (the Streamlit container port defined on line 22). |