## Pontos bem básicos

-Builder: Para fazer o deploy ou eu crio em package, ou rodo o comando ./mvn package, onde gerará a pasta target .jar
que será o arquivo de deploy.

- Deploy:
- Para rodar uma aplicação

```bash
java: java -jar "nome ou caminho do arquivo"
```

Receita básica para criar um ambiente java em docker file:

```dockerfile
FROM eclipse-temurin:17-jdk-alpine as builder
WORKDIR application
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
COPY src src
RUN ./mvnw package -DskipTests
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} application.jar
RUN java -Djarmode=layertools -jar application.jar extract


FROM eclipse-temurin:17-jre-alpine
WORKDIR application
COPY --from=builder application/dependencies/ ./
COPY --from=builder application/spring-boot-loader/ ./
COPY --from=builder application/snapshot-dependencies/ ./
COPY --from=builder application/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]

```

Depois, cria-se uma imagem na máquina:

```bash
`docker build -t matheus/codechella:1.0 .`
```

## Configuração do application-prod.properties:

```text
spring.datasource.url=${DATASOURCE_URL}
spring.datasource.username=${DATASOURCE_USERNAME}
spring.datasource.password=${DATASOURCE_PASSWORD}

spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=false

spring.mail.host=${MAIL_HOST}
spring.mail.username=${MAIL_USER}
spring.mail.password=${MAIL_PASSWORD}
spring.mail.port=587
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
spring.mail.properties.mail.smtp.starttls.required=true

app.security.jwt.secret=${APP_JWT_SECRET}

```

Configuração básica do docker-compose:

```dockerfile

version: '3'
services:
  mysql:
    image: mysql:8.0.36
    env_file: ./env/mysql.env
    volumes:
      - ./mysql-data:/var/lib/mysql
    restart: unless-stopped
    healthcheck:
      test: mysqladmin ping -h 127.0.0.1 -u $$MYSQL_USER --password=$$MYSQL_PASSWORD
      interval: 5s
      timeout: 5s
      retries: 10

  app:
    build:
      context: .
    env_file: ./env/app.env
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy

volumes:
  mysql-data:

```

Como colocou os envs, deve-se criar dois para configuração, para variáveis de ambiente:
app.env:

```text
SPRING_PROFILES_ACTIVE=prod
SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/codechella
SPRING_DATASOURCE_USERNAME=codechella_user
SPRING_DATASOURCE_PASSWORD=codechella_pwd
MAIL_HOST=smtp.gmail.com
MAIL_USER=mail_user
MAIL_PASSWORD=mail_pwd
APP_JWT_SECRET=jwt_secret

```

mysql.env

```text
MYSQL_RANDOM_ROOT_PASSWORD=true
MYSQL_DATABASE=codechella
MYSQL_USER=codechella_user
MYSQL_PASSWORD=codechella_pwd

```

Depois rodas os container com o docker compose:

```bash
docker compose up --build
```

## Condiguração nginx:

Deve adicionar o ngnix no docker-compose.yml e configurar um arquivo nginx.conf dentro de uma pasta nginx, claso, se
tratado de aws ec2:

```text
server {
    listen 80;
    server_name app;
    
    location / {
        proxy_pass http://app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

caso tenha ssl:

```text
server {
    listen 80;
    server_name app;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name app;

    ssl_certificate /etc/nginx/certificates/certificado.crt;
    ssl_certificate_key /etc/nginx/certificates/chave.key;

    location / {
        proxy_pass http://app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Também deve-se adiciona o docker-compose o apontamento para o arquivo de configuração:

```dockerfile
version: '3'
services:
  mysql:
    image: mysql:8.0.36
    env_file: ./env/mysql.env
    volumes:
      - ./mysql-data:/var/lib/mysql
    restart: unless-stopped
    healthcheck:
      test: mysqladmin ping -h 127.0.0.1 -u $$MYSQL_USER --password=$$MYSQL_PASSWORD
      interval: 5s
      timeout: 5s
      retries: 10

  app:
    image: aluracursos/codechella:latest
    env_file: ./env/app.env
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy

  nginx:
    image: nginx:stable-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx:/etc/nginx/conf.d
    restart: unless-stopped
    depends_on:
      - app

volumes:
  mysql-data:
```

## Pool de conexões

é um mecanismo que gerencia um conjunto de conexões pré-estabelecidas com um banco de dados. Essas conexões ficam "em
espera" em uma área da memória, prontas para serem usadas pelas aplicações.

Quando uma aplicação precisa acessar o banco de dados, ela solicita uma conexão ao pool. Se houver uma conexão
disponível, o pool a fornece para a aplicação. Caso contrário, a aplicação pode esperar até que uma conexão fique livre
ou, em alguns casos, o pool pode criar uma nova conexão.

Após a aplicação concluir sua operação com o banco de dados, ela retorna a conexão ao pool, onde ela fica disponível
para outras aplicações.

Essa técnica traz diversos benefícios, como:
**Melhora na performance:** Reduz o tempo de espera para estabelecer uma nova conexão, já que a conexão já está pronta
para ser usada.
**Redução de recursos: Diminui** o número de conexões abertas simultaneamente, liberando recursos do banco de dados e da
aplicação.
**Gerenciamento de conexões:** Permite controlar o número máximo de conexões abertas, evitando sobrecarga no banco de
dados.

O pool de conexões é uma técnica fundamental para garantir a performance e a escalabilidade de aplicações que acessam
bancos de dados.

### Como fazer isso com o spginboot

Usa-se o hikaricp para manipular a quantidades de conexões:

```properties
spring.datasource.hikari.minimum-idle=25
spring.datasource.hikari.maximum-pool-size=50
spring.datasource.hikari.connectionTimeout=10000
spring.datasource.hikari.idleTimeout=600000
spring.datasource.hikari.maxLifetime=1800000
```

Vamos analisar cada uma delas:
**spring.datasource.hikari.minimum-idle=25**: Essa propriedade define o número mínimo de conexões que o HikariCP deve
manter abertas no pool. Neste caso, o HikariCP sempre manterá pelo menos 25 conexões ativas, mesmo que não haja nenhuma
requisição em andamento. Isso garante que a aplicação tenha conexões disponíveis para atender a picos de demanda.
**spring.datasource.hikari.maximum-pool-size=50**: Essa propriedade define o número máximo de conexões que o HikariCP
pode manter abertas no pool. Se o número de conexões ativas atingir esse limite, o HikariCP não abrirá novas conexões
até que alguma conexão seja liberada. Isso evita que o pool de conexões consuma muitos recursos do banco de dados.
**spring.datasource.hikari.connectionTimeout=10000**: Essa propriedade define o tempo máximo que o HikariCP espera para
estabelecer uma nova conexão com o banco de dados. Se a conexão não for estabelecida dentro desse tempo, o HikariCP
lança uma exceção. Neste caso, o tempo limite é de 10 segundos (10000 milissegundos).
**spring.datasource.hikari.idleTimeout=600000**: Essa propriedade define o tempo máximo que uma conexão pode ficar
inativa no pool antes de ser fechada. Se uma conexão ficar inativa por mais tempo que esse limite, o HikariCP a fecha
para liberar recursos. Neste caso, o tempo limite é de 10 minutos (600000 milissegundos).
**spring.datasource.hikari.maxLifetime=1800000**: Essa propriedade define o tempo máximo que uma conexão pode permanecer
no pool, independente de estar ativa ou inativa. Após esse tempo, a conexão é fechada e uma nova é criada. Neste caso, o
tempo limite é de 30 minutos (1800000 milissegundos).

## Indices e tabela:

Um índice é uma estrutura de dados que armazena valores de uma ou mais colunas de uma tabela, juntamente com seus
respectivos ponteiros para as linhas da tabela.

Imagine que o índice é como um dicionário, onde as chaves são os valores das colunas e os valores são os ponteiros para
as linhas da tabela.

Quando você faz uma consulta com um filtro em uma coluna que tem um índice, o banco de dados pode usar o índice para
encontrar os valores que correspondem ao filtro. Em seguida, ele usa os ponteiros para encontrar as linhas da tabela que
correspondem aos valores encontrados no índice.

Isso é muito mais rápido do que procurar linha por linha na tabela, pois o índice permite que o banco de dados "salte"
para as linhas que correspondem ao filtro.

Por exemplo, se você tem uma tabela de clientes com um índice na coluna "nome", e você quer encontrar todos os clientes
com o nome "João", o banco de dados pode usar o índice para encontrar rapidamente os valores "João" no índice. Em
seguida, ele usa os ponteiros para encontrar as linhas da tabela que correspondem aos valores "João" encontrados no
índice.

Análise de indices:
O explain e o explain analyze são ferramentas do MySQL que te ajudam a entender como o banco de dados executa suas
consultas SQL.

O explain mostra um plano detalhado da consulta, incluindo índices usados, número de linhas examinadas, etc. Já o
explain analyze vai além, executando a consulta e mostrando o tempo de cada etapa do plano de execução.

Com essas informações, você pode identificar consultas que consomem muitos recursos e otimizá-las!

## Ponto de atenção

ORMs facilitam o acesso ao banco de dados, mas podem gerar consultas complexas e ineficazes, impactando o desempenho da
aplicação. É crucial monitorar as consultas geradas para identificar e otimizar o acesso ao banco de dados,
especialmente em ambientes de produção com grande volume de dados.

## Outro ponto importante abordado na aula

O operador LIKE no SQL é usado para procurar padrões em textos, pode ser um "vilão" da performance, principalmente em
tabelas grandes.
Aí, a aula te dá dicas para otimizar as consultas com LIKE, para otimizar uma coluna extra para concatenar os dados que
você quer pesquisar.

### Estudar novamente, réplica de leitura

### Cache

I
O cache é como um pequeno armário perto da entrada da biblioteca, onde ficam os livros mais populares. Quando você
precisa de um livro, você olha primeiro no armário. Se o livro estiver lá, você o pega rapidamente. Se não estiver, você
precisa ir até as prateleiras da biblioteca para procurá-lo.

No mundo da computação, o cache funciona de forma similar. Ele armazena dados que são frequentemente acessados, como
informações de um site ou dados de um banco de dados. Quando uma aplicação precisa de um dado, ela primeiro verifica o
cache. Se o dado estiver lá, ele é recuperado rapidamente. Se não estiver, a aplicação precisa ir até o local onde o
dado está armazenado, como um banco de dados, para buscá-lo.

O cache é uma técnica muito útil para melhorar a performance de aplicações, pois ele reduz o tempo necessário para
acessar dados.

configurando dockerfile para o redis:

```dockerfile
redis:
  image: redis:7.2.4
  restart: unless-stopped
```

No spring, adiciona como depedência o redis e o cache:

```text
	<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-cache</artifactId>
		</dependency>
```

Tem que ter uma imagem do redis ou o redis instalados localmente e no servidor de produção

depois, deve-se configurar na application.properties o redis:

```text
data.redis.host=redis
data.redis.port=6379
```

Depois, deve ativar na main que deseja fazer cache:

```java

@EnableCaching
public class CodechellaApplication {

    public static void main(String[] args) {
        SpringApplication.run(CodechellaApplication.class, args);
    }

}
```

E, após, deve escolher sabiamente qual evento que salvar em cache, no exemplo foi o listar:
NOtas: a anotação **@Cacheable(value = "proximosEventos")** determina o que eu quero cachear, o valor dentro seria o
nome do cache, já a **@CacheEvict(value = "proximosEventos", allEntries = true)** é usada para situação onde quero
renovar um cache, assim, toda vez que cadastrar o evento, o cache é reinicializado.
Ademais, após, todos os dados que recebem o cache devem implemetar o serialize, desde de que não sejam nativas do java,
como enums, strings, LocalDate, exemplos abaixo:

```java

@Cacheable(value = "proximosEventos")
public List<DadosEvento> listarProximosEventos() {
    var proximosEventos = eventoRepository.findAllByDataAfter(LocalDateTime.now());
    return proximosEventos.stream().map(DadosEvento::new).toList();
}

@CacheEvict(value = "proximosEventos", allEntries = true)
public DadosEvento cadastrar(DadosCadastroEvento dadosCadastro) {
    var eventoJaCadastrado = eventoRepository.existsByNomeIgnoringCase(dadosCadastro.nome());
    if (eventoJaCadastrado) {
        throw new RegraDeNegocioException("Evento já cadastrado com esse nome!");
    }

    var ingressos = criarIngressos(dadosCadastro.ingressos());
    var evento = new Evento(dadosCadastro, ingressos);
    this.eventoRepository.save(evento);

    return new DadosEvento(evento);
}


//--------- implementação do serialize:

public record DadosEvento(Long id, String nome, String descricao, LocalDateTime data, CategoriaEvento categoria,
                          Endereco endereco) implements Serializable {

    public DadosEvento(Evento evento) {
        this(evento.getId(), evento.getNome(), evento.getDescricao(), evento.getData(), evento.getCategoria(), evento.getEndereco());
    }

}

@Embeddable
public record Endereco(
        @NotBlank(message = "Cidade é obrigatória!")
        String cidade,
        @NotBlank(message = "UF é obrigatória!")
        String uf,
        @NotBlank(message = "Logradouro é obrigatório!")
        String logradouro,
        @NotBlank(message = "Bairro é obrigatório!")
        String bairro,
        @NotBlank(message = "CEP é obrigatório!")
        String cep,
        String numero,
        String complemento
) implements Serializable {
}
```

Com isso, o cache funcionará.

# Invalidação de Cache

## Estratégias de Invalidação de Cache

Além da invalidação manual ao inserir, atualizar ou excluir registros no banco de dados, existem outras estratégias para
garantir que o cache reflita os dados mais recentes:

- **Timeout**: Define um tempo de expiração para os objetos em cache. Após esse tempo, os objetos são automaticamente
  removidos e uma nova consulta ao banco de dados é realizada para obter os dados atualizados.
- **Tamanho do Cache**: Limita o número máximo de objetos que podem ser armazenados em cache. Quando o limite é
  atingido, os objetos menos utilizados são removidos para liberar espaço para novos.

## Políticas de Invalidação de Cache

As políticas de invalidação determinam quais objetos são removidos do cache para liberar espaço. As políticas mais
comuns são:

- **Least Frequently Used (LFU - Menos Frequente Utilizado)**: Remove os objetos que foram acessados com menos
  frequência. Objetos raramente acessados são mais propensos a serem removidos.
- **Least Recently Used (LRU - Menos Recentemente Utilizado)**: Remove os objetos que não foram acessados recentemente.
  Objetos não acessados por um longo tempo são os primeiros a serem removidos.

### Considerações

Ao escolher uma política de invalidação de cache, leve em consideração as características da aplicação e dos dados,
visando otimizar a utilização dos recursos e melhorar o desempenho geral.

# Considerações para o Uso de Cache

Ao utilizar cache em uma aplicação, é essencial levar em conta os seguintes aspectos:

## Seleção de Dados

- Nem todas as consultas se beneficiam do cache.
- Evite cachear consultas que retornam dados que mudam frequentemente ou envolvem operações de escrita.
- Selecione cuidadosamente quais consultas serão cacheadas para evitar inconsistências nos dados e sobrecarga no
  sistema.

## Tamanho e Expiração do Cache

- Defina um tamanho máximo para o cache para evitar consumo excessivo de memória e impactos no desempenho.
- Estabeleça um tempo de expiração adequado para os objetos em cache, equilibrando entre manter dados atualizados e
  evitar servir informações desatualizadas aos usuários.

## Monitoramento

- Monitorar o uso e a eficácia do cache é fundamental para garantir que ele esteja otimizando o desempenho da aplicação.

## Testes

- Realize testes para avaliar o impacto do cache na aplicação.
- Experimente diferentes configurações de cache e cenários de uso para identificar possíveis problemas de desempenho e
  ajustar o cache conforme necessário.

## Considerações Finais

O cache é uma ferramenta poderosa para melhorar o desempenho da aplicação, mas deve ser usado com cautela. Avalie
criteriosamente quais consultas podem se beneficiar do cache e ajuste as configurações de acordo com as necessidades
específicas da aplicação.

## Balanceamento de carga com o nginx

Para lidar com essa demanda, você pode abrir mais uma lanchonete, ou seja, escalar horizontalmente. Você terá mais
espaço, mais funcionários e mais recursos para atender a todos os clientes.

Escalabilidade Vertical: Agora, imagine que você quer melhorar a qualidade do seu atendimento na lanchonete. Você pode
comprar um forno maior, contratar um cozinheiro experiente e investir em um sistema de pedidos online. Isso significa
escalar verticalmente, ou seja, melhorar os recursos da sua lanchonete para oferecer um serviço de melhor qualidade.

## Como fazer:

```dockerfile

app-1: &app
  image: aluracursos/codechella:latest
  env_file: /env/app.env
  restart: unless-stopped
  depends_on:
     mysql:
      condition: service_healthy
            
app-2:
  <<: *app

nginx:
  image: nginx:stable-alpine
  ports:
    - "80:80"
  volumes:
    -./nginx:/etc/nginx/conf.d
  restart: unless-stopped
  depends_on:
    - app-1
    - app-2
        
```

agora no nginx:

```text

upstream servers {
  server app-1:8080;
  server app-2:8080;
}

server {
listen 80;

location / {
  proxy_pass http://servers
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
 }
}

```

## IMPORTANTE

Para limitar o docker e consumir menos memória, usase o limit no compose ou direto:

```prompt
docker run --cpu 1 --memory 512m nginx

```

```dockerfile
redis:
  image: redis:7.2.4
  restart: unless-stopped
  deploy:
    resources:
      limits:
        cpus: '1'
        memory: 512M

```

## Estratégias:

# Estratégias de Balanceamento de Carga

O Nginx oferece diversas estratégias de balanceamento de carga, permitindo ajustar a distribuição de requisições entre
os servidores backend conforme as necessidades da aplicação.

## Principais Estratégias

- **Round Robin**: Estratégia padrão do Nginx, que distribui as requisições de forma sequencial entre os servidores,
  garantindo uma distribuição equitativa ao longo do tempo.

- **Least Connections (Menor número de conexões)**: Direciona as requisições para o servidor com o menor número de
  conexões ativas no momento. Ideal para ambientes onde os servidores possuem capacidades diferentes ou onde o tempo de
  resposta pode variar.

- **IP Hash (Hash do endereço IP do cliente)**: Utiliza o hash do IP do cliente para determinar o servidor que receberá
  as requisições, garantindo que todas as requisições de um mesmo cliente sejam direcionadas ao mesmo servidor. Útil
  para manter consistência em aplicações que requerem sessões persistentes.

- **Least Time (Menor tempo de resposta)**: Direciona as requisições para o servidor com o menor tempo de resposta
  recente, otimizando o desempenho ao preferir os servidores mais rápidos.

## Exemplo de Configuração

A configuração das estratégias de balanceamento de carga é feita no arquivo de configuração do Nginx. Abaixo está um
exemplo de configuração utilizando a estratégia de **Least Connections**:

```nginx
upstream servers {
  least_conn;
  server app-1:8080 weight=1;
  server app-2:8080 weight=2;
}

server {
  listen 80;
  location / {
      proxy_pass http://servers;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      //esse é só para diferenciar os containers  
      add_header X-Backend-Server $upstream_addr;
  }
}

```
