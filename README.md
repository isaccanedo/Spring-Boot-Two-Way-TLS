# 2 way SSL with spring boot

Exemplo de código de cliente e servidor para mostrar como obter a configuração de SSL de 2 vias com certificado autoassinado.

# 1. Introdução 
Nos últimos anos, o microsserviço mudou o panorama do desenvolvimento de aplicativos. Ele aumentou drasticamente a velocidade de desenvolvimento, implantação e tempo de lançamento no mercado. Embora o microsserviço tenha muitas virtudes para desenvolvedores de aplicativos, há certos itens importantes que precisamos pensar quando se trata de como dois ou mais microsserviços se comunicam entre si com segurança.


Proteger um aplicativo da web se tornou a parte mais crítica do desenvolvimento. Na verdade, muitas organizações estão adotando o princípio de design da Segurança e forçando as equipes de desenvolvimento de aplicativos a criarem uma arquitetura construída em torno da segurança de ponta a ponta de seus aplicativos. Hoje, vou falar sobre uma dessas abordagens - SSL de 2 vias - com relação a microsserviços construídos usando Spring Boot.

# 2. O que é SSL bidirecional?
Tradicionalmente, a maioria de nós está familiarizada com SSL unilateral. Nesse formulário, o servidor apresenta seu certificado ao cliente e o cliente o adiciona à sua lista de certificados confiáveis. E assim, o cliente pode conversar com o servidor.
SSL bidirecional é o mesmo princípio, mas em ambos os sentidos. ou seja, tanto o cliente quanto o servidor devem estabelecer confiança entre si usando um certificado confiável. Nessa forma de handshake digital, o servidor precisa apresentar um certificado para se autenticar ao cliente e o cliente deve apresentar seu certificado ao servidor. O diagrama abaixo pode ajudá-lo a entendê-lo um pouco melhor.

# 3. O que estamos usando?

```
- Java 1.8
- Spring Boot 2.1.2
- keytool - já vem com a instalação do jdk.
```


# 4. Aplicação
Criaremos 2 aplicativos Spring Boot. Idealmente, podemos chamá-lo de cliente e servidor, mas apenas porque estamos usando o spring boot e de acordo com o princípio do microsserviço, é melhor ter um aplicativo de gateway voltado para todos os microsserviços subjacentes.

Iremos nos referir ao cliente como gateway e o servidor como ms - para microsserviço, obviamente.
Todos estão familiarizados com a configuração de um aplicativo Spring Boot simples, então, vou pular essa parte aqui e ir direto ao que precisamos para fazer com que ele se comunique usando SSL de 2 vias.

# 5. Criação dos Certificados

### 5.1. Criar um certificado de cliente autoassinado

Usaremos o comando da ferramenta-chave para isso.

```
keytool -genkeypair -alias nt-gateway -keyalg RSA -keysize 2048 -storetype JKS -keystore nt-gateway.jks -validity 3650 -ext SAN=dns:localhost,ip:127.0.0.1
```

A última parte no comando da ferramenta principal é muito crítica, pois o certificado autoassinado criado sem entradas de SAN não funciona com o Chrome e Safari.

### 5.2. Criar certificado de servidor autoassinado

```
keytool -genkeypair -alias nt-ms -keyalg RSA -keysize 2048 -storetype JKS -keystore nt-ms.jks -validity 3650 -ext SAN=dns:localhost,ip:127.0.0.1
```



### 5.3. Crie um arquivo de certificado público a partir do certificado do cliente

Agora que criamos certificados de cliente e servidor, precisamos configurar a confiança entre os dois. Para fazer isso, importaremos o certificado do cliente para os certificados confiáveis do servidor e vice-versa. Mas antes de fazermos isso, precisamos extrair o certificado público de cada arquivo jks.

```
keytool -export -alias nt-gateway -file nt-gateway.crt -keystore nt-gateway.jks
Enter keystore password:
Certificado armazenado em arquivo <nt-gateway.crt>
```



### 5.4. Criar arquivo de certificado público a partir do certificado do servidor

```
keytool -export -alias nt-ms -file nt-ms.crt -keystore nt-ms.jks
Enter keystore password:
Certificado armazenado em arquivo <nt-ms.crt>
```

Agora, teremos que importar o certificado do cliente para o armazenamento de chaves do servidor e o certificado do servidor para o arquivo de armazenamento de chaves do cliente.

### 5.5. Importar certificado do cliente para o arquivo jks do servidor

```
keytool -import -alias nt-gateway -file nt-gateway.crt -keystore nt-ms.jks
```



### 5.6. Importar certificado do servidor para o arquivo jks do cliente

```
keytool -import -alias nt-ms -file nt-ms.crt -keystore nt-gateway.jks
```


# 6. Configure o servidor para SSL de 2 vias

- Copie o arquivo jks do servidor final (no meu caso, nt-ms.jks) para a pasta src / main / resources / do aplicativo nt-ms.
- Adicione as entradas mostradas abaixo em application.yml (ou application.properties. Mas eu prefiro .yml)
- Crie uma classe de controlador com ponto de extremidade REST para atender à solicitação de entrada

E é basicamente isso do lado do servidor.

# 7. Configure o cliente para SSL de 2 vias:

Agora, isso requer mais algumas mudanças do que o lado do servidor, já que a comunicação https será iniciada a partir daqui. Mas não se preocupe. Vamos passo a passo.
- Primeiro, copie jks do cliente final (no meu caso, nt-gateway.jks) para src / main / resources / pasta
- Em seguida, adicione as entradas mostradas abaixo em application.yml
- Precisamos adicionar a dependência abaixo em nosso pom. Não se preocupe, saberemos como usá-los na próxima etapa.
- Felizmente, o Spring Boot vem com uma classe RestTemplate legal para comunicação http. Usaremos essa classe para nossa chamada https do aplicativo cliente para o servidor. E como vamos com SSL de 2 vias, precisamos configurar este RestTemplate para usar o armazenamento confiável com certificado de servidor.
- Assim que terminarmos com isso, vamos adicionar uma propriedade em application.yml para informar a url para o endpoint do aplicativo do servidor chamar.
- Depois disso, vamos criar uma classe de controlador com 2 métodos:

E isso é basicamente tudo na configuração do aplicativo cliente.

# 8. Executando o aplicativo

Compile o código do cliente e do servidor e inicie o aplicativo. Você verá que ambos os aplicativos iniciarão na porta definida em application.yml com https

# 9. Problemas
Mas ainda há um problema. Como você testa isso? Se você tentar acessar o aplicativo com https://url localhost, seu navegador reclamará sobre a necessidade do certificado do cliente !!! Por quê? Acessamos todos esses aplicativos https no mundo até agora sem nenhum problema. Então, o que há de tão especial em nosso aplicativo?

Porque é SSL de 2 vias. Quando acessamos o url do gateway no navegador, nosso navegador se torna o cliente de nosso aplicativo de gateway e, portanto, nosso aplicativo da web de gateway solicitará ao navegador que apresente um certificado para autenticação.

# 10. Solução

Para superar isso, teremos que importar um certificado para o nosso navegador. Mas nosso navegador não consegue entender um arquivo .jks. Em vez disso, ele entende o formato PKCS12. Então, como convertemos o arquivo .jks para o formato PKCS12? Novamente, o comando keytool para o resgate !!

```
keytool -importkeystore -srckeystore nt-ms.jks -destkeystore nt-ms.p12 -srcstoretype JKS -deststoretype PKCS12 -srcstorepass nt-service -deststorepass nt-service -srcalias nt-ms -destalias nt-ms -srckeypass nt-service -destkeypass nt-service -noprompt
```

Para importar este arquivo .p12 no mac, você precisará importá-lo nas chaves de login.
- Acesso de chaveiro aberto
- Clique em login em “chaveiros” e “Certificados” na categoria
- Arraste e solte o arquivo .p12 aqui. Ele solicitará a senha do arquivo .p12. Entre e adicione.
- Clique duas vezes no certificado que você acabou de enviar e em "Confiar" e selecione a opção "Sempre confiar". Isso solicitará a senha do keychain de login. Entre e prossiga.
- Feche as janelas do navegador, abra e limpe os cookies / cache e clique em https: // localhost: 9001 / nt-gw / ms-data e você receberá um aviso de erro “Conexão não privada”.
- Clique em “Mostrar detalhes”
- Clique em “visite este site” e você verá a tela abaixo.
- Uma etapa final. Você precisará inserir sua senha de login do mac.
- E é isso. Você poderá carregar o método. E se você verificar os logs de seu gateway e aplicativo ms, verá entradas de depuração apropriadas.

# 11. Conclusão
Neste breve tutorial, aprendemos a configurar uma conexão SSL de 2 vias com certificado autoassinado.



