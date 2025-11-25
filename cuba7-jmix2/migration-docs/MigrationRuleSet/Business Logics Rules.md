# Data Access Layer

## DataManager Usage

```java
// CUBA  
DataManager dataManager = AppBeans.get(DataManager.class);  

LoadContext<Customer> context = LoadContext.create(Customer.class)  
    .setQuery(LoadContext.createQuery("select c from sales_Customer c"))  
    .setView("customer-view");  
    
List<Customer> customers = dataManager.loadList(context);  
```

```java
// Jmix  
@Autowired  
private DataManager dataManager;  


List<Customer> customers = dataManager.load(Customer.class)  
      .query("select c from sales_Customer c")  
      .fetchPlan("customer-fetchPlan")  
      .list();
```

### Rest client

Any cuba third-party clients should be replaced with RestTemplate OR RestClient (If possible, **prioritized**)

1. Added to config default module's bean
```java
@Bean("companyid_RestTemplate")
    public org.springframework.web.client.RestTemplate restTemplate() {
        return new org.springframework.web.client.RestTemplate();
    }

```

2. Use and replace where it was used:
```java
@Autowired
private RestTemplate restTemplate;


  public Account getAccountByCode(String code) {
      String url = UriComponentsBuilder.fromHttpUrl(customerProfileServiceUrl + "/v2/accounts/")
              .queryParam("code", code)
              .toUriString();
      Account account = null;
      try {
          AccountResponse response = restTemplate.getForObject(url, AccountResponse.class);
          if(response != null) {
              account = response.getAccount();
          }
      } catch (Exception e) {
          throw new RuntimeException("Can't fetch account entity", e)
      }

      return account;
  }
```

### Dependencies injection

By default try to avoid @Autowired, instead use constructor injection:

```java

class ServiceA {
   private final ServiceDependentB b;

   public ServiceA(ServiceDependentB b) {
      this.b = b;
   }

   public Object getSomeJob() {
       val res = b.getSomeJob();
       return mutateSomeJobByAservice(res);
   }
}

```

### Remove @Config bean, this no need in Spring

### Properties:

- if property is simple, you can use:
```java
@Value("${main.url}")
private String mainUrl;
```

- If properties are very hard structed, then:

```java

@Component
@ConfigurationProperties("confapp")
class ApplicationProperties {
   private SubPropOne = new SubPropOne();

  class SubPropOne {
     private String fieldOne;
     private String fieldTwo;

     private SubNested subNest = new SubNested();

     class SubNested {
        private Int valVal;

         // getters and setters
     }

     // getters and setters
  }

  // getters and setters
}
```

Props:
```
# divided by dot, if we using camel case in java / kotlin by it's fields names
confapp.sub-props-one.field-one = asda
confapp.sub-props-one.field-two = asdaasd
confapp.sub-props-one.sub-nest.val-val = 123
```
