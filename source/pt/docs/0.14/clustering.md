title: Cluster
---
O framework Moleculer suporta várias arquiteturas de software.

## Arquitetura Monolítica
Nesta versão, todos os serviços estão rodando no mesmo nó como um monolítico. Não há latência de rede e não há módulo de transporte. _As chamadas locais são as mais rápidas._

![Arquitetura Monolítica](assets/architectures/monolith.svg)

## Arquitetura de microsserviços
Esta é a bem conhecida arquitetura dos microsserviços quando todos os serviços funcionam em nós individuais e se comunicam através do módulo de transporte. Neste caso, a latência de rede não é insignificante. No entanto, os seus serviços podem ser escalados de forma resiliente e com tolerância a falhas.

![Arquitetura de microsserviços](assets/architectures/microservices.svg)

## Arquitetura mista
Neste caso, estamos executando serviços relacionados agrupados no mesmo nó. Isto combina as vantagens das arquiteturas monolíticas e de microsserviços. Por exemplo, se o serviço `posts` chama muitas vezes o serviço `users`, poderiam ser colocados no mesmo nó, para que a latência de rede entre estes serviços seja reduzida. Se o nó estiver sobrecarregado, basta escalar.

![Arquitetura mista](assets/architectures/mixed.svg)

{% note info Tip %}
O ServiceBroker primeiro tenta chamar as instâncias locais do serviço (se existir) para reduzir as latências de rede. Esta lógica pode ser desativada em [opções do broker](configuration.html#Broker-options) com a propriedade `preferLocal: false` abaixo da chave `registry`.
{% endnote %}

## Como escolher
Você escolhe entre monolítico e microsserviços por causa da fase de desenvolvimento de sua aplicação? Você escolhe uma abordagem monolítica porque é fácil de desenvolver? Você prefere arquitetura de microsserviços porque ela é confiável e altamente escalável? Com Moleculer você não precisa escolher. Podemos ter o melhor de ambas as abordagens. Durante o desenvolvimento de seu aplicativo carregar todos os serviços em um único nó. Dessa forma você pode rapidamente debugar e testar a lógica da sua aplicação. Quando estiver pronto, basta distribuir seus serviços por vários nós. Não se preocupe, você não precisa alterar uma única linha de código dos seus serviços. Basta selecionar um módulo de transporte: carregar um serviço por nó e está feito. Sua aplicação está rodando em arquitetura de microsserviços.