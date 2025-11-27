# Common Migration Rules

### Imports

```java
// CUBA (before)
import com.haulmont.cuba.core.entity.\*;  
import com.haulmont.cuba.core.global.\*;  
import com.haulmont.cuba.gui.screen.\*;  
import javax.persistence.\*;  
```

```java
// Jmix (after)  
import io.jmix.core.entity.\*;  
import io.jmix.core.\*;  
import io.jmix.flowui.view.\*;  
import jakarta.persistence.\*;
```

### Build Configuration

```groovy
// build.gradle (CUBA)  
dependencies {  
    appComponent("com.haulmont.cuba:cuba-global:7.2-SNAPSHOT")  
}  
```

```groovy
// build.gradle (Jmix)  
dependencies {  
    implementation 'io.jmix.core:jmix-core-starter'  
    implementation 'io.jmix.flowui:jmix-flowui-starter'  
    implementation 'io.jmix.flowui:jmix-flowui-data-starter'  
    implementation 'io.jmix.datatools:jmix-datatools-starter'  
}
```

### Spring Configuration

```
@Config // <- In Jmix this is not needed, DONT DO IT  
public interface JmixConfig {  
    String APPLICATION = "myapp-conditions-addon";  
    String NAMESPACE = "myapp_";  
}
```

### Configs for multi-data stores

```java
@Configuration
@ComponentScan(basePackages = "com.company.myapp")
@ConfigurationPropertiesScan
@JmixModule(dependsOn = {EclipselinkConfiguration.class, FlowuiConfiguration.class})
@PropertySource(
        name = "com.company.myapp",
        value = "classpath:/com/company/myapp/module.properties"
)
public class MyappConfiguration {

    @Bean("myapp_MyappViewControllers")
    public ViewControllersConfiguration screens(final ApplicationContext applicationContext,
                                                final AnnotationScanMetadataReaderFactory metadataReaderFactory) {
        final ViewControllersConfiguration viewControllers
                = new ViewControllersConfiguration(applicationContext, metadataReaderFactory);
        viewControllers.setBasePackages(Collections.singletonList("com.company.myapp"));
        return viewControllers;
    }

    // datastore1 db connection
    @Bean
    @ConfigurationProperties("datastore1.datasource")
    public DataSourceProperties datastore1DataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean("datastore1LiquibaseProperties")
    @ConfigurationProperties(prefix = "datastore1.liquibase")
    public LiquibaseProperties datastore1LiquibaseProperties() {
        return new LiquibaseProperties();
    }

    @Bean("datastore1Liquibase")
    public SpringLiquibase datastore1Liquibase(
            @Qualifier("datastore1DataSource") DataSource dataSource,
            @Qualifier("datastore1LiquibaseProperties") LiquibaseProperties liquibaseProperties
    ) {
        return JmixLiquibaseCreator.create(dataSource, liquibaseProperties);
    }

    @Bean("datastore1DataSource")
    @ConfigurationProperties("datastore1.datasource.hikari")
    public DataSource datastore1DataSource(
            @Qualifier("datastore1DataSourceProperties")
            DataSourceProperties props
    ) {
        return props.initializeDataSourceBuilder()
                .type(com.zaxxer.hikari.HikariDataSource.class)
                .build();
    }

    @Bean("datastore1EntityManagerFactory")
    public JmixEntityManagerFactoryBean datastore1EntityManagerFactory(
            @Qualifier("datastore1DataSource") DataSource dataSource,
            JpaVendorAdapter jpaVendorAdapter,
            DbmsSpecifics dbmsSpecifics,
            JmixModules jmixModules,
            Resources resources
    ) {
        return new JmixEntityManagerFactoryBean(
                "datastore1", dataSource, jpaVendorAdapter, dbmsSpecifics, jmixModules, resources
        );
    }

    @Bean("datastore1TransactionManager")
    public JmixTransactionManager datastore1TransactionManager(
            @Qualifier("datastore1EntityManagerFactory") EntityManagerFactory emf) {
        return new JmixTransactionManager("datastore1", emf);
    }

    // datastore2

    @Bean
    @ConfigurationProperties("datastore2.datasource")
    public DataSourceProperties datastore2DataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean("datastore2LiquibaseProperties")
    @ConfigurationProperties(prefix = "datastore2.liquibase")
    public LiquibaseProperties datastore2LiquibaseProperties() {
        return new LiquibaseProperties();
    }

    @Bean("datastore2Liquibase")
    public SpringLiquibase datastore2Liquibase(
            @Qualifier("datastore2DataSource") DataSource dataSource,
            @Qualifier("datastore2LiquibaseProperties") LiquibaseProperties liquibaseProperties
    ) {
        return JmixLiquibaseCreator.create(dataSource, liquibaseProperties);
    }

    @Bean("datastore2DataSource")
    @ConfigurationProperties("datastore2.datasource.hikari")
    public DataSource datastore2DataSource(
            @Qualifier("datastore2DataSourceProperties")
            DataSourceProperties props
    ) {
        return props.initializeDataSourceBuilder()
                .type(com.zaxxer.hikari.HikariDataSource.class)
                .build();
    }

    @Bean("datastore2EntityManagerFactory")
    public JmixEntityManagerFactoryBean datastore2EntityManagerFactory(
            @Qualifier("datastore2DataSource") DataSource dataSource,
            JpaVendorAdapter jpaVendorAdapter,
            DbmsSpecifics dbmsSpecifics,
            JmixModules jmixModules,
            Resources resources
    ) {
        return new JmixEntityManagerFactoryBean(
                "datastore2", dataSource, jpaVendorAdapter, dbmsSpecifics, jmixModules, resources
        );
    }

    @Bean("datastore2TransactionManager")
    public JmixTransactionManager datastore2TransactionManager(
            @Qualifier("datastore2EntityManagerFactory") EntityManagerFactory emf) {
        return new JmixTransactionManager("datastore2", emf);
    }

    @Bean("myapp_ResourcePropertyStorage")
    public ResourcePropertyStorage resourcePropertyStorage() {
        return new ResourcePropertyStorage("datastore2");
    }
}
```

And properties:

```properties
\# Main data store
main.datasource.url=jdbc:postgresql://localhost:5432/maindb
main.datasource.username=root
main.datasource.password=root
main.datasource.driver-class-name=org.postgresql.Driver

\# Liquibase main
main.liquibase.change-log=com/haulmont/myapp/liquibase/changelog.xml
main.liquibase.enabled=true

\# List data stores
jmix.core.additional-stores=datastore1,datastore2

\# === datastore1 DataSource ===
datastore1.datasource.url=jdbc:postgresql://localhost:5432/datastore1
datastore1.datasource.username=root
datastore1.datasource.password=root
datastore1.datasource.driver-class-name=org.postgresql.Driver
# optional
datastore1.datasource.hikari.maximum-pool-size=10


\# Liquibase datastore1
datastore1.liquibase.enabled=true
datastore1.liquibase.change-log=com/company/myapp/liquibase/datastore1-changelog.xml

\# === datastore2 DataSource ===
datastore2.datasource.url=jdbc:postgresql://localhost:5432/datastore2
datastore2.datasource.username=root
datastore2.datasource.password=root
datastore2.datasource.driver-class-name=org.postgresql.Driver
# optional
datastore2.datasource.hikari.maximum-pool-size=10


\# Liquibase datastore2
datastore2.liquibase.enabled=true
datastore2.liquibase.change-log=com/company/myapp/liquibase/datastore2-changelog.xml
```

## Messages

In cuba for each "translatable" java classes (such as screens, entities) we created an message.props per each folder. Now, in Jmix we have only ONE root message.properties for each lang (e.g. messages_de.properties).

In cuba was:

```
conditionJsonScreen.caption=Condition JSON
menuCaption=Condition Json Screen
```

In Jmix we add a "group" (usually a package name) and `/` before the message key":

```
## Entity keys
com.company.myapp.entity.partner/Partner=Partner
com.company.myapp.entity.partner/Partner.id=ID

## View(Screens) keys
com.company.myapp.view.partner/PartnerListView.title=Partner list view
com.company.myapp.view.partner/column.enabled=Enabled
com.company.myapp.view.partner/tabs.integrators=Integrators
com.company.myapp.view.partner/tabs.suppliers=Suppliers
```

## Common classes

Migrate usages of common classes as follows:

- `com.haulmont.bali.util.Preconditions` -> `io.jmix.core.common.util.Preconditions`
- `com.haulmont.cuba.core.global.DatatypeFormatter` -> `io.jmix.core.metamodel.datatype.DatatypeFormatter`
- `com.haulmont.cuba.core.entity.KeyValueEntity`  -> `io.jmix.core.entity.KeyValueEntity`

Replace invocation of `com.haulmont.cuba.core.global.Metadata#getTools` method to get `MetadataTools` interface with direct injection of `MetadataTools` bean using `@Autowired`.

## Logger

Replace injection of logger like `@Inject private Logger log;` with static field as follows:
```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

class SampleBean {
    private static final Logger log = LoggerFactory.getLogger(SampleBean.class);
```

## Security permission checking

If the source CUBA project uses `com.haulmont.cuba.core.global.Security` interface methods, such as `isEntityOpPermitted`, introduce the following bean in Jmix project and use it instead:

```java
package com.company.myapp;

import io.jmix.core.Metadata;
import io.jmix.core.metamodel.model.MetaClass;
import io.jmix.core.security.AccessDeniedException;
import io.jmix.security.constraint.PolicyStore;
import io.jmix.security.constraint.SecureOperations;
import io.jmix.security.model.EntityPolicyAction;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class SecurityPermissions {

    @Autowired
    private Metadata metadata;
    @Autowired
    private PolicyStore policyStore;
    @Autowired
    private SecureOperations secureOperations;

    public boolean isEntityOpPermitted(Class<?> entityClass, EntityPolicyAction entityPolicyAction) {
        MetaClass metaClass = metadata.getClass(entityClass);
        switch (entityPolicyAction) {
            case CREATE -> {
                return secureOperations.isEntityCreatePermitted(metaClass, policyStore);
            }
            case UPDATE -> {
                return secureOperations.isEntityUpdatePermitted(metaClass, policyStore);
            }
            case READ -> {
                return secureOperations.isEntityReadPermitted(metaClass, policyStore);
            }
            case DELETE -> {
                return secureOperations.isEntityDeletePermitted(metaClass, policyStore);
            }
            default -> throw new UnsupportedOperationException();
        }
    }

    public void checkSpecificPermission(String permissionId) {
        if (!secureOperations.isSpecificPermitted(permissionId, policyStore))
            throw new AccessDeniedException("specific", permissionId);
    }
}
```