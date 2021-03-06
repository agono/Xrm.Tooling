# be(e) develop CrmSvcUtil.Extensions

[![Build status](https://ci.appveyor.com/api/projects/status/h08kj4jw0ogp8gex?svg=true)](https://ci.appveyor.com/project/JJauss/xrm-tooling)


## Intention
The CrmSvcUtil.exe tool which is provided by the Dynamics CRM SDK, allows to customize the process of the code generation. Out of the box, the code generation has some impediments. 

- [X] No capability to filter generated entities and attributes
- [X] Generated code is stressed with the publisher prefix (including undescores)
- [X] When customizing is not compatible with C# code some magic rules are applied to create names. No manual mapping is possible
- [ ] Enumerations are not gnerated for OptionSets
- [ ] Not possible to inject custom base classes for Entity.
- [ ] Code generation source cannot be persistet (cached) and reused during build

## How it works
The CrmSvcUtil.Extensions package contains an assembly with public types, which can bu used to inject into the CrmSvcUtil.exe execution pipeline.
Microsoft describes this process under [Create extensions for the code generation tool](https://msdn.microsoft.com/en-us/library/hh547384.aspx).

### Using the CrmSvcUtil.Extension package

1) Create the project where your generated code should be placed
1) Install and configure the Microsoft.CrmSdk.CoreTools 

```powershell
Install-Package Microsoft.CrmSdk.CoreTools
```
3) Configure the CrmSvcUtil.exe tool as described in [Create early bound entity classes with the code generation tool (CrmSvcUtil.exe)](https://msdn.microsoft.com/en-us/library/gg327844.aspx)
4) Run the first time against you master customization system to generate all entities without modifications.
1) Add the CrmSvcUtil.Extension package to you project:
```powershell
Install-Pack Beedevelop.Xrm.Tooling.CrmSvcUtil.Extensions
```
   As a result of the installation the assembly *Beedev.Xrm.CrmSvcUtil.Extensions.dll* will be copied to the `\bin\coretols` folder

6) Inject/configure the services you want to use to modify the generated code.
  The Services to be injected are placed into a well named namespace

| Parameter               | CrmSvcUtil Service-Interface    | be(e) develop CrmSvcUtil.Extensions Namespace         |
| ----------------------- | ------------------------------- | ----------------------------------------------------- |
| codecustomization       | ICustomizeCodeDomService        | *TBD*                                                 |
| codewriterfilter        | ICodeWriterFilterService        | Beedev.Xrm.CrmSvcUtil.Extensions.Filter               |
| codewritermessagefilter | ICodeWriterMessageFilterService | *TBD*                                                 |
| metadataproviderservice | IMetadataProviderService        | *TBD*                                                 |
| codegenerationservice   | ICodeGenerationService          | *TBD*                                                 |
| namingservice           | INamingService                  | Beedev.Xrm.CrmSvcUtil.Extensions.Naming.NamingService |

7) Configure the Extensions you want to use. (you can use either to commandline or the CrmSvcUtil.exe.config appSettings-Section to inject the services)

## Configure be(e) develop CrmSvcUtil.Extensions

The CrmSvcUtil.Extensions uses a Standard .NET ConfigurationSection to locate it's configuration sections.
All Configuration values have a default value which reflects the default behavior of the CrmSvcUtil.exe tool or disable the service.

To start configuring the CrmSvcUtil.Extensions you have to add the section in the CrmSvcUtil.exe.config file.
```xml
  <configSections>
    <section name="ServiceExtensions" type="Beedev.Xrm.CrmSvcUtil.Extensions.Configuration.ServiceExtensionsConfigurationSection, Beedev.Xrm.CrmSvcUtil.Extensions"/>
  </configSections>
```
If you want to configure the injected services using the _appSettings_-Section in the _CrmSvcUtil.exe.config_ you can copy from the following lines:

```xml
<appSettings>
  <add key="codewriterfilter" value="Beedev.Xrm.CrmSvcUtil.Extensions.Filter.RegExFilterService, Beedev.Xrm.CrmSvcUtil.Extensions"/>
  <add key="namingservice" value="Beedev.Xrm.CrmSvcUtil.Extensions.Naming.NamingService, Beedev.Xrm.CrmSvcUtil.Extensions"/>
</appSettings>
```

Now you are able to add configuration values as described below:
```xml
  <ServiceExtensions>
    <Filtering>
      <EntityFilter>
        <!-- configuration goes here -->
      </EntityFilter>
      <AttributeFilter>
        <!-- default filter which matches all entities and all attributes -->
        <!--<clear/>-->
        <add entity="account" expression=".*" ignoreCase="true"/>
      </AttributeFilter>
    </Filtering>
    <Naming>
      <Publisher>
        <add name="beedev_" action="remove">
      </Publisher>
      <Mapping>
        <add from="sample_entity" to="MyEntity" type="entity"/>
        <add from="sample_entity.foo" to="Bar" type="attribute"/>
      </Mapping>      
    </Naming>
  </ServiceExtensions>
```
### Filtering
The _Filtering_-Section allows you to configure the beavior of the filtering services
#### EntityFilter
The _EntityFilter_-Section allows you to configure which entities should be part of the generated code classes.
> By default the _EntityFilter_-Section contains the _FilterExpression_ "`.*`", which means that all entities are part of the output.

If you want to filter the generated output to special entities, first you should clear the collection using the `<clear/>` statement as the first child. After that the _EntityFilter_-Section allows you to add or remove _FilterElements_ the _regular expression_ in the `expression` attribute must match the entity(ies) _LogicalName_ that you want to be included in the output. The `ignoreCase` attribute (which defaults to true) you can provide the _ignoreCase_ option for the _regex_ evaluation.
```xml
<EntityFilter>
  <!-- clear the default filter which will generate all entities (.*)-->
  <clear/>
  <!-- generate the account entity-->
  <add expression="^account$" ignoreCase="true"/>
  <!-- generate all entities which matches opportinuty in it's logical name-->
  <add expression="opportunity" ignoreCase="true"/>
</EntityFilter>
```

#### AttributeFilter
The _AttributeFilter_-Section allows you to configure which attributes should be part of the generated code. The _AttributeFilter_-Section works a little bit different then the _EntityFilter_-Section. By default there is no rule defined, which results in *every attribute* is generated. In most cases you will define _expressions_ which exclude attributes on a specific entity.
```xml
<AttributeFilter>
  <!-- add all attributes for the account entity -->
  <add entity="account" expression=".*" ignoreCase="true"/>
  <!-- prevent _CurrencyBase_ attributes to be generated -->
  <add entity="opportunity" expression="(?!base)$">
</AttributeFilter>
```
### Naming
#### Publisher
The _Publisher_-Section configures how _Publisher_Prefixed should be handled.
* The _name_-Attribute identifies the *CRM-Publisher*-Prefix
* The _action_-Attribute configures the translation of the prefix

| Action | Description                             |
| ------ | --------------------------------------- |
| Remove | Removes the prefix from the Member Name |

```xml
      <Publisher>
        <!-- remove the beedev_ prefix from the entities -->
        <add name="beedev_" action="Remove"/>
      </Publisher>
```

The above configuration reults in the following code:

```csharp
/// <summary>
/// 
/// </summary>
[System.Runtime.Serialization.DataContractAttribute()]
[Microsoft.Xrm.Sdk.Client.EntityLogicalNameAttribute("beedev_sampleentity")]
[System.CodeDom.Compiler.GeneratedCodeAttribute("CrmSvcUtil", "8.2.1.8676")]
public partial class SampleEntity : Microsoft.Xrm.Sdk.Entity, System.ComponentModel.INotifyPropertyChanging, System.ComponentModel.INotifyPropertyChanged
{
  
  /// <summary>
  /// Default Constructor.
  /// </summary>
  public SampleEntity() : 
      base(EntityLogicalName)
  {
  }
  
  public const string EntityLogicalName = "beedev_sampleentity";

  // ...


  /// <summary>
  /// The name of the custom entity.
  /// </summary>
  [Microsoft.Xrm.Sdk.AttributeLogicalNameAttribute("beedev_name")]
  public string Name
  {
    get
    {
      return this.GetAttributeValue<string>("beedev_name");
    }
    set
    {
      this.OnPropertyChanging("Name");
      this.SetAttributeValue("beedev_name", value);
      this.OnPropertyChanged("Name");
    }
  }
}
``` 

#### Mapping

The _Mapping_-Section allows a detailed control for the naming of special CRM Element like Entities or Attributes.
You should avoid to map CRM Names to different Names in C# code, but some times it's nessecary. All the mapping is done using the LogicalNames instead of the SchemaNames!

* the _from_-Attribute should CRM LogicalName of the Element. For Attributes it's a combined name with _._ as a separator. (Here it's &lt;EntityLogicalName&gt;.&lt;AttributeLogicalName&gt;)
* the _to_-Attribute is the name you want to see in code.
* the _type_-Attribute should specify the type of the CRM Element you want to address. (defaults to entity)


```xml
      <Mapping>
        <add from="sample_entity" to="MyEntity" type="entity"/>
        <add from="sample_entity.foo" to="Bar" type="attribute"/>
      </Mapping>
```
