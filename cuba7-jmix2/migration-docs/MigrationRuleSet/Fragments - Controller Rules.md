# Migrating Screen Fragments (CUBA → Jmix 2.x)

## Overview

Fragments are reusable UI parts that can be embedded into screens (CUBA) or views (Jmix). Migrating fragments from CUBA to Jmix involves changes in how fragments are defined, included in screens, and interact with data. Below we outline the key differences with minimal text and **maximum code examples** for clarity.

## Defining a Fragment: Class and XML

In CUBA, a fragment controller is a class annotated with `@UiController` and `@UiDescriptor`, extending `ScreenFragment`. Its UI layout is defined in an XML descriptor with a root `<fragment>` element containing a `<layout>`.

In Jmix, a fragment controller uses the `@FragmentDescriptor` annotation (no explicit ID), extends `Fragment<…>` (generic base class), and its XML uses `<fragment>` with a `<content>` section (since any component can be the root).

```java
// CUBA fragment controller
@UiController("demo_AddressFragment")           // identifier for fragment
@UiDescriptor("address-fragment.xml")          // XML layout
public class AddressFragment extends ScreenFragment {
    // ... fragment logic
}
```

```xml
<!-- CUBA fragment XML (address-fragment.xml) -->
<fragment xmlns="http://schemas.haulmont.com/cuba/screen/fragment.xsd">
    <layout>
        <textField id="cityField" caption="City"/>
        <textField id="zipField" caption="Zip"/>
    </layout>
</fragment>
```

```java
// Jmix fragment controller
@FragmentDescriptor("address-fragment.xml")     // path to XML layout
public class AddressFragment extends Fragment<FormLayout> {
    // ... fragment logic
}
```

```xml
<!-- Jmix fragment XML (address-fragment.xml) -->
<fragment xmlns="http://jmix.io/schema/flowui/fragment">
    <content>
        <formLayout>
            <textField id="cityField" label="City"/>
            <textField id="zipcodeField" label="Zipcode"/>
        </formLayout>
    </content>
</fragment>
```

**Key differences:** In CUBA, the fragment is identified by a string ID (`demo_AddressFragment` above), while in Jmix no such ID is needed – the fragment is referenced by its **class**. Also, Jmix uses a `<content>` wrapper instead of `<layout>` (but both frameworks require exactly one root layout element inside the fragment).

## Including Fragments in a Screen/View

### Declarative Inclusion (XML)

**CUBA:** Use the `<fragment>` element with a `screen` attribute matching the fragment's `@UiController` ID. This can be placed inside any container of a host screen's layout (even the top-level layout).

```xml
<!-- CUBA host screen XML -->
<window ... xmlns="http://schemas.haulmont.com/cuba/screen/window.xsd">
    <layout>
        <groupBox id="addressBox" caption="Address">
            <fragment screen="demo_AddressFragment"/> <!-- include fragment by ID -->
        </groupBox>
    </layout>
</window>
```

**Jmix:** Use the `<fragment>` element with a `class` attribute specifying the fragment's **fully-qualified class name**. This is placed in the host view's layout (e.g., inside a `details` or any container).

```xml
<!-- Jmix host view XML -->
<view ... xmlns="http://jmix.io/schema/flowui/view">
    <layout>
        <details id="addressDetails" summaryText="Address" opened="true">
            <fragment class="com.company.project.view.address.AddressFragment"/> <!-- include fragment by class -->
        </details>
    </layout>
</view>
```

**Important:** In CUBA, the fragment ID in `screen="..."` must exactly match the controller's `@UiController` value. In Jmix, you must use the correct class name (if you refactor or move the fragment class, update this XML). Jmix's strict class reference means no more arbitrary string IDs for fragments – the Java class is the identifier.

### Programmatic Inclusion (Using `Fragments` Bean)

Both frameworks allow adding fragments dynamically in the screen controller using a `Fragments` bean:

* **CUBA:** Inject `Fragments` and call `fragments.create(hostScreen, FragmentClass.class)`. The returned fragment controller provides a `getFragment()` method to get the visual component, which you then add to a container.
* **Jmix:** Autowire `Fragments` and call `fragments.create(this, FragmentClass.class)`. The fragment instance can be added directly to a layout component (the fragment itself represents the UI component).

**Example:**

```java
// CUBA host screen controller (programmatic fragment addition)
@Inject
private Fragments fragments;
@Inject
private GroupBoxLayout addressBox;

@Subscribe
private void onInit(InitEvent event) {
    AddressFragment addrFragment = fragments.create(this, AddressFragment.class);
    addressBox.add(addrFragment.getFragment()); // add visual component to layout
}
```

```java
// Jmix host view controller (programmatic fragment addition)
@Autowired
private Fragments fragments;
@ViewComponent
private Details addressDetails;

@Subscribe
public void onInit(InitEvent event) {
    AddressFragment addrFragment = fragments.create(this, AddressFragment.class);
    addressDetails.add(addrFragment); // fragment instance is added directly
}
```

Notice that in CUBA we call `getFragment()` on the fragment controller to retrieve the UI component, whereas in Jmix the fragment object can be added directly (it extends a Component internally). Also, we use `@ViewComponent` in Jmix to inject UI components (like `Details`), versus `@Inject` in CUBA.

## Dependency Injection in Fragment Controllers

Fragment controllers follow the same injection rules as other controllers:

* **CUBA:** Use `@Inject` for both UI components and beans. For example, injecting a UI field or a `CollectionLoader` in a fragment uses `@Inject`.
* **Jmix:** Use `@ViewComponent` for UI components defined in the fragment XML, and `@Autowired` (or constructor injection) for services and other Spring beans. This aligns with the new composition approach (UI components vs. logic beans injection).

For example, a CUBA fragment controller might have:

```java
@UiController("demo_AddressFragment")
public class AddressFragment extends ScreenFragment {

    @Inject
    private TextField<String> cityField;

    @Inject
    private CollectionLoader<City> citiesDl;

    // ...
}
```

In Jmix, the equivalent would be:

```java
@FragmentDescriptor("address-fragment.xml")
public class AddressFragment extends Fragment<FormLayout> {

    @ViewComponent
    private EntityComboBox<City> cityField;

    @ViewComponent
    private CollectionLoader<City> citiesDl;

    @Autowired
    private Messages messages; // example of injecting a bean

    // ...
}
```

**Note:** View-scoped UI beans like `DataContext` or `CollectionLoader` defined in the fragment's XML should be injected with `@ViewComponent` in Jmix (they are part of the fragment's view), whereas global singletons (e.g. `DataManager`, `Notifications`) use `@Autowired`.

## Passing Parameters to Fragments

Often you need to pass data from the host screen to the fragment (e.g., an entity instance or configuration). Both CUBA and Jmix support setting properties on the fragment controller via public setter methods:

* Define public setters in the fragment controller for each parameter.
* If adding fragment programmatically, call these setters before adding the fragment to the layout.
* If including fragment declaratively in XML, use a `<properties>` sub-element inside the `<fragment>` element.

**CUBA example (passing parameters):**

```java
// In fragment controller (CUBA)
public void setCustomer(Customer customer) {
    this.customer = customer;
}
```

```xml
<!-- In host screen XML (CUBA) -->
<fragment screen="demo_AddressFragment">
    <properties>
        <property name="customer" ref="customerDc"/>        <!-- passing data container -->
        <property name="note" value="Additional Info"/>     <!-- passing a literal value -->
    </properties>
</fragment>
```

Here `ref="customerDc"` passes the data container (or component) from the host screen. CUBA uses `ref` for references and `value` for literals.

**Jmix example (passing parameters):**

```java
// In fragment controller (Jmix)
public void setCustomerDc(CollectionContainer<Customer> customerDc) {
    customersTable.setItems(customerDc);
}

public void setNote(String note) {
    noteLabel.setText(note);
}
```

```xml
<!-- In host view XML (Jmix) -->
<fragment class="com.company.project.view.AddressFragment">
    <properties>
        <property name="customerDc" value="customersDc" type="CONTAINER_REF"/> <!-- container reference -->
        <property name="note" value="Additional Info"/>                         <!-- literal value -->
    </properties>
</fragment>
```

In Jmix, the `properties` element uses a `type` attribute for references (e.g. `CONTAINER_REF` for data container, `COMPONENT_REF` for UI components) instead of a separate `ref` attribute. Both approaches achieve the same: calling the corresponding setter with the given value/reference **before** the fragment's initialization.

## Data in Fragments and Provided Containers

Fragments can have their own data containers and loaders defined in the fragment XML, just like a screen/view. They can also leverage data from the host screen via *provided* containers:

* **Own Data:** You can declare `<data>` elements inside the fragment XML for entities specific to the fragment. Both CUBA and Jmix will merge fragment data into the host's `DataContext` (so all changes are saved together when the host screen is committed). However, **automatic data loading annotations (`@LoadDataBeforeShow`)** are not honored inside fragments. You must load data manually (e.g., by subscribing to a host event) if needed.
* **Provided Data Containers:** Mark a container or loader in the fragment XML with `provided="true"` to indicate it should be supplied by the host. The host screen/view must have a data container or loader with the **same `id`**. At runtime, the fragment will use the host's instance. Attributes like `class` or `query` in the fragment's provided data element are ignored (they exist only for design-time support).

**Example: using host's data container in fragment (CUBA vs Jmix):**

CUBA host screen defines an `addressDc`, and fragment expects it:

```xml
<!-- CUBA host screen -->
<data>
    <instance id="addressDc" class="com.company.demo.entity.Address"/>
</data>
<layout>
    <fragment screen="demo_AddressFragment"/>
</layout>
```

Fragment XML (CUBA) marking `addressDc` as provided and also having its own collection:

```xml
<fragment ...>
    <data>
        <instance id="addressDc"
                  class="com.company.demo.entity.Address"
                  provided="true"/> <!-- provided from host -->
        <collection id="citiesDc" class="com.company.demo.entity.City">
            <loader id="citiesLd">
                <query><![CDATA[select e from demo_City e]]></query>
            </loader>
        </collection>
    </data>
    <layout>
        <lookupField id="cityField"
                     optionsContainer="citiesDc"
                     dataContainer="addressDc"
                     property="city"/>
        <textField id="zipField"
                   dataContainer="addressDc"
                   property="zip"/>
    </layout>
</fragment>
```

Jmix host view similarly provides data:

```xml
<!-- Jmix host view -->
<data>
    <instance id="addressDc" class="com.company.onboarding.entity.Address">
        <loader id="addressDl"/>
    </instance>
    <collection id="citiesDc" class="com.company.onboarding.entity.City">
        <loader id="citiesDl" query="select e from City e"/>
    </collection>
</data>
<layout>
    <fragment class="com.company.onboarding.view.AddressFragment"/>
</layout>
```

Fragment XML (Jmix) with provided containers:

```xml
<fragment ...>
    <data>
        <instance id="addressDc"
                  class="com.company.onboarding.entity.Address"
                  provided="true"/>
        <collection id="citiesDc"
                    class="com.company.onboarding.entity.City"
                    provided="true"/>
    </data>
    <content>
        <formLayout dataContainer="addressDc">
            <entityComboBox id="cityField"
                            itemsContainer="citiesDc"
                            property="city"/>
            <textField id="zipcodeField" property="zipcode"/>
        </formLayout>
    </content>
</fragment>
```

In both, `provided="true"` means the fragment will look for `addressDc` (and `citiesDc` in the Jmix example) in the host. The UI components in the fragment (like `cityField`) are bound to those containers. This is useful to have fragment UI update based on host data or to edit an entity from the host inside a fragment.

⚠️ **Note:** Because fragments share the host's `DataContext`, changes made in fragment UI (e.g. editing `addressDc`) are reflected in the host and will be saved when the host screen is saved. Also, fragments currently do not support placing **facets** (non-visual controllers like `DataLoadCoordinator`) inside them. This means you cannot auto-trigger fragment data loads via `<dataLoadCoordinator>` in the fragment; instead, handle data loading in code. Typically, you subscribe to the host screen's events from the fragment controller to load or initialize fragment data. For example, in Jmix:

```java
@Subscribe(target = Target.HOST_CONTROLLER)
protected void onHostBeforeShow(View.BeforeShowEvent event) {
    citiesDl.load(); // load data when host view is about to show
}
```

(similarly in CUBA, using `@Subscribe(target = Target.PARENT_CONTROLLER)` for host's `BeforeShowEvent`). This ensures fragment data is ready when the UI is displayed.

## Conclusion

Migrating fragments from CUBA to Jmix involves adjusting the controller annotations, XML structure, and usage patterns. By comparing the code side-by-side, we see that Jmix favors **explicit class references** and Spring's injection model, while preserving the core fragment concept of reusable UI parts. Keep these differences and pitfalls in mind – especially the **XML schema changes and the way data loading is handled** – to smoothly transition your fragments to Jmix 2.x.
