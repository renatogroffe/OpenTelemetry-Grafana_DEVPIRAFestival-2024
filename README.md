# OpenTelemetry-Grafana_DEVPIRAFestival-2024
Conteúdos sobre OpenTelemetry + Grafana de apresentação realizada durante a edição 2024 do DEVPIRA Festival.

---

Título da apresentação: **Observabilidade e Monitoramento de Aplicações com OpenTelemetry + stack Grafana**

Data: **07/12/2024 (sábado)**

Tecnologias e tópicos abordados: **OpenTelemetry, Grafana, Docker, Docker Compose, Linux, Grafana Tempo, Loki, Prometheus, PostgreSQL, .NET, ASP.NET Core...**

Número de participantes: **50 pessoas (estimativa)**

O DEVPIRA Festival é um evento presencial promovido pelo pessoal do DEVPIRA, uma das comunidades técnicas mais ativas do interior de São Paulo. A cada são centenas de participantes e dezenas de palestrantes, abordando tecnologias e práticas em alta no mercado de tecnologia.

Site do evento: **https://www.devpira.com.br/eventos/devpira-festival-2024**

Local: **PECEGE - Rua Cezira Giovanoni Moretti, 580 - Santa Rosa - Piracicaba-SP - CEP: 13414-157**

Deixo aqui meus agradecimentos ao **Alexandre Ballestero**, ao **Fábio Baldin**, ao **Murilo Beltrame** e demais organizadores por todo o apoio para que eu partipasse como palestrante de mais uma edição do **DEVPIRA Festival**.

---

![Renato palestrando 1](img/og-09.jpeg)

![Renato palestrando 2](img/og-10.jpeg)

![Renato palestrando 3](img/og-11.jpeg)

![Renato palestrando 4](img/og-03.jpeg)

![Renato palestrando 5](img/og-01.jpeg)

![Renato palestrando 5](img/og-02.jpeg)

![Divulgação](img/divulgacao.png)

![Programação](img/programacao.png)

Outras fotos podem ser encontrados neste [**diretório**](/img/).

---

## Implementação - Detalhes

### Ambiente - Docker Compose

Arquivo docker-compose.yaml:

```yaml
# Exemplo utilizado como base para a criacao deste script: https://github.com/grafana/tempo/blob/main/example/docker-compose/otel-collector/readme.md
# Documentacao: https://grafana.com/docs/tempo/latest/getting-started/docker-example/
# Configurando o acesso a logs a partir de um trace (arquivo grafana-datasources.yaml): https://grafana.com/docs/grafana/latest/datasources/tempo/configure-tempo-data-source/#provision-the-data-source

services:

  # Tempo runs as user 10001, and docker compose creates the volume as root.
  # As such, we need to chown the volume in order for Tempo to start correctly.
  init:
    image: &tempoImage grafana/tempo:latest
    user: root
    entrypoint:
      - "chown"
      - "10001:10001"
      - "/var/tempo"
    volumes:
      - ../tempo-data:/var/tempo

  tempo:
    image: *tempoImage
    command: [ "-config.file=/etc/tempo.yaml" ]
    volumes:
      - ./tempo.yaml:/etc/tempo.yaml
      - ../tempo-data:/var/tempo
    ports:
      #- "14268"  # jaeger ingest
      - "3200:3200"   # tempo
      - "4317:4317"  # otlp grpc
      - "4318:4318"  # otlp http
      #- "9411"   # zipkin
    depends_on:
      - init

  # And put them in an OTEL collector pipeline...
  otel-collector:
    image: otel/opentelemetry-collector:0.86.0
    command: [ "--config=/etc/otel-collector.yaml" ]
    volumes:
      - ./otel-collector.yaml:/etc/otel-collector.yaml

  prometheus:
    image: prom/prometheus:latest
    command:
      - --config.file=/etc/prometheus.yaml
      - --web.enable-remote-write-receiver
      - --enable-feature=exemplar-storage
      - --enable-feature=native-histograms
    volumes:
      - ./prometheus.yaml:/etc/prometheus.yaml
    ports:
      - "9090:9090"

  loki:
    image: grafana/loki:2.9.2
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:2.9.2
    volumes:
      - /var/log:/var/log
    command: -config.file=/etc/promtail/config.yml

  grafana:
    image: grafana/grafana:11.0.0
    volumes:
      - ./grafana-datasources.yaml:/etc/grafana/provisioning/datasources/datasources.yaml
    environment:
      - GF_PATHS_PROVISIONING=/etc/grafana/provisioning
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_AUTH_DISABLE_LOGIN_FORM=true
      - GF_FEATURE_TOGGLES_ENABLE=traceqlEditor
    ports:
      - "3000:3000"

  postgres:
    image: postgres
    volumes:
      - ./BaseContagemPostgreSql.sql:/docker-entrypoint-initdb.d/1-basecontagem.sql
    environment:
      POSTGRES_PASSWORD: "Postgres2024!"
    ports:
      - "5432:5432"
```

Arquivo grafana-datasources.yaml:

```yaml
apiVersion: 1

datasources:
- name: Tempo
  type: tempo
  access: proxy
  orgId: 1
  url: http://tempo:3200
  basicAuth: false
  isDefault: true
  version: 1
  editable: false
  apiVersion: 1
  uid: tempo
  jsonData:
    httpMethod: GET
    serviceMap:
      datasourceUid: prometheus
    tracesToLogsV2:
      # Field with an internal link pointing to a logs data source in Grafana.
      # datasourceUid value must match the uid value of the logs data source.
      datasourceUid: 'Loki'
      spanStartTimeShift: '-1h'
      spanEndTimeShift: '1h'
      filterByTraceID: true
      filterBySpanID: true
      customQuery: false
      query: 'method="$${__span.tags.method}"'
- name: Prometheus
  type: prometheus
  uid: prometheus
  access: proxy
  orgId: 1
  url: http://prometheus:9090
  basicAuth: false
  isDefault: false
  version: 1
  editable: false
  jsonData:
    httpMethod: GET
- name: Loki
  type: loki
  access: proxy 
  orgId: 1
  url: http://loki:3100
  basicAuth: false
  isDefault: false
  version: 1
  editable: false
  jsonData:
    derivedFields:
      - datasourceUid: tempo
        matcherRegex: tid=(\w+)
        name: TraceId
        url: $${__value.raw}
```

Arquivo otel-collector.yaml:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
exporters:
  otlp:
    endpoint: tempo:4317
    tls:
      insecure: true
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlp]
```

Arquivo prometheus.yaml:

```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: [ 'localhost:9090' ]
  - job_name: 'tempo'
    static_configs:
      - targets: [ 'tempo:3200' ]
```

Arquivo tempo.yaml:

```yaml
stream_over_http_enabled: true
server:
  http_listen_port: 3200
  log_level: info

query_frontend:
  search:
    duration_slo: 5s
    throughput_bytes_slo: 1.073741824e+09
  trace_by_id:
    duration_slo: 5s

distributor:
  receivers:                           # this configuration will listen on all ports and protocols that tempo is capable of.
    jaeger:                            # the receives all come from the OpenTelemetry collector.  more configuration information can
      protocols:                       # be found there: https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver
        thrift_http:                   #
        grpc:                          # for a production deployment you should only enable the receivers you need!
        thrift_binary:
        thrift_compact:
    zipkin:
    otlp:
      protocols:
        http:
        grpc:
    opencensus:

ingester:
  max_block_duration: 5m               # cut the headblock when this much time passes. this is being set for demo purposes and should probably be left alone normally

compactor:
  compaction:
    block_retention: 1h                # overall Tempo trace retention. set for demo purposes

metrics_generator:
  registry:
    external_labels:
      source: tempo
      cluster: docker-compose
  storage:
    path: /var/tempo/generator/wal
    remote_write:
      - url: http://prometheus:9090/api/v1/write
        send_exemplars: true
  traces_storage:
    path: /var/tempo/generator/traces

storage:
  trace:
    backend: local                     # backend configuration to use
    wal:
      path: /var/tempo/wal             # where to store the wal locally
    local:
      path: /var/tempo/blocks

overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics, local-blocks] # enables metrics generator
      generate_native_histograms: both
```

### API que utiliza PostgreSQL

Packages utilizados:
- Microsoft.AspNetCore.OpenApi
- Scalar.AspNetCore
- Npgsql.EntityFrameworkCore.PostgreSQL
- Npgsql.EntityFrameworkCore.PostgreSQL.NodaTime
- Npgsql.OpenTelemetry
- OpenTelemetry.Exporter.Console
- OpenTelemetry.Exporter.OpenTelemetryProtocol
- OpenTelemetry.Extensions.Hosting
- OpenTelemetry.Instrumentation.AspNetCore
- OpenTelemetry.Instrumentation.Http
- OpenTelemetry.Instrumentation.Runtime
- Serilog.AspNetCore
- Serilog.Enrichers.Span
- Serilog.Sinks.Grafana.Loki

Configuração do OpenTelemetry e Loki:

```csharp
builder.Services.AddSerilog(new LoggerConfiguration()
    .WriteTo.Console()
    .WriteTo.GrafanaLoki(
        builder.Configuration["Loki:Uri"]!,
        new List<LokiLabel>()
        {
            new()
            {
                Key = "service_name",
                Value = OpenTelemetryExtensions.ServiceName
            },
            new()
            {
                Key = "using_database",
                Value = "true"
            }
        })
    .Enrich.WithSpan(new SpanOptions() { IncludeOperationName = true, IncludeTags = true })
    .CreateLogger());

builder.Services.AddDbContext<ContagemContext>(options =>
{
    options.UseNpgsql(
        builder.Configuration.GetConnectionString("BaseContagem"),
        o => o.UseNodaTime());
});

var resourceBuilder = ResourceBuilder.CreateDefault()
    .AddService(serviceName: OpenTelemetryExtensions.ServiceName,
        serviceVersion: OpenTelemetryExtensions.ServiceVersion);
builder.Services.AddOpenTelemetry()
    .WithTracing((traceBuilder) =>
    {
        traceBuilder
            .AddSource(OpenTelemetryExtensions.ServiceName)
            .SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation()
            .AddNpgsql()
            .AddOtlpExporter()
            .AddConsoleExporter();
    });
```

Geração de Spans:

```csharp
        using var activity1 = OpenTelemetryExtensions.ActivitySource
            .StartActivity("GerarValorContagem")!;

        int valorAtualContador;
        using (ContagemLock.EnterScope())
        {
            _contador.Incrementar();
            valorAtualContador = _contador.ValorAtual;
        }
        activity1.SetTag("valorAtual", valorAtualContador);
        _logger.LogInformation($"Contador - Valor atual: {valorAtualContador}");

        var resultado = new ResultadoContador()
        {
            ValorAtual = _contador.ValorAtual,
            Local = _contador.Local,
            Kernel = _contador.Kernel,
            Mensagem = _configuration["Saudacao"],
            Framework = _contador.Framework
        };
        activity1.Stop();

        using var activity2 = OpenTelemetryExtensions.ActivitySource
            .StartActivity("RegistrarRetornarValorContagem")!;

        _repository.Insert(resultado);
        _logger.LogInformation($"Registro inserido com sucesso! Valor: {valorAtualContador}");

        activity2.SetTag("valorAtual", valorAtualContador);
        activity2.SetTag("horario", $"{DateTime.UtcNow.AddHours(-3):HH:mm:ss}");

        return resultado;
```

### Exemplo com Trace + Logs

Visualizando um Span, com a possibilidade de consulta aos logs:

![trace + span](docs/loki-01.png)

Consultando os logs de um Span:

![logs loki](docs/loki-02.png)

Grafo gerado automaticamente pelo Grafana, contendo as dependências envolvidas numa amostragem de traces:

![grafo dependencias](docs/graph-01.png)
