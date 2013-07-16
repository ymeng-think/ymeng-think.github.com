---
layout: post
title: "如何在项目透明地引入Feature Toggle"
date: 2013-07-14 18:22
comments: true
author: Meng Yu
published: false
categories: ContinuousDelivery
Tags: [ContinuousDelivery, FeatureToggle]

---

在前几期的InfoQ专栏中刊登了一篇名为“使用功能开关更好地实现持续部署”的文章，文中讲解了特性开关与Spring的集成应用。但如果项目没有依赖Spring，又如何很好地使用特性开关呢？又如何透明地导入之呢？

接下来就对我们在项目中使用特性开关来屏蔽尚不能发布的功能这一实践进行剖析，？？？？

### 问题

我们团队正在开发开发一款在线保险产品，该产品下包含若干品牌，每个品牌有不同的目标用户群，但所提供的服务基本相同。当第一个品牌正式上线后，我们就面临一个很大挑战——既要修正上线后发现的Bug，又要继续为其它品牌继续添加新特性，且这些特性不能暂时不能反映到已上线的品牌中。

最终我们决定选择[特性开关](http://martinfowler.com/bliki/FeatureToggle.html)来解决这个问题。

### 挑战

为了能比较完美地解决我们的问题，我们所期待的特性开关应具有如下特性：

1. *新加入的特型开关不能对现有的业务结构产生影响。*<br />
   清晰的业务逻辑，简单的代码结构是保证项目长期可维护性的基础。如果特性开关的添加使业务逻辑变得复杂而不易理解，那么特性开关就在一定程度上破坏了项目的可维护性。

   添加特性开关前：
   
		inputVehicleDetails()
		createQuote()
		inputPersonalDetails()
		buyInsurance()

   添加特性开关后：

		inputVehicleDetails()
		if (brandA.isActive()) {	// Feature Toggle
			createQuote()
		}
		inputPersonalDetails()
		buyInsurance()

   可以看到，简单的条件分支虽然实现了特性开关，即部分代码只有在满足条件下才会执行，但却破坏了原有清晰的业务结构...
   
2. *当前程序中已存在了某些类似特性开关的判断，所以新加入的特性开关应能明显区分于这些代码。*

   添加特性开关前：
   
   		inputVehicleDetails()
		createQuote()
		if (currentBrand == brandB) {
			inputPersonalDetails()
		}
		buyInsurance()

   添加特性开关后：
   
   		inputVehicleDetails()
		if (brandA.isActive()) {	// Feature Toggle
			createQuote()
		}
		if (currentBrand == brandB) {	// logic that depends on currentBrand
			inputPersonalDetails()
		}
		buyInsurance()
   		 
   可以看到，新添加的特性开关（brandA.isActive()）与原有业务判定逻辑（currentBrand == brashA）十分相似，很难区分。那么，很可能被用混。
  
3. *新加入的特性开关应具有可扩展性，以便更好地对业务做出表达。*

   如：
   
   		@Brand('BrandA')
   		inputVehicleDetails() { ... }
   		
   		@Area('Austrralia')
   		inputPersonalDetails() { ... }
   		
   		inputVehicleDetails() 		// execute when current brand is 'BrandA'
		createQuote()
		inputPersonalDetails()		// execute when current location is 'Australia'
		buyInsurance()
		
	为了更加清晰地表达业务需要，我们创建了@Brand和@Area两种特性开关，但它们所依赖的核心代码应该使完全相同的。
	
4. *使用新加入的特性开关可以容易地调整需要屏蔽的功能。*

   当前inputVehicleDetails()方法只对BrandA生效，即对于其它品牌则是被屏蔽状态：
   
   		@Brand('BrandA')
   		inputVehicleDetails() { ... }
   
		inputVehicleDetails()
		createQuote()
		inputPersonalDetails()
		buyInsurance()

   如果希望BrandB也具有inputVehicleDetails()功能，则只需要将BrandB加入到@Brand标注中：

		@Brand({'BrandA', 'BrandB'})
      	inputVehicleDetails() { ... }
      	
      	...

5. *当最终所有品牌都上线后，要能很方便地将特性开关移除。*

		inputVehicleDetails() { ... }
		
	只需要简单的移除标注信息，则所有的品牌就都具有了inputVehicleDetails()功能。
	
6. *能够方便地进行测试。*
	

### 方案

#### 假想问题
由于多数日本人不喜欢番茄酱，所以日本麦当劳的汉堡中是没有蕃茄酱的；而世界上其它地方的人却都很喜欢番茄酱的味道。

下面我们有代码的形式对这个问题进行描述：

McDonalds.java

	public class McDonalds {
	
    	private Country country;
	
		public McDonalds(Country country) {
			this.country = country;
		}

    	public String makeHamburg() {
			StringBuilder desc = new StringBuilder();

			Material material = createMaterial();
			desc.append(material.bread());
			desc.append(material.sauce());
			desc.append(material.lettuce());
			desc.append(material.cutlet());
			desc.append(material.bread());
	
			return desc.toString();
		}
		
		...
	}
	
Material.java
	
	class Material {
	
    	public String bread() {
			return "Bread|";
		}

		public String lettuce() {
			return "Lettuce|";
		}
	
    	public String cutlet() {
			return "Meat|";
		}
	
		public String sauce() {
			return "TomatoSauce|";
		}
	}

ProxyToggleTest.java
	
	public class ProxyToggleTest {
	
    	@Test
		public void should_not_put_tomato_sauce_in_japan() {
    		McDonalds mcDonalds = new McDonalds(Japan);
			
        	String hamburgerDesc = mcDonalds.makeHamburg();
	
			assertThat(hamburgerDesc, is("Bread|Lettuce|Meat|Bread|"));
		}
	
		@Test
		public void should_not_put_tomato_sauce_in_other_countries_of_world() {
			McDonalds mcDonalds = new McDonalds(Others);
	
			String hamburgerDesc = mcDonalds.makeHamburg();
	
			assertThat(hamburgerDesc, is("Bread|TomatoSauce|Lettuce|Meat|Bread|"));
		}
	}

#### 解决方案一 —— 使用动态代理创建Decorator类，实现AOP功能

[动态代理](http://www.javaworld.com/jw-11-2000/jw-1110-proxy.html)利用Java的反射机制，动态创建指定对象的地代理对象，从而实现AOP功能。

使用动态代理实现特性开关的参考代码如下：

ProxyGenerator.java
    
    public class ProxyGenerator<T> {
    
        private Class<T> targetClass;
        private Object[] constructorArgs;
    
        public ProxyGenerator(Class<T> targetClass, Object[] constructorArgs) {
            this.targetClass = targetClass;
            this.constructorArgs = constructorArgs;
        }
    
        public T generate(MethodFilter methodFilter, MethodHandler methodHandler) {
            Class<?>[] argTypes = extractTypes(constructorArgs);
    
            ProxyFactory factory = new ProxyFactory();
            factory.setSuperclass(targetClass);
            factory.setFilter(methodFilter);
    
            try {
                return (T) factory.create(argTypes, constructorArgs, methodHandler);
            } catch (NoSuchMethodException e) {
                throw new RuntimeException("Can not find constructor");
            } catch (InstantiationException e) {
                throw new RuntimeException("Can not initialize action object");
            } catch (IllegalAccessException e) {
                throw new RuntimeException("Can not call constructor");
            } catch (InvocationTargetException e) {
                throw new RuntimeException("Can not call constructor");
            }
        }
    
        private Class<?>[] extractTypes(Object[] constructorArgs) {
            Constructor<?> constructor = new ConstructorFinder(targetClass, constructorArgs).find();
            return constructor.getParameterTypes();
        }
    
    }
    
ProxyGenerator类使用动态代理创建目标类的装饰器(Decorator)类。
    
Area.java

	package my.think.proxy.sample.domain.noshery;

	import javassist.util.proxy.MethodFilter;
	import javassist.util.proxy.MethodHandler;
	import my.think.proxy.ProxyGenerator;

	import java.lang.reflect.InvocationTargetException;
	import java.lang.reflect.Method;

	public final class Area {
	
		private Country country;

	    public static Area area(Country country) {
    	    return new Area(country);
	    }
	
		public <T> T create(Class<T> targetClass) {
        	return create(targetClass, new Object[0]);
	    }
	
		public <T> T create(Class<T> targetClass, Object[] args) {
        	return new ProxyGenerator<T>(targetClass, args)
    	            .generate(new MarkedByLocation(), new LocationDependingHandler());
	    }

	    private Area(Country country) {
    	    this.country = country;
	    }

		private static class MarkedByLocation implements MethodFilter {
			@Override
			public boolean isHandled(Method method) {
				return method.getAnnotation(Location.class) != null;
			}
		}

		private class LocationDependingHandler implements MethodHandler {
			@Override
			public Object invoke(
				Object target, Method method, Method methodDelegation, Object[] args) throws Throwable {
				Location markedLocation = method.getAnnotation(Location.class);
				if (markedLocation == null) {
					return originalInvocation(target, methodDelegation, args);
				}

				if (markedLocation.value() == country) {
					return originalInvocation(target, methodDelegation, args);
				}

				return new DefaultValue(method.getReturnType()).value();
			}

			private Object originalInvocation(Object target, Method method, Object[] args)
					throws IllegalAccessException, InvocationTargetException {
				return method.invoke(target, args);
			}
		}

   		...
	}

Area类用于实现对当前地区信息的判断。Area类调用ProxyGenerator类来创建目标类的代理对象，并通过传入LocationDependingHandler类将关于地区的判断插入到特定方法的前面（被@Location所标记的方法）。
	
接下来，只需要在McDonalds类和Material类中添加少许代码即可实现特性开关。

McDonalds.java

	public class McDonalds {
    	...
		
		private Material createMaterial() {
			return area(country).create(Material.class);
		}
	}
	
Material.java
	
	class Material {
		...
	
    	@Location(Others)
		public String sauce() {
			return "TomatoSauce|";
		}
	}

#### 解决方案二 —— 使用AspectJ动态编译，实现AOP功能

[AspectJ](http://eclipse.org/aspectj/)是一个面向切面的框架，它扩展了Java语言。AspectJ定义了AOP语法,所以它有一个专门的编译器用来生成遵守Java字节编码规范的Class文件。
	
FeatureToggleAspect.java

	@Aspect
	public class FeatureToggleAspect {

    	@Around("methodProxy(toggleRunner)")
	    public Object beforeExecute(ProceedingJoinPoint joinPoint, ToggleRunner toggleRunner) throws Throwable {
	        ProceedingResult processingResult = execute(joinPoint, toggleRunner);
	        if (processingResult.shouldBeExecuted()) {
	            return joinPoint.proceed();
	        }

    	    return processingResult.getDefaultValue();
	    }

    	@Pointcut(value = "@annotation(runner)")
	    public void methodProxy(ToggleRunner runner) {
    	}

	    private ProceedingResult execute(ProceedingJoinPoint joinPoint, ToggleRunner toggleRunner) {
    	    try {
        	    Runner runner = toggleRunner.value().newInstance();
            	MethodSignature signature = (MethodSignature) joinPoint.getSignature();

	            return runner.execute(signature, joinPoint.getArgs());
    	    } catch (Exception e) {
        	    throw new RuntimeException("Runner should have a default constructor.", e);
	        }
    	}
	}
	
被@Aspect标注的FeatureToggleAspect类是实现动态编译的基础。FeatureToggleAspect类通过@ToggleRunner标注得到为目标方法特别指定的Runner类。

LocationDependingRunner.java

	public class LocationDependingRunner implements Runner {

    	@Override
	    public ProceedingResult execute(MethodSignature signature, Object[] args) {
    	    Location annotation = signature.getMethod().getAnnotation(Location.class);
        	if (annotation == null) {
            	return shouldBeExecuted();
	        }

    	    Country currentCountry = (Country) args[0];
        	Country expectedScope = annotation.value();
	        if (currentCountry == expectedScope) {
    	        return shouldBeExecuted();
        	}

	        return new ProceedingResult(false, "");
    	}

	    private ProceedingResult shouldBeExecuted() {
    	    return new ProceedingResult(true, null);
	    }
	}
	
LocationDependingRunner类是Runner接口的一个实现类，因而它可以作为@ToggleRunner的参数被标记在代码中，如：@ToggleRunner(LocationDependingRunner.class)。LocationDependingRunner类提供了对于Country的判定（这里我们假定方法的第一个参数会提供Country信息），当我们将其标记在特定方法上，则这些方法在编译后就会被加入该条件判定，从而实现特性开关的要求。
	
配置文件 pom.xml
		...
	    <dependencies>
    	    <dependency>
        	    <groupId>org.aspectj</groupId>
	            <artifactId>aspectjrt</artifactId>
	            <version>1.7.3</version>
    	    </dependency>
        	<dependency>
            	<groupId>my.think</groupId>
	            <artifactId>aspect-runner</artifactId>
    	        <version>1.0-SNAPSHOT</version>
	        </dependency>
	    </dependencies>

    	<build>
        	<plugins>
				...
        	    <plugin>
                	<groupId>org.codehaus.mojo</groupId>
	                <artifactId>aspectj-maven-plugin</artifactId>
    	            <version>1.4</version>
	                <configuration>
    	                <verbose>true</verbose>
	                    <privateScope>true</privateScope>
	                    <complianceLevel>1.6</complianceLevel>
	                    <aspectLibraries>
    	                    <aspectLibrary>
         	                    <groupId>my.think</groupId>
	                            <artifactId>aspect-runner</artifactId>
                      	    </aspectLibrary>
	                    </aspectLibraries>
    	            </configuration>
        	        <executions>
            	        <execution>
                	        <goals>
								<goal>compile</goal>
								<goal>test-compile</goal>
	                        </goals>
    	                </execution>
        	        </executions>
            	</plugin>
	        </plugins>
    	</build>
    	
将aspect-runner包，即包含FeatureToggleAspect类的包，作为特殊的编译步骤配置在pom.xml文件中。

接下来，只需要在McDonalds类和Material类中添加少许代码即可实现特性开关。

McDonalds.java

	public class McDonalds {
    	...

	    public String makeHamburg() {
			...
			
        	Material material = new Material();
	        desc.append(material.sauce(country));

			...
    	}
	}

Material.java

	class Material {
    	...

    	@ToggleRunner(LocationDependingRunner.class)
	    @Location(Others)
    	public String sauce(Country country) {
        	return "TomatoSauce|";
	    }
	}
	
如上面已经谈过的，@ToggleRunner(LocationDependingRunner.class)标注说明了sauce()方法在编译时需要特别“关照”。@Location()则表明该方法只能对日本以外地区生效。

通过上面的实例介绍，可以看到无论是“动态代理”还是“动态编译”都很好地达成了“挑战”中我们对特性开关的要求，并且隐藏了底层实现细节。但是两种方案仍都略有不足之处：

* 在“动态代理”中，我们使用如下方式创建Material对象，未能做到完全透明。

		Material material = area(country).create(Material.class);
		
* 在“动态编译”中，为了能够获得国家信息，将Country作为sauce()方法的第一个参数，尽管该方法本身并不需要它。这在一定程度上影响了业务的准确表达。

		String sauce(Country country) { ... }

在我们当前的项目中，业务的清晰准确表达是第一优先的，所以我们最终选择使用“动态代理”的方式实现特性开关。


### 在项目中的应用

仿照上面的例子，我们在项目中创建了：
	
1. Brands.java - 用于标记当前代码所属的品牌。
2. 
