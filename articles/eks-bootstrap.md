## Despliege y aprovisionamiento de EKS con Terraform y ArgoCD

Cuando desplegamos un clusters de Kubernetes, en este caso en AWS seguramente tendremos que añadir otros componentes adicionales para sacarle todo el provecho y si bien en cierto EKS nos permite a través de [addons](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html) cubrir una buena listas de funcionalidades, hay muchas de ellas que deben manejarse por separado, como [external-dns](https://github.com/kubernetes-sigs/external-dns) o [external-secrets](https://external-secrets.io/latest/). En este tutorial voy a mostrarte como desplegar este cluster usando Terraform y automáticamente aprovisionar los elementos adicionales utilizando ArgoCD.

### Lo que vamos a necesitar

Antes de nada estos son los requisitos que debes tener antes de seguir este tutorial.

- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [aws-cli(https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [terraform](https://developer.hashicorp.com/terraform/install)
- [Cuenta de AWS](https://repost.aws/es/knowledge-center/create-and-activate-aws-account)

Una vez asegurado todos los requisitos, recuerda configurar las credenciales del `aws-cli`. 

### Terraform 

Para crear los recursos de infraestructura vamos a utilizar `Terraform` que aunque ultimadamente se haya hecho noticia de forma no muy positiva, sigue siendo la mejor herramientas de infraestructura como código según mi opinión. Lo primero será agregar los proveedores. 

```hcl
  terraform {
    required_providers {
      aws = {
        source  = "hashicorp/aws"
        version = "5.47.0"
      }
      kustomization = {
        source  = "kbst/kustomization"
        version = "0.9.5"
      }
      kubernetes = {
        source  = "hashicorp/kubernetes"
        version = "2.29.0"
      }
      null = {
        source  = "hashicorp/null"
        version = "3.2.2"
      }
      local = {
        source  = "hashicorp/local"
        version = "2.5.1"
      }
    }
  }
```

Más adelante vamos a configurar cara proveedor pero por ahora vamos apoyarnos en [modulos](https://developer.hashicorp.com/terraform/language/modules) para crear tanto la red como nuestro cluster EKS de forma que no tengamos que ir uno a uno creando los recursos necesarios.

En el caso de la VPC.

```hcl 
/* datasource */

data "aws_caller_identity" "current" {}
data "aws_availability_zones" "current" {}

/* network settings */
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "5.8.1"
    
  name = "${local.prefix}-vpc"
  cidr = local.network.cidr

  azs             = local.network.az
  private_subnets = [for i in range(length(local.network.az)) : cidrsubnet(local.network.cidr, 8, i)]
  public_subnets  = [for i in range(length(local.network.az)) : cidrsubnet(local.network.cidr, 8, i + 3)]

  # single NAT per network
  enable_nat_gateway     = true
  single_nat_gateway     = true
  one_nat_gateway_per_az = true

  tags = local.default_tags
}

```
Aunque esto no se enfoca en la parte de red, puedes notar el uso de la función `cidrsubnet` que me ayuda a calcular las subnets de forma sequencial y automática.

y luego nuestro cluster.

```hcl
/* EKS Cluster */
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "${local.prefix}-eks"
  cluster_version = "1.29"

  cluster_endpoint_public_access = true

  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent = true
    }
  }

  vpc_id                   = module.vpc.vpc_id
  subnet_ids               = module.vpc.private_subnets
  control_plane_subnet_ids = module.vpc.private_subnets


  eks_managed_node_groups = {
    demo = {
      min_size     = 1
      max_size     = 2
      desired_size = 2

      instance_types = ["t3.medium"]
      capacity_type  = "ON_DEMAND"
    }
  }

  # Cluster access entry
  # To add the current caller identity as an administrator
  enable_cluster_creator_admin_permissions = true


  tags = local.default_tags
}

```
Con el uso de estos dos modulos somos capaces de crear tanto nuestra red y nuestro cluster pero antes no olvidamos configurar los proveedores.

```hcl
/* Provider Configuration */
resource "null_resource" "kubectl" {
  provisioner "local-exec" {
    command = "aws eks --region ${var.region} update-kubeconfig --name ${module.eks.cluster_name} --profile ${var.profile} --kubeconfig $(pwd)/.kube/config"
  }
  depends_on = [module.eks]
}

# Generate Kubeconfig
data "local_file" "kubeconfig" {
  filename   = ".kube/config"
  depends_on = [null_resource.kubectl]
}

provider "kustomization" {
  #kubeconfig_path = data.local_file.kubeconfig.filename
  kubeconfig_raw = data.local_file.kubeconfig.content
}

# This data source is necessary to configure the Kubernetes provider
data "aws_eks_cluster" "cluster" {
  name       = module.eks.cluster_name
  depends_on = [module.eks]
}

data "aws_eks_cluster_auth" "cluster" {
  name       = module.eks.cluster_name
  depends_on = [module.eks]
}

# In case of not creating the cluster, this will be an incompletely configured, unused provider, which poses no problem.
provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
  token                  = data.aws_eks_cluster_auth.cluster.token
}

provider "aws" {
  region  = var.region
  profile = var.profile
}

```
No te preocupes por el orden, puedes colocar todo este código en un mismo archivo `main.tf`. A diferencia de lenguajes como Python, Terraform intentará ejecutar esto en una forma ordenada, es decir, primero la red antes que crear el cluster, sin embargo, nosotros podemos controlar o indicar las dependencias de un recursos con otro, a través del uso de `depends_on`. Si te fijas existen `null_resource`, `local_file` y `datasource` que dependen de que se ejecute el modulo de EKS; los proveedores no aceptan agregar dependencia por tanto hacemos uso de los recursos antes mencionados para evitar que los proveedores intenten conectarse antes que nuestro cluster este listo.

Ya tenemos lo principal pero también necesitamos generar IAM roles que permitan a componentes como `external-secrets` conectarse con el servicios de [AWS Secret Manager](https://aws.amazon.com/secrets-manager/). Te dejo un ejemplo de como se hace esto con Terraform.

```hcl
# External Secrets
data "aws_iam_policy_document" "external_secrets_trust_policy" {
  statement {
    sid     = "externalsecret"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    effect  = "Allow"

    principals {
      identifiers = ["arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/${replace(module.eks.cluster_oidc_issuer_url, "https://", "")}"]
      type        = "Federated"
    }

    condition {
      test     = "StringEquals"
      values   = ["system:serviceaccount:external-secrets:external-secrets"]
      variable = "${replace(module.eks.cluster_oidc_issuer_url, "https://", "")}:sub"
    }
  }
}

resource "aws_iam_role" "external_secrets" {
  name               = "${local.prefix}-external-secrets-role"
  assume_role_policy = data.aws_iam_policy_document.external_secrets_trust_policy.json
}

data "aws_iam_policy_document" "external_secrets_access" {
  statement {
    effect = "Allow"
    actions = [
      "secretsmanager:GetResourcePolicy",
      "secretsmanager:GetSecretValue",
      "secretsmanager:DescribeSecret",
      "secretsmanager:ListSecretVersionIds"
    ]
    resources = [
      "arn:aws:secretsmanager:*:${data.aws_caller_identity.current.account_id}:secret:*"
    ]
  }
}

resource "aws_iam_policy" "external_secrets" {
  name   = "${local.prefix}-es-access"
  policy = data.aws_iam_policy_document.external_secrets_access.json
}

resource "aws_iam_policy_attachment" "external_secrets" {
  name       = "${local.prefix}-es-attachment"
  roles      = [aws_iam_role.external_secrets.name]
  policy_arn = aws_iam_policy.external_secrets.arn
}

```
Lo más importante este en la `Trust Policy` esta es la que permite que una service account en nuestro cluster puede acceder secret manager desde nuestro cluster. La politica posee una condición en la que `"system:serviceaccount:external-secrets:external-secrets"` solo una service account en el namespace `external-secrets` y con nombre igual puede usar asumir este IAM Role, por tanto cuando instalemos este componente luego debemos asegurarnos que coincida.

### Instalación ArgoCD 

Sino sabes que es ArgoCD, es una herramienta de Continuous Delivery para kubernetes que sigue el patrón GitOps, en donde un repositorio git es la fuente de la verdad; puedes leer más [acá](https://argo-cd.readthedocs.io/en/stable/). Para instalarlo, lo haremos también con Terraform, siguiendo las recomendaciones de la documentación.

```hcl
#argocd namespace
resource "kubernetes_namespace_v1" "argocd" {
  metadata {
    name = "argocd"
  }

  depends_on = [module.eks, null_resource.kubectl]
}

data "kustomization_build" "argo" {
  path = "bootstrap/argocd"
}

resource "kustomization_resource" "argocd" {
  for_each = data.kustomization_build.argo.ids

  manifest = data.kustomization_build.argo.manifests[each.value]


  depends_on = [module.eks, null_resource.kubectl, kubernetes_namespace_v1.argocd]
}
```
Como puedes ver arriba, lo primero será crear el namespace donde será instalado y luego con [kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) aplicaremos la instalación, que como puedes ver en el recurso hay una ruta hacia `bootstrap/argocd`. El archivo  `kustomization.yaml` se ve como así:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: argocd

namespace: argocd

resources:
  - https://raw.githubusercontent.com/argoproj/argo-cd/v2.10.9/manifests/install.yaml
  - base/github-repos.yaml
  - base/infra.yaml
```
En la sección de `resources` hay una url donde estan todos los manifiestos que instalan los distintos componentes de argoCD; no te preocupes por el resto, lo veremos más adelante.

### Componentes del Cluster via GitOps 

Aunque no hemos hecho ningún `terraform apply` en código ya tenemos tanto la red, nuestro cluster EKS y la instalación de ArgoCD. Ahora ya podemos instalar los componentes de nuestro cluster. Lo primero será indicarle a ArgoCD que repositorio de Git vamos a utilizar para instalar nuestros componentes:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: eks-bootstrap
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/davejfranco/eks-bootstrap

```
Lo hacemos mediante la creación de un `secret` pero no cualquier secret, la etiqueta `argocd.argoproj.io/secret-type: repository` le dice a Argo que esto es un repositorio. Le indicamos que es de tipo:`git` y la url, que por cierto es donde podrás ver todo junto como se ve. Finalmente indicamos como instalar los componentes de nuestro cluster.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: infra
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc
  project: default

  source:
    path: bootstrap/core
    repoURL: https://github.com/davejfranco/eks-bootstrap
    targetRevision: main

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```
Definimos una aplicación argo a la que le decimos que en el repositorio; que debe ser creado con anterioridad, dentro del directorio `bootstrap/core` busque definiciones de aplicaciones. Ejemplo tenemos dentro del directorio `bootstrap/core/external-secrets` esta definición de aplicación.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets
  namespace: argocd
spec:
  project: default
  source:
    chart: external-secrets
    repoURL: https://charts.external-secrets.io
    targetRevision: 0.9.17
    helm:
      releaseName: external-secrets
      valuesObject:
        serviceAccount:
          create: true
          annotations:
            eks.amazonaws.com/role-arn: arn:aws:iam::444106639146:role/argo-external-secrets-role
  destination:
    server: "https://kubernetes.default.svc"
    namespace: external-secrets
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```
Cuando Terraform luego de crear todos los componentes de infraestructura y finalmente pueda aplicar el recurso de kustomize, entonces creará todos los componentes que se encuentren definidos como aplicaciones dentro de esto directorio `bootstrap/core` siempre y cuando el código este en el branch `main`.

### Es la hora de aplicar 

Antes que nada hacemos `terraform init` para descargar todos los proveedores que tenemos definidos en el proyecto. Finalmante `terraform apply`


