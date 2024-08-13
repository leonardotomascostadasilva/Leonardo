---
title: "Implementando Pipelines e Validação em C# com FluentValidation, MediatR e Serilog"
tags:
  - PoCs
---

Neste artigo, vamos explorar como configurar e utilizar pipelines e validação em uma aplicação ASP.NET Core, usando as bibliotecas FluentValidation, MediatR e Serilog. Essas ferramentas permitem construir aplicações robustas, com uma arquitetura bem organizada e de fácil manutenção. A seguir, detalharemos a configuração do projeto, implementando validação rápida de requisições e logging detalhado.

## Configuração Inicial do Projeto

### Setup do Serilog para Logging

Começamos configurando a aplicação para usar Serilog, uma biblioteca robusta de logging. Serilog facilita a captura de logs tanto em console quanto em arquivos, e permite configurar diferentes níveis de log para diferentes ambientes.

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "System": "Warning",
      "Microsoft": "Warning"
    }
  },
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information"
    },
    "WriteTo": [
      {
        "Name": "Console"
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/myapp.txt",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 7
        }
      }
    ]
  }
}
```
```csharp
var builder = WebApplication.CreateBuilder(args);

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .WriteTo.Console()
    .WriteTo.File("logs/myapp.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();

builder.Host.UseSerilog();
```
Com essa configuração, garantimos que todos os eventos importantes da aplicação sejam capturados e persistidos, o que é crucial para depuração e monitoramento.

### Adicionando Validação com FluentValidation
Para manter a integridade dos dados que transitam pela nossa aplicação, utilizamos FluentValidation. Esta biblioteca nos permite definir regras de validação de forma clara e declarativa. No exemplo abaixo, criamos um validador para o comando `CreateMessageCommand`, assegurando que campos essenciais sejam preenchidos.

```csharp
using FluentValidation;
using PoC_PipelineBehavior.Commands;

namespace PoC_PipelineBehavior.Validators
{
    public class CreateMessageValidator : AbstractValidator<CreateMessageCommand>
    {
        public CreateMessageValidator()
        {
            RuleFor(a => a.Description)
               .NotEmpty()
               .WithMessage("Descrição é obrigatória.");

            RuleFor(a => a.Topic)
               .NotEmpty()
               .WithMessage("Topic é obrigatório.");

            RuleFor(a => a.Body)
               .NotEmpty()
               .WithMessage("Corpo é obrigatório.");
        }
    }
}
```

### Implementando o Pipeline Behavior com MediatR

Para garantir que as requisições passem por etapas de validação e logging antes de serem processadas, configuramos um pipeline behavior utilizando MediatR. O MediatR facilita a separação de preocupações ao delegar o processamento de requisições para handlers específicos.

#### FailFastRequestBehavior

Este comportamento valida as requisições antes que elas sejam processadas. Se houver falhas, o processamento é interrompido, e os erros são retornados imediatamente.

```csharp
using FluentValidation;
using FluentValidation.Results;
using MediatR;
using PoC_PipelineBehavior.Contracts;

namespace PoC_PipelineBehavior.PipelinesBehavior
{
    public class FailFastRequestBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
        where TRequest : IRequest<TResponse> where TResponse : Response
    {
        private readonly IEnumerable<IValidator<TRequest>> _validators;

        public FailFastRequestBehavior(IEnumerable<IValidator<TRequest>> validators)
        {
            _validators = validators;
        }

        public Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken)
        {
            var context = new ValidationContext<TRequest>(request);
            var failures = _validators.Select(v => v.Validate(context))
                                      .SelectMany(result => result.Errors)
                                      .Where(f => f != null)
                                      .ToList();

            return failures.Any() ? Errors(failures) : next();
        }

        private static Task<TResponse> Errors(IEnumerable<ValidationFailure> failures)
        {
            var response = new Response();
            foreach (var failure in failures)
            {
                response.AddError(failure.ErrorMessage);
            }
            return Task.FromResult(response as TResponse);
        }
    }
}
```

#### LoggingBehavior
Este comportamento registra logs antes de uma requisição ser processada, capturando tanto a requisição quanto a resposta. Isso ajuda a monitorar e rastrear o fluxo de dados na aplicação.

```csharp
using MediatR;
using Serilog;

namespace PoC_PipelineBehavior.PipelinesBehavior
{
    public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    {
        public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken)
        {
            Log.Information("Handling {HandlingName} - Request: {@Request}", typeof(TRequest).Name, request);
            var response = await next();
            return response;
        }
    }
}
```

### Configuração dos Serviços no ASP.NET Core

Após definir os behaviors, é necessário configurá-los no contêiner de serviços da aplicação.

```csharp
var assembly = Assembly.GetExecutingAssembly();
AssemblyScanner.FindValidatorsInAssembly(assembly)
               .ForEach(result => builder.Services.AddScoped(result.InterfaceType, result.ValidatorType));

builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
builder.Services.AddScoped(typeof(IPipelineBehavior<,>), typeof(FailFastRequestBehavior<,>));

builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(assembly));
```
Essa configuração assegura que todos os comandos passem pelos comportamentos de logging e validação, proporcionando uma aplicação mais robusta e fácil de manter.


### Exemplo de Uso: Endpoint para Criação de Mensagem
Por fim, implementamos um endpoint simples para demonstrar o uso do pipeline behavior. Este endpoint recebe um comando `CreateMessageCommand`, valida os dados, loga a requisição e processa a criação da mensagem.

```csharp
using MediatR;
using Microsoft.AspNetCore.Mvc;
using PoC_PipelineBehavior.Commands;

namespace PoC_PipelineBehavior.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class MessageController : ControllerBase
    {
        private readonly IMediator _mediator;

        public MessageController(IMediator mediator)
        {
            _mediator = mediator;
        }

        [HttpPost]
        public async Task<IActionResult> CreateAsync([FromBody] CreateMessageCommand createMessageCommand, CancellationToken cancellationToken)
        {
            var response = await _mediator.Send(createMessageCommand, cancellationToken);
            if (response.Errors.Any())
            {
                return BadRequest(response.Errors);
            }
            return Ok(response.Result);
        }
    }
}
```
## Conclusão
Integrar MediatR, FluentValidation e Serilog no ASP.NET Core ajuda a criar uma arquitetura mais organizada, modular e fácil de manter. Com esses behaviors de validação e logging no pipeline de requisições, você garante que as operações de negócio sejam feitas de forma consistente e segura. Usar MediatR e PipelineBehavior traz vários benefícios, como separar a lógica de negócios, centralizar validações e logging, além de facilitar a manutenção e a extensão da aplicação. 