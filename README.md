# Pedidos API

API Mule 4 de pedidos e itens. Roda na porta **8082**.

Parte do conjunto Day Zero, tres APIs que se encaixam num fluxo de negocio:
[clientes-api](https://github.com/igordiascardoso/clientes-api) (8081) ->
[pedidos-api](https://github.com/igordiascardoso/pedidos-api) (8082) ->
[estoque-api](https://github.com/igordiascardoso/estoque-api) (8083).

## Healthcheck

```
GET http://localhost:8082/api/health
```

Nao devolve um `UP` fixo: confere a **integridade pedido<->itens** - nenhum pedido
sem item, nenhum item orfao (sem SKU ou com quantidade <= 0). Reporta as duas
contagens; qualquer uma diferente de zero derruba para `DEGRADED`.

```json
{
  "status": "UP",
  "api": "pedidos-api",
  "versao": "1.0.0",
  "porta": 8082,
  "detalhes": {
    "storePedidos": "UP",
    "pedidosCarregados": 1,
    "itensOrfaos": 0,
    "pedidosSemItem": 0,
    "dependenciaClientesApi": "NAO_VERIFICADO"
  }
}
```

## Endpoints

| Metodo | Caminho | O que faz |
|---|---|---|
| GET | /api/health | healthcheck (acima) |
| GET | /api/pedidos | lista, com filtro por situacao e cliente |
| POST | /api/pedidos | abre um pedido em RASCUNHO |
| GET | /api/pedidos/{pedidoId} | detalha com os itens |
| GET | /api/pedidos/{pedidoId}/itens | lista os itens |
| POST | /api/pedidos/{pedidoId}/itens | acrescenta item (so em RASCUNHO) |
| PUT | /api/pedidos/{pedidoId}/confirmacao | confirma e reserva estoque |
| PUT | /api/pedidos/{pedidoId}/cancelamento | cancela |

## Estrutura do RAML

O layout e o que o Anypoint Studio e o Design Center geram - spec quebrada por
responsabilidade, nao um arquivo monolitico:

```
src/main/resources/api/
  pedidos-api.raml
  exchange.json     aponta o main + o GAV (groupId/assetId/version)
  types/            um DataType por arquivo
  traits/           libraries e traits reutilizaveis (paginacao)
  examples/         exemplos JSON referenciados por !include
```

Os flows APIKit em [src/main/mule/pedidos-api.xml](src/main/mule/pedidos-api.xml) foram gerados a
partir do RAML pelo scaffold headless do `ponte`.

## Rodar

```
mvn package
```

Ou pela extensao, sem abrir o Studio:

```
ponte scaffold repo    # gera os flows que faltam a partir do RAML
ponte runrepo          # sobe a API
ponte killrun          # encerra
```

A porta sai de [src/main/resources/config-dev.properties](src/main/resources/config-dev.properties)
via `${http.port}`, entao trocar de porta nao exige mexer no XML.

## O RAML vem do Exchange

O `pom.xml` consome o RAML como dependencia Maven do Exchange (`classifier: raml`), e o
`apikit:config` referencia o asset por GAV — o padrao de um projeto real vindo do Design
Center:

```xml
<dependency>
  <groupId>1b88a96a-3c95-4a05-b047-79e7a13b7a96</groupId>
  <artifactId>pedidos-api</artifactId>
  <version>1.0.0</version>
  <classifier>raml</classifier>
  <type>zip</type>
</dependency>
```

**Isso exige credencial Maven do Exchange** no seu `settings.xml`. Sem ela o `mvn package`
falha com `401 Unauthorized` ao resolver o RAML — o login do `anypoint-cli` nao serve aqui,
sao credenciais separadas.

Para editar o RAML localmente sem depender do Exchange a cada build, use o apontamento da
extensao:

```
ponte pararepo raml-local    # pom.xml e apikit:config passam a ler a sua pasta
ponte pararepo raml-undo     # volta a depender do Exchange (antes de commitar)
```

## Versoes

Validadas contra o Mule Runtime 4.12.2 / Java 17:

| Peca | Versao |
|---|---|
| Mule Runtime | 4.12.2 |
| `mule-maven-plugin` | 4.10.1 |
| `mule-http-connector` | 1.11.0 |
| `mule-apikit-module` | 1.11.7 |
