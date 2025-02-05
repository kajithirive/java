---

copyright:
  years: 2019
lastupdated: "2019-04-22"

keywords: dependency injection, cdi jakarta, cdi beans, cdi scopes, bean lifecycle, context injection microprofile, microprofile cdi

subcollection: java

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}
{:tip: .tip}
{:note: .note}
{:important: .important}

# Contextos e injeção de dependências (CDI, JSR-365)
{: #mp-cdi}

O CDI é um elemento central em Jakarta-EE e MicroProfile, pois ele é usado para ligar vários componentes. O CDI define **contextos** para especificar e gerenciar os ciclos de vida do bean gerenciado e usa **injeção de dependência** para satisfazer as dependências declaradas de uma maneira segura e dinâmica. O CDI também usa um evento fracamente acoplado e um modelo de interceptor e fornece um mecanismo de extensão móvel poderoso e flexível para outras estruturas para definir seus próprios beans CDI ou atualizar os componentes existentes.

As especificações do MicroProfile (por exemplo, MicroProfile Config, MicroProfile JWT) tomam a abordagem de CDI primeiro, dependendo do mecanismo de extensão do CDI para aprimorar os padrões existentes (como JAX-RS) com novos recursos. O CDI 1.2 faz parte da liberação do MicroProfile 1.0 e depois disso (o MicroProfile 2.0 move-se para o CDI 2.0).

## Beans CDI
{: #cdi-bean}

O CDI opera em _beans_. Os beans CDI são criados usando [Anotações de definição de bean](https://docs.jboss.org/cdi/spec/2.0/cdi-spec.html){: new_window}![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo"). Quase todo plain old Java&trade object (POJO) que tem um construtor sem argumentos ou um construtor com a anotação `@Inject` pode ser um bean.

As anotações que definem os beans CDI devem ser descobertas. A varredura de anotação do CDI pode ser ativada usando um arquivo `beans.xml` na pasta `META-INF` para um archive .jar ou na pasta `WEB-INF` para um archive .war. Esse arquivo pode estar vazio ou pode conter algo como o conteúdo a seguir:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://xmlns.jcp.org/xml/ns/javaee"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
            http://xmlns.jcp.org/xml/ns/javaee/beans_1_1.xsd"
      version="1.1" bean-discovery-mode="all">
  // enable interceptors; decorators; alternatives
</beans>
```
{: codeblock}

A presença de um arquivo `beans.xml` vazio ou um `beans.xml` com `bean-discovery-mode="all"` faz com que todos os objetos potenciais se tornem beans CDI. Caso contrário, apenas os objetos com anotações de definição de bean CDI serão beans CDI.

## Escopos de CDI
{: #cdi-scopes}

Anotações de tipo de escopo do CDI controlam o ciclo de vida do bean:

* Use a anotação de nível de classe `@ApplicationScoped` se for necessário que uma instância exista enquanto o aplicativo estiver ativo.
* Use a anotação de nível de classe `@RequestScoped` se uma nova instância do bean precisar ser criada para cada solicitação (uma solicitação do servlet, por exemplo).
* Use a anotação de nível de classe `@Dependent` se a nova instância pertencer a outro objeto. Uma instância de um bean dependente nunca é compartilhada entre clientes diferentes ou pontos de injeção diferentes. Ela é instanciada quando o objeto ao qual ela pertence é criado e, em seguida, destruída quando o objeto a que ela pertence é destruído.
* Se nenhuma anotação de nível de classe for definida, `@Dependent` será o padrão.

O contêiner CDI cria e destrói instâncias de bean de acordo com seu escopo definido, associa-as ao contexto apropriado e, em seguida, injeta-as como dependências em outros objetos.

## Injeção de bean CDI
{: #cdi-inject}

O CDI injeta beans definidos em outros componentes por meio de `@Inject`. Por exemplo, o seguinte POJO `MyBean` é um bean CDI:

```java
@ApplicationScoped
public class MyBean {
    int i=0;
    public String sayHello() {
        return "MyBean hello " + i++;
    }
}
```
{: codeblock}

Como é `@ApplicationScoped`, somente uma instância é criada. No seguinte, `MyRestEndPoint` é `@RequestScoped`, o que significa que uma instância é criada para cada solicitação. O CDI injeta a mesma instância `MyBean` em cada um.

```java
@RequestScoped
@Path("/hello")
public class MyRestEndPoint {
    @Inject MyBean bean;

    @GET
    @Produces (MediaType.TEXT_PLAIN)
    public String sayHello() {
        return bean.sayHello();
    }
}
```
{: codeblock}

O CDI está gerenciando o ciclo de vida desses beans:

* use um método anotado com `@PostConstruct` em vez de um construtor para inicializar seu bean; Ele é chamado após o contêiner do CDI injetar dependências e o bean associado no contexto adequado.
* use um método anotado com `@PreDestroy` para limpar o seu bean; Ele é chamado antes que as dependências sejam removidas.
