# UI View Descriptors Data Section Migration Rules 

## Common cases

### Jmix:
```xml
<data readOnly="true">
        <collection id="zonesDc"
                    class="com.company.addon.registry.entity.Zone">
            <view extends="_local">
                <property name="name"/>
            </view>
            <loader id="zonesDl">
                <query>
                    <![CDATA[select e from cmpny_Zone e]]>
                </query>
            </loader>
        </collection>
    </data>
```
### Cuba:

```xml
<data readOnly="true">
        <collection id="zonesDc"
                    class="com.company.addon.registry.entity.Zone">
            <fetchPlan extends="_base"/>
            <loader id="zonesDl">
                <query>-->
                    <![CDATA[select e from cmpny_Zone e]]>
                </query>
            </loader>
        </collection>
    </data>
```

## Uncommon cases

```xml
 <data>
        <instance id="platformDc"
                  class="com.company.addon.registry.entity.Platform"
                  view="platform-view">

            <collection id="settingsDc" property="settings"/>

            <collection id="integratorProductMappingsDc" property="integratorProductMappings">
                <collection id="integratorProductMappingValuesDc" property="values">
                    <collection id="integratorProductMappingConditionalValuesDc" property="conditionalValues"/>
                </collection>
            </collection>

            <collection id="supplierProductMappingsDc" property="supplierProductMappings">
                <collection id="supplierProductMappingValuesDc" property="values">
                    <collection id="supplierProductMappingConditionalValuesDc" property="conditionalValues"/>
                </collection>
            </collection>

            <collection id="integratorAccountMappingsDc" property="integratorAccountMappings">
                <collection id="integratorAccountMappingValuesDc" property="values">
                    <collection id="integratorAccountMappingConditionalValuesDc" property="conditionalValues"/>
                </collection>
            </collection>

            <collection id="supplierAccountMappingsDc" property="supplierAccountMappings">
                <collection id="supplierAccountMappingValuesDc" property="values">
                    <collection id="supplierAccountMappingConditionalValuesDc" property="conditionalValues"/>
                </collection>
            </collection>

            <loader id="platformDl"/>
        </instance>

        <collection id="productsDc" class="com.company.addon.registry.entity.Product">
            <loader id="productsDl"/>
        </collection>
    </data>
```

### For Jmix:

```xml
<data>
        <instance id="platformDc"
                  class="com.company.addon.registry.entity.Platform">
          
            <fetchPlan extends="platform-fetchPlan"/>

            <collection id="settingsDc" property="settings"/>

            <collection id="integratorProductMappingsDc" property="integratorProductMappings">
                <collection id="integratorProductMappingValuesDc" property="values">
                    <collection id="integratorProductMappingConditionalValuesDc" property="conditionalValues"/>
                </collection>
            </collection>

            <collection id="supplierProductMappingsDc" property="supplierProductMappings">
                <collection id="supplierProductMappingValuesDc" property="values">
                    <collection id="supplierProductMappingConditionalValuesDc" property="conditionalValues"/>
                </collection>
            </collection>

            <collection id="integratorAccountMappingsDc" property="integratorAccountMappings">
                <collection id="integratorAccountMappingValuesDc" property="values">
                    <collection id="integratorAccountMappingConditionalValuesDc" property="conditionalValues"/>
                </collection>
            </collection>

            <collection id="supplierAccountMappingsDc" property="supplierAccountMappings">
                <collection id="supplierAccountMappingValuesDc" property="values">
                    <collection id="supplierAccountMappingConditionalValuesDc" property="conditionalValues"/>
                </collection>
            </collection>

            <loader id="platformDl"/>
        </instance>

        <collection id="productsDc" class="com.company.addon.registry.entity.Product">
            <loader id="productsDl"/>
        </collection>
    </data>
```


## Fragments
If we are using fragments and want to place nested data containers that already loaded in owner screen, we need to mark fragment`s (children's) containers as `provided=true`

### For CUBA:
```xml
<data>
        <instance id="supplierDc"
                  class="com.company.addon.registry.entity.Supplier"
                  provided="true">
            <collection id="partnerPlatformSettingsDc" property="platformSettings" provided="true"/>

            <collection id="supplierProductMappingsDc" property="supplierProductsMapping" provided="true">
                <collection id="supplierProductMappingValuesDc" property="values" provided="true">
                    <collection id="supplierProductMappingConditionalValuesDc" property="conditionalValues" provided="true"/>
                </collection>
            </collection>

            <collection id="supplierAccountMappingsDc" property="supplierAccountsMapping" provided="true">
                <collection id="supplierAccountMappingValuesDc" property="values" provided="true">
                    <collection id="supplierAccountMappingConditionalValuesDc" property="conditionalValues" provided="true"/>
                </collection>
            </collection>
        </instance>

        <collection id="integrationJobProvidersDc"
                    class="com.company.addon.registry.entity.IntegrationJobProvider"
                    provided="true"/>

        <collection id="platformsDc"
                    class="com.company.addon.registry.entity.Platform"
                    provided="true"/>
    </data>
```

### For Jmix:

```xml
<data>
        <instance id="supplierDc"
                  class="com.company.addon.registry.entity.Supplier"
                  provided="true">
            <fetchPlan extends="supplier-fetchPlan"/>
            <collection id="partnerPlatformSettingsDc" property="platformSettings" provided="true"/>

            <collection id="supplierProductMappingsDc" property="supplierProductsMapping" provided="true">
                <collection id="supplierProductMappingValuesDc" property="values" provided="true">
                    <collection id="supplierProductMappingConditionalValuesDc" property="conditionalValues"
                                provided="true"/>
                </collection>
            </collection>

            <collection id="supplierAccountMappingsDc" property="supplierAccountsMapping" provided="true">
                <collection id="supplierAccountMappingValuesDc" property="values" provided="true">
                    <collection id="supplierAccountMappingConditionalValuesDc" property="conditionalValues"
                                provided="true"/>
                </collection>
            </collection>
        </instance>

        <collection id="integrationJobProvidersDc"
                    class="com.company.addon.registry.entity.IntegrationJobProvider"
                    provided="true"/>

        <collection id="platformsDc"
                    class="com.company.addon.registry.entity.Platform"
                    provided="true"/>
    </data>
```
