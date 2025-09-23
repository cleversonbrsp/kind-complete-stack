## Laboratório Kubernetes local com Kind + Ingress + MetalLB

Este projeto sobe um cluster Kubernetes local com 3 nós usando Kind, instala o NGINX Ingress Controller e configura o MetalLB para simular serviços `LoadBalancer`. Inclui um exemplo de aplicação e acesso local via `/etc/hosts`. Também é possível usar um serviço de domínio externo para expor publicamente, se desejar (não detalhado aqui).

- **Kind** (Kubernetes in Docker): `https://kind.sigs.k8s.io/`

### Por que usar este setup
- **Realista**: mesmos componentes usados em produção (Ingress, LoadBalancer).
- **Rápido e reprodutível**: `make rebuild` cria tudo do zero em minutos.
- **Sem custo de cloud**: ótimo para desenvolvimento, POCs e estudos.
- **Pronto para CI**: fácil criar clusters efêmeros para testes E2E.

### Componentes incluídos
- 3 nodes (1 control-plane, 2 workers)
- NGINX Ingress Controller (classe padrão `nginx`)
- MetalLB com IP pool para Services `LoadBalancer`
- Manifests de demo (`hello-ingress.yaml`)

---

## Requisitos
- Docker e permissões para executar containers
- `kind`, `kubectl`, `helm`

---

## Estrutura do repositório

```text
kind-complete-stack/
├── Makefile
├── kind-cluster.yaml
├── deploy-nginx-ingress.sh
├── deploy-metallb.sh
├── hello-ingress.yaml
├── metrics-server.yaml
└── README.md
```

---

## Como subir o cluster

```bash
make rebuild
```

O comando executa, nesta ordem: destruir cluster anterior, criar novo (`kind-cluster.yaml`), instalar Ingress NGINX e MetalLB.

Se ocorrer erro de porta 80/443 ocupada, veja "Troubleshooting > Portas 80/443 ocupadas" abaixo.

---

## Comandos úteis (Makefile)

- `make up`: cria o cluster a partir de `kind-cluster.yaml`
- `make ingress`: instala o NGINX Ingress Controller
- `make metallb`: instala e configura o MetalLB (IPAddressPool + L2Advertisement)
- `make destroy`: remove o cluster
- `make rebuild`: recria tudo do zero

Observação: os alvos `demo` e `hosts` não estão ativos no `Makefile`. Você pode aplicar a demo manualmente com `kubectl apply -f hello-ingress.yaml` e gerenciar o `/etc/hosts` conforme orientações abaixo.

---

## Formas de acessar sua aplicação

### Desenvolvimento local via `/etc/hosts` (com MetalLB)
1. Suba o cluster: `make rebuild`
2. Instale a demo (opcional):
   ```bash
   kubectl apply -f hello-ingress.yaml
   ```
3. Descubra o IP do LoadBalancer do Ingress NGINX:
   ```bash
   kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
   ```
4. Adicione o host no `/etc/hosts` (substitua `$IP`):
   ```bash
   echo "$IP hello.local" | sudo tee -a /etc/hosts
   ```
5. Acesse: `http://hello.local`

> Dica: Você pode alterar o `host` no `hello-ingress.yaml` para outro domínio local, ex.: `app.local`.

### Uso de um serviço de domínio (opcional)
Você pode utilizar um serviço de domínio externo (por exemplo, um provedor de DNS) para apontar um nome como `www.seu-dominio.com` para este cluster. Essa configuração depende do seu ambiente e provedor e não é detalhada aqui.

---

## Troubleshooting

### Portas 80/443 ocupadas ao criar o cluster
Sintoma (Kind/Docker):

```
failed to bind host port for 0.0.0.0:80 ... address already in use
```

Causa: o `kind-cluster.yaml` mapeia as portas 80 e 443 do host para o node de controle. Se o host já tem `nginx`, `apache` ou outro processo na 80/443, a criação falha.

Opções de correção:
- Parar o serviço do host e recriar o cluster:
  ```bash
  sudo systemctl stop nginx apache2 httpd traefik caddy haproxy 2>/dev/null || true
  make rebuild
  ```
- Alterar as portas do host (ex.: 8080/8443):
  ```yaml
  # kind-cluster.yaml
  nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 8080
      - containerPort: 443
        hostPort: 8443
  ```
  Acesse via `http://localhost:8080`.
- Remover os `extraPortMappings` e usar apenas o IP do MetalLB (recomendado para este stack). Veja a seção 
  "Desenvolvimento local via `/etc/hosts`" para mapear o host para o IP do Ingress.

Como identificar quem está usando a porta:
```bash
sudo lsof -nP -iTCP:80 -sTCP:LISTEN
sudo lsof -nP -iTCP:443 -sTCP:LISTEN
```

### MetalLB IP Pool
O script `deploy-metallb.sh` cria um `IPAddressPool` e um `L2Advertisement` com um range padrão. Ajuste o range conforme a sub-rede da rede Docker `kind`:
```bash
docker network inspect kind | grep Subnet
```
Edite o range no script se necessário.

### Demo e domínios customizados
Altere o host no `hello-ingress.yaml` para o domínio desejado (ex.: `www.devops.lab.com.br`). É possível integrar um serviço de domínio externo para acesso público, se necessário.

---

## Limpeza
```bash
make destroy
```

---

## Notas
- O Kind não suporta webhooks admission do ingress-nginx por padrão; o script de instalação desativa os webhooks de admission.
- Para habilitar métricas e HPA, você pode aplicar `metrics-server.yaml` (ajuste se necessário para o seu ambiente).

