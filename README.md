# WebClient OAuth2 Token SSL certificate

## Configuration yaml
```$xslt
spring:
  security:
    oauth2:
      client:
        registration:
          example:
            client-id: "your client id"
            client-secret: "your client credentials"
            client-authentication-method: basic
            authorization-grant-type: client_credentials
            scope: "scopes"
          xiang:
            client-id: "your client id"
            client-secret: "your client credentials"
            client-authentication-method: basic
            authorization-grant-type: client_credentials
            scope: "scopes"
        provider:
          xiang:
            token-uri: https://magiciendecode.fr
          example:
            token-uri: https://magiciendecode.fr
```

## WebClient
* build an oauth2WebClient
```$xslt
    @Bean
    fun oAuth2AuthorizedClientManager(
            clientRegistrationRepository: ClientRegistrationRepository,
            oAuth2AuthorizedClientRepository: OAuth2AuthorizedClientRepository,
            sslTokenRestTemplate: RestTemplate
    ): OAuth2AuthorizedClientManager {
        val oAuth2AuthorizedClientProvider = OAuth2AuthorizedClientProviderBuilder
                .builder()
                .clientCredentials()
                .build()

        val authorizedClientManager =
                DefaultOAuth2AuthorizedClientManager(clientRegistrationRepository, oAuth2AuthorizedClientRepository)
        authorizedClientManager.setAuthorizedClientProvider(oAuth2AuthorizedClientProvider)

        return authorizedClientManager
    }
```
```$xslt
    @Bean
    fun oauth2webClient(oAuth2AuthorizedClientManager: OAuth2AuthorizedClientManager): WebClient {
        val oAuth2Client = ServletOAuth2AuthorizedClientExchangeFilterFunction(oAuth2AuthorizedClientManager)

        return WebClient.builder()
                .apply(oAuth2Client.oauth2Configuration())
                .build()
    }
```
use case
```$xslt
    val result = oauth2webClient.get()
            .uri("uri")
            .attributes(clientRegistrationId("xiang"))
            .retrieve()
            .bodyToMono(String::class.java)
            .block()
```
* build an oauth2WebClient with certificate / sslOauth2webClient
As `DefaultClientCredentialsTokenResponseClient` use `RestTemplate`,
if we want to use sslOauth2webClient, we need inject a sslRestTemplate
```$xslt
    @Bean
    fun httpComponentsClientHttpRequestFactory(): HttpComponentsClientHttpRequestFactory {
        val clientStore = KeyStore.getInstance("PKCS12")
        val password = keyStorePassword.toCharArray()
        clientStore.load(certificate.inputStream, password)
        val sslContext = SSLContextBuilder()
            .loadKeyMaterial(clientStore, password)
            .build()
        val sslConnectionSocketFactory = SSLConnectionSocketFactory(sslContext)
        val httpClient = HttpClients.custom().setSSLSocketFactory(sslConnectionSocketFactory).build()
        return HttpComponentsClientHttpRequestFactory(httpClient)
    }
```
```$xslt
    @Bean
    fun sslTokenRestTemplate(
        builder: RestTemplateBuilder,
        httpComponentsClientHttpRequestFactory: HttpComponentsClientHttpRequestFactory
    ): RestTemplate {

        val restTemplate = RestTemplate(
            listOf(
                FormHttpMessageConverter(),
                OAuth2AccessTokenResponseHttpMessageConverter()
            )
        )

        restTemplate.requestFactory = httpComponentsClientHttpRequestFactory
        return restTemplate
    }
```
According to documentation, use custom client `sslClientCredentialsTokenResponseClient`
```$xslt
    @Bean
    fun sslOAuth2AuthorizedClientManager(
            clientRegistrationRepository: ClientRegistrationRepository,
            oAuth2AuthorizedClientRepository: OAuth2AuthorizedClientRepository,
            sslTokenRestTemplate: RestTemplate
    ): OAuth2AuthorizedClientManager {

        val sslClientCredentialsTokenResponseClient = DefaultClientCredentialsTokenResponseClient()
        sslClientCredentialsTokenResponseClient.setRestOperations(sslTokenRestTemplate)

        val oAuth2AuthorizedClientProvider = OAuth2AuthorizedClientProviderBuilder
                .builder()
                .clientCredentials { configurer ->
                    configurer.accessTokenResponseClient(
                            sslClientCredentialsTokenResponseClient
                    )
                }
                .build()

        val authorizedClientManager =
                DefaultOAuth2AuthorizedClientManager(clientRegistrationRepository, oAuth2AuthorizedClientRepository)
        authorizedClientManager.setAuthorizedClientProvider(oAuth2AuthorizedClientProvider)

        return authorizedClientManager
    }
```
```$xslt
    @Bean
    fun sslOauth2webClient(sslOAuth2AuthorizedClientManager: OAuth2AuthorizedClientManager): WebClient {
        val oAuth2Client = ServletOAuth2AuthorizedClientExchangeFilterFunction(sslOAuth2AuthorizedClientManager)
        val clientHttpConnector = buildClientHttpConnector(certificate.inputStream, keyStorePassword)

        return WebClient.builder()
                .clientConnector(clientHttpConnector)
                .apply(oAuth2Client.oauth2Configuration())
                .build()
    }
```

## RestTemplate
```$xslt
    @Bean
    fun httpComponentsClientHttpRequestFactory(): HttpComponentsClientHttpRequestFactory {
        val clientStore = KeyStore.getInstance("PKCS12")
        val password = keyStorePassword.toCharArray()
        clientStore.load(certificate.inputStream, password)
        val sslContext = SSLContextBuilder()
            .loadKeyMaterial(clientStore, password)
            .build()
        val sslConnectionSocketFactory = SSLConnectionSocketFactory(sslContext)
        val httpClient = HttpClients.custom().setSSLSocketFactory(sslConnectionSocketFactory).build()
        return HttpComponentsClientHttpRequestFactory(httpClient)
    }
```
RestTemplate with certificate / sslRestTemplate
```$xslt
    @Bean
    fun sslRestTemplate(
        builder: RestTemplateBuilder,
        httpComponentsClientHttpRequestFactory: HttpComponentsClientHttpRequestFactory
    ): RestTemplate {
        return builder.requestFactory { httpComponentsClientHttpRequestFactory }.build()
    }
```
Normal RestTemplate
```$xslt
    @Bean
    fun restTemplate(builder: RestTemplateBuilder): RestTemplate =
        builder.requestFactory { HttpComponentsClientHttpRequestFactory() }.build()
```

### Reference Documentation
* [Spring Security 5.2 Reference](https://docs.spring.io/spring-security/site/docs/current/reference/)

### Additional Links
These additional references should also help you:
* [Magicien De Code](https://magiciendecode.fr)

