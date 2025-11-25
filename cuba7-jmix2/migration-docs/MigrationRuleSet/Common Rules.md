# Core Architecture Changes

### Java Packages



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
    String APPLICATION = "shamrock-conditions-addon";  
    String NAMESPACE = "shcnds_";  
}
```



### Configs for multi-data stores

```java
@Configuration
@ComponentScan(basePackages = "com.haulmont.shamrock.affiliatesregistry")
@ConfigurationPropertiesScan
@JmixModule(dependsOn = {EclipselinkConfiguration.class, FlowuiConfiguration.class})
@PropertySource(
        name = "com.haulmont.shamrock.affiliatesregistry",
        value = "classpath:/com/haulmont/shamrock/affiliatesregistry/module.properties"
)
public class AffiliatesregistryConfiguration {

    @Bean("shaffls_AffiliatesregistryViewControllers")
    public ViewControllersConfiguration screens(final ApplicationContext applicationContext,
                                                final AnnotationScanMetadataReaderFactory metadataReaderFactory) {
        final ViewControllersConfiguration viewControllers
                = new ViewControllersConfiguration(applicationContext, metadataReaderFactory);
        viewControllers.setBasePackages(Collections.singletonList("com.haulmont.shamrock.affiliatesregistry"));
        return viewControllers;
    }

    // affiliates db connection
    @Bean
    @ConfigurationProperties("affiliatesregistry.datasource")
    public DataSourceProperties affiliatesregistryDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean("affiliatesregistryLiquibaseProperties")
    @ConfigurationProperties(prefix = "affiliatesregistry.liquibase")
    public LiquibaseProperties affiliatesregistryLiquibaseProperties() {
        return new LiquibaseProperties();
    }

    @Bean("affiliatesregistryLiquibase")
    public SpringLiquibase affiliatesregistryLiquibase(
            @Qualifier("affiliatesregistryDataSource") DataSource dataSource,
            @Qualifier("affiliatesregistryLiquibaseProperties") LiquibaseProperties liquibaseProperties
    ) {
        return JmixLiquibaseCreator.create(dataSource, liquibaseProperties);
    }

    @Bean("affiliatesregistryDataSource")
    @ConfigurationProperties("affiliatesregistry.datasource.hikari")
    public DataSource affiliatesregistryDataSource(
            @Qualifier("affiliatesregistryDataSourceProperties")
            DataSourceProperties props
    ) {
        return props.initializeDataSourceBuilder()
                .type(com.zaxxer.hikari.HikariDataSource.class)
                .build();
    }

    @Bean("affiliatesregistryEntityManagerFactory")
    public JmixEntityManagerFactoryBean affiliatesregistryEntityManagerFactory(
            @Qualifier("affiliatesregistryDataSource") DataSource dataSource,
            JpaVendorAdapter jpaVendorAdapter,
            DbmsSpecifics dbmsSpecifics,
            JmixModules jmixModules,
            Resources resources
    ) {
        return new JmixEntityManagerFactoryBean(
                "affiliatesregistry", dataSource, jpaVendorAdapter, dbmsSpecifics, jmixModules, resources
        );
    }

    @Bean("affiliatesregistryTransactionManager")
    public JmixTransactionManager affiliatesregistryTransactionManager(
            @Qualifier("affiliatesregistryEntityManagerFactory") EntityManagerFactory emf) {
        return new JmixTransactionManager("affiliatesregistry", emf);
    }

    // affiliates shamrock

    @Bean
    @ConfigurationProperties("affiliatesregistryshamrock.datasource")
    public DataSourceProperties affiliatesregistryshamrockDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean("affiliatesregistryshamrockLiquibaseProperties")
    @ConfigurationProperties(prefix = "affiliatesregistryshamrock.liquibase")
    public LiquibaseProperties affiliatesregistryshamrockLiquibaseProperties() {
        return new LiquibaseProperties();
    }

    @Bean("affiliatesregistryshamrockLiquibase")
    public SpringLiquibase affiliatesregistryshamrockLiquibase(
            @Qualifier("affiliatesregistryshamrockDataSource") DataSource dataSource,
            @Qualifier("affiliatesregistryshamrockLiquibaseProperties") LiquibaseProperties liquibaseProperties
    ) {
        return JmixLiquibaseCreator.create(dataSource, liquibaseProperties);
    }

    @Bean("affiliatesregistryshamrockDataSource")
    @ConfigurationProperties("affiliatesregistryshamrock.datasource.hikari")
    public DataSource affiliatesregistryshamrockDataSource(
            @Qualifier("affiliatesregistryshamrockDataSourceProperties")
            DataSourceProperties props
    ) {
        return props.initializeDataSourceBuilder()
                .type(com.zaxxer.hikari.HikariDataSource.class)
                .build();
    }

    @Bean("affiliatesregistryshamrockEntityManagerFactory")
    public JmixEntityManagerFactoryBean affiliatesregistryshamrockEntityManagerFactory(
            @Qualifier("affiliatesregistryshamrockDataSource") DataSource dataSource,
            JpaVendorAdapter jpaVendorAdapter,
            DbmsSpecifics dbmsSpecifics,
            JmixModules jmixModules,
            Resources resources
    ) {
        return new JmixEntityManagerFactoryBean(
                "affiliatesregistryshamrock", dataSource, jpaVendorAdapter, dbmsSpecifics, jmixModules, resources
        );
    }

    @Bean("affiliatesregistryshamrockTransactionManager")
    public JmixTransactionManager affiliatesregistryshamrockTransactionManager(
            @Qualifier("affiliatesregistryshamrockEntityManagerFactory") EntityManagerFactory emf) {
        return new JmixTransactionManager("affiliatesregistryshamrock", emf);
    }

    @Bean("shaffls_ResourcePropertyStorage")
    public ResourcePropertyStorage resourcePropertyStorage() {
        return new ResourcePropertyStorage("shamrock-affiliates-registry");
    }
}
```

And properties:

```properties
\# Main data store
main.datasource.url=jdbc:postgresql://localhost:5432/shamrock
main.datasource.username=shamrock
main.datasource.password=shamrock
main.datasource.driver-class-name=org.postgresql.Driver

\# Liquibase main
main.liquibase.change-log=com/haulmont/shamrock/main/liquibase/changelog.xml
main.liquibase.enabled=true

\# List data stores
jmix.core.additional-stores=productregistry,affiliatesregistry,affiliatesregistryshamrock

\# === PRODUCTREGISTRY DataSource ===
productregistry.datasource.url=jdbc:postgresql://localhost:5432/product_registry
productregistry.datasource.username=shamrock
productregistry.datasource.password=shamrock
productregistry.datasource.driver-class-name=org.postgresql.Driver
# optional
productregistry.datasource.hikari.maximum-pool-size=10


\# Liquibase PRODUCTREGISTRY
productregistry.liquibase.enabled=true
productregistry.liquibase.change-log=com/haulmont/shamrock/productregistry/liquibase/changelog.xml

\# === AFFILIATES DataSource ===
affiliatesregistry.datasource.url=jdbc:postgresql://localhost:5432/affiliates_registry
affiliatesregistry.datasource.username=shamrock
affiliatesregistry.datasource.password=shamrock
affiliatesregistry.datasource.driver-class-name=org.postgresql.Driver
# optional
affiliatesregistry.datasource.hikari.maximum-pool-size=10


\# Liquibase AFFILIATES
affiliatesregistry.liquibase.enabled=true
affiliatesregistry.liquibase.change-log=com/haulmont/shamrock/affiliatesregistry/liquibase/changelog.xml

\# === AFFILIATES-SHAMROCK DataSource ===
affiliatesregistryshamrock.datasource.url=jdbc:postgresql://localhost:5432/affiliates_registry_shamrock
affiliatesregistryshamrock.datasource.username=shamrock
affiliatesregistryshamrock.datasource.password=shamrock
affiliatesregistryshamrock.datasource.driver-class-name=org.postgresql.Driver
# optional
affiliatesregistryshamrock.datasource.hikari.maximum-pool-size=10


\# Liquibase AFFILIATES REGISTRY
affiliatesregistryshamrock.liquibase.enabled=true
affiliatesregistryshamrock.liquibase.change-log=com/haulmont/shamrock/affiliatesregistry/liquibase_shamrock/changelog.xml
```


## Messages

In cuba for each "translatable" java classes (such as screens, entities) we created an message.props per each folder. Now, in Jmix we have only ONE root message.properties for each lang (e.g. messages_de.properties).

In cuba was:

```
conditionJsonScreen.caption=Condition JSON
menuCaption=Condition Json Screen
```

In Jmix we appending "fnq (packages + / + key / name)":

```
## Entity keys
com.haulmont.shamrock.affiliatesregistry.entity.partner/Partner=Partner
com.haulmont.shamrock.affiliatesregistry.entity.partner/Partner.id=ID

## View(Screens) keys
com.haulmont.shamrock.affiliatesregistry.view.partner/PartnerListView.title=Partner list view
com.haulmont.shamrock.affiliatesregistry.view.partner/column.enabled=Enabled
com.haulmont.shamrock.affiliatesregistry.view.partner/tabs.integrators=Integrators
com.haulmont.shamrock.affiliatesregistry.view.partner/tabs.suppliers=Suppliers
```

