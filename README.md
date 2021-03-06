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

### Fluxo abstrato

Vejamos como os papéis interagem. O fluxo é iniciado quando o _client_ necessita de um recurso que está protegido por OAuth.

![](abstract-flow.drawio.svg)

- (A): O _client_ necessita de uma autorização de uso (_authorization grant_), que só pode ser dada pelo _resource owner_. Há diversas maneiras de realizar essa solicitação (preferencialmente via _authorization server_, e não diretamente como nesse fluxo abstrato), e 4 tipos diferentes de _grants_.
- (B): O _client_ recebe um _authorization grant_, uma credencial representando uma autorização de uso dada pelo _resource owner_.
- (C): O _client_ envia o _grant_ ao _authorization server_, e espera um _access token_ em troca.
- (D): O _authorization server_ autentica o _client_ e valida a _grant_, e emite um _access token_.
- (E): O _client_ solicita o recurso desejado ao _resource server_ usando o _access token_ como prova de permissão.
- (F): O _resource server_ valida o _access token_ e entrega o recurso solicitado.

### _Grants_

Um _authorization grant_ é uma credencial dada pelo _resource owner_ a um _client_ para acessar um recurso protegido.

- _Authorization Code_: usa o _authorization server_ como intermediário entre o _client_ e o _resource owner_, aumentando a segurança em situações onde o _resource owner_ está online.
- _Implicit_: como o anterior, mas com um fluxo simplificado, em troca de uma segurança reduzida. Não é recomendado.
- _Resource Owner Password Credentials_: permite usar usuário/senha do _resource owner_ diretamente pelo _client_. Deve ser utilizado somente em situações específicas de alta confiança entre _resource owner_ e _client_.
- _Client Credentials_: utilizado quando o _client_ também é o _resource owner_, como em comunicação M2M (_machine to machine_) sem interação humana.

### _Access Token_

Os _access tokens_ são uma abstração para dados de autorização (como usuário/senha, por exemplo).

São formados por uma _string_ que representa uma autorização emitida pelo _resource owner_ permitindo ao _client_ ter acesso a um recurso. Essencialmente são identificadores, _strings_ opacas ao cliente, sem significado, mas que podem implicar escopos de uso e prazo de validade garantidos pelo _authorization server_ e o _resource server_. Podem também conter informações auto-verificáveis (como dados e assinatura). O formato mais usado atualmente é o _bearer token_, ou token "_ao portador_", codificado como JWT (_JSON Web Token_).

### _Refresh Token_

São como os _access tokens_, mas com a finalidade de obter um novo _access token_, e não o acesso a um recurso. São usados para renovar autorizações mesmo após o final de sua validade.

![](refresh.drawio.svg)

### Criptografia

OAuth foi desenhado para ser utilizado com TLS.

### Redirecionamentos

Não há exigência de método, porém o mais comum é utilizar HTTP 302.

### _Client Registration_

_Clients_ devem ser previamente identificados no _authorization server_ através de:

- um tipo de _client_;
- uma ou mais URLs de redirecionamento;
- quaisquer outros metadados requeridos pelo _authorization server_.

### Tipos de _clients_

_Clients_ diferem em relação à sua habilidade de armazenar algum tipo de credencial em seu _deploy_ que possa ser utilizada para verificação de identidade junto ao _authorization server_. Essas credenciais são chamadas _client secrets_ e não podem ser acessíveis pelos usuários das aplicações, logados ou não.

- _Confidential Clients_: aplicações implantadas no _backend_ (como PHP, .NET ou Python, por exemplo) que podem guardar segredos em forma de _API Keys_ em arquivos de configuração ou variáveis de ambiente (podemos pensar as _API keys_ como as senhas das aplicações).
- _Public Clients_: aplicações que não tem essa capacidade, pois rodam nos dispositivos dos usuários, como um app _mobile_, uma SPA, um app de TV ou um software rodando em um dispositivo IoT.
- _Credentialed Clients_: aplicações que confirmam sua identidade através de uma chave privada e um certificado únicos no cliente para obter _tokens_ vinculados unicamente a esse cliente. Incluído na versão 2.1 do OAuth.

_Confidential Clients_ podem solicitar autorização a um _authorization server_ usando suas credenciais secretas únicas. Esses segredos permitem que o cliente seja identificado exclusivamente, sem a interação com o usuário. Sem eles, não há como saber se a chamada veio de um _client_ real, ou de alguém se passando por ele.

Caso os segredos vazem, um atacante pode utilizá-los para se passar pelo cliente. É importante que a URL de redirecionamento de volta ao cliente também não esteja sob controle de um atacante.

Os _authorization servers_ podem ter políticas diferenciadas para clientes confidenciais e públicos. Por exemplo, solicitar as credenciais ao usuário no momento do login, possivelmente utilizando mais de um fator, em caso de _public clients_. Também é comum incluir tempos de expiração nos _access tokens_ e alguma regra envolvendo _refresh tokens_ para manter o usuário logado por períodos prolongados.

Um exemplo de _credentialed client_ é um app _mobile_ instalado via loja de aplicativos. No seu primeiro uso, ele ainda não pode ser autenticado, pois ele não pode armazenar um segredo no momento do _deploy_. Ele faz um processo conhecido como _dynamic client registration_ para conseguir um segredo único, que fica armazenado somente naquele dispositivo e pode ser utilizado para identificar chamadas daquela aplicação naquele dispositivo específico. O _client_ ainda pode não estar identificado, mas é possível garantir que todas as solicitações com o mesmo _token_ vieram do mesmo _client_.

_Clients_ que possuem credenciais (_confidential_ ou _credentialed_) devem usar essas credenciais sempre que possível. É recomendado o uso de métodos assimétricos como mTLS ([RFC8705](https://datatracker.ietf.org/doc/html/rfc8705)) ou OpenID (`private_key_jwt`), de forma que o _authorization server_ não necessite manter chaves simétricas dos clientes.

_Clients_ que possuam o _client secret_ (como o usuário/senha do _resource owner_, por exemplo) devem usar _HTTP Basic authentication scheme_, não devem transmitir os dados via _query string_, e o _authorization server_ deve garantir que TLS está sendo usado.

#### _Client Profiles_

OAuth é desenhado para atender 3 perfis de clientes:

- _web applications_: um _confidential client_ rodando em um _web server_. Geralmente interagem com o _resource owner_ através de uma interface HTML intermediada por um _user-agent_ rodando em um dispositivo controlado pelo usuário. Seu código e dados só são visíveis àqueles com acesso ao servidor. Exemplos: _backends_ de aplicações web com interface _server-rendered_.
- _browser-based applications_: um _public client_ em que o software é baixado de um servidor e executado em um _user-agent_ rodando em um dispositivo controlado pelo usuário. Seu código e dados não podem ser protegidos, ou são explicitamente públicos.  Exemplos: aplicações _frontend_ acessando APIs, SPAs.
- _native applications_: um _public client_ instalado e executado nativamente em um dispositivo controlado pelo usuário. Assume-se que seu código e dados estáticos (incluídos na instalação) são passíveis de serem extraídos, mas os dados dinâmicos (gerados durante a execução ou recebidos de terceiros) podem receber um tratamento com proteção aceitável (variando em cada plataforma). Exemplos: aplicações _mobile_ nativas, aplicações _desktop_.

#### _Client Identifier_

Um _client_ registrado recebe um identificador, chamado _client ID_ (`client_id`). A geração do ID deve ser feita pelo _authorization server_, e nunca pelo _client_. É composta por uma _string_ de tamanho e formatos livres, e deve ser única por _authorization server_.

### _User Consent_

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

Nesses casos o _authorization server_ não retorna um _access token_, mas sim um _authorization code_ com tempo de expiração bastante curto. Ele pode então ser trocado por um _access token_ usando o _back channel_. Para se garantir que o solicitante do _access token_ é o mesmo que solicitou o _authorization code_, no redirecionamento à _consent screen_ também é esperado o _client secret_. Porém, _public clients_ não o possuem, então é utilizado a extensão PKCE, _Proof Key for Code Exchange_ (lê-se como _pixy_: _pic-si_). Antes da primeira chamada, o _public client_ gera um segredo único, que será utilizado em substituição ao _client secret_, garantindo que o solicitante do fluxo é o mesmo que receberá o _access token_ em troca do _authorization code_.

_Na realidade, a extensão PKCE é tão poderosa que é recomendado seu uso em todas as situações, mesmo em confidential clients, evitando assim ataques do tipo [Authorization Code Injection](https://tools.ietf.org/id/draft-ietf-oauth-security-topics-12.html#rfc.section.4.5)._

Isso ainda não resolve o fato de que a identidade do _client_ não pode ser comprovada, mas somente a correlação entre o solicitante do login e o recebedor do _access token_. Um atacante ainda poderá tentar se passar pela aplicação e realizar todo o fluxo, já que toda a informação nesse fluxo é pública.

Para reduzir esse risco, há uma _whitelist_ de URLs de redirecionamento permitidas que deve ser configurado no _authorization server_. Isso garante que o redirecionamento contendo o _authorization code_ via _front channel_ só será realizado para origens confiáveis.

Essas URL, chamadas _redirect URLs_, podem ser URLs como `https://xyzapp.com/redirect` no caso de SPAs, mas em caso de aplicações nativas, como apps _mobile_ ou _desktop_, podem ser _URL schemas_ como `xyzapp://redirect`. URLs são únicas (garantido pelos registros DNS), porém não há registro global centralizado de _URL schemas_. Mesmo que algumas lojas garantam que não há dois apps registrados com o mesmo _schema_, ele não deve ser utilizado para garantir a identidade do _client_. Algumas lojas permitem que aplicações recebam redirecionamentos usando URLs, desde que seja confirmada a posse do URL (via _Digital Assets Link_, por exemplo).

Assim, nos casos de SPA ou aplicações nativas, não há uma maneira totalmente segura de garantir a identidade da aplicação. Assim, em _public clients_ só se pode garantir que a aplicação que recebeu autorização de acesso é a mesma que a solicitou e recebeu consentimento do usuário. Tenha isso em mente ao definir os tempos de vida dos _tokens_ e a dispensa da tela de consentimento.

## CIAM - _Customer Identity and Access Management_

Há várias soluções para o papel de _authorization servers_, comerciais e gratuitas. Segue algumas.

Serviços dedicados (SaaS/IDaaS): 
- [Okta](https://www.okta.com/)
- [Auth0](https://auth0.com/)

Serviços integrados a soluções de nuvem:
- [Amazon Cognito](https://aws.amazon.com/pt/cognito/)
- [Google Cloud Identity Platform](https://cloud.google.com/identity-platform)
- [Azure Active Directory](https://azure.microsoft.com/pt-br/services/active-directory/) e [Azure Active Directory B2C](https://azure.microsoft.com/pt-br/services/active-directory/external-identities/b2c/#overview)

_On-premise_:
- [Keycloak](https://www.keycloak.org/)
- [WSO2](https://wso2.com/)
- [Ory Hydra](https://www.ory.sh/hydra/docs/)

_Frameworks_ para desenvolvimento:
- [Duende IdentityServer](https://duendesoftware.com/products/identityserver)

Escolha um deles, e faça as configurações necessárias.

Será necessário se cadastrar como desenvolvedor, e em sua conta registrar as suas aplicações _clients_. Os _clients_ registrados receberão credenciais para serem usadas em fluxos OAuth, contendo um `client_id` (público) e, para _confidential clients_, um `client_secret` (privado). Ao registrar, você indicará as _redirect URLs_ em uma _whitelist_ (evite usar curingas ou padrões). Pode ser necessário indicar um nome público (para a _consent screen_), logo, descrição e links para termos de serviço e privacidade.

As informações e configurações variam por provedor, podendo adaptar-se de acordo com o tipo de _client_ solicitado.

## _Clientes OAuth_

Vamos discutir fluxos para diversos cenários.

### Aplicações _backend_

Aplicações _backend_ podem necessitar acessar APIs que estão protegidas por OAuth para buscar recursos necessários para o seu processamento. Como não possuem nenhum usuário operando interativamente, não há interesse no uso de _consent screens_. Porém, esse tipo de aplicação costuma ser implantada em servidores nos quais o usuário não tem acesso direto, sendo assim ideais _confidential clients_.

Exemplo: Alice usa o `chrome` para acessar a aplicação `myapp_mvc`. Ela é uma aplicação ASP.NET MVC que roda no _backend_, e necessita consumir um serviço em `whatever_api`, protegido por OAuth através do _authorization server_ do Okta.

![](confidential-client-001.drawio.svg)

O _confidential client_ `myapp_mvc` é registrado em `okta` e obtém um `client_id` e um `client_secret`, que são incluídos em seu _deploy_ (`client_id` no fonte, `client_secret` em uma variável de ambiente).

Alice não está logada ainda, e clica em uma opção que necessita de um dado que deve ser buscado em `whatever_api`, porém só está disponível para usuários autenticados.

![](confidential-client-002.drawio.svg)

1. Alice solicita um dado privado ao clicar em um botão na página usando o Chrome. O _user agent_ trafega a solicitação até o _backend_, que não pode obtê-la e entregá-la sem a autenticação do usuário. 
2. Para obter a autorização, o cliente envia ao navegador um pedido de redirecionamento para o _authorization server_, contendo uma solicitação de autorização. Nessa solicitação trafega o identificador do cliente, o nível de acesso necessário, e a URL para redirecionamento (que deve trazer de volta ao _backend_). O objetivo é obter um _authorization code_.
3. O _authorization server_ redireciona para a tela de consentimento, aguardando credenciais e permissão de acesso.
4. O usuário provê interativamente credenciais válidas e permissão para o cliente obter os dados, que são enviados diretamente ao _authorization server_.
5. Em troca das credenciais válidas, o _authorization server_ solicita redirecionamento para o URL indicado, juntamente com um _authorization code_. O navegador redireciona para o _backend_. Até agora, toda a comunicação se deu pelo _front channel_. O usuário continuará aguardando o término da solicitação.
6. Em posse do _authorization code_, o cliente pode anexar o seu _client secret_ e trocá-los por um _access token_. Essa comunicação é feita pelo _back channel_, diretamente entre o cliente e o _authorization server_.
7. O _authorization server_ devolve um _access token_ (e possivelmente um _refresh token_), que permitirá ao cliente obter recursos no _resource server_.
8. Ainda no _back channel_, o cliente solicita o recurso desejado usando o _access token_.
9. O _resource server_ valida o _access token_ e devolve o recurso desejado.
10. De posse do dado, o cliente envia a página com o conteúdo solicitado por Alice.

Caso exista suporte a PKCE, ele será gerado no passo 2 (trafega hash, ou _code challenge_), e conferido no passo 6 (trafega texto claro, ou _code verifier_). Caso não exista suporte, pode-se utilizar `state` para se proteger de ataques CSRF (aleatório em 2, igual na resposta em 3).

## Materiais adicionais

- [Keycloak local com `podman-compose`](keycloak/README.md)

## Referências

- IETF. RFC 6749 - The OAuth 2.0 Authorization Framework. [https://datatracker.ietf.org/doc/html/rfc6749](https://datatracker.ietf.org/doc/html/rfc6749).
- IETF. The OAuth 2.1 Authorization Framework (RFC draft, version 3). [https://datatracker.ietf.org/doc/html/draft-parecki-oauth-v2-1-03](https://datatracker.ietf.org/doc/html/draft-parecki-oauth-v2-1-03).
- IETF. RFC 6750 - The OAuth 2.0 Authorization Framework: Bearer Token Usage. [https://datatracker.ietf.org/doc/html/rfc6750](https://datatracker.ietf.org/doc/html/rfc6750).
- IETF. RFC 7519 - JSON Web Token (JWT) [https://datatracker.ietf.org/doc/html/rfc7519](https://datatracker.ietf.org/doc/html/rfc7519).
- IETF. RFC 7636 - Proof Key for Code Exchange by OAuth Public Clients. [https://datatracker.ietf.org/doc/html/rfc7636](https://datatracker.ietf.org/doc/html/rfc7636).
- IETF. OAuth 2.0 Security Best Current Practice (RFC draft, version 19). [https://datatracker.ietf.org/doc/draft-ietf-oauth-security-topics/](https://datatracker.ietf.org/doc/draft-ietf-oauth-security-topics/).
- PARECKI, Aaron. The Nuts and Bolts of OAuth 2.0. Udemy. [https://www.udemy.com/course/oauth-2-simplified](https://www.udemy.com/course/oauth-2-simplified).
- Digital Ocean. An Introduction to OAuth 2. [https://www.digitalocean.com/community/tutorials/an-introduction-to-oauth-2](https://www.digitalocean.com/community/tutorials/an-introduction-to-oauth-2). ([versão pt-BR](https://www.digitalocean.com/community/tutorials/uma-introducao-ao-oauth-2-pt)).
- Okta. Developer Docs - Concepts. [https://developer.okta.com/docs/concepts/](https://developer.okta.com/docs/concepts/).
