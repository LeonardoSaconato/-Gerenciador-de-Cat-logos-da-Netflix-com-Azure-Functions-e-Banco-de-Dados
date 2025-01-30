
# Netflix Catalog Manager

Este projeto utiliza Azure Functions e Cosmos DB para criar, listar, buscar, atualizar e excluir filmes e séries de um catálogo. A solução oferece um backend serverless escalável e econômico para gerenciar o catálogo de filmes e séries.


## Estrutura de Diretórios Projeto:

netflix-catalog-manager/
├── .gitignore
├── function-app/
│   ├── CatalogManagerFunctionApp.csproj
│   ├── Functions/
│   │   ├── CreateCatalogFunction.cs
│   │   ├── GetCatalogFunction.cs
│   │   ├── UpdateCatalogFunction.cs
│   │   ├── DeleteCatalogFunction.cs
│   │   └── SearchByGenreFunction.cs
├── host.json
├── local.settings.json
└── README.md


## Passos para Configuração

### Passo 1: Instalar o Azure Functions Tools

Se você ainda não tem o Azure Functions Tools instalado, execute o seguinte comando:

```bash
npm install -g azure-functions-core-tools@4 --unsafe-perm true
Passo 2: Criar um Novo Azure Function App
Você pode criar um novo Function App utilizando o Visual Studio, Visual Studio Code ou a linha de comando. Para criar pelo terminal, siga os passos abaixo:

## Crie o Function App:
bash

func init netflix-catalog-manager --worker-runtime dotnet
Navegue até o diretório criado:
bash
Copiar
Editar
cd netflix-catalog-manager
Passo 3: Criar o Banco de Dados - Azure Cosmos DB
No portal do Azure, crie uma instância do Azure Cosmos DB utilizando a API Core (SQL). Esta API é ideal para dados JSON e NoSQL.

Crie o Container chamado Catalogs, utilizando genre como chave de particionamento.

No portal do Azure, obtenha a connection string do Cosmos DB. Ela será usada para conectar ao Cosmos DB no código. Guarde a string de conexão e as credenciais.

Passo 4: Configuração do local.settings.json
O arquivo local.settings.json é necessário para armazenar as configurações de ambiente durante o desenvolvimento local. Configure o arquivo com a chave de conexão do Cosmos DB, conforme o exemplo abaixo:

json

{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "CosmosDBConnectionString": "<YOUR_COSMOS_DB_CONNECTION_STRING>"
  }
}

Passo 5: Criar as Funções
O projeto possui várias funções para manipular o catálogo de filmes e séries. Essas funções estão localizadas na pasta Functions/ dentro de function-app/.

5.1: Função de Cadastro (Create)
Crie a função CreateCatalogFunction.cs para adicionar filmes/séries ao banco de dados:

C#

using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System.Net.Http;
using System.Threading.Tasks;

public static class CreateCatalogFunction
{
    [FunctionName("CreateCatalog")]
    public static async Task<HttpResponseMessage> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestMessage req,
        [CosmosDB(
            databaseName: "NetflixCatalog",
            collectionName: "Catalogs",
            ConnectionStringSetting = "CosmosDBConnectionString")] IAsyncCollector<dynamic> catalogItems,
        ILogger log)
    {
        var requestBody = await req.Content.ReadAsStringAsync();
        var movie = JsonConvert.DeserializeObject<Movie>(requestBody);

        // Adicionando o item no Cosmos DB
        await catalogItems.AddAsync(movie);
        
        log.LogInformation($"Filme {movie.Title} adicionado ao catálogo.");
        
        return new HttpResponseMessage(HttpStatusCode.Created);
    }
}

public class Movie
{
    public string Id { get; set; }
    public string Title { get; set; }
    public string Genre { get; set; }
    public string Description { get; set; }
    public string ReleaseDate { get; set; }
}

5.2: Função de Listagem (Get)
Crie a função GetCatalogFunction.cs para listar todos os filmes/séries:

C#

using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System.Collections.Generic;
using System.Net.Http;
using System.Threading.Tasks;

public static class GetCatalogFunction
{
    [FunctionName("GetCatalog")]
    public static async Task<HttpResponseMessage> Run(
        [HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequestMessage req,
        [CosmosDB(
            databaseName: "NetflixCatalog",
            collectionName: "Catalogs",
            ConnectionStringSetting = "CosmosDBConnectionString")] IEnumerable<Movie> movies,
        ILogger log)
    {
        log.LogInformation("Consultando catálogo.");

        var result = JsonConvert.SerializeObject(movies);
        return new HttpResponseMessage(HttpStatusCode.OK)
        {
            Content = new StringContent(result, System.Text.Encoding.UTF8, "application/json")
        };
    }
}

5.3: Função de Busca por Gênero (Search)
Crie a função SearchByGenreFunction.cs para buscar filmes/séries por gênero:

C#

using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System.Linq;
using System.Net.Http;
using System.Threading.Tasks;

public static class SearchByGenreFunction
{
    [FunctionName("SearchByGenre")]
    public static async Task<HttpResponseMessage> Run(
        [HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequestMessage req,
        [CosmosDB(
            databaseName: "NetflixCatalog",
            collectionName: "Catalogs",
            ConnectionStringSetting = "CosmosDBConnectionString")] IEnumerable<Movie> movies,
        ILogger log)
    {
        var genre = req.GetQueryNameValuePairs()
            .FirstOrDefault(q => string.Compare(q.Key, "genre", true) == 0)
            .Value;

        var filteredMovies = movies.Where(m => m.Genre.ToLower() == genre.ToLower()).ToList();
        var result = JsonConvert.SerializeObject(filteredMovies);

        log.LogInformation($"Consultando filmes do gênero {genre}.");

        return new HttpResponseMessage(HttpStatusCode.OK)
        {
            Content = new StringContent(result, System.Text.Encoding.UTF8, "application/json")
        };
    }
}

5.4: Função de Atualização (Update)
Crie a função UpdateCatalogFunction.cs para atualizar um filme/série:

C#

using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System.Net.Http;
using System.Threading.Tasks;

public static class UpdateCatalogFunction
{
    [FunctionName("UpdateCatalog")]
    public static async Task<HttpResponseMessage> Run(
        [HttpTrigger(AuthorizationLevel.Function, "put")] HttpRequestMessage req,
        [CosmosDB(
            databaseName: "NetflixCatalog",
            collectionName: "Catalogs",
            ConnectionStringSetting = "CosmosDBConnectionString")] IAsyncCollector<Movie> catalogItems,
        ILogger log)
    {
        var requestBody = await req.Content.ReadAsStringAsync();
        var updatedMovie = JsonConvert.DeserializeObject<Movie>(requestBody);

        await catalogItems.AddAsync(updatedMovie);

        log.LogInformation($"Filme {updatedMovie.Title} atualizado.");

        return new HttpResponseMessage(HttpStatusCode.OK);
    }
}

5.5: Função de Exclusão (Delete)
Crie a função DeleteCatalogFunction.cs para excluir um filme/série:

C#

using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using System.Net.Http;
using System.Threading.Tasks;

public static class DeleteCatalogFunction
{
    [FunctionName("DeleteCatalog")]
    public static async Task<HttpResponseMessage> Run(
        [HttpTrigger(AuthorizationLevel.Function, "delete")] HttpRequestMessage req,
        [CosmosDB(
            databaseName: "NetflixCatalog",
            collectionName: "Catalogs",
            ConnectionStringSetting = "CosmosDBConnectionString")] IAsyncCollector<Movie> catalogItems,
        ILogger log)
    {
        var requestBody = await req.Content.ReadAsStringAsync();
        var movie = JsonConvert.DeserializeObject<Movie>(requestBody);

        // Lógica para excluir filme (pode ser uma chamada CosmosDB.DeleteAsync ou similar)
        log.LogInformation($"Filme {movie.Title} excluído.");

        return new HttpResponseMessage(HttpStatusCode.NoContent);
    }
}

Passo 6: Testar Localmente
Execute o comando abaixo para testar localmente:

bash

func start
Passo 7: Fazer o Deploy para o Azure
Depois de testar localmente, você pode fazer o deploy da sua aplicação para o Azure:

bash

az functionapp deployment source config-zip --src <path-to-your-zip-file> --name <function-app-name> --resource-group <resource-group-name>


## Conclusão
Com isso, você terá um backend serverless utilizando Azure Functions e Cosmos DB para gerenciar o catálogo de filmes e séries. A solução oferece alta escalabilidade e baixo custo, já que você só paga pelo tempo de execução das funções.
Esse projeto pode ser facilmente expandido com novas funcionalidades, como autenticação de usuários ou recomendações de filmes com base no histórico de visualizações.
