---
ms.openlocfilehash: 17c93f353c84ea2db28cd2e0203d30c5f320e36e
ms.sourcegitcommit: 141fe5c30dea84029ef61cf82558c35f2a744b65
ms.translationtype: MT
ms.contentlocale: pt-BR
ms.lasthandoff: 12/11/2020
ms.locfileid: "49655230"
---
<!-- markdownlint-disable MD002 MD041 -->

Neste tutorial, você criará uma função simples do Azure que implementa funções de gatilho HTTP que chamam o Microsoft Graph. Essas funções abrangerão os seguintes cenários:

- Implementa uma API para acessar a caixa de entrada de um usuário usando a autenticação de [fluxo em nome de](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-on-behalf-of-flow) .
- Implementa uma API para assinar e cancelar a assinatura de notificações na caixa de entrada de um usuário, usando o uso de [credenciais de cliente conceder](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow) autenticação de fluxo.
- Implementa um webhook para receber [notificações de alteração](https://docs.microsoft.com/graph/webhooks) do Microsoft Graph e acessar dados usando o fluxo de concessão de credenciais do cliente.

Você também criará um aplicativo simples de página única (SPA) JavaScript para chamar as APIs implementadas na função do Azure.

## <a name="create-azure-functions-project"></a>Criar projeto de funções do Azure

1. Abra a interface de linha de comando (CLI) em um diretório onde você deseja criar o projeto. Execute o seguinte comando.

    ```Shell
    func init GraphTutorial --dotnet
    ```

1. Altere o diretório atual em sua CLI para o diretório **GraphTutorial** e execute os seguintes comandos para criar três funções no projeto.

    ```Shell
    func new --name GetMyNewestMessage --template "HTTP trigger" --language C#
    func new --name SetSubscription --template "HTTP trigger" --language C#
    func new --name Notify --template "HTTP trigger" --language C#
    ```

1. Abra **local.settings.jsem** e adicione o seguinte ao arquivo para permitir CORS de `http://localhost:8080` , a URL para o aplicativo de teste.

    ```json
    "Host": {
      "CORS": "http://localhost:8080"
    }
    ```

1. Execute o comando a seguir para executar o projeto localmente.

    ```Shell
    func start
    ```

1. Se tudo estiver funcionando, você verá o seguinte resultado:

    ```Shell
    Http Functions:

        GetMyNewestMessage: [GET,POST] http://localhost:7071/api/GetMyNewestMessage

        Notify: [GET,POST] http://localhost:7071/api/Notify

        SetSubscription: [GET,POST] http://localhost:7071/api/SetSubscription
    ```

1. Verifique se as funções estão funcionando corretamente abrindo o navegador e navegando até as URLs de função mostradas na saída. Você deverá ver a seguinte mensagem em seu navegador: `This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response.` .

## <a name="create-single-page-application"></a>Criar um aplicativo de página única

1. Abra a CLI em um diretório onde você deseja criar o projeto. Crie um diretório chamado **TestClient** para manter seus arquivos HTML e JavaScript.

1. Crie um novo arquivo chamado **index.html** no diretório **TestClient** e adicione o código a seguir.

    :::code language="html" source="../demo/TestClient/index.html" id="indexSnippet":::

    Isso define o layout básico do aplicativo, incluindo uma barra de navegação. Ele também adiciona o seguinte:

    - [Bootstrap](https://getbootstrap.com/) e seu JavaScript de suporte
    - [FontAwesome](https://fontawesome.com/)
    - [Biblioteca de autenticação da Microsoft para JavaScript (MSAL.js) 2,0](https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-browser)

    > [!TIP]
    > A página inclui um favicon ( `<link rel="shortcut icon" href="g-raph.png">` ). Você pode remover essa linha ou pode baixar o arquivo de **g-raph.png** do [GitHub](https://github.com/microsoftgraph/g-raph).

1. Crie um novo arquivo chamado **Style. css** no diretório **TestClient** e adicione o código a seguir.

    :::code language="css" source="../demo/TestClient/style.css":::

1. Crie um novo arquivo chamado **ui.js** no diretório **TestClient** e adicione o código a seguir.

    :::code language="javascript" source="../demo/TestClient/ui.js" id="uiJsSnippet":::

    Este código usa JavaScript para renderizar a página atual com base no modo de exibição selecionado.

### <a name="test-the-single-page-application"></a>Testar o aplicativo de página única

> [!NOTE]
> Esta seção inclui instruções para usar o [dotnet-serve](https://github.com/natemcmaster/dotnet-serve) para executar um servidor http de teste simples em sua máquina de desenvolvimento. Não é necessário usar essa ferramenta específica. Você pode usar qualquer servidor de teste que preferir para servir o diretório **TestClient**

1. Execute o seguinte comando em sua CLI para instalar o **dotnet-atendimento**.

    ```Shell
    dotnet tool install --global dotnet-serve
    ```

1. Altere o diretório atual em sua CLI para o diretório **TestClient** e execute o seguinte comando para iniciar um servidor http.

    ```Shell
    dotnet serve -h "Cache-Control: no-cache, no-store, must-revalidate" -p 8080
    ```

1. Abra o navegador e vá até `http://localhost:8080`. A página deve renderizar, mas nenhum dos botões funciona no momento.

## <a name="add-nuget-packages"></a>Adicionar pacotes NuGet

Antes de prosseguir, instale alguns pacotes NuGet adicionais que serão usados posteriormente.

- [Microsoft. Azure. Functions. Extensions](https://www.nuget.org/packages/Microsoft.Azure.Functions.Extensions) para habilitar a injeção de dependência no projeto de funções do Azure.
- [Microsoft.Extensions.Configuration. Usersecrets](https://www.nuget.org/packages/Microsoft.Extensions.Configuration.UserSecrets) para ler a configuração do aplicativo do [repositório de segredo de desenvolvimento do .net](https://docs.microsoft.com/aspnet/core/security/app-secrets).
- [Microsoft. Graph](https://www.nuget.org/packages/Microsoft.Graph/) para fazer chamadas para o Microsoft Graph.
- [Microsoft. Identity. Client](https://www.nuget.org/packages/Microsoft.Identity.Client/) para autenticação e gerenciamento de tokens.
- [Microsoft. IdentityModel. Protocols. OpenIdConnect](https://www.nuget.org/packages/Microsoft.IdentityModel.Protocols.OpenIdConnect) para recuperar a configuração de OpenID para validação de token.
- [System. IdentityModel. Tokens. JWT](https://www.nuget.org/packages/System.IdentityModel.Tokens.Jwt) para validar tokens enviados para a API Web.

1. Altere o diretório atual em sua CLI para o diretório **GraphTutorial** e execute os seguintes comandos.

    ```Shell
    dotnet add package Microsoft.Azure.Functions.Extensions --version 1.0.0
    dotnet add package Microsoft.Extensions.Configuration.UserSecrets --version 3.1.5
    dotnet add package Microsoft.Graph --version 3.8.0
    dotnet add package Microsoft.Identity.Client --version 4.15.0
    dotnet add package Microsoft.IdentityModel.Protocols.OpenIdConnect --version 6.7.1
    dotnet add package System.IdentityModel.Tokens.Jwt --version 6.7.1
    ```
