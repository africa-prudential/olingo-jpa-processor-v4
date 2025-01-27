# How to create a Singleton?

OdataV4 offers the ability to state that an Entity Set has only one element. This is called a Singleton.

We designed our service in a way that every user is represented by a Person. Now we like to give easy access to the data of the currently logged on user. To do so, the service shall be enhanced by a singleton called _Me_.

The JPA Processor provides an annotation to describe how a JPA entity should be converted: `@EdmEntityType`. In case the annotation is missing, the JPA entity is converted into an OData Entity Type and an Entity Set. To state that a singleton shall be created, two options are provided:

1. An option to trigger a generation of an OData Entity Type and a Singleton: `@EdmEntityType(as = EdmTopLevelElementRepresentation.AS_SINGLETON)`
2. An option to create a Singleton on top of an existing OData Entity Type: `@EdmEntityType(as = EdmTopLevelElementRepresentation.AS_SINGLETON_ONLY)`

As we want to put our singleton on top of the Person, we choose the second option and create a subclass of `Person`:

```Java
import javax.persistence.Entity;
import javax.persistence.Table;

import com.sap.olingo.jpa.metadata.core.edm.annotation.EdmEntityType;
import com.sap.olingo.jpa.metadata.core.edm.annotation.EdmTopLevelElementRepresentation;

@Entity(name = "Me")
@EdmEntityType(as = EdmTopLevelElementRepresentation.AS_SINGLETON_ONLY,
    extensionProvider = CurrentUserQueryExtension.class)
@Table(schema = "\"Trippin\"", name = "\"Person\"")
public class CurrentUser extends Person {

}
```

As the Person can have multiple entries, there has to be a way to define how to select the correct record. As it is not possible to do this on the database by creating a view, another option is needed. `@EdmEntityType` has the parameter `extensionProvider` that takes an implementation of `EdmQueryExtensionProvider`. With this we are able to provide an JPA Expression, which is used to extend the WHERE condition to select the singleton.

As a first step we add a constant to `Person` that provides the name of the attribute that we need to select the current user:

```Java
public class Person {

  public static final String USER_NAME_ATTRIBUTE = "userName";
```

Having done this we can use it in our implementation of `EdmQueryExtensionProvider`, which we place also in our model package:

```Java
import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.Expression;
import javax.persistence.criteria.From;

import com.sap.olingo.jpa.metadata.api.JPARequestParameterMap;
import com.sap.olingo.jpa.metadata.core.edm.annotation.EdmQueryExtensionProvider;

public class CurrentUserQueryExtension implements EdmQueryExtensionProvider {
  public static final String USER_NAME = "UserName";
  private final JPARequestParameterMap parameter;

  public CurrentUserQueryExtension(final JPARequestParameterMap parameter) {
    super();
    this.parameter = parameter;
  }

  @Override
  public Expression<Boolean> getFilterExtension(final CriteriaBuilder cb, final From<?, ?> from) {
    final String userName = (String) parameter.get(USER_NAME);
    if (userName == null)
      return cb.isNull(from.get(Person.USER_NAME_ATTRIBUTE));
    return cb.equal(from.get(Person.USER_NAME_ATTRIBUTE), userName);
  }

}
```

Unfortunately we are still not done. One thing is missing. If we want ot filter on the current user, we need to know the user currently logged on. To keep it simple, we use the Authentication header for this. We have to extract the user name from the header and provide it to `CurrentUserQueryExtension`. This is done in the service configuration, in class `ProcessorConfiguration`:

```Java
  @Bean
  @Scope(scopeName = SCOPE_REQUEST)
  public JPAODataRequestContext requestContext(final EntityManagerFactory emf) {
    final HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes())
        .getRequest();
    final String userName = determineUserName(request);
    return JPAODataRequestContext.with()
        .setParameter(CurrentUserQueryExtension.USER_NAME, userName)
        .setCUDRequestHandler(new JPAExampleCUDRequestHandler())
        .setTransactionFactory(new TransactionFactory(emf.createEntityManager()))
        .setDebugSupport(new DefaultDebugSupport())
        .build();
  }

  private String determineUserName(final HttpServletRequest request) {
    final Enumeration<String> authHeader = request.getHeaders("authorization");
    if (authHeader.hasMoreElements()) {
      final String authBase64 = authHeader.nextElement().split(" ")[1];
      final String auth = new String(Base64.getDecoder().decode(authBase64));
      return auth.split(":")[0];
    }
    return "";
  }
```

Now that we have everything together, we can start our service. First step is, to have a look at the metadata:

http://localhost:9010/Trippin/v1/$metadata

The Entity Container should look like this:

```XML
<EntityContainer Name="TrippinContainer">
  <EntitySet Name="People" EntityType="Trippin.Person">
    <NavigationPropertyBinding Path="Trips" Target="Trips"/>
  </EntitySet>
  <EntitySet Name="Airports" EntityType="Trippin.Airport"/>
  <EntitySet Name="Trips" EntityType="Trippin.Trip"/>
  <Singleton Name="Me" Type="Trippin.Person">
    <NavigationPropertyBinding Path="Trips" Target="Trips"/>
  </Singleton>
</EntityContainer>
```

Now we can retrieve the current user:

http://localhost:9010/Trippin/v1/Me

In case the request contains the header _authorization : Basic cnVzc2VsbHdoeXRlOndpY2h0aWc=_, we get the data for _russellwhyte_.