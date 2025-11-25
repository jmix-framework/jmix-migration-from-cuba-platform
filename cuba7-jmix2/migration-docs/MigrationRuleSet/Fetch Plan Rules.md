## FetchPlan Migration (Views → FetchPlans)

### Overview

In CUBA, entity graphs are called **Views**. In Jmix 2.x, they are called **FetchPlans**. The concept is similar but the
API and XML structure have changed.

### XML-based FetchPlans

If you are putting new fetch-plan.xml into resources/base.pkg/ folder, then don't forget also add fetch-plans into application properties(or module.properties, same folder):

jmix.core.fetch-plans-config=com/haulmont/shamrock/affiliatesregistry/fetch-plans.xml


#### CUBA View Definition

```xml
<!-- views.xml -->
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<views xmlns="http://schemas.haulmont.com/cuba/view.xsd">

    <view class="com.company.sales.entity.Customer"
          name="customer-browse"
          extends="_local">
        <property name="status"/>
    </view>

    <view class="com.company.sales.entity.Customer"
          name="customer-edit"
          extends="_local">
        <property name="status"/>
        <property name="orders" view="_minimal">
            <property name="orderNumber"/>
            <property name="orderDate"/>
        </property>
    </view>

    <view class="com.company.sales.entity.Order"
          name="order-with-customer"
          extends="_local">
        <property name="customer" view="_instance-name"/>
        <property name="items" view="_minimal">
            <property name="product" view="_instance-name"/>
            <property name="quantity"/>
        </property>
    </view>

</views>
```

#### Jmix FetchPlan Definition

```xml
<!-- fetch-plans.xml -->
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<fetchPlans xmlns="http://jmix.io/schema/core/fetch-plans">

    <fetchPlan class="com.company.sales.entity.Customer"
               name="customer-list"
               extends="_base">
        <property name="status"/>
    </fetchPlan>

    <fetchPlan class="com.company.sales.entity.Customer"
               name="customer-detail"
               extends="_base">
        <property name="status"/>
        <property name="orders" fetchPlan="_base">
            <property name="orderNumber"/>
            <property name="orderDate"/>
        </property>
    </fetchPlan>

    <fetchPlan class="com.company.sales.entity.Order"
               name="order-with-customer"
               extends="_base">
        <property name="customer" fetchPlan="_instance_name"/>
        <property name="items" fetchPlan="_base">
            <property name="product" fetchPlan="_instance_name"/>
            <property name="quantity"/>
        </property>
    </fetchPlan>

</fetchPlans>
```

### Inline FetchPlans in Views

```xml
<!-- CUBA -->
<data>
    <collection id="customersDc"
                class="com.company.sales.entity.Customer"
                view="customer-browse">
        <loader id="customersDl">
            <query>select c from sales$Customer c</query>
        </loader>
    </collection>
</data>

        <!-- Jmix -->
<data>
<collection id="customersDc"
            class="com.company.sales.entity.Customer">
    <fetchPlan extends="_base">
        <property name="status"/>
        <property name="createdBy"/>
    </fetchPlan>
    <loader id="customersDl">
        <query>select c from Customer c</query>
    </loader>
</collection>
</data>
```

### Fetch plan mapping

### Built-in FetchPlans Comparison

| CUBA View        | Jmix FetchPlan   | Description                                                 | Usage                  |
|------------------|------------------|-------------------------------------------------------------|------------------------|
| `_local`         | `_base`          | All local (non-reference) attributes including audit fields | Default for list views |
| `_minimal`       | `_instance_name` | Only field(s) marked with `@InstanceName`                   | Lookup/combo boxes     |
| `_instance-name` | `_instance_name` | Only instance name field(s)                                 | Display names only     |
| `_base`          | `_base`          | Same as `_local` in CUBA                                    | Most common usage      |

### ⚠️ CRITICAL DIFFERENCE: `_minimal` Behavior Changed

```java
// CUBA: _minimal loads ALL local attributes (same as _local)
@Entity(name = "sales$Customer")
public class Customer extends StandardEntity {
    @Column(name = "NAME")
    private String name;          // ✅ loaded with _minimal

    @Column(name = "EMAIL")
    private String email;         // ✅ loaded with _minimal

    @Column(name = "PHONE")
    private String phone;         // ✅ loaded with _minimal

    @ManyToOne
    private CustomerType type;    // ❌ NOT loaded with _minimal
}

// Jmix: _instance_name loads ONLY @InstanceName field
@JmixEntity
@Entity(name = "Customer")
public class Customer {
    @InstanceName
    @Column(name = "NAME")
    private String name;          // ✅ loaded with _instance_name

    @Column(name = "EMAIL")
    private String email;         // ❌ NOT loaded with _instance_name

    @Column(name = "PHONE")
    private String phone;         // ❌ NOT loaded with _instance_name

    @ManyToOne
    private CustomerType type;    // ❌ NOT loaded with _instance_name
}
```

### Programmatic FetchPlan Creation

```java
// CUBA
@Inject
private ViewRepository viewRepository;

View<Customer> view = viewRepository.getView(Customer.class, "customer-edit");

// Or create programmatically
View<Customer> customView = new View<>(Customer.class)
        .addProperty("name")
        .addProperty("email")
        .addProperty("orders", new View<>(Order.class)
                .addProperty("orderNumber")
                .addProperty("orderDate"));

List<Customer> customers = dataManager.load(Customer.class)
        .query("select c from sales$Customer c")
        .view(customView)
        .list();

// Jmix
@Autowired
private FetchPlanRepository fetchPlanRepository;

FetchPlan fetchPlan = fetchPlanRepository.getFetchPlan(Customer.class, "customer-detail");

// Or create programmatically
FetchPlan customFetchPlan = fetchPlanBuilder.of(Customer.class)
        .add("name")
        .add("email")
        .add("orders", fetchPlanBuilder.of(Order.class)
                .add("orderNumber")
                .add("orderDate")
                .build())
        .build();

List<Customer> customers = dataManager.load(Customer.class)
        .query("select c from Customer c")
        .fetchPlan(customFetchPlan)
        .list();
```

### System FetchPlans Mapping

| CUBA View        | Jmix FetchPlan   | Description                          |
|------------------|------------------|--------------------------------------|
| `_local`         | `_base`          | All local (non-reference) attributes |
| `_minimal`       | `_base`          | Same as _base                        |
| `_instance-name` | `_instance_name` | Only @InstanceName field             |
