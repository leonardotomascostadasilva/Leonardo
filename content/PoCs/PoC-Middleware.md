---
title: "Implementando Middleware Personalizado em ASP.NET Core: Logging Simplificado para Suas Aplicações"
tags:
  - PoCs
---

## O que é Middleware em ASP.NET Core?
Middleware é um conceito fundamental no ASP.NET Core, responsável por gerenciar o fluxo de requisições e respostas em uma aplicação web. Cada requisição HTTP passa por uma cadeia de middlewares antes de chegar ao seu destino final, permitindo que você insira lógica personalizada em diversos pontos do pipeline de execução.

## Benefícios de Usar Middleware Personalizado
- **Reutilização de Código**: Middlewares encapsulam funcionalidades que podem ser facilmente reutilizadas em diferentes partes da aplicação.
- **Isolamento de Lógica**: Permitem separar preocupações, isolando funcionalidades específicas como autenticação, logging, ou manipulação de exceções.
- **Flexibilidade e Controle**: Você pode criar middlewares que inspecionam, alteram ou condicionam o processamento de requisições e respostas de maneira centralizada.

## Implementação de um Middleware de Logging em ASP.NET Core

Vamos explorar a criação de um middleware personalizado para logging em uma aplicação ASP.NET Core. Esse middleware registrará informações sobre as requisições e respostas, facilitando o monitoramento e depuração da aplicação.

```csharp
namespace PoC_Middleware.Middlewares
{
    public class LoggingMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<LoggingMiddleware> _logger;

        public LoggingMiddleware(RequestDelegate next, ILogger<LoggingMiddleware> logger)
        {
            _next = next;
            _logger = logger;
        }

        public async Task InvokeAsync(HttpContext context)
        {
            _logger.LogInformation("Processing request: {Path}", context.Request.Path);

            await _next(context);

            _logger.LogInformation("Response status code: {StatusCode}", context.Response.StatusCode);
        }
    }
}
```
- **RequestDelegate**: Representa o próximo middleware no pipeline de execução. O `LoggingMiddleware` chama `_next(context)` para garantir que a requisição continue sua jornada.
- **ILogger**: A injeção do `ILogger<LoggingMiddleware>` permite que o middleware registre informações relevantes sobre a requisição e a resposta.
- **InvokeAsync**: Método principal onde ocorre o registro das informações. Antes de continuar para o próximo middleware, ele registra o caminho da requisição; após a execução do próximo middleware, ele registra o código de status da resposta.

## Integrando o Middleware na Aplicação
```csharp
using PoC_Middleware;
using PoC_Middleware.Middlewares;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

app.UseMiddleware<LoggingMiddleware>();

app.MapPost("/api/v1", async (CancellationToken cancellationToken) =>
{
    return Results.Created();
});

app.Run();
```

## Como Funciona?
- **Configuração Inicial**: O código começa configurando a aplicação ASP.NET Core, adicionando suporte à documentação com Swagger.
- **Pipeline de Execução**: Após configurar o Swagger e o redirecionamento HTTPS, o middleware de logging (LoggingMiddleware) é adicionado ao pipeline.
- **Rota Mapeada**: Uma rota de exemplo é definida usando MapPost, onde uma requisição POST é tratada e um status 201 Created é retornado.

## Conclusão
Criar um middleware personalizado em ASP.NET Core é uma maneira poderosa de adicionar funcionalidades transversais à sua aplicação. Com o exemplo de middleware de logging, você tem uma base sólida para monitorar o tráfego HTTP e depurar problemas de forma mais eficaz. Esse padrão pode ser estendido para incluir outras funcionalidades, como manipulação de exceções, autenticação, e muito mais, proporcionando um controle granular sobre o fluxo de requisições e respostas na sua aplicação.