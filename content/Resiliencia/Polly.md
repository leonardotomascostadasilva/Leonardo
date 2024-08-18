---
title: "Polly"
tags:
  - resiliência
---

Polly é uma biblioteca .NET voltada para resiliência e tratamento de falhas em aplicações. Ela permite que você implemente políticas como retries (tentativas automáticas de refazer uma operação que falhou), circuit breakers (interromper temporariamente tentativas de operação após uma série de falhas), timeouts, e outras estratégias de resiliência. A ideia principal do Polly é ajudar a criar aplicações mais robustas e tolerantes a falhas, especialmente ao lidar com operações que dependem de recursos externos, como chamadas de API, acesso a banco de dados, ou serviços de terceiros.

## Principais Funcionalidades do Polly:

### Retries (Tentativas Automáticas):
- Polly permite que você configure uma política de retry para que, se uma operação falhar, ela seja tentada novamente automaticamente. Você pode definir o número de tentativas, o intervalo entre elas e até mesmo uma lógica de "backoff" (por exemplo, aumentar o intervalo entre as tentativas a cada falha).

### Circuit Breaker:

- Um circuit breaker é uma proteção que desativa temporariamente a operação depois de várias falhas consecutivas. Isso evita sobrecarregar um serviço que já está instável, dando tempo para ele se recuperar antes de permitir novas tentativas.

### Timeouts:

- Com Polly, você pode definir um tempo limite para uma operação. Se a operação demorar mais do que o tempo especificado, ela será interrompida, evitando que sua aplicação fique presa esperando indefinidamente.

### Bulkhead Isolation (Isolamento de Compartimentos):

- Essa estratégia limita o número de operações simultâneas que podem ser executadas, prevenindo que uma falha em massa em uma operação afete outras partes da aplicação.

### Fallbacks:

- Permite definir uma ação alternativa a ser tomada quando todas as tentativas de uma operação falharem, como retornar um valor padrão ou usar um serviço alternativo.

## Exemplo de Uso
Imagine que você está consumindo uma API externa que pode estar sujeita a instabilidades. Usar Polly com uma política de retry pode ajudar sua aplicação a lidar com essas instabilidades, tentando a operação novamente em caso de falhas temporárias:

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .RetryAsync(3, onRetry: (exception, retryCount) =>
    {
        Console.WriteLine($"Tentativa {retryCount} falhou: {exception.Message}");
    });

var response = await retryPolicy.ExecuteAsync(() =>
    httpClient.GetAsync("https://api.exemplo.com/dados"));

```

Neste exemplo, se a chamada à API falhar por causa de uma HttpRequestException, Polly tentará novamente até três vezes antes de desistir.

## Por que usar Polly?
Polly é uma ferramenta poderosa para garantir que sua aplicação seja resiliente a falhas temporárias, especialmente em ambientes distribuídos, onde há muitas dependências externas. Usando Polly, você pode melhorar a disponibilidade e a confiabilidade de suas aplicações, garantindo que elas continuem funcionando mesmo diante de problemas temporários em serviços dependentes.