# UI UX Migration Rules

Short, practical rules for keeping **layout and UX** close to CUBA when you migrate to **Jmix Flow UI**.
Minimum text, maximum “before → after” snippets.

---

## 1. VBox / HBox: always control padding & spacing

In Flow UI, `vbox` / `hbox` are Vaadin layouts with **different defaults** than CUBA.
If you just “rename tags”, you’ll suddenly get extra gaps around content.

### 1.1. Simple vertical layout

**CUBA:**

```xml
<!-- CUBA -->
<vbox id="mainBox" spacing="true" margin="false">
    <label value="Header"/>
    <table id="customersTable" width="100%"/>
</vbox>
```

**Naive Jmix (❌ will look “fatter” than CUBA):**

```xml
<!-- Jmix - naive -->
<vbox id="mainBox">
    <label text="Header"/>
    <dataGrid id="customersDataGrid" width="100%"/>
</vbox>
```

**Correct Jmix (close to CUBA):**

```xml
<!-- Jmix - recommended -->
<vbox id="mainBox"
      padding="false"
      spacing="true"
      margin="false">
    <label text="Header"/>
    <dataGrid id="customersDataGrid" width="100%"/>
</vbox>
```

Key points:

* **Always set `padding="false"`** on `vbox`/`hbox` unless you really want extra inner space.
* Explicitly set `spacing` / `margin` to match old screen behavior.

---

### 1.2. Nested layouts (toolbars, filters, etc.)

**CUBA:**

```xml
<!-- CUBA -->
<vbox id="wrapper" spacing="true">
    <hbox id="filterBox" spacing="true" margin="false">
        <label value="Filter by:"/>
        <lookupField id="statusFilter"/>
    </hbox>

    <groupTable id="ordersTable" width="100%"/>
</vbox>
```

**Jmix – bad (toolbar suddenly padded):**

```xml
<!-- Jmix - naive -->
<vbox id="wrapper">
    <hbox id="filterBox">
        <label text="Filter by:"/>
        <comboBox id="statusFilter"/>
    </hbox>

    <dataGrid id="ordersTable" width="100%"/>
</vbox>
```

**Jmix – good (explicit padding/spacing):**

```xml
<!-- Jmix - recommended -->
<vbox id="wrapper"
      padding="false"
      spacing="true"
      margin="false">

    <hbox id="filterBox"
          padding="false"
          spacing="true"
          margin="false"
          alignItems="CENTER">
        <label text="Filter by:"/>
        <comboBox id="statusFilter"/>
    </hbox>

    <dataGrid id="ordersTable" width="100%"/>
</vbox>
```

Rule of thumb:

* **Every `hbox` / `vbox` you add → immediately think: `padding="false"`?**
* For compact toolbars: also set `spacing="false"` if you want tight buttons.

---

## 2. GroupBox with caption → Details section

In CUBA `groupBox` с `caption` был универсальной секцией формы. В Flow UI такие блоки обычно делаются через `<details>`:
получаем **заголовок + возможность сворачивать**.

### 2.1. Form section with header

**CUBA:**

```xml
<!-- CUBA -->
<groupBox id="contactGroup"
          caption="Contact info"
          collapsable="true"
          expanded="true">
    <form id="contactForm" width="100%" columns="2">
        <column>
            <textField id="emailField" caption="Email"/>
            <textField id="phoneField" caption="Phone"/>
        </column>
        <column>
            <textField id="skypeField" caption="Skype"/>
        </column>
    </form>
</groupBox>
```

**Jmix:**

```xml
<!-- Jmix -->
<details id="contactDetails"
         opened="true"
         summaryText="Contact info">
    <formLayout id="contactForm"
                width="100%"
                responsiveSteps="1, 2">
        <textField id="emailField" label="Email"/>
        <textField id="phoneField" label="Phone"/>
        <textField id="skypeField" label="Skype"/>
    </formLayout>
</details>
```

Why this is nicer in Flow UI:

* Header is `summaryText` → looks like a modern collapsible block.
* You can collapse sections on long forms by default.
* No need for extra `hbox/vbox` just to show a title.

---

### 2.2. “Fake groupBox” in CUBA → Details in Jmix

**CUBA pattern:**

```xml
<!-- CUBA: header + vbox inside -->
<groupBox id="paymentGroup" caption="Payment settings">
    <vbox spacing="true">
        <checkBox id="activeField" caption="Active"/>
        <lookupField id="currencyField" caption="Currency"/>
    </vbox>
</groupBox>
```

**Jmix version:**

```xml
<!-- Jmix: just Details -->
<details id="paymentDetails"
         opened="true"
         summaryText="Payment settings">
    <vbox padding="false" spacing="true">
        <checkbox id="activeField" label="Active"/>
        <comboBox id="currencyField" label="Currency"/>
    </vbox>
</details>
```

Rule:

* **Section with a title? → use `<details>` in Jmix**, not a random `hbox` + `label`.

---

## 3. When NOT to use groupBox in Flow UI

In Jmix Flow UI `groupBox` exists, but:

* It draws borders and extra padding.
* Over-using it makes the screen “heavy” and nested.

### 3.1. “Visual card” in CUBA → simple vbox in Jmix

**CUBA:**

```xml
<!-- CUBA: just to add border / background -->
<groupBox id="summaryBox" caption="" showAsPanel="true">
    <vbox spacing="true">
        <label value="Totals"/>
        <label id="totalAmountLabel"/>
    </vbox>
</groupBox>
```

**Jmix (better UX):**

```xml
<!-- Jmix: card-like container -->
<vbox id="summaryBox"
      classNames="jmix-card"
      padding="true"
      spacing="true">
    <label text="Totals"/>
    <label id="totalAmountLabel"/>
</vbox>
```

Or, if you want a title:

```xml
<details id="summaryDetails"
         opened="true"
         summaryText="Totals">
    <vbox padding="false" spacing="true">
        <label id="totalAmountLabel"/>
    </vbox>
</details>
```

Rule:

* Need *section* → use `details`.
* Need just a *card* → use `vbox` with `classNames="jmix-card"` (or your own style).
* Use `groupBox` only when you really want the old “frame” look.

---

## 4. Form layout & labels: keep forms readable

Big migration gotcha: labels / captions moved from `caption` → `label` attributes, and layout changed from `form` to `formLayout`.

### 4.1. Classic two-column form

**CUBA:**

```xml
<!-- CUBA -->
<form id="customerForm" width="100%" columns="2">
    <column>
        <textField id="nameField" caption="Name"/>
        <textField id="codeField" caption="Code"/>
    </column>
    <column>
        <lookupField id="statusField" caption="Status"/>
        <textArea id="commentField" caption="Comment"/>
    </column>
</form>
```

**Jmix:**

```xml
<!-- Jmix -->
<formLayout id="customerForm"
            width="100%"
            responsiveSteps="1, 2">
    <textField id="nameField" label="Name"/>
    <textField id="codeField" label="Code"/>
    <comboBox id="statusField" label="Status"/>
    <textArea id="commentField" label="Comment"/>
</formLayout>
```

UX rules here:

* Always set `width="100%"` on `formLayout` inside the view’s main layout.
* Use `responsiveSteps` instead of explicit `<column>` blocks.
* Don’t wrap each field in extra `hbox` unless you really need custom alignment.

---

## 5. Scrolling: scrollBox → scroller

Long forms behave differently if you rely on browser scroll only.
CUBA often used `scrollBox`. In Flow UI use `<scroller>` explicitly.

**CUBA:**

```xml
<!-- CUBA -->
<scrollBox id="scrollBox" width="100%" height="100%">
    <form id="longForm" columns="2">
        <!-- many fields -->
    </form>
</scrollBox>
```

**Jmix:**

```xml
<!-- Jmix -->
<scroller id="scrollBox"
          width="100%"
          height="100%">
    <formLayout id="longForm"
                width="100%"
                responsiveSteps="1, 2">
        <!-- many fields -->
    </formLayout>
</scroller>
```

Rule:

* If CUBA had `scrollBox` around your content → keep `scroller` in Jmix.
  Otherwise you can easily end up with **cut content** or **double scrollbars**.

---

## TL;DR – cheat sheet

When migrating layouts CUBA → Jmix Flow UI:

1. **Every vbox/hbox:**

   * Set `padding="false"` (and `spacing` / `margin` explicitly) to match old spacing.
2. **Sections with titles:**

   * Replace `groupBox caption="..."` with `<details opened="true" summaryText="...">`.
3. **Visual cards / frames:**

   * Prefer `vbox classNames="jmix-card"` instead of “empty” groupBox.
4. **Forms:**

   * `form` → `formLayout` with `responsiveSteps`, `label="..."` instead of `caption`.
5. **Scrolling:**

   * `scrollBox` → `scroller` to keep UX of long forms.
