---
title: "Processamento Assíncrono com Channels em ASP.NET Core: Um Guia Passo a Passo"
tags:
  - PoCs
---

No mundo do desenvolvimento de software, especialmente quando falamos de aplicações que lidam com grandes volumes de dados ou eventos, é fundamental garantir que o processamento seja eficiente e escalável. Uma técnica eficaz para atingir isso em uma API ASP.NET Core é o uso de `System.Threading.Channels` para processamento assíncrono. Neste post, vou te mostrar como você pode implementar isso, com direito a código de exemplo e explicações detalhadas.

## O Que São Channels?
Antes de mais nada, vamos entender o que são Channels. Channels são uma estrutura de dados na qual você pode enviar e receber mensagens de forma assíncrona. Eles são altamente eficientes para comunicação entre diferentes partes de uma aplicação que precisam processar mensagens ou eventos em paralelo.

## Configurando a API ASP.NET Core
Vamos começar configurando uma aplicação ASP.NET Core simples. Abaixo, temos o código de configuração da aplicação:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddSingleton<MyQueue>();
builder.Services.AddHostedService<MyQueueProcessor>();
builder.Services.AddHostedService<MyQueueProcessor2>();
builder.Services.AddHostedService<MyQueueProcessor3>();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();
app.UseHttpsRedirection();

app.MapPost("/api/channel", async (MyQueue queue, CancellationToken cancellation) =>
{
    Parallel.ForEach(Enumerable.Range(0, 1000), async i =>
    {
        await queue.Writer.WriteAsync(new MyEvent() { Description = $"test-{i}" }, cancellation);
    });

    return Results.Created();
});

app.Run();
```
Aqui, estamos criando uma API que expõe um endpoint POST que envia 1000 eventos para uma fila (`Channel`). Esses eventos são processados por três serviços hospedados (`MyQueueProcessor`, `MyQueueProcessor2`, e `MyQueueProcessor3`), que consomem os eventos de forma assíncrona.

## Entendendo a Fila: MyQueue

A `MyQueue` é uma classe simples que encapsula um `Channel`. Veja como ela é implementada:

```csharp
public class MyQueue
{
    private readonly Channel<MyEvent> _channel = Channel.CreateUnbounded<MyEvent>();

    public ChannelWriter<MyEvent> Writer => _channel.Writer;
    public ChannelReader<MyEvent> Reader => _channel.Reader;
}
```
Aqui, utilizamos `Channel.CreateUnbounded<MyEvent>()`, que cria um canal sem limite de buffer. Isso significa que o canal pode armazenar quantos eventos forem necessários até que um consumidor esteja pronto para processá-los.

## Processando Eventos com `backgroud service`
Os eventos que chegam na fila precisam ser processados, e para isso, configuramos três `backgroud service`. Esses serviços são representados pelas classes `MyQueueProcessor`, `MyQueueProcessor2`, e `MyQueueProcessor3`:

```csharp
public sealed class MyQueueProcessor(MyQueue queue) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await QueueProcessorExtensions.CreateQueueProcessorAsync("MyQueueProcessor", queue, stoppingToken);
    }
}
```
O que esses serviços fazem? Eles leem os eventos da fila e os processam de forma assíncrona. Vamos ver o método de extensão que realiza essa tarefa:

```csharp
public static class QueueProcessorExtensions
{
    public static async Task CreateQueueProcessorAsync(string processor, MyQueue queue, CancellationToken stoppingToken)
    {
        await foreach (var @event in queue.Reader.ReadAllAsync(stoppingToken))
        {
            await Task.Delay(1000, stoppingToken);
            Console.WriteLine($"{processor}-{@event.Description} - {queue.Reader.Count}");
        }
    }
}
```
Esse método itera sobre todos os eventos disponíveis no ChannelReader e os processa com um delay de 1 segundo, simulando uma operação que exige tempo, como uma consulta a um banco de dados ou uma chamada a um serviço externo.

## Benefícios e Aplicações
O uso de `Channels` em uma API ASP.NET Core traz diversos benefícios:

- **Escalabilidade**: Ao distribuir a carga de processamento entre vários consumidores, você pode lidar com um grande número de eventos sem sobrecarregar a aplicação.

- **Simplicidade**: A configuração e o uso dos `Channels` são diretos, o que torna essa abordagem fácil de implementar e entender.

- **Controle**: Com `backgroud service`, você tem controle total sobre como e quando os eventos são processados, podendo, inclusive, implementar lógica de retry ou manipulação de erros customizada.


## Conclusão
Integrar `Channels` ao seu projeto ASP.NET Core é uma maneira poderosa de gerenciar processamento assíncrono, especialmente em cenários onde alta escalabilidade e eficiência são necessárias. O exemplo acima fornece uma base sólida para começar a implementar essa solução em suas aplicações, garantindo que você possa lidar com grandes volumes de dados de maneira eficiente e controlada.