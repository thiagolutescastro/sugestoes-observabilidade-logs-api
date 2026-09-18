# Sugestões para Implementação de Logs de Startup e Acesso em APIs

## Resultado esperado

```text
📋 [endpoint] [pvd] - PUT /api/sessao/transbordo - SessaoController.realizarTransbordoSessoes
📋 [endpoint] [pub] - GET /api/public/email/descadastro - DescadastroEmailController.descadastrar
````
```

## 1. Alterar `WebSecurity.java`

### O que faz

Adicionar a lista estática de strings `PUBLIC_PATH_PATTERNS`, reutilizando os mesmos padrões já utilizados no método `permitAll()` do filtro de segurança.

### Exemplo

```
````java
public static final List<String> PUBLIC_PATH_PATTERNS = List.of(
    SecurityConstants.SIGN_UP_URL,
    SecurityConstants.PUBLIC_URL,
    "/api/usuario/esqueci-senha/**",
    "/api/usuario/email/**",
    "/actuator/**",
    "/h2-console/**",
    "/favicon.ico",
    "/wss-socket/**",
    "/swagger-ui.html",
    "/v3/api-docs/**",
    "/swagger-ui/**"
);
````
```

## 2. Criar o `record` `EndpointInfoDTO.java`

Adicionar um objeto com os campos utilizados na representação do log:

- `metodoHttp`
- `uri`
- `controller`
- `metodoJava`
- `visibilidade`

### Exemplo

```
````java
package br.com.claribot.model.DTO.observability;

public record EndpointInfoDTO(
    String metodoHttp,
    String uri,
    String controller,
    String metodoJava,
    String visibilidade
) {}
````
```

## 3. Criar a interface e a implementação do serviço

Sugestões de nomes:

- `EndpointInventoryService.java`
- `EndpointInventoryServiceImpl.java`

A interface deve conter o método `listarTodos()`, que retorna uma `List<EndpointInfoDTO>`.

O serviço deve:

1. Ler o `RequestMappingHandlerMapping`, sem chamar nenhum endpoint.
2. Montar uma lista com todos os endpoints registrados.
3. Classificar cada endpoint como `pub` (público) ou `pvd` (privado), comparando suas URIs com `WebSecurity.PUBLIC_PATH_PATTERNS`.
4. Ordenar os resultados por controller, método Java, URI e método HTTP.

### Interface do serviço

```
````java
package br.com.claribot.service.observability;

import java.util.List;

public interface EndpointInventoryService {

    List<EndpointInfoDTO> listarTodos();
}
````
```

### Implementação do serviço

```
````java
package br.com.claribot.service.impl.observability;

import ...;

@Service
@RequiredArgsConstructor
public class EndpointInventoryServiceImpl implements EndpointInventoryService {

    private static final String VISIBILIDADE_PUBLICA = "pub";
    private static final String VISIBILIDADE_PRIVADA = "pvd";

    private final RequestMappingHandlerMapping requestMappingHandlerMapping;
    private final AntPathMatcher antPathMatcher = new AntPathMatcher();

    @Override
    public List<EndpointInfoDTO> listarTodos() {
        Map<RequestMappingInfo, HandlerMethod> mapeamentos =
                requestMappingHandlerMapping.getHandlerMethods();

        List<EndpointInfoDTO> endpoints = new ArrayList<>();

        for (Map.Entry<RequestMappingInfo, HandlerMethod> entry : mapeamentos.entrySet()) {
            HandlerMethod handlerMethod = entry.getValue();
            String controller = handlerMethod.getBeanType().getSimpleName();
            String metodoJava = handlerMethod.getMethod().getName();

            for (String metodoHttp : metodosHttp(entry.getKey())) {
                for (String uri : uris(entry.getKey())) {
                    endpoints.add(new EndpointInfoDTO(
                            metodoHttp,
                            uri,
                            controller,
                            metodoJava,
                            visibilidade(uri)
                    ));
                }
            }
        }

        endpoints.sort(
                Comparator.comparing(EndpointInfoDTO::controller)
                        .thenComparing(EndpointInfoDTO::metodoJava)
                        .thenComparing(EndpointInfoDTO::uri)
                        .thenComparing(EndpointInfoDTO::metodoHttp)
        );

        return endpoints;
    }

    private String visibilidade(String uri) {
        boolean publico = WebSecurity.PUBLIC_PATH_PATTERNS.stream()
                .anyMatch(padrao -> antPathMatcher.match(padrao, uri));

        return publico ? VISIBILIDADE_PUBLICA : VISIBILIDADE_PRIVADA;
    }

    private Iterable<String> metodosHttp(RequestMappingInfo info) {
        var metodos = info.getMethodsCondition().getMethods();

        if (metodos.isEmpty()) {
            return List.of("ANY");
        }

        return metodos.stream()
                .map(Enum::name)
                .toList();
    }

    private Iterable<String> uris(RequestMappingInfo info) {
        var pathPatterns = info.getPathPatternsCondition();

        if (pathPatterns != null && !pathPatterns.getPatterns().isEmpty()) {
            return pathPatterns.getPatterns()
                    .stream()
                    .map(PathPattern::getPatternString)
                    .toList();
        }

        var patterns = info.getPatternsCondition();

        if (patterns != null && !patterns.getPatterns().isEmpty()) {
            return patterns.getPatterns();
        }

        return List.of("/");
    }
}
````
```

> **Observação:** Os imports foram representados por `import ...;` e devem ser ajustados conforme a estrutura do projeto.

## 4. Criar a classe `EndpointInventoryStartupLogger.java`

### O que faz

Ao receber o evento `ApplicationReadyEvent`, percorre o `EndpointInventoryService` e registra uma linha de log para cada endpoint no seguinte formato:

```
````text
📋 [endpoint] [pub/pvd] - MÉTODO uri - Controller.método
````
```

### Exemplo

```
````java
package br.com.claribot.config.observability;

import ...;

@Slf4j
@Component
@RequiredArgsConstructor
public class EndpointInventoryStartupLogger {

    private final EndpointInventoryService endpointInventoryService;

    @EventListener(ApplicationReadyEvent.class)
    public void logarInventario() {
        for (EndpointInfoDTO endpoint : endpointInventoryService.listarTodos()) {
            log.info(
                    "📋 [endpoint] [{}] - {} {} - {}.{}",
                    endpoint.visibilidade(),
                    endpoint.metodoHttp(),
                    endpoint.uri(),
                    endpoint.controller(),
                    endpoint.metodoJava()
            );
        }
    }
}
````
```

## 5. Criar a classe `RequestAccessLogInterceptor.java`

### O que faz

Criar um `HandlerInterceptor` que registra cada requisição real no seguinte formato:

```
````text
✅/❌ [request] - MÉTODO uri - Controller.método Status {código}-{NOME}
````
```

- `✅`: requisição concluída com sucesso.
- `❌`: requisição com erro.
- O status é exibido no formato `código-NOME`, por exemplo, `200-OK` ou `404-NOT_FOUND`.

### Exemplo

```
````java
package br.com.claribot.config.observability;

import ...;

@Slf4j
@Component
public class RequestAccessLogInterceptor implements HandlerInterceptor {

    @Override
    public void afterCompletion(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            Exception ex
    ) {
        String endpointNome = handler instanceof HandlerMethod handlerMethod
                ? handlerMethod.getBeanType().getSimpleName()
                        + "." + handlerMethod.getMethod().getName()
                : String.valueOf(handler);

        int statusCode = response.getStatus();
        boolean sucesso = statusCode < 400 && ex == null;
        String icone = sucesso ? "✅" : "❌";
        String status = statusCode + "-" + nomeStatus(statusCode);

        log.info(
                "{} [request] - {} {} - {} Status {}",
                icone,
                request.getMethod(),
                request.getRequestURI(),
                endpointNome,
                status
        );
    }

    private String nomeStatus(int statusCode) {
        try {
            return org.springframework.http.HttpStatus.valueOf(statusCode).name();
        } catch (IllegalArgumentException e) {
            return "UNKNOWN";
        }
    }
}
````
```

## 6. Criar a classe `ObservabilityWebConfig.java`

### O que faz

Criar uma implementação de `WebMvcConfigurer` para registrar o `RequestAccessLogInterceptor` globalmente.

### Exemplo

```
````java
package br.com.claribot.config.observability;

import ...;

@Configuration
@RequiredArgsConstructor
public class ObservabilityWebConfig implements WebMvcConfigurer {

    private final RequestAccessLogInterceptor requestAccessLogInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(requestAccessLogInterceptor);
    }
}
```

## Fluxo esperado

1. Durante a inicialização da aplicação, o `EndpointInventoryStartupLogger` lista e registra todos os endpoints encontrados.
2. A classificação de visibilidade utiliza os padrões públicos já definidos no `WebSecurity`.
3. Durante a execução da aplicação, o `RequestAccessLogInterceptor` registra cada requisição recebida.
4. Os logs podem ser coletados e visualizados posteriormente no Grafana, conforme a configuração de observabilidade da aplicação.
