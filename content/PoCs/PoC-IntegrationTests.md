---
title: "Testes Integrados em C#: Explicação, Benefícios e Facilidade com Exemplos de Código"
tags:
  - PoCs
---

## O que são Testes Integrados?
Testes integrados são uma forma de garantir que diferentes partes de uma aplicação funcionem corretamente quando integradas. Diferente dos testes unitários, que verificam o comportamento de componentes isolados, os testes integrados validam o comportamento de um sistema completo ou subsistemas interconectados. Eles simulam interações reais entre diferentes módulos da aplicação, como o acesso a bancos de dados, serviços externos ou APIs internas.

## Benefícios dos Testes Integrados

- **Validação Completa**: Testes integrados ajudam a identificar problemas que surgem apenas quando múltiplos componentes são combinados. Isso garante que a aplicação funcione corretamente em um ambiente de produção.

- **Redução de Erros em Produção**: Ao simular cenários reais, os testes integrados ajudam a capturar erros que poderiam passar despercebidos em testes unitários, reduzindo o risco de falhas em produção.

- **Documentação**: Servem como uma forma de documentação viva, demonstrando como os componentes interagem e fornecendo exemplos de uso real da aplicação.

- **Confiança no Código**: Com uma suite de testes integrados robusta, os desenvolvedores podem fazer mudanças no código com mais confiança, sabendo que os principais fluxos da aplicação estão sendo validados.

## Implementação de Testes Integrados em C#
Vamos explorar uma implementação de testes integrados em C#, utilizando o framework .NET Core. Os exemplos a seguir demonstram como configurar e executar testes integrados, usando ferramentas como WireMock para simular respostas de APIs externas e FluentAssertions para validações.


### Configuração da Aplicação de Teste
Para iniciar, é necessário configurar uma aplicação web customizada para os testes. No exemplo abaixo, criamos uma `CustomWebApplicationFactory` que configura o ambiente de teste e carrega as configurações específicas para testes a partir do arquivo `testsAppsettings.json`.

```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.TestHost;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;

namespace PoCTests.Api.IntegrationTest.Commons
{
    public class CustomWebApplicationFactory : WebApplicationFactory<FakerStartup>
    {
        protected override IHostBuilder? CreateHostBuilder()
        {
            return Host
                .CreateDefaultBuilder()
                .ConfigureAppConfiguration((_, configuration) =>
                {
                    configuration.Sources.Clear();
                    configuration.SetBasePath(AppContext.BaseDirectory);
                    configuration.AddJsonFile("testsAppsettings.json", true, true);
                })
                .ConfigureWebHostDefaults(w => w.UseStartup<FakerStartup>().UseTestServer())
                .UseEnvironment("test");
        }
    }
}
```

### Configuração do Servidor Mock
O `MockServerFixture` usa o WireMock para simular respostas de APIs externas. Isso permite testar o comportamento da aplicação sem depender de serviços externos reais.

```csharp
using WireMock.Server;
using WireMock.Settings;

namespace PoCTests.Api.IntegrationTest.Commons
{
    public class MockServerFixture
    {
        private readonly WireMockServer _wireMockServer;
        public readonly GetApiFixture GetApiFixture;

        public MockServerFixture()
        {
            _wireMockServer = WireMockServer.Start(new WireMockServerSettings
            {
                Urls = ["http://+:8085"],
                StartAdminInterface = true
            });

            GetApiFixture = new GetApiFixture(_wireMockServer);

        }

        public void Reset()
        {
            _wireMockServer.ResetMappings();
        }
    }
}
```

### Classe HelperFixture
A classe `HelperFixture` age como um contêiner que gerencia a configuração do `CustomWebApplicationFactory` e do `MockServerFixture`, além de expor o cliente HTTP para realizar as requisições de teste.
```csharp
namespace PoCTests.Api.IntegrationTest.Commons
{
    public class HelperFixture : IDisposable
    {
        private readonly CustomWebApplicationFactory _webApplicationFactory = new();
        public readonly MockServerFixture MockServerFixture = new();
        public HttpClient Client => _webApplicationFactory.Server.CreateClient();

        public void Dispose()
        {
            MockServerFixture.Reset();
        }
    }
}
```
### CustomWebApplicationFactoryCollection
A `CustomWebApplicationFactoryCollection` é uma coleção de testes que agrupa os testes integrados que usam a `CustomWebApplicationFactory`, permitindo o gerenciamento da criação e disposição do contexto de teste.

```csharp
namespace PoCTests.Api.IntegrationTest.Commons
{
    [CollectionDefinition(nameof(CustomWebApplicationFactoryCollection), DisableParallelization = true)]
    public class CustomWebApplicationFactoryCollection : ICollectionFixture<HelperFixture>
    {
    }
}
```

### Classe GetApiFixture
A classe `GetApiFixture` é responsável por configurar as respostas simuladas da API externa, utilizando WireMock para criar os endpoints simulados e suas respectivas respostas.

```csharp
using System.Net;
using WireMock.RequestBuilders;
using WireMock.ResponseBuilders;
using WireMock.Server;

namespace PoCTests.Api.IntegrationTest.Commons
{
    public class GetApiFixture
    {
        private readonly WireMockServer _wireMockServer;

        public GetApiFixture(WireMockServer wireMockServer)
        {
            _wireMockServer = wireMockServer;
        }

        public void Execute(string path, HttpStatusCode statusCode, object? response = null)
        {
            if (response is not null)
            {
                _wireMockServer
               .Given(Request
                   .Create()
                   .WithPath(path)
                   .UsingGet())
               .RespondWith(Response
                   .Create()
                   .WithBodyAsJson(response)
                   .WithStatusCode(statusCode));
            }
            else
            {
                _wireMockServer
               .Given(Request
                   .Create()
                   .WithPath(path)
                   .UsingGet())
               .RespondWith(Response
                   .Create()
                   .WithStatusCode(statusCode));
            }
        }
    }
}
```

### Classe FakerStartup
A classe `FakerStartup` representa a configuração inicial do ambiente de testes. Ela simula o comportamento da aplicação real, permitindo que os testes sejam executados em um ambiente controlado.

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;

namespace PoCTests.Api
{
    public class FakerStartup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddScoped<ILogin, Login>();

            services
                .AddRefitClient<IValidate>()
                .ConfigureHttpClient(c =>
                {
                    var url = Configuration.GetSection("ValidateConfiguration:url").Value;
                    c.BaseAddress = new Uri(url);
                });
            services.AddControllers();
            services.AddEndpointsApiExplorer();
            services.AddRouting(options => options.LowercaseUrls = true);
        }

        public void Configure(IApplicationBuilder app)
        {
            app.UseHttpsRedirection();
            app.UseRouting();
            app.UseAuthorization();
            app.UseEndpoints(e =>
            {
                e.MapControllers();
            });
        }
    }
}
```

### Teste de Endpoint com Verificação de Resposta
Abaixo está um exemplo de teste que verifica se um endpoint retorna a resposta correta. O método `Execute` simula uma resposta da API externa e o teste valida se o endpoint da aplicação retorna o valor esperado.

```csharp
using AutoFixture;
using FluentAssertions;
using PoCTests.Api.Features.Login;
using PoCTests.Api.IntegrationTest.Commons;
using System.Net;

namespace PoCTests.Api.IntegrationTest.Features
{
    [Collection(name: nameof(CustomWebApplicationFactoryCollection))]
    public class EndpointIntegrationTest(HelperFixture helperFixture)
    {
        private readonly Fixture _fixture = new();

        [Fact]
        public async Task GetEndpoint_GivenRequestReceived_ThenReturnsOKWithHelloWorldContent()
        {
            // Arrange
            var expectedResponse = new LoginResponse
            {
                Description = "Hello-World"
            };

            helperFixture.MockServerFixture.GetApiFixture.Execute(
                path: "/api/v1/validate",
                statusCode: HttpStatusCode.OK,
                response: new ValidateResponse("Hello-World"));

            // Act
            var httpResponseMessage = await helperFixture.Client.GetAsync("/v1/login");

            // Assert
            var response = await httpResponseMessage.DeserializeAsync<LoginResponse>();
            response.Should().NotBeNull();
            response.Should().BeEquivalentTo(expectedResponse);
        }
    }
}
```

## Facilidade e Eficiência
O código apresentado facilita a criação de testes integrados ao fornecer uma estrutura reutilizável que pode ser aplicada a diferentes cenários de teste. Com a utilização de ferramentas como WireMock, é possível simular diversos comportamentos de APIs externas, eliminando a necessidade de ambientes de teste complexos ou de dependência de serviços de terceiros.

- **Reutilização**: A estrutura modular permite que diferentes partes do código sejam reutilizadas em vários testes, economizando tempo e esforço no desenvolvimento de novos cenários de teste.

- **Simulação Completa**: Com WireMock, é possível criar simulações completas de respostas de serviços externos, o que é essencial para testar como a aplicação se comporta sob diferentes condições.

- **Validações Fluent**: O uso de FluentAssertions torna as asserções nos testes mais claras e legíveis, facilitando a compreensão dos resultados esperados e reais.

## Conclusão
Testes integrados são uma parte essencial do desenvolvimento de software robusto e confiável. Em C#, com o suporte de ferramentas como WireMock, AutoFixture, e FluentAssertions, é possível construir uma suite de testes que cobre cenários complexos de integração, garantindo que a aplicação funcione corretamente em todas as situações. A estrutura apresentada oferece uma base sólida para implementar testes de integração de forma eficiente e escalável.