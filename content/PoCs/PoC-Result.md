---
title: "Implementando o Result Pattern em C#: Simplifique o Tratamento de Erros e Sucessos com Código Limpo"
tags:
  - PoCs
---

## O que é o Result Pattern?
O Result Pattern é uma abordagem que encapsula o resultado de uma operação, oferecendo uma forma estruturada de lidar tanto com sucessos quanto com falhas. Em vez de confiar em exceções para gerenciar erros ou retornar valores nulos, este padrão permite que você retorne um objeto contendo informações sobre o sucesso ou falha da operação, facilitando a manipulação e o fluxo de controle no código.

## Benefícios do Result Pattern

- **Clareza no Código**: Com o `Result<T>`, fica fácil identificar onde e como uma operação pode falhar, além de como os erros devem ser tratados.
- **Padronização**: Garante que todos os métodos sigam uma convenção uniforme ao lidar com sucesso ou falha, o que facilita a manutenção e a leitura do código.
- **Facilidade de Uso**: A conversão implícita entre o tipo de resultado e `Result<T>` permite escrever código mais limpo e intuitivo.
- **Confiança no Código**: Ao utilizar `Result<T>`, você pode ter certeza de que todos os caminhos de sucesso e falha são explicitamente tratados.

## Implementação do Result Pattern em C#
Vamos explorar uma implementação simples do Result Pattern em C#. O exemplo a seguir demonstra como configurar e utilizar o `Result<T>` para encapsular tanto o conteúdo de uma operação bem-sucedida quanto os detalhes de um erro, se houver.

```csharp
namespace PoC_Result
{
    public class Result<T> where T : class
    {
        public T Content { get; private set; }
        public bool IsFailed => Error is not null || Error is { };
        public ErrorDescription Error { get; private set; }

        private Result() { }

        private static Result<T> Success(T content)
        {
            return new Result<T>
            {
                Content = content
            };
        }

        private static Result<T> Failure(ErrorDescription error)
        {
            return new Result<T>
            {
                Error = error
            };
        }

        public static implicit operator Result<T>(T content)
        {
            return Success(content);
        }

        public static implicit operator Result<T>(ErrorDescription error)
        {
            return Failure(error);
        }
    }
}
```

## Como Funciona?

- **Content**: Armazena o resultado de uma operação bem-sucedida.
- **Error**: Descreve o erro caso a operação tenha falhado.
- **IsFailed**: Indica se a operação falhou, tornando a lógica de verificação de erros mais clara.
- **Conversão Implícita**: Facilita o uso do `Result<T>`, permitindo que o código seja mais direto e legível.

## Exemplo de Uso no Serviço de Teste

```csharp
namespace PoC_Result
{
    public record ErrorDescription(string Message);
    public class Error
    {
        public static ErrorDescription TestError = new("Ocorreu um erro durante os testes.");
    }

    public record TestOutput(string Output);

    public interface ITestService
    {
        Task<Result<TestOutput>> ExecuteAsync(string input);
    }

    public class TestService : ITestService
    {
        public async Task<Result<TestOutput>> ExecuteAsync(string input)
        {
            var obj = new TestOutput("test...");

            if (input.Equals("test"))
                return Error.TestError;

            return obj;
        }
    }
}
```

## Conclusão
O Result Pattern é uma abordagem poderosa para melhorar a clareza e a robustez do seu código, especialmente em aplicações onde o tratamento de erros e a padronização são críticos. Com este padrão, você ganha controle total sobre como os sucessos e falhas são gerenciados, permitindo a construção de aplicações mais previsíveis e fáceis de manter.

Além disso, este padrão facilita o teste e a manutenção do código, pois todas as operações retornam um resultado estruturado, eliminando a necessidade de verificações manuais e reduzindo a dependência de exceções para controle de fluxo.