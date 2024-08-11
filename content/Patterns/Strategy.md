---
title: "Strategy"
tags:
  - Patterns
---

O padrão Strategy é um padrão de design comportamental que permite definir uma família de algoritmos, encapsular cada um deles e torná-los intercambiáveis. Isso permite que o algoritmo varie independentemente dos clientes que o utilizam. Em outras palavras, o Strategy permite que você defina uma série de estratégias (ou algoritmos) encapsulados em classes separadas e use essas estratégias de forma intercambiável, com base nas necessidades do seu aplicativo.

### Componentes do Pattern Strategy:

```csharp
public interface IStrategy  
{  
    Task ExecuteAsync(UserInput input);  
}
```
A interface `IStrategy` define um contrato para as estratégias que serão implementadas. Ela contém o método `ExecuteAsync`, que recebe um objeto do tipo `UserInput` como parâmetro e retorna uma `Task`, indicando que a execução da estratégia será assíncrona. Este é o coração do Pattern Strategy, onde cada implementação concreta fornecerá uma lógica específica para lidar com o input do usuário.


```csharp
public  interface IMyStrategy1 : IStrategy{}
public class MyStrategy1 : IMyStrategy1  
{  
    public Task ExecuteAsync(UserInput input)  
    {        
	    Console.WriteLine(nameof(MyStrategy1));  
        return Task.CompletedTask;  
    }
}
```
Aqui, definimos uma implementação concreta da interface `IStrategy` através de `MyStrategy1`, que também implementa a interface `IMyStrategy1`. O método `ExecuteAsync` simplesmente escreve o nome da estratégia (`MyStrategy1`) no console e retorna uma `Task` concluída. Isso representa uma estratégia específica que pode ser escolhida para execução com base no input.

```csharp
public interface IMyStrategy2 : IStrategy{}
public class MyStrategy2 : IMyStrategy2  
{  
    public Task ExecuteAsync(UserInput input)  
    {        
	    Console.WriteLine(nameof(MyStrategy2));  
        return Task.CompletedTask;  
    }
}
```
De forma semelhante à `MyStrategy1`, `MyStrategy2` é uma implementação concreta da interface `IStrategy`, mas desta vez implementando a interface `IMyStrategy2`. Novamente, o método `ExecuteAsync` imprime o nome da estratégia (`MyStrategy2`) no console e retorna uma `Task` concluída. Cada estratégia pode ser usada em diferentes contextos com base no input recebido.

```csharp
public interface IExecuteStrategies  
{  
    Task Handle(UserInput input);  
}

public class ExecuteStrategies : IExecuteStrategies  
{  
    private readonly Dictionary<string, IStrategy> _strategies = new();  
    
    public ExecuteStrategies(IMyStrategy1 myStrategy1, IMyStrategy2 myStrategy2)  
    {        
	    _strategies.Add("PF", myStrategy1);  
        _strategies.Add("PJ", myStrategy2);  
    }    
    public async Task Handle(UserInput input)  
    {        
	    await _strategies[input.Type].ExecuteAsync(input);  
    }
}  
```
A classe `ExecuteStrategies` é responsável por orquestrar a execução das diferentes estratégias. Ela implementa a interface `IExecuteStrategies`, que define o método `Handle`. No construtor da classe, um dicionário `_strategies` é preenchido com as diferentes estratégias, mapeando cada uma a uma chave específica (`"PF"` ou `"PJ"`). Quando o método `Handle` é chamado, ele verifica o tipo de input (input.Type) e executa a estratégia correspondente.

```csharp
public class UserInput  
{  
    public string Type { get; set; } = default!;  
}
```
A classe `UserInput` representa o input do usuário, com uma única propriedade `Type` que determina qual estratégia será selecionada para execução. Este tipo de input é passado para o método `Handle` na classe `ExecuteStrategies`, onde ele é usado para decidir qual estratégia executar.

```csharp
app.MapGet("/", async (IExecuteStrategies executeStrategies) =>  
{  
    await executeStrategies.Handle(new UserInput() { Type = "PJ"});  
  
    Results.Ok();  
});
```
Neste trecho, uma rota GET é mapeada para a aplicação. Quando esta rota é acessada, o método `Handle` de `ExecuteStrategies` é chamado com um `UserInput` do tipo `"PJ"`. Isso aciona a execução da estratégia correspondente a `"PJ"`, neste caso, `MyStrategy2`. A injeção de dependência é usada para fornecer a instância de `IExecuteStrategies` automaticamente.

```csharp
services.AddTransienty<IMyStrategy1, MyStrategy1>
services.AddTransienty<IMyStrategy2, MyStrategy2>
services.AddTransienty<IExecuteStrategies, ExecuteStrategies>
```
Finalmente, o código configura a injeção de dependência no contêiner de serviços da aplicação. Aqui, `MyStrategy1`, `MyStrategy2` e `ExecuteStrategies` são registrados com a duração de vida `Transient`, o que significa que uma nova instância será criada a cada vez que forem solicitadas. Isso permite que as estratégias sejam injetadas e usadas de forma flexível em diferentes partes da aplicação.

### Benefícios do Pattern Strategy:

- **Encapsulamento**: Cada estratégia é encapsulada em sua própria classe, o que facilita a manutenção e evolução do código.
- **Flexibilidade**: As estratégias podem ser trocadas dinamicamente, permitindo que o sistema se adapte a diferentes requisitos sem alterações significativas no código.
- **Reutilização de código**: As estratégias podem ser reutilizadas em diferentes contextos, promovendo a reutilização e evitando duplicação de código.

Em resumo, o padrão Strategy é útil quando você tem diferentes algoritmos que podem ser aplicados a uma determinada situação e deseja que esses algoritmos sejam intercambiáveis, permitindo que o sistema seja flexível e adaptável a mudanças.