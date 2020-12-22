# A técnica de desenvolvimento "Fail Fast"

---
Título: A técnica de desenvolvimento "Fail Fast"
Autor: Anderson Lucas Silva de Oliveira
---

### Introdução
No processo de desenvolvimento, uma das principais tarefas que circundam o cotidiano dos engenheiros de software é a depuração de código "debugging", a qual, geralmente, é a tarefa que mais despende de tempo. Existem diversas técnicas que visam melhorar o processo de criação e manutenção de software, uma delas se chama "Fail Fast", a qual visa reduzir o tempo que levamos para encontrar um bug.

A técnica foi originalmente descrita através de um artigo do [Martin Fowler](https://pt.wikipedia.org/wiki/Martin_Fowler) intítulado "[Fail Fast](https://www.martinfowler.com/ieeeSoftware/failFast.pdf)" ao qual esse texto se baseia. Trata-se de uma abordagem onde sua principal premissa é "Falhar rápido e de forma clara". Esse texto retrata seus conceitos forma resumida e direcionada ao nosso contexto.

### O problema

Geralmente os sistemas são feitos de forma com que os erros não sejam mostrados ao usuário em sua forma “crua”, ao invés disso, mostramos mensagens de feedback, alerta, etc. Isso é muito correto e útil quando se trata de UX, todavia, se esse acobertamento de erro também acontece “por baixo dos panos”, isto é, no momento do desenvolvimento, oque temos é um erro difícil de encontrar e, consequentemente, de resolver. Não só isso, frequentemente nos deparamos com erros que são ignorados e propagados até os mais desconhecidos fluxos, que, mais cedo ou mais tarde, será encontrado por usuário e se tornará uma pedra no sapato do desenvolvedor encarregado de resolve-lo. Basta se lembrar daquela falha que quase todo programador já teve que lidar, aonde o tempo gasto em encontrar o problema foi infinitamente maior do tempo gasto em solucioná-lo.


Vamos a um exemplo prático. Uma "api" que contenha um componente de cupom de desconto, encarregado de tratar o código do cupom, validar sua integridade e recuperar seu desconto correspondente, vejamos:

### Exemplo 1 

```php
<?php
namespace app\Controllers\Coupon\CouponController;

class CouponController
{   
    public function __construct(
        CouponHandlerInterface $couponHandler, 
        CouponRepositoryInterface $couponRepository
    )
    {
        $this->couponHandler = $couponHandler;
        $this->couponRepository = $couponRepository;
        
    }

    public function getDiscount(string $rawCode): float
    {
        if (empty($rawCode)) {
            return 0.00;
        }

        $parsedCouponCode = $this->couponHandler->parseCode($rawCode);
        
        if (!$this->couponHandler->isValidCoupon($parsedCouponCode)) {
        	return 0.00
        }

        return $this->couponRepository->getValue($parsedCouponCode);
    }
}
```

Utilizei um exemplo simples para facilitar o entendimento, porém em casos reais, sabemos que os componentes que lidamos no dia a dia nem sempre são tão simples assim. O que temos é um "controller" que faz uso de 2 interfaces; "couponHandler" e "couponRepository" através do método "getDiscount", o qual retorna o valor do desconto com base no código do mesmo.

Agora, suponhamos que tenha ocorrido uma falha onde os cupons de descontos válidos eram retornados com seu valor sendo igual a 0,00. A primeira coisa que fazemos é ir ao banco e checar se o valor do desconto está correto, e temos a constatação que está, vamos supor que o valor do cupom seja 10,00 R$.

Analisando o código, podemos observar que existem dois pontos onde o processo pode retornar 0,00:


```php
<?php 

  if (empty($rawCode)) {
      return 0.00;
  }
```

Também temos:

```php
<?php
   if (!$this->couponHandler->isValidCoupon($parsedCouponCode)) {
      return 0.00
   }
```

Ao identificar esses pontos, fazemos uma série de testes no "endpoint" a fim de encontrarmos o erro, utilizando o mesmo código de cupom, com o mesmo banco da loja da falha com todos os cenários idênticos ao do problema descrito, mas, mesmo assim, o problema parece não existir, isso é, o cupom é retornado corretamente. Esse é um caso, simplificado por motivos didáticos, é muito comum que pode aumentar significativamente o tempo de resolução dos bugs. Acredito que todos os desenvolvedores, não importa o nível, passaram ou irão passar por uma situação parecida.

### A Solução

Prosseguindo no nosso exemplo, após muitas tentativas de reprodução, descobrimos que oque ocasionava o problema era um fluxo onde o "Client Side" enviava o código para a api em branco fazendo com que entrasse nesse fluxo, retornando "0.00":  


```php
   <?php 
   if (empty($rawCode)) {
      return 0.00;
   }
```

Mas até chegarmos ao foco do problema, muitas vezes, despendemos de horas e horas de analise, logs e mais logs e um tempo que poderia trazer mais valor ao usuário.

A abordagem Fail Fast se encaixaria muito bem nessa situação. Não faz sentido retornarmos um valor para um cupom onde seu código é vazio. O que devemos retornar é, segundo a técnica, um erro informando a quem está utilizando o trecho de código (interface) que algo deu errado o mais rápido possível e de forma clara. Aqui, a mensagem tem de ser clara para o desenvolvedor, pois não se trata do feedback final ao usuário, mas sim de algo interno utilizado para tratar os erros da aplicação, vejamos como poderia ficar:

```php
<?php
   if (empty($rawCode)) {
      throw new \InvalidArgumentException('Falha ao tentar processar cupom com o código vazio', 404).
   }
```

Como podemos notar, ao invés de retornar um valor para um cupom com o código vazio, o que fazemos é falhar rápido de forma clara. O que deverá ser mostrado ao usuário e como esse erro será abordado nos fluxos subsequentes é papel de quem está fazendo uso desse techo de código.

Assim, facilitamos futuros fluxos colaterais que possam vir a acontecer, pois ao invés de retornar um valor genérico, disparamos um erro de fácil identificação que facilita a analise e resolução do problema.

Podemos deixar ainda melhor utilizando um serviço de log, vejamos:

```php
<?php
   if (empty($rawCode)) {
      NRClient::addCustomParameter('EmptyCouponProcessing', /*algo que identifique a sessão vigente*/);
      throw new \InvalidArgumentException('Falha ao tentar processar cupom com o código vazio', 404);
   }

```

Assim, caso alguma ocorrência de erro falhe, podemos simplesmente checar se naquela sessão ocorreu algum desses erros assim elaborar uma implementação com base nos dados coletados.

Podemos seguir com a mesma abordagem em outros pontos do método:

```php
<?php
    if (!$this->couponHandler->isValidCoupon($parsedCouponCode)) {
       NRClient::addCustomParameter('InvalidCouponCode', $parsedCouponCode);
       throw new \InvalidArgumentException("Falha ao tentar processar cupom inválido {$parsedCouponCode}", 404);
    }

```

Muito melhor para o "debugging" do que um simples valor genérico, não é mesmo ?

### Conclusão

Portanto, vê-se que pequenas atitudes podem ser de grande valia no processo de fazer software. Dentre tantas outras abordagens, pode-se dizer que a Fail Fast é uma das que entrega resultado de forma mais rápida e simples possível. Sem dúvidas vale a pena fazer o uso da técnica ao criar ou dar manutenção em código já existente, com certeza irá facilitar o trabalho além de tornar o produto mais “saudável”.

### Rerências

- [Fail Fast, de Martin Fowler](https://www.martinfowler.com/ieeeSoftware/failFast.pdf) 







