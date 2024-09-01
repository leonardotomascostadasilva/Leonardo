---
title: "Testando Endpoints de API com K6: Configuração e Execução de Testes de Carga"
tags:
  - PoCs
---

Quando desenvolvemos APIs, garantir que elas possam lidar com uma carga significativa é crucial. Para isso, o K6 é uma ferramenta poderosa que ajuda a realizar testes de carga e performance em nossos endpoints. Neste exemplo, vamos configurar um teste de carga para uma API utilizando K6, explorando como definir a taxa de chegada de requisições, enviar payloads e gerar relatórios detalhados.

## Estrutura do Teste de Carga com K6
O script a seguir configura um teste de carga que simula requisições constantes para um endpoint da API. Vamos analisar cada parte do código e entender como ele contribui para a realização do teste.

## Configuração do Teste

O primeiro passo é configurar as opções do teste, que incluem a taxa de chegada de requisições, a duração do teste e o gerenciamento de VUs (Virtual Users):

```js
import http from 'k6/http';
import { check, fail } from 'k6';
import { sleep } from 'k6';
import { htmlReport } from "https://raw.githubusercontent.com/benc-uk/k6-reporter/main/dist/bundle.js";
import { textSummary } from "https://jslib.k6.io/k6-summary/0.0.1/index.js";

export const options = {
    scenarios: {
        main: {
            executor: 'constant-arrival-rate',
            duration: __ENV.duration,
            rate: __ENV.tps,
            timeUnit: '1s',
            preAllocatedVUs: __ENV.tps * 15,
            maxVUs: __ENV.tps * 30
        }
    }
}
```
Neste código, a configuração define um cenário de teste onde:

- **executor**: `'constant-arrival-rate'`: Mantém uma taxa constante de chegada de requisições.
- **duration**: Duração total do teste, definida como uma variável de ambiente.
- **rate**: Taxa de requisições por segundo, também configurada como variável de ambiente.
- **timeUnit**: Unidade de tempo para a taxa de chegada, que é definida como segundos.
- **preAllocatedVUs**: Número de VUs pré-alocados para o teste, determinado pela taxa de requisições.
- **maxVUs**: Número máximo de VUs que podem ser utilizados durante o teste.

## Função Principal do Teste
A função `default` é a função principal que será executada para cada VU:

```js
export default function(){
    const url = "https://localhost:7122/api/v1/sample";

    const params = {
        headers: {
            'Content-Type': 'application/json',
        },
    };

    const payload = JSON.stringify({
        "description": "test"
    });

    var response = http.post(url, payload, params);

    if (!check(response, {
        "[POST] is status created.": (r) => r.status === 200
    })){
        fail(response.status);
    }

    sleep(1);
}
```
Aqui, a função realiza o seguinte:

- **URL e Headers**: Define a URL do endpoint e os cabeçalhos da requisição.
- **Payload**: Cria um payload JSON com uma descrição para enviar no corpo da requisição.
- **Requisição POST**: Envia uma requisição POST para o endpoint especificado.
- **Validação**: Verifica se a resposta tem o status 200 OK. Se não, o teste falha.
- **Pausa**: Inclui uma pausa de 1 segundo entre as requisições para simular um intervalo realista.


## Gerando Relatórios
Por fim, o script configura a geração de relatórios para analisar os resultados do teste:

```js
export function handleSummary(data) {
    return {
      "post-result.html": htmlReport(data),
      stdout: textSummary(data, { indent: " ", enableColors: true }),
    };
}
```
A função handleSummary gera dois tipos de relatórios:

- **HTML Report**: Um relatório visual em HTML que pode ser visualizado em um navegador.
- **Text Summary**: Um resumo textual dos resultados, útil para análise rápida no terminal.

## Conclusão
O uso de K6 para realizar testes de carga permite simular diferentes cenários e medir o desempenho de sua API sob condições variadas. A configuração detalhada do teste ajuda a garantir que a API possa lidar com a carga esperada e identificar possíveis pontos de estrangulamento. Com os relatórios gerados, é possível obter insights valiosos para otimizar e escalar a API de maneira eficaz.