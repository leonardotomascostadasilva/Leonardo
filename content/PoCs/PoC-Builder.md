---
title: "Construindo Objetos com Builders em C#: Explicação, Benefícios e Exemplos Práticos"
tags:
  - PoCs
---

## O que são Builders?
Builders são um padrão de design utilizado para a criação de objetos complexos em linguagens de programação como C#. O objetivo principal desse padrão é permitir a construção de objetos de forma controlada e passo a passo, garantindo que o objeto criado esteja sempre em um estado válido e completo antes de ser utilizado.

## Benefícios de Utilizar Builders
- **Leitura e Manutenção do Código**: Com Builders, o código que cria objetos se torna mais legível e organizado. Ao invés de ter um construtor com vários parâmetros ou múltiplos métodos `Set`, o Builder permite que cada parte do objeto seja configurada em etapas.

- **Reutilização**: Com Builders, você pode encapsular a lógica de criação de objetos em uma única classe, reutilizando-a em todo o sistema sempre que precisar criar objetos semelhantes.

- **Flexibilidade**: A possibilidade de configurar objetos passo a passo oferece uma flexibilidade enorme, permitindo a criação de diferentes variações do mesmo objeto de forma intuitiva e organizada.

- **Validações Fluent**: As asserções e validações dos objetos construídos podem ser facilmente verificadas, garantindo que o objeto final esteja sempre em um estado válido e coerente com o esperado.

- **Código Limpo**: Builders ajudam a evitar a criação de múltiplos construtores sobrecarregados, reduzindo a complexidade do código e mantendo as classes mais enxutas e fáceis de entender.


## Implementação de Builders em C#
A seguir, veremos como implementar o padrão Builder em C# com exemplos práticos. Vamos construir um sistema simples que envolve três classes: `Content`, `Headers`, e `Message`. Cada uma dessas classes possui seu respectivo Builder, permitindo a construção de objetos de forma clara e fluida.

### Classe Content
A classe `Content` é responsável por armazenar informações sobre o conteúdo da mensagem, como `Id`, `Title`, e `Description`.

```csharp
namespace PoC_Builder
{
    public class Content
    {
        public string Id { get; set; }
        public string Description { get; set; }
        public string Title { get; set; }
    }
}
```

### Implementação do ContentBuilder
O `ContentBuilder` facilita a construção de objetos `Content`, permitindo que as propriedades sejam configuradas em etapas, seguindo a abordagem do padrão Builder.

```csharp
namespace PoC_Builder
{
    public class ContentBuilder
    {
        private Content Content = new();
        public static ContentBuilder Create() => new();

        private ContentBuilder() { }

        public ContentBuilder WithId(string id)
        {
            Content.Id = id;
            return this;
        }

        public ContentBuilder WithTitle(string title)
        {
            Content.Title = title;
            return this;
        }

        public ContentBuilder WithDescription(string description)
        {
            Content.Description = description;
            return this;
        }

        public Content Build()
        {
            return Content;
        }
    }
}
```
### Classe Headers e HeadersBuilder
A classe `Headers` armazena metadados sobre a mensagem, como `Type` e `Mark`. A implementação do `HeadersBuilder` segue a mesma lógica do `ContentBuilder`, facilitando a criação de objetos `Headers`.

```csharp
namespace PoC_Builder
{
    public class Headers
    {
        public string Type { get; set; }
        public string Mark { get; set; }
    }
}

namespace PoC_Builder
{
    public class HeadersBuilder
    {
        private Headers headers = new();
        public static HeadersBuilder Create() => new();

        private HeadersBuilder() { }

        public HeadersBuilder WithType(string type)
        {
            headers.Type = type;
            return this;
        }

        public HeadersBuilder WithMark(string mark)
        {
            headers.Mark = mark;
            return this;
        }

        public Headers Build()
        {
            return headers;
        }
    }
}
```

### Classe Message e MessageBuilder
A classe `Message` é a entidade principal que combina `Headers` e `Content` em uma única estrutura. O `MessageBuilder` permite a criação de uma mensagem completa, configurando cada parte da mensagem (headers, conteúdo, etc.) de forma encadeada.

```csharp
namespace PoC_Builder
{
    public class Message
    {
        public string Key { get; set; }
        public DateTime SentOn { get; set; }
        public Headers Headers { get; set; }
        public Content Content { get; set; }
    }
}

namespace PoC_Builder
{
    public class MessageBuilder
    {
        private Message message = new();
        public static MessageBuilder Create() => new();

        private MessageBuilder() { }

        public MessageBuilder WithKey(string key)
        {
            message.Key = key;
            return this;
        }

        public MessageBuilder SentOn(DateTime dateTime)
        {
            message.SentOn = dateTime;
            return this;
        }

        public MessageBuilder WithHeaders(Action<HeadersBuilder> options)
        {
            var headers = HeadersBuilder.Create();
            options(headers);
            message.Headers = headers.Build();
            return this;
        }

        public MessageBuilder WithContent(Action<ContentBuilder> options)
        {
            var content = ContentBuilder.Create();
            options(content);
            message.Content = content.Build();
            return this;
        }

        public Message Build()
        {
            return message;
        }
    }
}
```
### Exemplo Prático de Uso
Abaixo, vemos um exemplo prático de como utilizar o `MessageBuilder` para criar um objeto `Message` completo. Este exemplo demonstra a flexibilidade e clareza proporcionadas pelo uso do padrão Builder.

```csharp
using PoC_Builder;
using System.Text.Json;

var message = MessageBuilder
    .Create()
    .WithKey("12345")
    .SentOn(DateTime.UtcNow)
    .WithHeaders(
        headers => headers
            .WithType("testType")
            .WithMark("testMark"))
    .WithContent(
        content => content
            .WithId(Guid.NewGuid().ToString())
            .WithTitle("test title")
            .WithDescription("test description")) 
    .Build();

Console.WriteLine(JsonSerializer.Serialize(message));
```

## Facilidade e Eficiência
Os Builders apresentados facilitam a criação de objetos complexos, garantindo que todas as dependências sejam configuradas corretamente. Além disso, essa abordagem resulta em código mais limpo e modular, que pode ser facilmente reutilizado em diferentes partes do sistema.

## Conclusão
O padrão Builder é uma poderosa ferramenta para construção de objetos complexos em C#. Com ele, podemos criar objetos de forma controlada e flexível, mantendo o código legível e fácil de manter. Os exemplos apresentados mostram como é possível aplicar esse padrão de forma prática e eficiente, garantindo que as necessidades de construção de objetos sejam atendidas de maneira clara e organizada.