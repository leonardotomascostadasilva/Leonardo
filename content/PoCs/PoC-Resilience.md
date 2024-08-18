---
title: "Resiliência em APIs com Polly no ASP.NET Core: Uma Abordagem Prática"
tags:
  - PoCs
---


No desenvolvimento de APIs, a resiliência é um aspecto crucial para garantir que sua aplicação continue a funcionar de maneira confiável, mesmo diante de falhas temporárias em serviços externos. Uma maneira eficaz de implementar resiliência é utilizando a biblioteca [[Resiliencia/Polly|Polly]], que fornece políticas de resiliência como retries, circuit breakers, timeouts, entre outras. Neste post, vou te mostrar como configurar e utilizar o [[Resiliencia/Polly|Polly]] em uma API ASP.NET Core, com foco em retries e timeouts, garantindo que sua aplicação seja robusta e preparada para lidar com falhas.

## Configurando a API ASP.NET Core com Polly
Vamos começar configurando uma aplicação ASP.NET Core simples que usa [[Resiliencia/Polly|Polly]] para adicionar resiliência ao consumir um serviço externo. Aqui está o código de configuração:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHttpClient();
builder.Services.AddSingleton<WeatherService>();

builder.Services.AddResiliencePipeline("default", x =>
{
    x.AddRetry(new RetryStrategyOptions
    {
        ShouldHandle = new PredicateBuilder().Handle<Exception>(),
        Delay = TimeSpan.FromSeconds(2),
        MaxRetryAttempts = 2,
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        OnRetry = static args =>
        {
            Console.WriteLine("OnRetry, Attempt: {0} - {1}", args.AttemptNumber, args.Outcome.Exception.Message);

            return default;
        }
    })
    .AddTimeout(TimeSpan.FromSeconds(30));
});

var app = builder.Build();

app.MapGet("/weather/{city}", async (string city, WeatherService weatherService) =>
{
    var weather = await weatherService.GetWeatherAsync(city);
    return weather is null ? Results.NotFound() : Results.Ok(weather);
});

app.Run();
```

Neste exemplo, estamos configurando uma API que expõe um endpoint GET `/weather/{city}`, que consulta um serviço externo para obter dados meteorológicos. Usamos o [[Resiliencia/Polly|Polly]] para aplicar uma política de retry e timeout, garantindo que, em caso de falha temporária, a requisição será repetida algumas vezes antes de retornar um erro.


## O Serviço de Clima: WeatherService
O `WeatherService` é responsável por fazer a chamada HTTP ao serviço externo, utilizando a política de resiliência configurada:

```csharp
public class WeatherService(
    IHttpClientFactory httpClientFactory,
    ResiliencePipelineProvider<string> pipelineProvider)
{
    public async Task<WeatherResponse?> GetWeatherAsync(string city)
    {
        try
        {
            var url = $"https://api.opeweathermap.org/data/2.5/weather?q={city}";

            var httpClient = httpClientFactory.CreateClient();

            var pipeline = pipelineProvider.GetPipeline("default");

            var weatherResponse = await pipeline.ExecuteAsync(async ct => await httpClient.GetAsync(url, ct));

            if (!weatherResponse.IsSuccessStatusCode)
            {
                return null;
            }

            return await weatherResponse.Content.ReadFromJsonAsync<WeatherResponse>();
        }
        catch (Exception e)
        {
            Console.WriteLine(e);
            return null;
        }
    }
}
```
Neste código, utilizamos o `IHttpClientFactory` para criar uma instância de `HttpClient` e fazemos a chamada ao serviço externo usando a pipeline de resiliência que configuramos anteriormente. Em caso de falha, [[Resiliencia/Polly|Polly]] tentará novamente a requisição, conforme definido na política de retry, e também aplicará um timeout para evitar que a chamada fique indefinidamente pendente.

## Conclusão
Adicionar resiliência às suas APIs utilizando [[Resiliencia/Polly|Polly]] é uma maneira eficaz de garantir que sua aplicação possa lidar com falhas temporárias em serviços externos. O exemplo acima demonstra como configurar retries e timeouts de maneira simples e eficaz, ajudando a melhorar a robustez e confiabilidade de suas aplicações em produção.

