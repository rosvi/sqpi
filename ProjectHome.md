# Spring Injection into Quartz's Job Beans #
Spring provides a nice way to utilize quartz's libraries to create Jobs and populate their properties via a Map.

One of the problem's with Spring's implementation is that it keeps the instance of the Object in every cycle of the Job, explicitly declaring it as Prototype doesn't do the trick.

The problem exists up to spring 2.5.6 ( I call it a problem, but Spring doesn't seem to recognize it as a problem for now )

In our project we needed to "hack" the property injection, and to make it a little generic is always good.

## Ok, after this little appetizer , here's the beef - ##

### First, the XML Configuration ###
```
<bean id="mockJobBean" class="org.springframework.scheduling.quartz.JobDetailBean">
  <property name="jobClass" value="com.test.MockJobBean"/>
  <property name="jobDataAsMap">
    <util:map>
	<entry key="filesHandler" value-ref="trifectaFilesHandler"/> // I want this to 
                                                                    //  be Prototype!!!
        ...
    </util:map>
  </property>
</bean>

 <!-- so I'll define it in the scope, that will do the trick ;^) -->
<bean id="trifectaFilesHandler" class="com.project.handlers.TrifectaFilesHandler" scope="prototype"> 
</bean>


```

**Defined above is a simple Quartz Job Bean.
> As can see, I've explicitly declared the FilesHandler implementation class to be
> _prototype_.**

### The Quartz Job Bean Class : ###
```

public class ZipFilesHandlerJob extends QuartzJobBean implements ApplicationContextAware {
  private ApplicationContext ctx;
  private FilesHandler filesHandler;
  ...

 @Inject
  private JobBeansFactory jobBeansFactory;
 
  
  public void setFilesHandler(FilesHandler filesHandler) {
   this.filesHandler = filesHandler;
  }
	
 @Override
  protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        try {
	     Application.injectDependencies(this);
	     jobBeansFactory.setApplicationContext(ctx);
             jobBeansFactory.injectMembers(this, context.getMergedJobDataMap());
	     switch (stateKeeper.getFailureState().getStatus()) {
		   case 0:
                        log.info("There Was a Failure in Previous Directory Scan");
                        filesHandler.init();
			break;
		    case 1:
            		...
			break;
                        ...
  		    }
        	catch (IOException e) {
	        	...
		} catch (Throwable throwable) {
			...
		}
		finally {
		        ...
		}
          }
     	  catch (SchedulerException e) {
                        ...
	  }
	}

  public void setApplicationContext(ApplicationContext applicationContext)
 throws BeansException {
		ctx = applicationContext;
	}
}
```

**to override Spring's IOC I've done a couple of things :**

**1.** using Guice to inject a _JobFactory_ .

**3.** Setting the _ApplicationContext_ object to the _JobFactory_ (passing the _ApplicationContext_ is vital!)

**4.** Calling the _InjectMembers_ from the _JobFactory_ to inject the desired objects to the job )

### The Job Beans Factory class ###

```
@Singleton
public class JobBeansFactory {

 private ApplicationContext applicationContext;
  public void injectMembers(Object instance, JobDataMap jobDataMap) {
	Collection<JobProperty> properties = getProperties(instance);
	Map<String, Object> values = getBeans(properties, jobDataMap);
	for (final JobProperty property : properties) {
		Object value = values.get(property.getName());
		property.setValue(instance, value);
	}
 }

  private Collection<JobProperty> getProperties(Object instance) {
	Class<?> type = instance.getClass();
	Collection<JobProperty> properties = this.jobProperties.get(type);
	if (properties.isEmpty()) {
		for (final Method method : type.getDeclaredMethods()) {
			if (ObjectNamingUtil.isPropertySetterMethod(method)) {
				jobProperties.put(type, new JobProperty(method));
			}
		}
		properties = this.jobProperties.get(type);
	}
	return properties;
 }

  ...

```

### Last but not least, the Job Property class ###
```
public class JobProperty {

	private final Method setterMethod;
	private final String name;

	public JobProperty(Method setterMethod) {
		this.setterMethod = setterMethod;
		name = ObjectNamingUtil.getPropertyName(setterMethod);
	}

	public void setValue(Object jobInstance, Object value) {
		try {
			ReflectionUtil.invokeMethod(jobInstance, setterMethod, value);
		} catch (Throwable throwable) {
			throw new IDISystemException(throwable);
		}
	}

	public String getName() {
		return name;
	}
}
```

**Some explanations :** The ObjectNamingUtil is a simple naming utility that helps with Beans naming conventions.
**The ReflectionUtil is much like Spring's and Apache's ReflectionUtil.**

So the general idea is to use Spring's Injection for 2 things -
**1.** To inject the ApplicationContext for us.
**2.** To inject own defined Beans

It's a simple process of getting the Beans names - some from the Definitions in the XML
and others from their Runtime Name, then getting the beans using the Injected
ApplicationContext to retrieve the defined instances of the beans.

### Conclusion... ###
The only problem with this design is performance, in this scenario I'm only injecting
2 objects ( 3 if you consider the ApplicationContext that gets passed around )
We're actually overriding Springs injections in each Job's Cycle,
So until Spring realize how important this requirement is, the overhead is pretty tolerable.

Eldad.d
Java Developer/Software Engineer.