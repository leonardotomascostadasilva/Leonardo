---
tags:
  - Artigos
---

O padrão Strategy é um padrão de design comportamental que permite definir uma família de algoritmos, encapsular cada um deles e torná-los intercambiáveis. Isso permite que o algoritmo varie independentemente dos clientes que o utilizam. Em outras palavras, o Strategy permite que você defina uma série de estratégias (ou algoritmos) encapsulados em classes separadas e use essas estratégias de forma intercambiável, com base nas necessidades do seu aplicativo.

### Componentes do Pattern Strategy:

```csharp
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
  
public interface IExecuteStrategies  
{  
    Task Handle(UserInput input);  
}
```

```csharp
public interface IStrategy  
{  
    Task ExecuteAsync(UserInput input);  
}
```

```csharp
public class MyStrategy1 : IMyStrategy1  
{  
    public Task ExecuteAsync(UserInput input)  
    {        
	    Console.WriteLine(nameof(MyStrategy1));  
        return Task.CompletedTask;  
    }
}
public  interface IMyStrategy1 : IStrategy{}
```

```csharp
public class MyStrategy2 : IMyStrategy2  
{  
    public Task ExecuteAsync(UserInput input)  
    {        
	    Console.WriteLine(nameof(MyStrategy2));  
        return Task.CompletedTask;  
    }
}
public interface IMyStrategy2 : IStrategy{}
```

```csharp
public class UserInput  
{  
    public string Type { get; set; } = default!;  
}
```

```csharp
app.MapGet("/", async (IExecuteStrategies executeStrategies) =>  
{  
    await executeStrategies.Handle(new UserInput() { Type = "PJ"});  
  
    Results.Ok();  
});
```

```csharp
service.AddTransienty<IMyStrategy1, MyStrategy1>
service.AddTransienty<IMyStrategy2, MyStrategy2>
service.AddTransienty<IExecuteStrategies, ExecuteStrategies>
```
### Benefícios do Pattern Strategy:

- **Encapsulamento**: Cada estratégia é encapsulada em sua própria classe, o que facilita a manutenção e evolução do código.
- **Flexibilidade**: As estratégias podem ser trocadas dinamicamente, permitindo que o sistema se adapte a diferentes requisitos sem alterações significativas no código.
- **Reutilização de código**: As estratégias podem ser reutilizadas em diferentes contextos, promovendo a reutilização e evitando duplicação de código.

Em resumo, o padrão Strategy é útil quando você tem diferentes algoritmos que podem ser aplicados a uma determinada situação e deseja que esses algoritmos sejam intercambiáveis, permitindo que o sistema seja flexível e adaptável a mudanças.