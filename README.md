# Dicas para o Desafio 3: proposta, transação e fatura.


	Esse documento é um compilado do material de apoio, documentação e stack 
over flow. A ideia desse repositório foi economizar tempo nessa parte de configurações (principalmente nos assuntos que fazem maior intersecção com o setup de infraestrutura) e deixar mais tempo pro código. Aqui no README está um tutorial rápido de como colocar todas essas ferramentas no projeto. O ponto aqui não é aprofundar, é só conectar esse ferramental com o projeto e deixar funcionando. Nos arquivos de cada ferramenta, deixei algumas features extras que achei interessantes.


### PRINCIPAL

1 - Prometheus
2 - Grafana
3 - Kafka
4 - Vault
5 - Jaeger
6 - Docker 
7 - Kubernetes
8 - Keycloak


### EXTRA

9 - Open Api 3.0 (antigo Swagger)
10 - Knative

- O Knative aqui mais pra facilitar mesmo a parte do Kubernetes


### 1 - Prometheus

Documentação: https://prometheus.io/

localhost:9090/targets


### 2 - Grafana

Documentação: https://grafana.com/docs/grafana/latest/

- localhost:3000

- no primeiro acesso => 
    - login: admin
    - senha: admin



### 3 - Kafka

Documentação: https://kafka.apache.org/documentation/


3.1. pom.xml

```

<dependency>
  <groupId>org.springframework.kafka</groupId>
  <artifactId>spring-kafka</artifactId>
</dependency>


```


3.2 application.properties


```

spring.kafka.bootstrap-servers=${KAFKA_HOST:localhost:9092}
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.group-id=${KAFKA_CONSUMER_GROUP_ID:fatura}
spring.kafka.consumer.auto-offset-reset=${KAFKA_AUTO-OFFSET-RESET:latest}
spring.kafka.topic.transactions=${KAFKA_TOPIC:transacoes}

```


3.3 classe de configurações na API



```


@Configuration
@EnableKafka
public class ConfiguracoesKafka {


    private final KafkaProperties kafkaProperties;

    public ConfiguracoesKafka(KafkaProperties kafkaProperties) {
        this.kafkaProperties = kafkaProperties;
    }


    public Map<String, Object> consumerConfigurations() {

        Map<String, Object> properties = new HashMap<>();
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaProperties.getBootstrapServers());
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, kafkaProperties.getConsumer().getGroupId());
        properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, kafkaProperties.getConsumer().getAutoOffsetReset());

        return properties;

    }


    @Bean
    public ConsumerFactory<String, RecebeTransacao> transactionConsumerFactory() {

        StringDeserializer stringDeserializer = new StringDeserializer();

        JsonDeserializer<RecebeTransacao> jsonDeserializer =
                new JsonDeserializer<>(RecebeTransacao.class, false);

        return new DefaultKafkaConsumerFactory<>(consumerConfigurations(), stringDeserializer, jsonDeserializer);
    }


    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, RecebeTransacao> kafkaListenerContainerFactory() {

        ConcurrentKafkaListenerContainerFactory<String, RecebeTransacao> factory =
                new ConcurrentKafkaListenerContainerFactory<>();

        factory.setConsumerFactory(transactionConsumerFactory());

        return factory;
    }


}

```

3.4. Criar uma classe para consumir as mensagens do Kafka Producer

```

    @KafkaListener(topics="${spring.kafka.topic.transactions}")
    public void consume(RecebeTransacao transacaoRecebida)  {

        // nesse ponto já temos os dados enviados pelo Producer e podemos
        // realizar todas as operações pedidas nos requisitos

    }



```


### 4 - Vault

Documentação: https://www.vaultproject.io/docs


Integrando ao projeto:


4.1. adicionar a dependência no pom.xml


```

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
    <version>2.2.5.RELEASE</version>
    <scope>runtime</scope>
</dependency>


```

4.2. criar um arquivo bootstrap.yml


```

spring:
 application:
   name: fatura
 cloud:
   vault:
     token: ee413645-dbe8-4848-afc6-6bb2768ada75
     scheme: http


```

- esse spring.application.name vai ser o nome utilizado na hora de definir o segredo no vault.

- o token veio definido no docker-compose.yml


4.3. definir o segredo no nosso cofre, o vault.

- Um exemplo seria:

> docker exec -it 5dcab36cd82f vault kv put secret/fatura DB_USERNAME=keycloak
> DB_PASSWORD=password

- Aqui estamos falando para o docker ir lá no container 5dcab36cd82f e executar esse comando no bash:

> vault kv put secret/fatura DB_USERNAME=keycloak DB_PASSWORD=password


4.4. variáveis de ambiente na nossa API

- O DB_USERNAME e DB_PASSWORD estarão como variáveis de ambiente 
lá no application properties, assim:

```
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}

```


Pronto, agora o Vault já está funcionando.


### 5 - Jaeger


Documentação: https://www.jaegertracing.io/docs/1.20/

http://localhost:16686


5.1. Application.properties

```

opentracing.jaeger.udp-sender.host=${JAEGGER_HOST:localhost}
opentracing.jaeger.udp-sender.port=${JAEGGER_PORT:5775}

```

### 6 - Docker - criar container

Documentação: https://docs.docker.com/get-started/


6.1. Criar Dockerfile:

```

## Builder Image
FROM maven:3.6.3-jdk-11 AS builder
COPY src /usr/src/app/src
COPY pom.xml /usr/src/app
RUN mvn -f /usr/src/app/pom.xml clean package -DskipTests


## Runner Image
FROM openjdk:11
COPY --from=builder /usr/src/app/target/fatura-0.0.1-SNAPSHOT.jar 
/usr/app/app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/usr/app/app.jar"]


```

6.2. construir o container

> docker build -t bootcamp/proposta .


### 7 - Kubernetes



### 8 - Keycloak

Documentação: https://www.keycloak.org/documentation


8.1 Application.properties


```

spring.security.oauth2.resourceserver.jwt.issuer-uri=${KEYCLOAK_ISSUER_URI:http:
//localhost:18080/auth/realms/nosso-cartao}

spring.security.oauth2.resourceserver.jwt.jwk-set-uri=${KEYCLOAK_JWKS_URI:http://
localhost:18080/auth/realms/nosso-cartao/protocol/openid-connect/certs}


```

## EXTRA


### 9 - Open Api 3.0


9.1. pom.xml


```
        <dependency>
			<groupId>org.springdoc</groupId>
			<artifactId>springdoc-openapi-ui</artifactId>
			<version>1.4.8</version>
		</dependency>

```


9.2. application.properties


```

# open api 3 doc
springdoc.swagger-ui.path=/swagger-ui.html

```


### 10 - Knative