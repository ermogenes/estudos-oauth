# Segurança de Aplicações: Identidade

## Conceitos

### O problema

Ao acessar uma API, uma aplicação solicita ao usuário sua identificação e sua senha, via UI. As credenciais são conferidas contra uma _whitelist_, e o usuário então é autenticado e pode receber as suas informações. As credenciais precisam ser mantidas no cliente e reenviadas/reconferidas a cada solicitação (através de _cookies_ de sessão, por exemplo). O processo é conhecido como _Basic Auth_.

Esse processo é especialmente ingênuo quando pensamos em integrar o _login_ com outras aplicações (SSO - _Single Sign On_), como aplicações de terceiros (um cliente de Twitter, ou uma aplicação web que salve no seu Google Drive, por exemplo). Não parece seguro informar suas credenciais para terceiros que devem armazená-las e mantê-las seguras. A cada mudança de senha será necessário atualizá-las em todos os terceiros.

Surgiu a necessidade de se disponibilizar acesso a APIs sem que os clientes necessitem guardar as credenciais do usuário, ou sequer conhecê-las.

Além disso, cada mudança no processo de autenticação (como a adição de mecanismo de MFA, por exemplo) exige a alteração de todas as aplicações envolvidas.

Nesse cenário, nasce o padrão OAuth em 2007. Em 2012 a segunda versão foi finalizada, simplificando o processo para o desenvolvedor e resolvendo questões relevantes para a segurança de aplicações _mobile_ nativas.

### Autenticação e Autorização

A ideia principal por trás da solução trazida pelo OAuth2 é que sempre que uma aplicação necessitar identificar um usuário, ela não solicitará suas credenciais diretamente, mas o redirecionará para um terceiro confiável, chamado de _servidor de identidade_. Ele disponibilizará um UI padronizada para fornecimento de credenciais, e retornará para a aplicação solicitante uma informação de sucesso/falha, e uma maneira de continuar acessando os recursos por um período de tempo (chamada de _Access Token_).

Dessa forma, as aplicações podem utilizar recursos das APIs sem conhecer as credenciais do usuário, nem o processo de autenticação.

Pode parecer estranho, mas pense no exemplo de uma locadora de automóveis. Você se identifica ao alugar o carro, e recebe uma chave, que te dá acesso ao carro. Nem a chave nem o carro sabem quem é você, ou quais documentos você mostrou para consegui-los.

Caso seja necessário que a aplicação cliente obtenha informações sobre o usuário autenticado (o chamado _ID Token_), é necessário que o servidor de identidade também dê suporte a outra tecnologia associada ao OAuth2, chamada OpenID Connect.

Assim, tenha em mente:

A tecnologia | | faz | que trada de
--- | ---| --- | ---
<img src="oauth-icon.png" alt="OAuth" height="50"> | OAuth | autenticação | _acessar APIs_
<img src="openid-icon.png" alt="OpenID" height="50"> | OpenID Connect | autorização | _identificar usuários_

### Os atores e seus papéis

- **Usuário** ou **_Resource Owner_**: o proprietário da conta.
- **Dispositivo** ou **_User Agent_**: (telefone, TV, navegador) utilizado pelo usuário para executar ou acessar uma aplicação.
- **Aplicação** ou **_Client_**: app, _site_ ou outro software que é utilizado pelo usuário no dispositivo, e que acessa recursos em uma API.
- **API** ou **_Resource Server_**: entrega recursos à aplicações, à pedido do usuário. Local onde os dados estão armazenados.
- **Servidor de identidade** ou **_Authorization server_**: identifica o usuário de forma independente e retorna um _access token_ ao _client_, que pode utilizá-lo para solicitar recursos à API.

Exemplo 1: Maria (_resource owner_) usa o Chrome (_user agent_) para acessar o site XYZ (_client_), que usa segurança baseada em _cookies_. Ele exibe uma tela própria solicitando as credenciais do usuário (usuário e senha), que são enviadas para o servidor _backend_ (_API_, no papel de _authorization server_), que valida e retorna um _cookie de sessão_. Esse _cookie_ é reenviado a cada solicitação de dados ao _backend_ (_API_, no papel de _resource server_).

Exemplo 2: Tereza (_resource owner_) usa o Firefox (_user agent_) para acessar o site JKL (_client_) que usa segurança OAuth, e clica em 'Entrar com o Google'. Ele redireciona o usuário para o Google (_authorization server_) que mostra uma tela solicitando as credenciais do usuário (usuário e senha). Após validação, retorna um _access token_ ao redirecionar de volta ao _client_. Esse _token_ é reenviado a cada solicitação de dados ao _backend_ (_API_, no seu papel funcional), que possui algum mecanismo para verificar se o _access token_ é válido.

### Tipos de _clients_

_Clients_ diferem em relação à sua habilidade de armazenar algum tipo de credencial em seu _deploy_ que possa ser utilizada para verificação de identidade junto ao _authorization server_. Essas credenciais são chamadas _client secrets_ e não podem ser acessíveis pelos usuários das aplicações, logados ou não.

- _Confidential Clients_: aplicações implantadas no _backend_ (como PHP, .NET ou Python, por exemplo) que podem guardar segredos em forma de _API Keys_ em arquivos de configuração ou variáveis de ambiente (podemos pensar as _API keys_ como as senhas das aplicações).
- _Public Clients_: aplicações que não tem essa capacidade, pois rodam nos dispositivos dos usuários, como um app _mobile_, uma SPA, um app de TV ou um software rodando em um dispositivo IoT.
- _Credentialed Clients_: aplicações que confirmam sua identidade através de uma chave privada e um certificado únicos no cliente para obter _tokens_ vinculados unicamente a esse cliente.

_Confidential Clients_ podem solicitar autorização a um _authorization server_ usando suas credenciais secretas únicas. Esses segredos permitem que o cliente seja identificado exclusivamente, sem a interação com o usuário. Sem eles, não há como saber se a chamada veio de um _client_ real, ou de alguém se passando por ele.

Caso os segredos vazem, um atacante pode utilizá-los para se passar pelo cliente. É importante que a URL de redirecionamento de volta ao cliente também não esteja sob controle de um atacante.

Os _authorization servers_ podem ter políticas diferenciadas para clientes confidenciais e públicos. Por exemplo, solicitar as credenciais ao usuário no momento do login, possivelmente utilizando mais de um fator, em caso de _public clients_. Também é comum incluir tempos de expiração nos _access tokens_ e alguma regra envolvendo _refresh tokens_ para manter o usuário logado por períodos prolongados.

Um exemplo de _credentialed client_ é um app _mobile_ instalado via loja de aplicativos. No seu primeiro uso, ele ainda não pode ser autenticado, pois ele não pode armazenar um segredo no momento do _deploy_. Ele faz um processo conhecido como _dynamic client registration_ para conseguir um segredo único, que fica armazenado somente naquele dispositivo e pode ser utilizado para identificar chamadas daquela aplicação naquele dispositivo específico. O _client_ ainda pode não estar identificado, mas é possível garantir que todas as solicitações com o mesmo _token_ vieram do mesmo _client_.

### _User consent_

Vejamos o caso de um _client_ que forneça sua própria tela de login, fazendo que o usuário entre com suas credenciais. Seria solicitado um _token_ com essas credenciais enviadas em uma chamada POST com usuário, senha, identificação do cliente e o tipo de login (`grant_type=password`). O _client_ realiza uma troca com o _authorization server_: as credenciais por um _access token_.

Do ponto de vista do _authorization server_, não há como saber se é o usuário tentando se logar legitimamente, se o _client_ está reutilizando as credenciais do último login (e solicitando recursos os quais você não deu permissão), ou se é atacante que conseguiu as credenciais de alguma forma.

O que falta nesse fluxo é a garantia ao _authorization server_ de que foi mesmo o usuário que iniciou o login, e que ele está ciente das permissões que são solicitadas. Isso é feito usando as _consent screens_.

Ao realizar um login usando uma tela de consentimento disponibilizada pelo _authorization server_, podemos garantir que é o usuário que está tentando logar (através, por exemplo, de MFA) e deixar claro a ele as permissões solicitadas pelo _client_. As credenciais são fornecidas diretamente ao _authorization server_, e nunca repassadas ao _client_. A única coisa que ela recebe é um _access token_ que somente permite o uso de recursos com o consentimento do usuário. Esse _token_ é repassado redirecionando para uma URL listada nas configurações do _authorization server_, de forma que mesmo que uma solicitação seja forjada, ela não pode ser entregue a um _client_ malicioso.

Nesse processo, os _authorization server_ podem pular a interação quando provém de _confidential clients_, já que a identidade já é previamente confirmada.

### _Front_ e _Back Channels_

Conexões HTTPS diretas, como um `fetch` são realizadas no que chamamos de _back channel_. Elas garantem que, uma vez estabelecida a conexão, a mensagem é segura e confiável, e não pode ter sua origem forjada.

Em contraste, podemos realizar transferência de dados usando o chamado _front channel_, um redirecionamento realizado por um navegador. Quando digitamos um endereço na barra de navegação, clicamos em um link ou fazemos um redirecionamento em uma página usando `window.location` (ou equivalente), estamos terceirizando a chamada ao navegador.

No _front channel_ não há comunicação direta entre a aplicação e o _authorization server_, a origem não sabe se o pacote foi entregue, e o destino não sabe se foi o usuário que a originou, se foi alterado no transporte, ou se o endereço de redirecionamento é legítimo.

Sendo assim, a maneira mais segura de transitar dados em OAuth é usando o _back channel_ sempre que possível. O _front channel_ é usado somente quando não é possível fazer de outra forma, como por exemplo, nas _consent screens_.

Apesar de constante na especificação, não é recomendado o uso do _implicit flow_. Nele o _client_ envia pelo _front channel_ um pedido de tela de consentimento, e recebe de volta, novamente pelo _front channel_ um _access token_. Isso só deve ser usado caso não seja possível realizar chamadas de _back channel_ (por falta de suporte a CORS, por exemplo).

### _Application Identity_

Cada _client_ deve possuir uma identidade única, chamada _client ID_. Ela será utilizada para informar ao _authorization server_ quem é o _client_ solicitante.

Para _public clients_, não há muito que se possa fazer para garantir a origem do fluxo, já que a tela de consentimento e o redirecionamento de retorno são chamadas _front channel_, e não há uma _API Key_ única envolvida.

Nesses casos o _authorization server_ não retorna um _access token_, mas sim um _authorization code_ com tempo de expiração bastante curto. Ele pode então ser trocado por um _access token_ usando o _back channel_. Para se garantir que o solicitante do _access token_ é o mesmo que solicitou o _authorization code_, no redirecionamento à _consent screen_ também é esperado o _client secret_. Porém, _public clients_ não o possuem, então é utilizado a extensão PKCE, _Proof Key for Code Exchange_ (lê-se como _pixie_: _pic-si_). Antes da primeira chamada, o _public client_ gera um segredo único, que será utilizado em substituição ao _client secret_, garantindo que o solicitante do fluxo é o mesmo que receberá o _access token_ em troca do _authorization code_.

Isso ainda não resolve o fato de que a identidade do _client_ não pode ser comprovada, mas somente a correlação entre o solicitante do login e o recebedor do _access token_. Um atacante ainda poderá tentar se passar pela aplicação e realizar todo o fluxo, já que toda a informação nesse fluxo é pública.

Para reduzir esse risco, há uma _whitelist_ de URLs de redirecionamento permitidas que deve ser configurado no _authorization server_. Isso garante que o redirecionamento contendo o _authorization code_ via _front channel_ só será realizado para origens confiáveis.

Essas URL, chamadas _redirect URLs_, podem ser URLs como `https://xyzapp.com/redirect` no caso de SPAs, mas em caso de aplicações nativas, como apps _mobile_ ou _desktop_, podem ser _URL schemas_ como `xyzapp://redirect`. URLs são únicas (garantido pelos registros DNS), porém não há registro global centralizado de _URL schemas_. Mesmo que algumas lojas garantam que não há dois apps registrados com o mesmo _schema_, ele não deve ser utilizado para garantir a identidade do _client_. Algumas lojas permitem que aplicações recebam redirecionamentos usando URLs, desde que seja confirmada a posse do URL (via _Digital Assets Link_, por exemplo).

Assim, nos casos de SPA ou aplicações nativas, não há uma maneira totalmente segura de garantir a identidade da aplicação. Assim, em _public clients_ só se pode garantir que a aplicação que recebeu autorização de acesso é a mesma que a solicitou e recebeu consentimento do usuário. Tenha isso em mente ao definir os tempos de vida dos _tokens_ e a dispensa da tela de consentimento.

## Referências

- PARECKI, Aaron. The Nuts and Bolts of OAuth 2.0. Udemy. [https://www.udemy.com/course/oauth-2-simplified](https://www.udemy.com/course/oauth-2-simplified).


