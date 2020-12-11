---
ms.openlocfilehash: eb227079656e2a57550511c3abfacb49935fe46a
ms.sourcegitcommit: 141fe5c30dea84029ef61cf82558c35f2a744b65
ms.translationtype: MT
ms.contentlocale: pt-BR
ms.lasthandoff: 12/11/2020
ms.locfileid: "49655237"
---
<!-- markdownlint-disable MD002 MD041 -->

Neste exercício, você concluirá a implementação das funções do Azure `SetSubscription` e `Notify` atualizará o aplicativo de teste para inscrever-se e cancelar a assinatura das alterações na caixa de entrada de um usuário.

- A `SetSubscription` função atuará como uma API, permitindo que o aplicativo de teste crie ou exclua uma [assinatura](https://docs.microsoft.com/graph/webhooks) para alterações na caixa de entrada de um usuário.
- A `Notify` função atuará como o webhook que recebe notificações de alteração geradas pela assinatura.

Ambas as funções usarão o [fluxo de concessão de credenciais do cliente](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow) para obter um token somente de aplicativo para chamar o Microsoft Graph. Como um administrador concedeu consentimento de administrador aos escopos de permissão necessários, nenhuma interação do usuário será necessária para obter o token.

## <a name="add-client-credentials-authentication-to-the-azure-functions-project"></a>Adicionar autenticação de credenciais de cliente ao projeto de funções do Azure

Nesta seção, você implementará o fluxo de credenciais do cliente no projeto de funções do Azure para obter um token de acesso compatível com o Microsoft Graph.

1. Abra a CLI no diretório que contém **GraphTutorial. csproj**.

1. Adicione a ID e o segredo do aplicativo webhook ao repositório secreto usando os seguintes comandos. Substitua `YOUR_WEBHOOK_APP_ID_HERE` pela ID do aplicativo do **webhook da função Graph Azure**. Substitua `YOUR_WEBHOOK_APP_SECRET_HERE` pelo segredo do aplicativo que você criou no portal do Azure para o **webhook de função do Azure Graph**.

    ```Shell
    dotnet user-secrets set webHookId "YOUR_WEBHOOK_APP_ID_HERE"
    dotnet user-secrets set webHookSecret "YOUR_WEBHOOK_APP_SECRET_HERE"
    ```

### <a name="create-a-client-credentials-authentication-provider"></a>Criar um provedor de autenticação de credenciais de cliente

1. Crie um novo arquivo no diretório **./GraphTutorial/Authentication** chamado **ClientCredentialsAuthProvider.cs** e adicione o código a seguir.

    :::code language="csharp" source="../demo/GraphTutorial/Authentication/ClientCredentialsAuthProvider.cs" id="AuthProviderSnippet":::

Reserve um tempo para considerar o que o código no **ClientCredentialsAuthProvider.cs** faz.

- No construtor, ele inicializa um **ConfidentialClientApplication** do `Microsoft.Identity.Client` pacote. Ele usa as `WithAuthority(AadAuthorityAudience.AzureAdMyOrg, true)` `.WithTenantId(tenantId)` funções e para restringir a audiência de login apenas à organização especificada do Microsoft 365.
- Na `GetAccessToken` função, ele chama `AcquireTokenForClient` para obter um token para o aplicativo. O fluxo de token de credenciais do cliente é sempre não interativo.
- Ele implementa a `Microsoft.Graph.IAuthenticationProvider` interface, permitindo que essa classe seja passada no construtor do `GraphServiceClient` para autenticar solicitações de saída.

## <a name="update-graphclientservice"></a>Atualizar GraphClientService

1. Abra **GraphClientService.cs** e adicione a propriedade a seguir à classe.

    :::code language="csharp" source="../demo/GraphTutorial/Services/GraphClientService.cs" id="AppGraphClientMembers":::

1. Substitua a função `GetAppGraphClient` existente pelo seguinte.

    :::code language="csharp" source="../demo/GraphTutorial/Services/GraphClientService.cs" id="AppGraphClientFunctions":::

## <a name="implement-notify-function"></a>Implementar função Notify

Nesta seção, você implementará a `Notify` função, que será usada como a URL de notificação para notificações de alteração.

1. Crie um novo diretório no diretório **GraphTutorials** chamado **modelos**.

1. Crie um novo arquivo no diretório **modelos** chamado **ResourceData.cs** e adicione o código a seguir.

    :::code language="csharp" source="../demo/GraphTutorial/Models/ResourceData.cs" id="ResourceDataSnippet":::

1. Crie um novo arquivo no diretório **modelos** chamado **ChangeNotification.cs** e adicione o código a seguir.

    :::code language="csharp" source="../demo/GraphTutorial/Models/ChangeNotification.cs" id="ChangeNotificationSnippet":::

1. Crie um novo arquivo no diretório **modelos** chamado **NotificationList.cs** e adicione o código a seguir.

    :::code language="csharp" source="../demo/GraphTutorial/Models/NotificationList.cs" id="NotificationListSnippet":::

1. Abra **./GraphTutorial/Notify.cs** e substitua todo o conteúdo pelo seguinte.

    :::code language="csharp" source="../demo/GraphTutorial/Notify.cs" id="NotifySnippet":::

Reserve um tempo para considerar o que o código no **Notify.cs** faz.

- A `Run` função verifica a presença de um `validationToken` parâmetro de consulta. Se esse parâmetro estiver presente, ele processará a solicitação como uma [solicitação de validação](https://docs.microsoft.com/graph/webhooks#notification-endpoint-validation)e responderá de acordo.
- Se a solicitação não for uma solicitação de validação, a carga JSON será desserializada em um `NotificationList` .
- Cada notificação na lista é verificada quanto ao valor de estado do cliente esperado e é processada.
- A mensagem que disparou a notificação é recuperada com o Microsoft Graph.

## <a name="implement-setsubscription-function"></a>Implementar função setsubscription

Nesta seção, você implementará a função setsubscription. Essa função atuará como uma API que é chamada pelo aplicativo de teste para criar ou excluir uma assinatura na caixa de entrada de um usuário.

1. Crie um novo arquivo no diretório **modelos** chamado **SetSubscriptionPayload.cs** e adicione o código a seguir.

    :::code language="csharp" source="../demo/GraphTutorial/Models/SetSubscriptionPayload.cs" id="SetSubscriptionPayloadSnippet":::

1. Abra **./GraphTutorial/SetSubscription.cs** e substitua todo o conteúdo pelo seguinte.

    :::code language="csharp" source="../demo/GraphTutorial/SetSubscription.cs" id="SetSubscriptionSnippet":::

Reserve um tempo para considerar o que o código no **SetSubscription.cs** faz.

- A `Run` função lê a carga JSON enviada na solicitação post para determinar o tipo de solicitação (inscrever-se ou cancelar inscrição), a ID de usuário a ser inscrita e a ID da assinatura para cancelar a assinatura.
- Se a solicitação for uma solicitação de assinatura, ela usará o SDK do Microsoft Graph para criar uma nova assinatura na caixa de entrada do usuário especificado. A assinatura notificará quando as mensagens forem criadas ou atualizadas. A nova assinatura é retornada na carga JSON da resposta.
- Se a solicitação for uma solicitação de cancelamento de assinatura, ela usará o SDK do Microsoft Graph para excluir a assinatura especificada.

## <a name="call-setsubscription-from-the-test-app"></a>Chamar setsubscription do aplicativo de teste

Nesta seção, você implementará as funções para criar e excluir assinaturas no aplicativo de teste.

1. Abra **./TestClient/azurefunctions.js** e adicione a função a seguir.

    :::code language="javascript" source="../demo/TestClient/azurefunctions.js" id="createSubscriptionSnippet":::

    Este código chama a `SetSubscription` função do Azure para inscrever e adiciona a nova assinatura à matriz de assinaturas na sessão.

1. Adicione a função a seguir para **azurefunctions.js**.

    :::code language="javascript" source="../demo/TestClient/azurefunctions.js" id="deleteSubscriptionSnippet":::

    Este código chama a `SetSubscription` função do Azure para desmarcar e remover a assinatura da matriz de assinaturas na sessão.

1. Se você não tiver o ngrok em execução, execute ngrok ( `ngrok http 7071` ) e copie a URL de encaminhamento HTTPS.

1. Adicione a URL do ngrok ao repositório de segredos do usuário executando o seguinte comando.

    ```Shell
    dotnet user-secrets set ngrokUrl "YOUR_NGROK_URL_HERE"
    ```

    > [!IMPORTANT]
    > Se você reiniciar o ngrok, será necessário repetir este comando para atualizar a URL do ngrok.

1. Altere o diretório atual em sua CLI para o diretório **./GraphTutorial** e execute o seguinte comando para iniciar a função do Azure localmente.

    ```Shell
    func start
    ```

1. Atualize o SPA e selecione o item de navegação de **assinaturas** . Insira uma ID de usuário para um usuário em sua organização do Microsoft 365 que tenha uma caixa de correio do Exchange Online. Isso pode ser o do usuário `id` (do Microsoft Graph) ou o do usuário `userPrincipalName` . Clique em **inscrever**.

1. A página é atualizada mostrando a nova assinatura na tabela.

1. Envie um email para o usuário. Após um breve período, a `Notify` função deve ser chamada. Você pode verificar isso na interface Web do ngrok ( `http://localhost:4040` ) ou na saída de depuração do projeto de função do Azure.

    ```Shell
    ...
    [7/8/2020 7:33:57 PM] The following message was created:
    [7/8/2020 7:33:57 PM] Subject: Hi Megan!, ID: AAMkAGUyN2I4N2RlLTEzMTAtNDBmYy1hODdlLTY2NTQwODE2MGEwZgBGAAAAAAA2J9QH-DvMRK3pBt_8rA6nBwCuPIFjbMEkToHcVnQirM5qAAAAAAEMAACuPIFjbMEkToHcVnQirM5qAACHmpAsAAA=
    [7/8/2020 7:33:57 PM] Executed 'Notify' (Succeeded, Id=9c40af0b-e082-4418-aa3a-aee624f30e7a)
    ...
    ```

1. No aplicativo de teste, clique em **excluir** na linha da tabela da assinatura. A página é atualizada e a assinatura não está mais na tabela.
