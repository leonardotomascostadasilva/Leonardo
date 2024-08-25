---
title: "Configurando Serviços Externos com Polly e Refit"
tags:
  - resiliência
---
Resiliência em sistemas distribuídos é um dos pilares mais importantes para garantir que uma aplicação seja capaz de lidar com falhas temporárias e outros tipos de erros de forma eficaz. No contexto de aplicações web modernas, especialmente aquelas que dependem de APIs externas, é essencial implementar políticas de resiliência que ajudem a mitigar os impactos de falhas intermitentes, garantindo assim uma experiência mais estável e confiável para os usuários.

## O que é resiliência em sistemas?
Resiliência é a capacidade de um sistema continuar operando corretamente em caso de falhas parciais. Isso pode incluir falhas de rede, indisponibilidade temporária de serviços externos, ou mesmo erros que acontecem de forma intermitente. A ideia é que o sistema consiga se recuperar ou, ao menos, degradar graciosamente para não afetar de forma significativa a experiência do usuário.

Existem várias abordagens para implementar resiliência, como Retry, Timeout, Circuit Breaker, Bulkhead Isolation, entre outras. Cada uma dessas estratégias tem um papel específico e pode ser combinada com outras para criar um mecanismo robusto contra falhas.

## Aplicando resiliência no ASP.NET Core com Polly e Refit

No ASP.NET Core, Polly é uma biblioteca popular que facilita a implementação de políticas de resiliência. Junto com Refit, uma biblioteca que simplifica o consumo de APIs RESTful, é possível criar uma configuração poderosa para lidar com falhas de comunicação.

Vamos explorar cada uma das políticas de resiliência e como elas podem ser configuradas:

### Retry
A política de Retry tenta executar a operação novamente um determinado número de vezes antes de falhar definitivamente. É ideal para falhas transitórias, como problemas temporários de rede.

Exemplo de código:
```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
```
### Timeout
Timeout define um tempo limite para a execução de uma operação. Se o tempo for excedido, a operação é cancelada.

Exemplo de código:

```csharp
var timeoutPolicy = Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(10));
```
### Circuit Breaker
Circuit Breaker impede que o sistema continue a tentar executar operações que provavelmente falharão, baseado em um histórico recente de falhas. Isso permite que o sistema evite sobrecarregar um serviço que está claramente com problemas, com isso é possível gerar uma possibilidade do sistema conseguir se recuperar.

Exemplo de código:
```csharp
var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(
        handledEventsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromMinutes(1)
    );
```
## Integrando Polly e Refit
A integração do Polly com Refit é bastante direta. Podemos aplicar as políticas que definimos ao configurar o HttpClient usado pelo Refit.

Exemplo de código:
```csharp
services.AddRefitClient<IExternalApi>()
    .ConfigureHttpClient(c => c.BaseAddress = new Uri("https://api.exemplo.com"))
    .AddPolicyHandler(timeoutPolicy)
    .AddPolicyHandler(retryPolicy)
    .AddPolicyHandler(circuitBreakerPolicy);
```

Aqui, as políticas de Retry, Timeout, e Circuit Breaker são aplicadas na ordem especificada. Isso significa que, primeiro, a política de Timeout será avaliada. Se o tempo limite for excedido, a operação falha sem tentar novamente. Em seguida, a política de Retry será tentada em caso de exceções transitórias, e, finalmente, a política de Circuit Breaker pode interromper as tentativas se houver muitas falhas consecutivas.

## Considerações Finais
Ao implementar políticas de resiliência em suas aplicações, é crucial entender o comportamento de cada uma e como elas interagem entre si. Testar essas políticas em um ambiente controlado e monitorar seu impacto é igualmente importante para garantir que elas realmente tragam benefícios ao seu sistema.