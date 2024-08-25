---
title: "Configurando Serviços Externos com Polly e Refit"
tags:
  - resiliência
---
No desenvolvimento de APIs, a resiliência é um aspecto crucial para garantir que sua aplicação continue a funcionar de maneira confiável, mesmo diante de falhas temporárias em serviços externos. Uma maneira eficaz de implementar resiliência é utilizando a biblioteca [[Resiliencia/Polly|Polly]], que fornece políticas de resiliência como retries, circuit breakers, timeouts, entre outras. Neste post, vou te mostrar como configurar e utilizar o [[Resiliencia/Polly|Polly]] em uma API ASP.NET Core, com foco em retries e timeouts, garantindo que sua aplicação seja robusta e preparada para lidar com falhas.

## Criando a Estrutura de Configuração
Para centralizar as configurações necessárias para a resiliência de serviços externos, criamos a classe `ExternalServices`. Esta classe encapsula as informações de URL, Token, e as configurações específicas de resiliência, permitindo uma configuração flexível e reutilizável:

```csharp
public class ExternalServices
{
    public string Url { get; set; } = string.Empty;

    public string Token { get; set; } = string.Empty;

    public Resilience Resilience { get; set; } = default!;
}

public class Resilience
{
    public int TimeoutInSeconds { get; set; }

    public int NumberOfRetrys { get; set; }

    public int HandledEventsAllowedBeforeBreaking { get; set; }

    public int DurationOfBreakInMinutes { get; set; }
}
```

## Configurando o Serviço Externo com Refit e Polly
Vamos agora integrar essa configuração em uma aplicação ASP.NET Core. Usaremos Refit para consumir o serviço externo e Polly para adicionar políticas de resiliência. O código a seguir ilustra essa configuração:

```csharp
public static class ExternalServicesExtensions
{
    public static IServiceCollection AddExternalService<TRefit>(this IServiceCollection services, string externalService, IConfiguration configuration) 
        where TRefit : class
    {
        var externalServices = new ExternalServices();
        configuration.GetSection($"ExternalServices:{externalService}").Bind(externalServices);

        services
            .AddRefitClient<TRefit>()
            .ConfigureHttpClient(c => c.BaseAddress = new Uri(externalServices.Url))
            .AddPolicyHandler(ResilienceExtensions.AddCircuitBreakerPolicy(externalServices.Resilience.HandledEventsAllowedBeforeBreaking, externalServices.Resilience.DurationOfBreakInMinutes, externalServices.Url))
            .AddPolicyHandler(ResilienceExtensions.AddRetryPolicy(externalServices.Resilience.NumberOfRetrys, externalServices.Url))
            .AddPolicyHandler(ResilienceExtensions.AddTimeoutPolicy(externalServices.Resilience.TimeoutInSeconds));

        return services;
    }
}
```

Aqui, o método `AddExternalService<TRefit>` é uma extensão que facilita a configuração de serviços externos. Ele permite definir políticas de resiliência específicas para cada serviço, simplificando a reutilização de código e a manutenção das configurações.

## Configurando Políticas de Resiliência
Agora, vamos detalhar como essas políticas de resiliência são configuradas utilizando Polly. Elas garantem que sua aplicação possa lidar melhor com falhas transitórias e problemas na comunicação com serviços externos.

```csharp
public static class ResilienceExtensions
{
    public static AsyncRetryPolicy<HttpResponseMessage> AddRetryPolicy(int numberOfRetrys, string url)
        => HttpPolicyExtensions
            .HandleTransientHttpError()
            .OrInner<TimeoutRejectedException>()
            .WaitAndRetryAsync(numberOfRetrys,
                retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                onRetry: (response, delay, retryCount, context) =>
                {
                    Log.Information("[Retry Policy] {RetryCount} - ({Url}) - {Message} - {PolicyKey}", retryCount, url, response.Exception?.Message, context.PolicyKey);
                });

    public static AsyncTimeoutPolicy<HttpResponseMessage> AddTimeoutPolicy(int timeoutInSeconds)
        => Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(timeoutInSeconds));

    public static AsyncCircuitBreakerPolicy<HttpResponseMessage> AddCircuitBreakerPolicy(int handledEventsAllowedBeforeBreaking, int durationOfBreakInMinutes, string url)
        => HttpPolicyExtensions
            .HandleTransientHttpError()
            .OrInner<TimeoutRejectedException>()
            .CircuitBreakerAsync(handledEventsAllowedBeforeBreaking, TimeSpan.FromMinutes(durationOfBreakInMinutes),
                onBreak: (exception, duration) =>
                {
                    Log.Information("[Circuit Breaker Policy] ({Url}) On break ->  Duration:{Duration} - Exception: {Exception}", url, duration.TotalMinutes, exception?.Result?.RequestMessage);
                },
                onReset: () =>
                {
                    Log.Information("[Circuit Breaker Policy] Circuit breaker reset. ({Url})", url);
                },
                onHalfOpen: () =>
                {
                    Log.Information("[Circuit Breaker Policy] Circuit breaker is in half-open state. ({Url})", url);
                });
}
```
Neste código, estamos configurando três políticas principais:

- **Retry Policy**: Implementa uma política de retry com backoff exponencial, registrando as tentativas de retry nos logs. Isso é crucial para lidar com falhas transitórias de rede ou de serviço.
- **Timeout Policy**: Define um tempo limite para a operação HTTP, evitando que as requisições fiquem pendentes indefinidamente. Essa política garante que o tempo de resposta seja previsível.
- **Circuit Breaker Policy**: Configura um circuito de proteção que abre após um número específico de falhas consecutivas. Isso evita que o serviço seja sobrecarregado por tentativas repetidas e falhas em cascata.

Essas políticas, quando combinadas, aumentam significativamente a robustez e a resiliência da sua aplicação, tornando-a mais preparada para lidar com instabilidades e falhas transitórias ao consumir serviços externos.

Se você quiser saber mais sobre o tema de resiliência, confira o tópico [[PoCs/PoC-Resilience]].