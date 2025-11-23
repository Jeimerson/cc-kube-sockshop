---

# Ambassador/Consul Connect E2E Kubernetes Demo

=============================================

## Propósito

Este repositório contém os arquivos de configuração e scripts para configurar um cluster Kubernetes com tudo o que você precisa para demonstrar ou explorar o Consul Connect.
Especificamente, ele demonstra os seguintes recursos:

* [Helm chart][helm-blog] para implantar agentes e servidores Consul com criptografia Gossip, o injector do Consul Connect e sincronização de catálogo entre Consul e Kubernetes.

* Injeção automática de [sidecars do Consul Connect][sidecars] em pods com uma simples anotação.

* Uma instância do [serviço HTTP echo][echo] e seu cliente, para testar a funcionalidade do Connect.

* Uma instância do serviço [QoTM da DataWire][qotm], também para testes.

* Instância funcional do demo de microsserviços [Weaveworks Sock Shop][sockshop], com Consul Connect mediando todas as conexões entre os serviços.

* [Ambassador][] da DataWire como gateway L7, roteando requisições da Internet para os proxies Connect com mTLS completo e usando Consul para descoberta de serviços.

---

## Pré-requisitos

Você também deve ter um cluster Kubernetes já em execução, com o utilitário `kubectl` configurado corretamente para falar com ele, e uma versão recente do [Helm][].
Em um Mac com Homebrew, basta digitar:

```
brew install kubernetes-helm kubernetes-cli
```

Crie um secret no Kubernetes com o conteúdo da sua licença do Consul Enterprise:

```
kubectl create secret generic consul-ent-license --from-file=bef1b5c5-4290-a854-a34b-af1651d5d41b.hclic
```

---

## Configuração

#### Helm e Tiller

Execute `tiller/helm-init.sh` para criar uma service account e instalar o serviço Tiller.

#### Kubernetes Dashboard (opcional)

Se preferir usar o dashboard em vez da CLI, você pode executar
`kube-dashboard/dashboard.sh` para instalar o dashboard e iniciar um proxy local.
Você pode então acessar o dashboard em:

```
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```

Se o dashboard pedir login, existem scripts auxiliares em `token-*.sh` que copiarão um token de autenticação para a área de transferência.

#### Servidores e clientes Consul

O script `consul/consul.sh` usa o Helm chart do Consul para implantar servidores e agentes Consul. O Helm chart faz o seguinte:

* Implanta 3 pods de servidor Consul, cada um com 20 GB de armazenamento (atualmente versão 1.4.2 OSS).

* Implanta pods de agente em cada host do cluster.

* Implanta o pod `consul-k8s` para sincronização do catálogo de serviços e injeção automática do Consul Connect.

O chart pode ser customizado editando `consul/values.yaml`, por exemplo:

* Se estiver testando com minikube, altere `server.replicas` e `server.bootstrapExpect` para 1.

* Você pode usar versões específicas de [Consul][consul-tags], [Envoy][envoy-tags] ou [`consul-k8s`][k8s-tags] alterando os campos `image`.

* Você pode expor a UI do Consul para fora do cluster mudando `ui.service.type` de `NodePort` para `LoadBalancer`, caso não queira usar o port forward do `kubectl`.

Se preferir implantar binários Enterprise em vez de OSS, faça o seguinte:

* Crie um secret no Kubernetes contendo sua chave de licença. Veja os comentários no início de `values.yaml`.

* Adicione o sufixo `-ent` na tag especificada em `global.image`, por exemplo `"consul:1.4.2-ent"`.

* Configure o nome do seu secret em `server.enterpriseLicense.secretName` e `server.enterpriseLicense.secretKey`. Use as entradas comentadas em `values.yaml` como guia.

Se algo der errado na implantação (ex.: erro de sintaxe no `values.yaml`), você pode usar o script `consul/clean.sh` para remover tudo e tentar novamente.

#### Serviço HTTP echo simples

Alguns documentos do Consul utilizam um serviço HTTP "echo" e seu cliente para demonstrar vários conceitos. Eles podem ser implantados com `simple/simple.sh`.

Para testar a conexão entre cliente e servidor, veja a seção “Testes e Demos”.

#### Carregar intentions no Consul Connect

O script `consul/intentions.sh` cria um conjunto padrão de intentions para permitir que as demos funcionem:

* Ambassador pode se comunicar com qualquer serviço.

* Os serviços `carts`, `orders`, `catalogue` e `user` do Sock Shop podem acessar seus próprios bancos de dados (`carts-db`, `orders-db`, etc.).

* O servidor web `front-end` do Sock Shop pode acessar os serviços `carts`, `orders`, `catalogue` e `user`.

* O cliente HTTP echo pode acessar o servidor echo.

* Todo o restante do tráfego é negado.

#### Demo Sock Shop

Execute o script `sockshop/weaveworks.sh` para implantar uma versão do Sock Shop personalizada para usar Consul Connect. Pouquíssimas mudanças foram necessárias — isso mostra como é fácil adaptar suas próprias aplicações ao Connect!

As mudanças feitas foram:

* No Helm chart, anotar os pods para a [injeção de sidecar][cartdb] e declarar as [dependências upstream][carts].

* Para cada serviço downstream, adicionar uma [variável de ambiente][user] ou uma [opção de linha de comando][catalogue] para instruir o serviço a procurar seus upstreams em localhost.

O único serviço que exigiu mudanças de código foi o `front-end`, pois os nomes dos serviços upstream estavam hard-coded. Aproveitei as variáveis de ambiente criadas pelo Connect, como mostrado [aqui][frontend].
Se você remover a injeção do Connect, o front-end voltará ao comportamento antigo.

#### Ambassador

Finalmente, implante o proxy Ambassador executando `ambassador/ambassador.sh`.
Isso instalará o próprio Ambassador e também o Consul Connector, que busca os certificados mTLS do Consul e os fornece ao serviço principal Ambassador.

---

## Testes e Demos

#### Descobrindo o endereço IP do Ambassador

Como gateway L7, o Ambassador expõe um endereço IP público. Você precisará dele para executar os testes abaixo.
Use o comando:

```
kubectl describe service ambassador
```

Procure por "LoadBalancer Ingress". Esse é o endereço IP público do seu serviço Ambassador.
Sempre que aparecer `AMBASSADOR_IP` nos exemplos, substitua por esse endereço.

#### Serviço HTTP echo simples

O pod cliente HTTP echo foi injetado com um proxy para se conectar ao servidor echo. Você pode verificar isso inspecionando o pod e observando as anotações:

```
consul.hashicorp.com/connect-inject=true
consul.hashicorp.com/connect-inject-status=injected
consul.hashicorp.com/connect-service=http-echo-client
consul.hashicorp.com/connect-service-upstreams=http-echo:1234
```

Você também verá que o pod contém um container extra chamado
`consul-connect-envoy-sidecar`. Esse é o proxy que transporta as conexões para o serviço upstream.

Para verificar a conexão, execute:

```
$ kubectl exec -it http-echo-client curl localhost:1234
"hello world"
```

Experimente alterar a intention de "allow" para "deny": o comando `curl` para imediatamente:

```
$ consul intention create -replace -deny '*' http-echo
$ kubectl exec -it http-echo-client curl localhost:1234
curl: (52) Empty reply from server
command terminated with exit code 52
```

E você pode permitir novamente com:

```
$ consul intention create -replace -allow '*' http-echo
```

Você também pode chamar o serviço via Ambassador:

```
$ curl http://AMBASSADOR_IP/echo/
```

#### Serviço QoTM do Ambassador

O Ambassador fornece um serviço “Quote of the Moment”.
Você pode testá-lo abrindo:

```
http://AMBASSADOR_IP/qotm/
```

#### Demo Sock Shop

Finalmente, a grande demonstração! Visite:

```
http://AMBASSADOR_IP/socks/
```

O fluxo de tráfego para servir a página é o seguinte:

![traffic flow](data_flow.png)

---

## Melhorias futuras

* [ ] Monitoramento com Prometheus
* [ ] SSL/TLS externo com Ambassador
* [ ] Bootstrapping de ACL

[sidecars]: https://www.consul.io/docs/platform/k8s/connect.html
[sockshop]: https://microservices-demo.github.io/
[helm]: https://helm.sh/
[helm-blog]: https://kubernetes.io/blog/2016/10/helm-charts-making-it-simple-to-package-and-deploy-apps-on-kubernetes/
[ambassador]: https://www.getambassador.io/
[connector]: https://www.getambassador.io/user-guide/consul-connect-ambassador/
[qotm]: https://github.com/datawire/qotm
[echo]: https://github.com/hashicorp/http-echo
[proxy]: http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/overview?namespace=default
[consul-tags]: https://hub.docker.com/_/consul?tab=tags
[k8s-tags]: https://hub.docker.com/r/hashicorp/consul-k8s/tags
[envoy-tags]: https://hub.docker.com/r/envoyproxy/envoy-alpine/tags
[me]: mailto:todd@hashicorp.com
[frontend]: https://github.com/tradel/front-end/blob/9c32e77828993c4571ac2219843a999e6e4e12cf/api/endpoints.js#L18-L35
[cartdb]: https://github.com/tradel/microservices-demo/blob/2bc270d61c993f8a1ae3c8a492cae504b7c3ade5/deploy/kubernetes/helm-chart/templates/cart-db-dep.yaml#L14-L15
[carts]: https://github.com/tradel/microservices-demo/blob/2bc270d61c993f8a1ae3c8a492cae504b7c3ade5/deploy/kubernetes/helm-chart/templates/carts-dep.yaml#L14-L16
[catalogue]: https://github.com/tradel/microservices-demo/blob/2bc270d61c993f8a1ae3c8a492cae504b7c3ade5/deploy/kubernetes/helm-chart/templates/catalogue-dep.yaml#L24
[user]: https://github.com/tradel/microservices-demo/blob/2bc270d61c993f8a1ae3c8a492cae504b7c3ade5/deploy/kubernetes/helm-chart/templates/user-dep.yaml#L31-L32

---
