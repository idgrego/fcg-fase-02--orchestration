# Desenvolvimento .NET no VS Code: Ambiente, Extensões e Criação de Projetos

Bem-vindo ao primeiro módulo do guia de **Desenvolvimento .NET no Visual Studio Code** do ecossistema **FIAP Cloud Games**. Este documento ensina como preparar um ambiente profissional de desenvolvimento C# / .NET 10 no VS Code, quais extensões oficiais da Microsoft instalar, como utilizar o novo formato de solução `.slnx` e como dominar a CLI do .NET para criar e pesquisar templates de projetos.

---

## 📑 Sumário

- [1. Por que Desenvolver em .NET no Visual Studio Code?](#1-por-que-desenvolver-em-net-no-visual-studio-code)
- [2. Configuração do Ambiente e Extensões Essenciais](#2-configuração-do-ambiente-e-extensões-essenciais)
  - [2.1. Verificação do .NET 10 SDK](#21-verificação-do-net-10-sdk)
  - [2.2. Extensões Oficiais da Microsoft](#22-extensões-oficiais-da-microsoft)
  - [2.3. Extensões Complementares de Alta Produtividade](#23-extensões-complementares-de-alta-produtividade)
- [3. Gestão de Soluções (.sln vs. .slnx Moderno)](#3-gestão-de-soluções-sln-vs-slnx-moderno)
  - [3.1. O Formato Clássico `.sln`](#31-o-formato-clássico-sln)
  - [3.2. O Novo Formato `.slnx` do .NET 10](#32-o-novo-formato-slnx-do-net-10)
- [4. Criação de Projetos por Template na .NET CLI](#4-criação-de-projetos-por-template-na-net-cli)
  - [4.1. Aplicativo de Console (`console`)](#41-aplicativo-de-console-console)
  - [4.2. Web API e Minimal APIs (`webapi`)](#42-web-api-e-minimal-apis-webapi)
  - [4.3. Biblioteca de Classes (`classlib`)](#43-biblioteca-de-classes-classlib)
  - [4.4. Serviço em Segundo Plano (`worker`)](#44-serviço-em-segundo-plano-worker)
  - [4.5. Testes Unitários (`xunit`, `nunit`, `mstest`)](#45-testes-unitários-xunit-nunit-mstest)
- [5. Pesquisa e Instalação de Novos Templates](#5-pesquisa-e-instalação-de-novos-templates)
  - [5.1. Listar Templates Locais (`dotnet new list`)](#51-listar-templates-locais-dotnet-new-list)
  - [5.2. Pesquisar Templates Online no NuGet (`dotnet new search`)](#52-pesquisar-templates-online-no-nuget-dotnet-new-search)
  - [5.3. Instalar e Desinstalar Templates da Comunidade](#53-instalar-e-desinstalar-templates-da-comunidade)
- [6. Navegação](#6-navegação)

---

## 1. Por que Desenvolver em .NET no Visual Studio Code?

Historicamente, o desenvolvimento .NET esteve associado ao Visual Studio tradicional (IDE pesada, exclusiva para Windows). Hoje, o **Visual Studio Code (VS Code)** é uma das ferramentas mais populares para engenharia de software corporativa em .NET:
- **Ultraleve e Rápido:** Inicialização em segundos e baixo consumo de memória RAM;
- **Multiplataforma:** O mesmo fluxo de trabalho funciona identicamente no Windows, Linux e macOS;
- **Integração com Docker e Kubernetes:** Depuração híbrida combinando código local e contêineres;
- **Suporte Oficial da Microsoft:** O compilador Roslyn e o **C# Dev Kit** trazem experiência completa de refatoração, IntelliSense inteligente e gerenciamento de soluções.

---

## 2. Configuração do Ambiente e Extensões Essenciais

### 2.1. Verificação do .NET 10 SDK
Antes de abrir o VS Code, certifique-se de que o **.NET SDK** está instalado e acessível no seu terminal:

```powershell
# Exibe a versão ativa do compilador .NET
dotnet --version

# Exibe detalhes de arquitetura, runtimes e SDKs instalados na máquina
dotnet --info
```

Se o comando retornar `10.0.x` (ou superior), seu ambiente de linha de comando está pronto.

---

### 2.2. Extensões Oficiais da Microsoft
Abra o menu de Extensões do VS Code (`Ctrl + Shift + X`) e instale o pacote oficial:

1. **C# Dev Kit (`ms-dotnettools.csdevkit`):**
   - Traz o **Solution Explorer** para a barra lateral do VS Code (idêntico ao Visual Studio).
   - Gerencia projetos, adição de referências e soluções sem necessidade de comandos manuais.
   - Integração nativa com o **Test Explorer** para rodar e depurar testes xUnit/NUnit com cliques visuais.
2. **C# (`ms-dotnettools.csharp`):**
   - Fornece realce de sintaxe (*Syntax Highlighting*), análise de código em tempo real (Roslyn), autocompletar inteligente (*IntelliSense*), *Go to Definition* (`F12`), *Find All References* (`Shift + F12`) e o depurador oficial `coreclr`.
3. **.NET Install Tool (`ms-dotnettools.vscode-dotnet-runtime`):**
   - Gerencia runtimes internos necessários pelas ferramentas do VS Code.

---

### 2.3. Extensões Complementares de Alta Produtividade

- **REST Client (`humao.rest-client`):** Permite disparar requisições HTTP e testar APIs diretamente de arquivos `.http` salvos no projeto (ex: `FCG02.UsersAPI.API.http`), sem abrir ferramentas externas como Postman.
- **Docker (`ms-azuretools.vscode-docker`):** Facilita a visualização, inspeção de logs e comandos em contêineres e imagens locais ([📖 Ver Guia de Docker](../docker/index.md)).
- **Kubernetes (`ms-kubernetes-tools.vscode-kubernetes-tools`):** Permite visualizar nós, Pods, Services e logs do cluster local ([📖 Ver Guia de Kubernetes](../kubernetes/index.md)).
- **NuGet Gallery (`pcislo.nuget-gallery`):** Interface visual amigável para buscar, atualizar e desinstalar pacotes NuGet nos arquivos `.csproj`.

---

## 3. Gestão de Soluções (.sln vs. .slnx Moderno)

Uma **Solution** é o contêiner raiz que agrupa múltiplos projetos (`.csproj`) interdependentes.

### 3.1. O Formato Clássico `.sln`
Por mais de duas décadas, o .NET utilizou o formato proprietário `.sln` (baseado em texto com sintaxes e GUIDs complexos):

```powershell
# Cria uma solução no formato tradicional:
dotnet new sln -n MinhaSolucao
```

### 3.2. O Novo Formato `.slnx` do .NET 10
No **.NET 9 e 10**, a Microsoft introduziu o formato **`.slnx`**:
- Estruturado em **XML limpo**, legível e sem GUIDs arcaicos;
- Muito mais fácil de resolver conflitos de merge no Git;
- Padrão moderno utilizado em todos os microsserviços do **FIAP Cloud Games** (ex: `FCG02.UsersAPI.slnx`).

```powershell
# Cria uma solução no novo formato moderno .slnx:
dotnet new slnx -n FCG02.UsersAPI
```

#### Exemplo do arquivo `FCG02.UsersAPI.slnx`:
```xml
<Solution>
  <Folder Name="/src/">
    <Project Path="src/FCG02.UsersAPI.Domain/FCG02.UsersAPI.Domain.csproj" />
    <Project Path="src/FCG02.UsersAPI.Application/FCG02.UsersAPI.Application.csproj" />
    <Project Path="src/FCG02.UsersAPI.Infrastructure/FCG02.UsersAPI.Infrastructure.csproj" />
    <Project Path="src/FCG02.UsersAPI.API/FCG02.UsersAPI.API.csproj" />
  </Folder>
  <Folder Name="/tests/">
    <Project Path="tests/FCG02.UsersAPI.UnitTests/FCG02.UsersAPI.UnitTests.csproj" />
  </Folder>
</Solution>
```

---

## 4. Criação de Projetos por Template na .NET CLI

A CLI `dotnet new` utiliza templates padronizados para gerar a estrutura de arquivos e pastas de cada tipo de aplicação.

### 4.1. Aplicativo de Console (`console`)
Ideal para utilitários de terminal, rotinas batch e migrações CLI:

```powershell
# Cria um projeto de console em C#
dotnet new console -n MeuUtilitario -o src/MeuUtilitario
```

### 4.2. Web API e Minimal APIs (`webapi`)
Gera a estrutura de uma API HTTP REST com ASP.NET Core:

```powershell
# Cria uma Web API utilizando Minimal APIs (padrão moderno .NET 10)
dotnet new webapi -n FCG02.UsersAPI.API -o src/FCG02.UsersAPI.API

# Caso queira desativar OpenAPI por padrão (não recomendado):
dotnet new webapi -n MinhaApi -o src/MinhaApi --no-openapi
```

### 4.3. Biblioteca de Classes (`classlib`)
Gera um projeto que compila para uma biblioteca reutilizável de DLLs (sem ponto de entrada `Main`), fundamental para as camadas de **Domain**, **Application** e **Infrastructure** da Clean Architecture:

```powershell
dotnet new classlib -n FCG02.UsersAPI.Domain -o src/FCG02.UsersAPI.Domain
dotnet new classlib -n FCG02.UsersAPI.Application -o src/FCG02.UsersAPI.Application
dotnet new classlib -n FCG02.UsersAPI.Infrastructure -o src/FCG02.UsersAPI.Infrastructure
```

### 4.4. Serviço em Segundo Plano (`worker`)
Gera uma aplicação baseada em `BackgroundService` contínuo, ideal para consumidores assíncronos de RabbitMQ ou mensageria:

```powershell
dotnet new worker -n FCG02.NotificationsWorker -o src/FCG02.NotificationsWorker
```

### 4.5. Testes Unitários (`xunit`, `nunit`, `mstest`)
Gera um projeto de testes automatizados com runners e dependências pré-configuradas:

```powershell
# Cria projeto de testes com xUnit (Padrão corporativo adotado no projeto)
dotnet new xunit -n FCG02.UsersAPI.UnitTests -o tests/FCG02.UsersAPI.UnitTests
```

---

## 5. Pesquisa e Instalação de Novos Templates

O ecossistema .NET possui um sistema extensível de templates que permite instalar aceleradores criados pela comunidade ou por grandes empresas.

### 5.1. Listar Templates Locais (`dotnet new list`)
Exibe todos os modelos instalados atualmente na sua máquina:

```powershell
dotnet new list
```

#### Para filtrar por tipo ou linguagem:
```powershell
# Lista apenas templates relacionados a web
dotnet new list web

# Lista apenas templates em C#
dotnet new list --language "C#"
```

---

### 5.2. Pesquisar Templates Online no NuGet (`dotnet new search`)
Você pode pesquisar templates publicados globalmente no repositório oficial da comunidade NuGet diretamente pelo seu terminal:

```powershell
# Pesquisar templates de Clean Architecture
dotnet new search clean-architecture

# Pesquisar templates de arquitetura orientada a microsserviços
dotnet new search microservice
```

---

### 5.3. Instalar e Desinstalar Templates da Comunidade
Quando encontrar um template útil no NuGet, instale-o com um comando:

```powershell
# 1. Instalar um pacote de template da comunidade (exemplo: Clean Architecture de Jason Taylor)
dotnet new install Clean.Architecture.Solution.Template

# 2. Criar um projeto a partir do template instalado
dotnet new ca-sln -n MeuProjetoCompleto

# 3. Listar e desinstalar um template que você não usa mais
dotnet new uninstall Clean.Architecture.Solution.Template
```

---

## 6. Navegação

- 📑 **[Ir para o Índice do Guia de VS Code (`index.md`)](index.md)**
- ➡️ **[Avançar para a Aula 02: Gerenciamento de Projetos, Referências e NuGet (`02-gerenciamento-de-projetos-referencias-e-nuget.md`)](02-gerenciamento-de-projetos-referencias-e-nuget.md)**
