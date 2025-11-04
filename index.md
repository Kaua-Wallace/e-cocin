# Documentação do Sistema e-cocin

Este documento descreve o fluxo geral do sistema e-cocin e detalha a aplicação dos fundamentos da Orientação a Objetos (OOP) no código-fonte.

## 1. Fluxo do Sistema

O sistema e-cocin é uma aplicação web construída com o framework Oat++ para C++. Ele segue uma arquitetura em camadas, com Controladores, Serviços e Repositórios, interagindo com um banco de dados SQLite. A arquitetura é projetada para ser modular e extensível, com uma clara separação de responsabilidades entre as camadas.

### 1.1. Inicialização da Aplicação (`ECocinApplication.cpp`)

O ponto de entrada da aplicação é o arquivo `ECocinApplication.cpp`. Aqui, as seguintes etapas são realizadas:

1.  **Migrações do Banco de Dados**: O banco de dados SQLite (`e-cocin.db`) é inicializado e as migrações são executadas para garantir que o esquema do banco de dados esteja atualizado.
2.  **Inicialização de Repositórios e Serviços**: Para cada entidade principal (Cliente, Produto, Endereço, Pedido), uma instância de repositório (e.g., `ClientRepositorySqlite`) e uma instância de serviço (e.g., `ClientService`) são criadas. Os serviços recebem os repositórios como dependências.
3.  **Configuração do Servidor Web (Oat++)**:
    *   O ambiente Oat++ é inicializado.
    *   Um `ObjectMapper` (para serialização/desserialização JSON) e um `HttpRouter` são criados.
    *   Controladores (e.g., `ClientController`) são instanciados, recebendo o `ObjectMapper` e o serviço correspondente como dependências.
    *   Os controladores são adicionados ao `HttpRouter`, mapeando as rotas da API para os métodos dos controladores.
    *   Um `HttpConnectionHandler` é configurado com o roteador.
    *   Um `ConnectionProvider` é criado para escutar requisições HTTP na porta 8000.
    *   O servidor Oat++ é iniciado, aguardando requisições.

### 1.2. Processamento de Requisições

Quando uma requisição HTTP chega ao servidor:

1.  **Mapeamento de Rotas**: O Oat++ mapeia a URL da requisição para um método específico em um dos controladores. Por exemplo, uma requisição `POST /orders` é direcionada para o método `createOrder` no `OrderController`.
2.  **Controlador**: O `HttpRouter` direciona a requisição para o método apropriado no `Controller` (e.g., `ClientController::createClient`). O controlador é responsável por:
    *   Receber os dados da requisição (geralmente em formato DTO - Data Transfer Object).
    *   Validar os dados de entrada.
    *   Chamar o método correspondente no `Service` para executar a lógica de negócio.
    *   Formatar a resposta e enviá-la de volta ao cliente.
3.  **Serviço**: O `Service` (e.g., `ClientService`) contém a lógica de negócio da aplicação. Ele é responsável por:
    *   Coordenar operações que podem envolver múltiplas entidades ou regras de negócio complexas.
    *   Interagir com um ou mais `Repository` para persistir ou recuperar dados.
    *   Aplicar validações de negócio adicionais.
4.  **Repositório**: O `Repository` (e.g., `ClientRepositorySqlite`) é responsável pela interação com a camada de persistência (neste caso, o banco de dados SQLite). Ele:
    *   Implementa a interface do repositório (`IClientRepository`).
    *   Traduz objetos de domínio para o formato do banco de dados e vice-versa.
    *   Executa operações CRUD (Create, Read, Update, Delete) no banco de dados.
5.  **Entidades de Domínio**: As entidades (e.g., `Client`, `Product`, `Order`) representam os objetos de negócio do sistema. Elas contêm os dados e a lógica de negócio intrínseca a esses objetos.

## 2. Fundamentos da Orientação a Objetos (OOP)

O projeto e-cocin faz uso extensivo dos princípios da Orientação a Objetos para organizar e estruturar o código. A seguir, detalhamos como cada princípio é aplicado, com exemplos adicionais.

### 2.1. Encapsulamento

O encapsulamento é o princípio de agrupar dados (atributos) e métodos (funções) que operam nesses dados em uma única unidade, a classe, e restringir o acesso direto a alguns dos componentes do objeto. No e-cocin, as classes de entidade no diretório `src/domain/entities/` são excelentes exemplos:

*   **Atributos Privados**: Os membros de dados (e.g., `id_`, `name_`, `email_` na classe `Client`) são declarados como `private`, impedindo o acesso direto de fora da classe.
*   **Métodos Públicos (Getters e Setters)**: O acesso e a modificação desses atributos são feitos exclusivamente através de métodos públicos (`public`) como `getId()`, `setName()`, `getEmail()`, etc. Isso permite controlar como os dados são acessados e modificados, garantindo a integridade do objeto.

**Exemplo (`src/domain/entities/Client.h`):**

```cpp
class Client {
private:
    long long id_;
    std::string name_;
    std::string email_;
    std::string cpf_;
    ecocin::core::Timestamp createDate_;

public:
    // Getters
    long long                 getId() const {return id_;}
    const std::string&        getName() const {return name_;}
    const std::string&        getEmail() const {return email_;}
    const std::string&        getCpf()   const {return cpf_;}

    // Setters
    void setId(long long id)                            {id_ = id;}
    void setName(const std::string& name)              { name_ = name; }
    void setEmail(const std::string& email)            { email_ = email; }
    void setCpf(const std::string& cpf)                { cpf_ = cpf; }
    void setCreateDate(ecocin::core::Timestamp createDate) { createDate_ = createDate; }
    // ... outros métodos
};
```

Outro exemplo pode ser visto na classe `Order`, onde a lógica de cálculo do total do pedido é encapsulada dentro da própria classe, garantindo que o total seja sempre consistente com a quantidade e o preço unitário.

**Exemplo (`src/domain/entities/Order.cpp`):**

```cpp
void Order::calculateTotal() {
    totalPrice_ = quantity_ * unitPrice_;
}
```

### 2.2. Construtores

Construtores são métodos especiais de uma classe que são chamados automaticamente quando um objeto dessa classe é criado. Eles são usados para inicializar o estado do objeto. As classes de entidade no e-cocin possuem múltiplos construtores para diferentes cenários de inicialização.

**Exemplo (`src/domain/entities/Client.cpp` e `src/domain/entities/Client.h`):**

```cpp
// Construtor padrão
Client::Client() : id_(-1), name_(""), email_(""), cpf_(""), createDate_(ecocin::core::Time::get  CurrentTimestamp()) {}

// Construtor com parâmetros para criação de um novo cliente
Client::Client(const std::string& name,
               const std::string& email,
               const std::string& cpf)
    : id_(-1), name_(trim_copy(name)), email_(trim_copy(email)), cpf_(trim_copy(cpf)),
      createDate_(ecocin::core::Time::getCurrentTimestamp()) {}

// Construtor completo para reconstruir um cliente do banco de dados
Client::Client(long long id,
               const std::string& name,
               const std::string& email,
               const std::string& cpf,
               ecocin::core::Timestamp createDate)
    : id_(id), name_(name), email_(email), cpf_(cpf), createDate_(createDate) {}
```
Estes construtores permitem criar objetos `Client` de diferentes maneiras, seja um objeto vazio, um novo objeto com dados iniciais, ou um objeto totalmente preenchido a partir de dados existentes.

A classe `OrderService` também demonstra o uso de construtores para injeção de dependência, recebendo referências para os repositórios necessários para sua operação.

**Exemplo (`src/services/OrderService.cpp`):**

```cpp
OrderService::OrderService(
  infra::repositories::sqlite::OrderRepositorySqlite& orderRepo,
  infra::repositories::sqlite::ClientRepositorySqlite& clientRepo,
  infra::repositories::sqlite::ProductRepositorySqlite& productRepo,
  infra::repositories::sqlite::AddressRepositorySqlite& addressRepo)
  : orderRepo_(orderRepo)
  , clientRepo_(clientRepo)
  , productRepo_(productRepo)
  , addressRepo_(addressRepo) {}
```

### 2.3. Herança

A herança é um mecanismo que permite que uma classe (subclasse ou classe derivada) herde propriedades e comportamentos de outra classe (superclasse ou classe base). No projeto e-cocin, a herança é utilizada de forma mais sutil através de interfaces (classes abstratas puras) para definir contratos, o que é uma forma de "herança de interface".

**Exemplo de Herança de Interface (`src/domain/repositories/IClientRepository.h`):**

```cpp
namespace ecocin::domain::repositories {

class IClientRepository {
public:
    virtual ~IClientRepository() = default; // Destrutor virtual

    virtual void create(Client& client) = 0;
    virtual std::shared_ptr<Client> findById(long long id) = 0;
    virtual std::shared_ptr<Client> findByEmail(const std::string& email) = 0;
    virtual std::vector<std::shared_ptr<Client>> findAll() = 0;
    virtual void update(const Client& client) = 0;
    virtual void remove(long long id) = 0;
};

} // namespace ecocin::domain::repositories
```

A classe `IClientRepository` é uma interface (uma classe abstrata com métodos puramente virtuais). Classes concretas, como `ClientRepositorySqlite`, herdam desta interface e são obrigadas a implementar todos os métodos virtuais puros.

**Exemplo de Implementação (`src/infra/repositories/sqlite/ClientRepositorySqlite.h`):**

```cpp
namespace ecocin::infra::repositories::sqlite {

class ClientRepositorySqlite : public ecocin::domain::repositories::IClientRepository {
private:
    ecocin::infra::db::SqliteConnection& connection_;

public:
    ClientRepositorySqlite(ecocin::infra::db::SqliteConnection& connection);

    void create(ecocin::domain::entities::Client& client) override;
    std::shared_ptr<ecocin::domain::entities::Client> findById(long long id) override;
    std::shared_ptr<ecocin::domain::entities::Client> findByEmail(const std::string& email) override;
    std::vector<std::shared_ptr<ecocin::domain::entities::Client>> findAll() override;
    void update(const ecocin::domain::entities::Client& client) override;
    void remove(long long id) override;
};

} // namespace ecocin::infra::repositories::sqlite
```
Aqui, `ClientRepositorySqlite` herda publicamente de `IClientRepository`, comprometendo-se a fornecer implementações para todos os métodos definidos na interface.

Além dos repositórios, a herança também é utilizada no framework Oat++ para a criação de controladores. A classe `OrderController` herda de `oatpp::web::server::api::ApiController`, o que permite que ela se integre ao sistema de roteamento do Oat++.

**Exemplo (`src/controllers/OrderController.h`):**

```cpp
class OrderController : public oatpp::web::server::api::ApiController {
    // ...
};
```

### 2.4. Polimorfismo

Polimorfismo significa "muitas formas" e permite que objetos de diferentes classes sejam tratados como objetos de uma classe comum (geralmente uma classe base ou interface). No e-cocin, o polimorfismo é fundamental para a arquitetura baseada em interfaces dos repositórios.

*   **Interfaces como Contratos**: As interfaces `IClientRepository`, `IProductRepository`, etc., definem um contrato para as operações de persistência.
*   **Injeção de Dependência**: Os serviços (e.g., `ClientService`) recebem uma referência ou ponteiro para a interface do repositório em seu construtor. Isso significa que o `ClientService` não se importa com a implementação concreta do repositório (se é SQLite, PostgreSQL, ou outro), apenas que ele adere ao contrato da interface.

**Exemplo (`src/services/ClientService.h`):**

```cpp
namespace ecocin::services {

class ClientService {
private:
    ecocin::domain::repositories::IClientRepository& clientRepository_;

public:
    ClientService(ecocin::domain::repositories::IClientRepository& clientRepository);

    // Métodos de serviço que usam clientRepository_
    void createClient(ecocin::controllers::dto::ClientDto& clientDto);
    std::shared_ptr<ecocin::controllers::dto::ClientOutDto> getClientById(long long id);
    // ...
};

} // namespace ecocin::services
```

No `ECocinApplication.cpp`, quando o `ClientService` é instanciado, ele recebe uma instância de `ClientRepositorySqlite` (que é um `IClientRepository`):

```cpp
auto clientRepo    = std::make_shared<ecocin::infra::repositories::sqlite::ClientRepositorySqlite>(cx);
auto clientService = std::make_shared<ecocin::services::ClientService>(*clientRepo);
```

Isso demonstra polimorfismo: o `ClientService` interage com `*clientRepo` através da interface `IClientRepository`, sem saber que a implementação subjacente é `ClientRepositorySqlite`. Se no futuro a aplicação precisar usar um banco de dados diferente, bastaria criar uma nova implementação de `IClientRepository` (e.g., `ClientRepositoryPostgres`) e injetá-la no `ClientService` sem modificar a lógica de negócio do serviço.

O polimorfismo também é evidente na forma como o `OrderService` utiliza diferentes repositórios para orquestrar a criação de um pedido. O serviço interage com `IClientRepository`, `IProductRepository` e `IAddressRepository` através de suas interfaces, sem conhecer os detalhes de implementação de cada um.

### 2.5. Abstração

A abstração foca em mostrar apenas as informações essenciais e esconder os detalhes de implementação. No e-cocin, isso é evidente em várias camadas:

*   **Interfaces de Repositório**: As interfaces `IClientRepository`, `IProductRepository`, etc., são abstrações que definem "o que" um repositório deve fazer (criar, encontrar, atualizar, remover) sem especificar "como" ele faz isso. Os detalhes de como a persistência é realizada (e.g., comandos SQL, ORM) são abstraídos para as classes de implementação concretas.
*   **DTOs (Data Transfer Objects)**: As classes DTO (e.g., `ClientDto`, `ClientOutDto`) no diretório `src/controllers/dto/` são abstrações dos dados que são transferidos entre as camadas da aplicação (Controlador, Serviço). Elas representam a forma dos dados para requisições e respostas da API, abstraindo os detalhes internos das entidades de domínio.

*   **Serviços**: A camada de serviço abstrai a lógica de negócio complexa. Por exemplo, o método `createByCpfSkuAndType` no `OrderService` abstrai toda a complexidade de encontrar um cliente por CPF, um produto por SKU, resolver o endereço de entrega e, finalmente, criar um pedido. O controlador que chama este método não precisa saber como essas operações são realizadas.

---

## 3. Uso de Ponteiros Inteligentes e `std::optional`

O projeto utiliza ponteiros inteligentes (`std::shared_ptr`) e `std::optional` para gerenciar a memória e representar a ausência de valor de forma segura e expressiva.

### 3.1. `std::shared_ptr` para Gerenciamento de Dependências

`std::shared_ptr` é um ponteiro inteligente que gerencia a vida útil de um objeto através de contagem de referências. No e-cocin, ele é usado principalmente para a injeção de dependências entre as camadas da aplicação.

*   **Onde é usado?**
    *   **`ECocinApplication.cpp`**: Ao criar instâncias de repositórios e serviços, `std::make_shared` é utilizado para alocar esses objetos no heap e encapsulá-los em `std::shared_ptr`.
    *   **Controladores**: Os controladores (e.g., `ClientController`, `OrderController`) recebem e armazenam `std::shared_ptr` para os seus respectivos serviços.

**Exemplo (`ECocinApplication.cpp`):**

```cpp
// Client Repo + Service
auto clientRepo    = std::make_shared<ecocin::infra::repositories::sqlite::ClientRepositorySqlite>(cx);
auto clientService = std::make_shared<ecocin::services::ClientService>(*clientRepo);

// ...

// O controlador recebe o serviço como um std::shared_ptr
auto controller = std::make_shared<ClientController>(objectMapper, clientService);
```

### 3.2. `std::optional` para Valores Opcionais

`std::optional` é um tipo que pode conter um valor ou estar vazio. Ele é usado para representar de forma explícita a possibilidade de ausência de um valor, sem recorrer a ponteiros nulos ou valores mágicos.

*   **Onde é usado?**
    *   **Retornos de Repositórios**: Métodos como `findById`, `findByCpf`, `findBySku` nos repositórios retornam `std::optional<Entity>`. Se a entidade for encontrada no banco de dados, o `std::optional` conterá o objeto; caso contrário, estará vazio.
    *   **Retornos de Serviços**: Os serviços propagam esse padrão, retornando `std::optional` para os controladores quando uma operação de busca pode não encontrar um resultado.


**Exemplo (`src/domain/repositories/IClientRepository.h`):**

```cpp
class IClientRepository {
public:
    // ...
    virtual std::optional<Client> findById(long long id) = 0;
    virtual std::optional<Client> findByCpf(const std::string& cpf) = 0;
    // ...
};
```

**Exemplo de Uso no Serviço (`src/services/ClientService.cpp`):**

```cpp
std::optional<Client> ClientService::getClientByCpf(const std::string& cpf) {
    return clientRepository_.findByCpf(cpf);
}
```
## Divisão do Projeto

* Kauã Wallace Silva Melo: responsáve de `controllers` ; relatório e GitHubPages

* KLismman Luan Cabral Silva: responsável por `entities`; apresentação e roteiro do vídeo

* Luiz Ribeiro da Silva Neto: responsável por `services`; relatório e GitHubPages

* Raphael Alves da Silva: responsável por `repositories` e banco de dados



