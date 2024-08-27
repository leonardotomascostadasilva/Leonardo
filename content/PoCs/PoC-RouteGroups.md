---
title: "Configurando Grupos de Endpoints com Suporte a Múltiplas Versões da API no ASP.NET Core"
tags:
  - PoCs
---


Quando estamos desenvolvendo APIs que precisam suportar múltiplas versões, uma prática comum é separar os endpoints por versão para facilitar a manutenção e evitar conflitos entre versões diferentes da mesma funcionalidade. Neste exemplo, vamos ver como configurar e organizar grupos de endpoints para múltiplas versões de uma API utilizando o ASP.NET Core.


## Estrutura da Solução
A solução está organizada em diferentes features, onde cada feature tem seus próprios endpoints. Essas features são agrupadas em versões da API usando grupos de endpoints. A seguir, vamos explorar o código que implementa essa estrutura.

### Configuração do Swagger para Múltiplas Versões
Primeiro, vamos configurar o Swagger para documentar as versões `v1` e `v2` da API:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new Microsoft.OpenApi.Models.OpenApiInfo
    {
        Version = "v1",
        Title = "API V1",
        Description = "Versão 1 da API"
    });

    options.SwaggerDoc("v2", new Microsoft.OpenApi.Models.OpenApiInfo
    {
        Version = "v2",
        Title = "API V2",
        Description = "Versão 2 da API"
    });

    options.DocInclusionPredicate((documentName, apiDescription) => apiDescription.RelativePath!.StartsWith($"api/{documentName}"));
});
```
Aqui, estamos criando duas documentações separadas para `v1` e `v2`. A opção `DocInclusionPredicate` garante que cada endpoint seja incluído na documentação correta com base na versão especificada no caminho.

### Mapeando Endpoints por Versão
Agora, vamos organizar os endpoints em grupos de versões dentro do nosso pipeline de requisições:

```csharp
var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI(options =>
{
    options.SwaggerEndpoint("/swagger/v1/swagger.json", "API V1");
    options.SwaggerEndpoint("/swagger/v2/swagger.json", "API V2");
});

app.MapEndpoints();

app.Run();
```
O método `MapEndpoints` é uma extensão que criamos para organizar e mapear os endpoints de acordo com suas versões:

```csharp
namespace PoC_RouteGroups.Commons
{
    public static class EndpointExtensions
    {
        public static void MapEndpoints(this WebApplication app)
        {
            var v1Endpoints = app.MapGroup("/api/v1");

            v1Endpoints
                .MapGroup("/sample")
                .WithTags("Sample")
                .MapEndpoint<Features.SampleFeatureOne.CreateSampleFeature>()
                .MapEndpoint<Features.SampleFeatureOne.GetSampleFeature>();

            var v2Endpoints = app.MapGroup("/api/v2");

            v2Endpoints
                .MapGroup("/sample")
                .WithTags("Sample")
                .MapEndpoint<Features.SampleFeatureOne.CreateSampleFeature>()
                .MapEndpoint<Features.SampleFeatureOne.GetSampleFeature>();

        }

        private static IEndpointRouteBuilder MapEndpoint<TEndpoint>(this IEndpointRouteBuilder app) where TEndpoint : IEndpoint
        {
            TEndpoint.Map(app);
            return app;
        }
    }
}
```
Neste exemplo, temos dois grupos de versões, `v1` e `v2`, cada um contendo seus próprios endpoints. A interface IEndpoint é utilizada para garantir que cada endpoint siga um padrão de mapeamento:

```csharp
namespace PoC_RouteGroups.Commons
{
    public interface IEndpoint
    {
        static abstract void Map(IEndpointRouteBuilder app);
    }
}
```

### Implementação dos Endpoints
Para cada versão da API, você pode ter diferentes implementações de endpoints. Vamos ver como isso é feito para duas features diferentes:

#### Feature 1
```csharp
namespace PoC_RouteGroups.Features.SampleFeatureOne
{
    public record SampleObject(string Description);
    
    public class CreateSampleFeature : IEndpoint
    {
        public static void Map(IEndpointRouteBuilder app)
        {
            app.MapPost("/", HandleAsync);
        }

        [ProducesResponseType(typeof(SampleObject), 200)]
        private static async Task<IResult> HandleAsync(SampleObject sampleObject)
        {
            return Results.Ok($"CreateSampleFeature - {sampleObject.Description}");
        }
    }

    public class GetSampleFeature : IEndpoint
    {
        public static void Map(IEndpointRouteBuilder app)
        {
            app.MapGet("/", HandleAsync);
        }

        [ProducesResponseType(typeof(string), 200)]
        private static async Task<IResult> HandleAsync()
        {
            return Results.Ok("GetSampleFeature");
        }
    }
}
```

#### Feature 2
```csharp
namespace PoC_RouteGroups.Features.SampleFeatureTwo
{
    public record SampleObject(string Description);
    
    public class CreateSampleFeature : IEndpoint
    {
        public static void Map(IEndpointRouteBuilder app)
        {
            app.MapPost("/", HandleAsync);
        }

        [ProducesResponseType(typeof(SampleObject), 200)]
        private static async Task<IResult> HandleAsync(SampleObject sampleObject)
        {
            return Results.Ok($"CreateSampleFeature - {sampleObject.Description}");
        }
    }

    public class GetSampleFeature : IEndpoint
    {
        public static void Map(IEndpointRouteBuilder app)
        {
            app.MapGet("/", HandleAsync);
        }

        [ProducesResponseType(typeof(string), 200)]
        private static async Task<IResult> HandleAsync()
        {
            return Results.Ok("GetSampleFeature");
        }
    }
}
```
Cada uma dessas features pode ser separada por versão e pode ter suas próprias implementações. Isso torna a API altamente modular e fácil de manter.

## Conclusão
A abordagem de separar endpoints por versão usando grupos no ASP.NET Core é uma prática eficaz para garantir que diferentes versões da API possam coexistir de forma organizada e escalável. Isso facilita a manutenção e a evolução da API sem causar impacto em versões anteriores.