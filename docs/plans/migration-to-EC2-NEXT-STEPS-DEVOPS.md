# Itens pendentes para o DevOps da edgebr

O cluster EC2 (`ubuntu@13.58.99.235`) já está com a infraestrutura base instalada e rodando.
TLS e Grafana Ingress já estão configurados (`https://grafana.k8s-ee.edge.net.br`).
Para finalizar a configuração, precisamos dos itens abaixo.

---

## Prioridade alta

### ~~1. GitHub App para ARC (self-hosted runners)~~ ✅ Recebido e configurado

---

### ~~2. DNS wildcard~~ ✅ Recebido e configurado

### ~~3. Credenciais AWS para TLS~~ ✅ Recebido e configurado

---

### ~~4. Fork do repositório~~ ✅ Recebido e configurado

Fork criado e secret `KUBECONFIG` configurado.

---

## Prioridade média

### ~~5. OAuth App para Grafana~~ ✅ Recebido e configurado

---

### ~~6. Token de acesso para limpeza automática~~ ✅ Recebido e configurado

---

## Resumo

| # | Item | O que entregar | Status |
|---|------|----------------|--------|
| 1 | ~~GitHub App (ARC)~~ | ~~App ID, Installation ID, `.pem`~~ | ✅ |
| 2 | ~~DNS wildcard~~ | ~~`*.k8s-ee.edge.net.br` → Elastic IP~~ | ✅ |
| 3 | ~~Credenciais AWS (TLS)~~ | ~~IAM Access Key + Hosted Zone ID + email ACME~~ | ✅ |
| 4 | ~~Fork do repositório~~ | ~~`edgebr/k8s-ephemeral-environments`~~ | ✅ |
| 5 | ~~OAuth App (Grafana)~~ | ~~Client ID, Client Secret~~ | ✅ |
| 6 | ~~Token de limpeza~~ | ~~PAT com leitura de PRs~~ | ✅ |
