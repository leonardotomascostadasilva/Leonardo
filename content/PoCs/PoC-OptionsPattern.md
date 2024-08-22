---
title: "ASP.NET Core: Configurando Serviços Externos com o Padrão Options"
tags:
  - PoCs
---

## Introdução ao Padrão Options
No desenvolvimento de aplicações ASP.NET Core, a gestão de configurações externas, como URLs de serviços, tokens de autenticação e parâmetros de resiliência, é uma prática comum e essencial. Uma abordagem poderosa para centralizar e validar essas configurações é o Padrão Options, que oferece uma maneira estruturada e tipada de acessar e validar dados de configuração.

Este artigo explora como configurar serviços externos em uma aplicação ASP.NET Core usando o Padrão Options, com exemplos práticos de código.


## Configurando Serviços Externos com o Padrão Options

### Estrutura do Projeto
Neste exemplo, vamos trabalhar com duas configurações de serviços externos (`ApiConfiguration` e `ApiConfiguration2`). Ambas as configurações derivam de uma classe base chamada `ExternalServices`, que define propriedades comuns como `Url`, `Token` e um objeto `Resilience` que controla o tempo de espera e o número de tentativas.

```csharp
using System.ComponentModel.DataAnnotations;

namespace PoC_OptionsPattern
{
    public abstract class ExternalServices
    {
        [Required, Url]
        public string Url { get; set; } = string.Empty;

        [Required]
        public string Token { get; set; } = string.Empty;

        [Required]
        public Resilience? Resilience { get; set; }
    }

    public class Resilience
    {
        [Required]
        public int TimeoutInSeconds { get; set; }

        [Required]
        public int NumberOfRetrys { get; set; }
    }
}

namespace PoC_OptionsPattern
{
    public class ApiConfiguration : ExternalServices
    {
        public const string ConfigurationSection = "ExternalServices:Api1";
    }

    public class ApiConfiguration2 : ExternalServices
    {
        public const string ConfigurationSection = "ExternalServices:Api2";
    }
}
```
### Configuração da Aplicação
Para carregar e validar as configurações, utilizamos o método `AddOptions<T>` no Program, onde configuramos cada serviço com sua respectiva seção de configuração.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services
    .AddOptions<ApiConfiguration>()
    .BindConfiguration(ApiConfiguration.ConfigurationSection)
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services
    .AddOptions<ApiConfiguration2>()
    .BindConfiguration(ApiConfiguration2.ConfigurationSection)
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Aqui, `AddOptions<T>` registra cada classe de configuração como um serviço no contêiner de dependências. O método `BindConfiguration` mapeia os valores da configuração do arquivo `appsettings.json` para as respectivas classes (`ApiConfiguration` e `ApiConfiguration2`). Além disso, `ValidateDataAnnotations` assegura que as propriedades obrigatórias sejam validadas na inicialização, evitando que a aplicação seja executada com configurações inválidas.

## Implementação do Endpoint para testar as Configurações

Vamos adicionar um endpoint que expõe as configurações carregadas para verificarmos se tudo está funcionando corretamente:

```csharp
var app = builder.Build();

app.UseHttpsRedirection();

app.MapGet("options", (IOptions<ApiConfiguration> apiConfiguration, IOptions<ApiConfiguration2> apiConfiguration2) =>
{
    var response = new
    {
        Api1 = apiConfiguration.Value,
        Api2 = apiConfiguration2.Value
    };
    return Results.Ok(response);
});

try
{
    app.Run();
}
catch (Exception e)
{
    Console.WriteLine(e);
}
```
Este endpoint GET retorna um objeto anônimo com as configurações de `ApiConfiguration` e `ApiConfiguration2`. Usando `IOptions<T>`, acessamos as configurações carregadas no momento da execução.

## Configuração no appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ExternalServices": {
    "Api1": {
      "Url": "https://localhost",
      "Token": "Bearer token",
      "Resilience": {
        "TimeoutInSeconds": 1,
        "NumberOfRetrys": 3
      }
    },
    "Api2": {
      "Url": "https://localhost",
      "Token": "Bearer token",
      "Resilience": {
        "TimeoutInSeconds": 1,
        "NumberOfRetrys": 3
      }
    }
  }
}
```

## Validação e Segurança
As anotações de dados, como `[Required]` e `[Url]`, garantem que todas as propriedades necessárias sejam fornecidas e estejam corretas. Isso previne que erros de configuração sejam ignorados, o que poderia levar a falhas em produção.

## Conclusão
O uso do Padrão Options no ASP.NET Core é uma prática poderosa para gerenciar e validar configurações de serviços externos de maneira centralizada e tipada. Ele não apenas simplifica o acesso às configurações, mas também garante que a aplicação seja iniciada apenas com valores válidos, reduzindo o risco de problemas em produção.