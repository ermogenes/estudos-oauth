# Segurança de Aplicações: Identidade

## O problema

Ao acessar uma API, uma aplicação solicita ao usuário sua identificação e sua senha, via UI. As credenciais são conferidas contra uma _whitelist_, e o usuário então é autenticado e pode receber as suas informações. As credenciais precisam ser mantidas no cliente e reenviadas/reconferidas a cada solicitação (através de _cookies_ de sessão, por exemplo). O processo é conhecido como _Basic Auth_.

Esse processo é especialmente ingênuo quando pensamos em integrar o _login_ com outras aplicações (SSO - _Single Sign On_), como aplicações de terceiros (um cliente de Twitter, ou uma aplicação web que salve no seu Google Drive, por exemplo). Não parece seguro informar suas credenciais para terceiros que devem armazená-las e mantê-las seguras. A cada mudança de senha será necessário atualizá-las em todos os terceiros.

Surgiu a necessidade de se disponibilizar acesso a APIs sem que os clientes necessitem guardar as credenciais do usuário, ou sequer conhecê-las.

Além disso, cada mudança no processo de autenticação (como a adição de mecanismo de MFA, por exemplo) exige a alteração de todas as aplicações envolvidas.

Nesse cenário, nasce o padrão OAuth em 2007. Em 2012 a segunda versão foi finalizada, simplificando o processo para o desenvolvedor e resolvendo questões relevantes para a segurança de aplicações _mobile_ nativas.

## Autenticação e Autorização

A ideia principal por trás da solução trazida pelo OAuth2 é que sempre que uma aplicação necessitar identificar um usuário, ela não solicitará suas credenciais diretamente, mas o redirecionará para um terceiro confiável, chamado de _servidor de identidade_. Ele disponibilizará um UI padronizada para fornecimento de credenciais, e retornará para a aplicação solicitante uma informação de sucesso/falha, e uma maneira de continuar acessando os recursos por um período de tempo (chamada de _Access Token_).

Dessa forma, as aplicações podem utilizar recursos das APIs sem conhecer as credenciais do usuário, nem o processo de autenticação.

Pode parecer estranho, mas pense no exemplo de uma locadora de automóveis. Você se identifica ao alugar o carro, e recebe uma chave, que te dá acesso ao carro. Nem a chave nem o carro sabem quem é você, ou quais documentos você mostrou para consegui-los.

Caso seja necessário que a aplicação cliente obtenha informações sobre o usuário autenticado (o chamado _ID Token_), é necessário que o servidor de identidade também dê suporte a outra tecnologia associada ao OAuth2, chamada OpenID Connect.

Assim, tenha em mente:

A tecnologia | | faz | que trada de
--- | ---| --- | ---
<img src="oauth-icon.png" alt="OAuth" height="50"> | OAuth | autenticação | _acessar APIs_
<img src="openid-icon.png" alt="OpenID" height="50"> | OpenID Connect | autorização | _identificar usuários_

## Os atores e seus papéis

- **Usuário** ou **_Resource Owner_**: o proprietário da conta.
- **Dispositivo** ou **_User Agent_**: (telefone, TV, navegador) utilizado pelo usuário para executar ou acessar uma aplicação.
- **Aplicação** ou **_Client_**: app, _site_ ou outro software que é utilizado pelo usuário no dispositivo, e que acessa recursos em uma API.
- **API** ou **_Resource Server_**: entrega recursos à aplicações, à pedido do usuário. Local onde os dados estão armazenados.
- **Servidor de identidade** ou **_Authorization server_**: identifica o usuário de forma independente e retorna um _access token_ à aplicação, que pode utilizá-lo para solicitar recursos à API.

Exemplo 1: Maria (_resource owner_) usa o Chrome (_user agent_) para acessar o site XYZ (_client_), que usa segurança baseada em _cookies_. Ele exibe uma tela própria solicitando as credenciais do usuário (usuário e senha), que são enviadas para o servidor _backend_ (_API_, no papel de _authorization server_), que valida e retorna um _cookie de sessão_. Esse _cookie_ é reenviado a cada solicitação de dados ao _backend_ (_API_, no papel de _resource server_).

Exemplo 2: Tereza (_resource owner_) usa o Firefox (_user agent_) para acessar o site JKL (_client_) que usa segurança OAuth, e clica em 'Entrar com o Google'. Ele redireciona o usuário para o Google (_authorization server_) que mostra uma tela solicitando as credenciais do usuário (usuário e senha). Após validação, retorna um _access token_ ao redirecionar de volta ao _client_. Esse _token_ é reenviado a cada solicitação de dados ao _backend_ (_API_, no seu papel funcional), que possui algum mecanismo para verificar se o _access token_ é válido.

### Tipos de _clients_

_Clients_ diferem em relação à sua habilidade de armazenar algum tipo de credencial em seu _deploy_ que possa ser utilizada para verificação de identidade junto ao _authorization server_. Essas credenciais são chamadas _client secrets_ e não podem ser acessíveis pelos usuários das aplicações, logados ou não.

- _Confidential Clients_: aplicações implantadas no _backend_ (como PHP, .NET ou Python, por exemplo) que podem guardar segredos em forma de _API Keys_ em arquivos de configuração ou variáveis de ambiente.
- _Public Clients_: aplicações que não tem essa capacidade, pois rodam nos dispositivos dos usuários, como um app _mobile_, uma SPA, um app de TV ou um software rodando em um dispositivo IoT.
- _Credentialed Clients_: aplicações que confirmam sua identidade através de uma chave privada e um certificado únicos no cliente para obter _tokens_ vinculados unicamente a esse cliente.

_Confidential Clients_ podem solicitar autorização a um _authorization server_ usando suas credenciais secretas únicas. Esses segredos permitem que o cliente seja identificado exclusivamente, sem a interação com o usuário. Sem eles, não há como saber se a chamada veio da aplicação cliente real, ou de alguém se passando por ela.

Caso os segredos vazem, um atacante pode utilizá-los para se passar pelo cliente. É importante que a URL de redirecionamento de volta ao cliente também não esteja sob controle de um atacante.

Os _authorization servers_ podem ter políticas diferenciadas para clientes confidenciais e públicos. Por exemplo, solicitar as credenciais ao usuário no momento do login, possivelmente utilizando mais de um fator, em caso de _public clients_. Também é comum incluir tempos de expiração nos _access tokens_ e alguma regra envolvendo _refresh tokens_ para manter o usuário logado por períodos prolongados.

Um exemplo de _credentialed client_ é um app _mobile_ instalado via loja de aplicativos. No seu primeiro uso, ele ainda não pode ser autenticado, pois ele não pode armazenar um segredo no momento do _deploy_. Ele faz um processo conhecido como _dynamic client registration_ para conseguir um segredo único, que fica armazenado somente naquele dispositivo e pode ser utilizado para identificar chamadas daquela aplicação naquele dispositivo específico. O cliente ainda pode não estar identificado, mas é possível garantir que todas as solicitações com o mesmo _token_ vieram do mesmo cliente.

<!-- ### _User consent_

Password grant
resource owner password flow -->