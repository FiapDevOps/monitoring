# monitoring
Deploy a spring boot app using prometheus to monitoring and consul for service discovery


1. Primeiro para testar a aplicação verifique o conteúdo do diretório build em relação aos seguintes pontos:

1.1. A versão original da aplicação é um projeto simples em java preparado para construção de um container com exibição da mensagem mais tradicional possível: "Hello Docker World", essa aplicação pode ser estudada neste guide: [https://spring.io/guides/gs/spring-boot-docker/#initial](https://spring.io/guides/gs/spring-boot-docker/#initial);

1.2. Para monitorar a aplicação utilizaremos o projeto micrometer fornecendo a informação de time series necessária a coleta do prometheus, essa implantação pode ser estudada na documentação do micrometer: [https://micrometer.io/docs/registry/prometheus](https://micrometer.io/docs/registry/prometheus);

1.3 Na pratica a implantação do micrometer consistiu em adicionar as dependências abaixo no arquivo pom.xml:

```sh
        <!-- Spring boot actuator to expose metrics endpoint -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!-- Micormeter core dependecy  -->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-core</artifactId>
        </dependency>
        <!-- Micrometer Prometheus registry  -->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>
```

1.4 Além disso também adicionamos um Dockerfile, pois o modelo descrito nesse Lab utiliza DockerCompose para entregar todos os serviços necessários usando uma camada de abstração de rede e evitando conflitos de endereço IP:

```sh
FROM openjdk:8-jdk-alpine
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
ENV JAVA_OPTS="-XX:+UnlockExperimentalVMOptions"
EXPOSE 8080
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -jar /app.jar" ]
```

2. Executando o build local da aplicação:

2.1 Para gerar o artefato que será utilizado no laboratório acesse o diretório do projeto e execute o build da aplicação:

```sh
cd $HOME/monitoring/build/initial
./mvnw package && java -jar target/gs-spring-boot-docker-0.1.0.jar
```

Se estiver executando uma versão local do projeto verifique a documentação oficial referente ao processo de build neste guide: [https://spring.io/guides/gs/spring-boot-docker/#initial](https://spring.io/guides/gs/spring-boot-docker/#initial);

> No caso de uma falha neste processo ainda teremos a alternativa de utilizar uma versão do artefato que foi preparada para o lab como um "fallback" disponível no registry [https://hub.docker.com/r/devfiap/gs-spring-boot-docker](https://hub.docker.com/r/devfiap/gs-spring-boot-docker);

2.2 Após o processo de build você verá um exemplo da aplicação rodando no endereço 127.0.0.1:8080:

| descrição                       | path                              |
|---------------------------------|-----------------------------------|
| Entrega da aplicação            | <IP-APP>:8080                     |
| Raiz do actuator (micrometer)   | <IP-APP>:8080/actuator            |
| Health do actuator (micrometer) | <IP-APP>:8080/actuator/health     |
| Métricas do prometheus          | <IP-APP>:8080/actuator/prometheus |

2.3 Pare a aplicação anterior utilziada no teste, ela será iniciada novamente dentro da arquitetura de rede onde está configurado o prometheus e os outros componentes do laboratório;


---

3. Inicie o conkuento de containers que fazem a composição do laboratório com Docker compose:

```sh
docker-compose up -d
```

Se o processo de build ocorrer conforme esperado e as imagens do prometheus e do segundo componente que trataremos no futuro forem baixadas teremos o seguinte cenário:

| descrição                            | path                              |
|--------------------------------------|-----------------------------------|
| Entrega da aplicação Java            | <IP-APP>:8080                     |
| Entrega da monitoração time series   | <IP-APP>:9090                     |


3.1 Em nosso modelos temos 3 targets configurados para expor métricas via time series, cada um deles é identificado por um job da configuração e podem ser consultados na instância onde rodamos nosso stack na path ":9090/targets";

4.0 Na configuração atual além do prometheus implementamos o consul como service-discovery, o serviço pode ser acessado diretamente pelo container:

```sh
docker exec monitoring_consul_1 consul members list
```

4.1 Na pratica o mecanismo mais comum para construir as integrações é interagir diretamente com a api rest do serviço:

```sh
curl http://127.0.0.1:8500/v1/status/leader
```

4.2 Para cadastrar a nossa aplicação utilizaremos a api rest do consul, o arquivo com os dados da aplicação segue o seguinte padrão:

```sh
{
  "Name": "spring",
  "Tags": ["java", "rc-v1"],
  "Address": "spring-app",
  "Port": 8080,
  "Meta": {
    "spring_version": "2.0",
    "openjdk_version": "11.0.7"
  },
  "EnableTagOverride": false,
  "Check": {
    "name": "HTTP API on port 8080",
    "http": "http://spring-app:8080/actuator/health",
    "method": "GET",
    "Interval": "10s",
    "Timeout": "5s"
  },
  "Weights": {
    "Passing": 10,
    "Warning": 1
  }
}
```

4.3 Faça um cadastro manual do serviço:

```sh
curl \
    --request PUT \
    --data @payload.json \
    http://127.0.0.1:8500/v1/agent/service/register?replace-existing-checks=true
```

> Execute este comando a partir da raiz deste projeto onde o arquivo payload.json foi criado;

Outros exemplos de cadastro de serviços podem ser obtidos na documentação do projeto referente a [registro de serviços](https://www.consul.io/api/agent/service.html);

