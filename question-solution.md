1.  According to theory we should not have basic configuration defined in application.yml file but in bootstrap.yml. Why we need have anything there?

```
According to theory we should not have basic configuration defined in application.yml file but in bootstrap.yml. 
Why we need have anything there? At least application has to know discovery server address to be able to invoke configuration 
server. In addition, we can override default parameters for configuration invoking, 
such as config server discovery name (the default is configserver), configuration name, profile and label. 
By default microservice tries to detect configuration with name equal to ${spring.application.name}, 
label equal to ‘master’ and profiles read from ${spring.profiles.active} property.
```
