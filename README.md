---

# Ambassador/Consul Connect E2E Kubernetes Demo

===============================================

## Propósito

Este repositório contém os arquivos de configuração e scripts para configurar um cluster Kubernetes com tudo o que você precisa para demonstrar ou explorar o Consul Connect.
Especificamente, ele demonstra os seguintes recursos:

* [Helm chart][helm-blog] para implantar agentes e servidores Consul com criptografia Gossip, o injector do Consul Connect e sincronização de catálogo entre Consul e Kubernetes.

* Injeção automática de [sidecars do Consul Connect][sidecars] em pods com uma simples anotação.

* Uma instância do [serviço HTTP echo][echo] e seu cliente, para testar a funcionalidade do Connect.

* Uma instância do serviço DataWire [QoTM][qotm], também para testes.

* Instância funcional do demo de microsserviços [Weaveworks Sock Shop][sockshop], com Consul Connect mediando todas as conexões entre os serviços.

* DataWire [Ambassador][] como o gateway L7, roteando requisições da Internet para os proxies Connect com TLS mútuo completo e usando Consul para descoberta de serviços.

---

## Pré-requisitos

Você também deve ter um cluster Kubernetes em funcionamento, com o utilitário `kubectl` configurado corretamente para se comunicar com ele, e uma versão recente do [Helm][].
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

Se você preferir usar o dashboard em vez da CLI, execute
`kube-dashboard/dashboard.sh` para instalar o dashboard e iniciar um proxy local.
Você poderá acessá-lo em:

```
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```

Se o dashboard pedir login, existem scripts auxiliares `token-*.sh` que copiarão um token de autenticação para a área de transferência.

#### Servidores e clientes Consul

O script `consul/consul.sh` usa o Helm chart do Consul para implantar servidores e agentes Consul. O chart faz o seguinte:

* Implanta 3 pods de servidor Consul, cada um com 20GB de armazenamento (versão atual 1.4.2 OSS).

* Implanta pods de agente em cada nó do cluster.

* Implanta o pod `consul-k8s` para sincronização do catálogo de serviços e injeção automática do Consul Connect.

O chart pode ser customizado de várias maneiras editando `consul/values.yaml`, por exemplo:

* Se estiver testando com minikube, altere `server.replicas` e `server.bootstrapExpect` para 1.

* Você pode usar versões específicas de [Consul][consul-tags], [Envoy][envoy-tags] ou [`consul-k8s`][k8s-tags] alterando os campos `image`.

* Você pode expor a UI do Consul para fora do cluster mudando `ui.service.type` de `NodePort` para `LoadBalancer`, caso prefira não usar o port forward do `kubectl`.

Se preferir implantar binários enterprise em vez do OSS, faça o seguinte:

* Crie um secret no Kubernetes contendo sua chave de licença. Veja os comentários no topo do `values.yaml`.

* Adicione o sufixo `-ent` à tag especificada em `global.image`, por exemplo `"consul:1.4.2-ent"`.

* Configure o nome do secret em `server.enterpriseLicense.secretName` e `server.enterpriseLicense.secretKey`.

Se algo der errado com a implantação (ex.: erros de sintaxe no `values.yaml`), use o script `consul/clean.sh` para limpar tudo e tentar novamente.

#### Serviço simples HTTP echo

Alguns documentos do Consul usam um serviço HTTP "echo" e seu cliente para demonstrar vários conceitos. Eles podem ser implantados com `simple/simple.sh`.

Para testar a conexão entre cliente e servidor, veja a seção "Testes e Demos" abaixo.

#### Carregar intentions no Consul Connect

O script `consul/intentions.sh` cria um conjunto padrão de intentions para permitir que as demos funcionem:

* Ambassador pode se comunicar com qualquer serviço.

* Os serviços `carts`, `orders`, `catalogue` e `user` do Sock Shop podem acessar seus próprios bancos de dados (`carts-db`, `orders-db`, etc).

* O servidor web `front-end` do Sock Shop pode acessar os serviços `carts`, `orders`, `catalogue` e `user`.

* O cliente HTTP echo pode acessar o servidor echo.

* Todo o restante do tráfego é negado.

#### Demo Sock Shop

Execute o script `sockshop/weaveworks.sh` para implantar uma versão do Sock Shop personalizada para usar o Consul Connect. Pouquíssimas mudanças foram necessárias — isso mostra como é fácil adaptar suas próprias aplicações para o Connect!

As mudanças foram:

* No Helm chart, anotar os pods para [injeção de sidecar][cartdb] e declarar as [dependências upstream][carts].

* Para cada serviço downstream, adicionar uma [variável de ambiente][user] ou uma [opção de linha de comando][catalogue] para instruir a buscar seus upstreams em localhost.

O único serviço que precisou de mudanças de código foi o `front-end`, porque os nomes dos serviços upstream estavam hard-coded. Usei as variáveis de ambiente criadas pelo Connect, como mostrado [aqui][frontend].
Se você remover a injeção Connect, o front-end voltará ao comportamento antigo.

#### Ambassador

Por fim, implante o proxy Ambassador executando `ambassador/ambassador.sh`.
Isso instalará o próprio Ambassador e também o Consul Connector, que recupera os certificados mTLS do Consul e os fornece ao serviço principal Ambassador.

---

## Testes e Demos

#### Encontrando o endereço IP do Ambassador

Como um gateway L7, o Ambassador expõe um endereço IP público. Você precisará dele para os testes abaixo.
Use:

```
kubectl describe service ambassador
```

Procure por “LoadBalancer Ingress”. Esse é o IP público do serviço Ambassador.
Sempre que aparecer `AMBASSADOR_IP` nos exemplos, substitua por esse valor.

#### Serviço HTTP echo simples

O pod cliente HTTP echo foi injetado com um proxy para se conectar ao servidor echo.
Você pode confirmar isso inspecionando o pod e vendo as anotações:

```
consul.hashicorp.com/connect-inject=true
consul.hashicorp.com/connect-inject-status=injected
consul.hashicorp.com/connect-service=http-echo-client
consul.hashicorp.com/connect-service-upstreams=http-echo:1234
```

Você também verá um container extra chamado `consul-connect-envoy-sidecar`.
Este é o proxy que transporta as conexões para o serviço upstream.

Para testar a conexão, execute dentro do container cliente:

```
$ kubectl exec -it http-echo-client curl localhost:1234
"hello world"
```

Altere a intention de “allow” para “deny” e o `curl` para de funcionar imediatamente:

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

Você também pode acessar o serviço via Ambassador:

```
$ curl http://AMBASSADOR_IP/echo/
```

#### Serviço QoTM do Ambassador

O Ambassador fornece um serviço “Quote of the Moment”.
Você pode testar acessando:

```
http://AMBASSADOR_IP/qotm/
```

#### Demo Sock Shop

Finalmente, o grande final! Visite:

```
http://AMBASSADOR_IP/socks/
```

O fluxo de tráfego para servir a página é assim:

![traffic flow](data_flow.png)

---

## Melhorias Futuras

* [ ] Monitoramento com Prometheus
* [ ] SSL/TLS externo com Ambassador
* [ ] Bootstrapping de ACL
