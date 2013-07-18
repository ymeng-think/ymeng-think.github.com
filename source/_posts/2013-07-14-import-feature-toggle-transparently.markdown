---
layout: post
title: "在项目透明地引入Feature Toggle"
date: 2013-07-14 18:22
comments: true
author: Meng Yu
published: false
categories: ContinuousDelivery
Tags: [ContinuousDelivery, FeatureToggle]

---

在前几期的InfoQ专栏中刊登了一篇名为“[使用功能开关更好地实现持续部署](http://www.infoq.com/cn/articles/function-switch-realize-better-continuous-implementations)”的文章，文中讲解了特性开关与Spring的集成应用。但如果项目没有依赖Spring，又如何很好地使用特性开关呢？又如何透明地引入之呢？

接下来就向大家介绍，在我们的项目中是如何使用特性开关透明地实现功能屏蔽的。


### 问题

我们团队正在开发开发一款在线保险产品，该产品下包含若干品牌，每个品牌有不同的目标用户群，但所提供的服务基本相同。当第一个品牌正式上线后，我们就面临一个很大挑战——既要修正上线后发现的Bug，又要继续为其它品牌继续添加新特性，且这些特性暂时不能反映到已上线的品牌中。

最终我们决定选择[特性开关](http://martinfowler.com/bliki/FeatureToggle.html)来解决这个问题。

要解决这个问题，我们可以选择“if… else… ”这样简单的特性开关模型。但是这样的特性开关又引入了其它问题：

1. *条件式特型开关会对现有的业务结构产生影响*

   清晰的业务逻辑，简单的代码结构是保证项目长期可维护性的基础。如果特性开关的添加使业务逻辑变得复杂而不易理解，那么特性开关就在一定程度上破坏了项目的可维护性。

   添加特性开关前：

``` java
	inputVehicleDetails();
	createQuote();
	inputPersonalDetails();
	buyInsurance();
```

   添加特性开关后：

``` java
	inputVehicleDetails();
	if (brandA.isActive()) {	// 条件式特性开关
		createQuote();
	}
	inputPersonalDetails();
	buyInsurance();
```

   可以看到，简单的条件分支虽然实现了特性开关——部分代码只有在满足条件时才会执行，但却破坏了原有清晰的业务结构...
   
2. *当前程序中如果已存在了某些类似特性开关的判断，条件式特性开关会造成逻辑混淆*

   添加特性开关前：
   
``` java
	inputVehicleDetails();
	createQuote();
	if (currentBrand == brandB) {
		inputPersonalDetails();
	}
	buyInsurance();
```

   添加特性开关后：

``` java
	inputVehicleDetails();
	if (brandA.isActive()) {	    // 特性开关
		createQuote();
	}
	if (currentBrand == brandB) {	// 原有依赖于品牌的条件判断
		inputPersonalDetails();
	}
	buyInsurance();
```
   		 
   可以看到，新添加入的条件式特性开关（brandA.isActive()）与原有业务判定逻辑（currentBrand == brashA）十分相似，很难区分，在使用过程中更是特别容易混淆。更糟的是，随着项目的不断深入，越来越多的条件式特性开关会被放置在代码中，代码中会充满“坏味道“，这样只会使混淆的情况进一步恶化！
   
3. *条件式特性开关并不具有可扩展性*
	
   条件式特性开关通常只是简单的条件判断，并不具有可扩展性。添加第一个条件判断与添加第十个需要写同样多的代码，并且由于判断逻辑越来越多，会令添加代码所用时间和维护成本成几何级数增长。
   
   例如：
   
``` java
	if (currentBrand == BrandA) {
		inputVehicleDetails();
	}
	if (currentBrand == BrandB) {
		createQuote();
	}
	if (currentBrand == BrandC) {
		inputPersonalDetails();
	}
	if (currentBrand == BrandD) {
		buyInsurance();
	}
```

通过上面的分析，我们可以看出简单的条件式特性开关对于我们的项目并不是最好的选择。那么我们所期待的特性开关应该具有那些特点呢？


### 期待的特点

为了更加完美地解决我们所遇到的问题，我们期待所使用的特性开关具有如下特点：

1. *不会对现有的业务结构产生影响* 
2. *不会与程序中已存在的逻辑判断相混淆*
3. *具有可扩展性*
4. *可以容易地调整需要屏蔽或开放的功能*
5. *当最终所有品牌都上线后，可以很方便地将特性开关移除*
6. *随意切换，便于测试*

所以，如果我们的特性开关如果能像下面代码所示的那样工作就好了。

``` java
	@Brand("BrandA")			// 将特性开关扩展为对“品牌”进行支持
	inputVehicleDetails() { ... }

	@Area("Austrralia")			// 将特性开关扩展为对“地区”进行支持
	inputPersonalDetails() { ... }
   	
	...
   	
	inputVehicleDetails() 		// 只有当前品牌为BrandA时，此方法才会被执行
	createQuote()
	inputPersonalDetails()		// 只有当前区域为澳洲时，此方法才会被执行
	buyInsurance()
```

太好了，上面的代码具有了我们所期待的全部特性，那么，我们究竟如何实现呢？


### 方案

通过上面的问题描述和期待特点分析，我们可以看出，特性开关作为一种基础结构不应与业务逻辑代码间存在强耦合的联系。我们既需要保持原有业务逻辑，又要在合适的位置将判断逻辑注入其中，这会使我们想到设计模式中的“代理(Proxy)模式”。

#### 1. 使用代理模式创建特性开关

> 代理模式: 为其他对象提供一种代理，并以控制对这个对象的访问。而对一个对象进行访问控制的一个原因是为了只有在我们确实需要这个对象时才对它进行创建和初始化。它是给某一个对象提供一个替代者(占位者)，使之在client对象和subject对象之间编码更有效率。

在实际应用中，我们可以创建一种名为“保护代理”的对象，即控制对象具有不同的访问权限。

{% img middle /images/tech/import_feature_toggle_transparently/proxy.png 458 248 'Create Object with Protection Proxy' 'protection proxy' %}

具体实现参见如下代码，ProxyGenerator类使用动态代理创建目标类的代理(Proxy)类。

ProxyGenerator.java

``` java
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
```
    
应用举例：

McDonalds.java

``` java
	public String makeHamburg() {
		...
		Material material = area(country).create(Material.class); // 创建Proxy对象
		desc.append(material.sauce(country));
		...
    }
```
    		
Material.java
	
``` java
	class Material {
		...
	
		@Location(Others)		// 标记需要加入判定条件的方法
		public String sauce() {
			return "TomatoSauce|";
		}
	}
```

此外，还可以通过控制编译器，在编译阶段将判定条件注入到生成的代码中。

#### 2. 使用ASpectJ动态编译创建特性开关

[AspectJ](http://eclipse.org/aspectj/)是一个面向切面的框架，它扩展了Java语言。AspectJ定义了AOP语法,所以它有一个专门的编译器用来生成遵守Java字节编码规范的Class文件。

通过AspectJ，我们可以将判定条件在编译时注入到代码中。

{% img middle /images/tech/import_feature_toggle_transparently/aspectj.png 432 259 'Compile Object with AspectJ' 'aspectj' %}

如下代码所示，FeatureToggleAspect类通过AspectJ在编译时将切入点(Runner接口的实现类)置入被ToggleRunner Annotation所标记的方法的前部。当方法被调用时，将首先执行Runner接口的实现类，对是否满足条件作出判断。如果满足，则原方法逻辑才会得到执行。

FeatureToggleAspect.java

``` java
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
```

ProceedingResult.java

``` java
	public class ProceedingResult {

		private boolean shouldBeExecuted;
		private Object defaultValue;

		public ProceedingResult(boolean shouldBeExecuted, Object defaultValue) {
			this.shouldBeExecuted = shouldBeExecuted;
			this.defaultValue = defaultValue;
		}

		public boolean shouldBeExecuted() {
			return shouldBeExecuted;
		}

		public Object getDefaultValue() {
			return defaultValue;
		}
	}
```

Runner.java

``` java
	public interface Runner {
		ProceedingResult execute(MethodSignature signature, Object[] args);
	}
```

ToggleRunner.java

``` java
	@Retention(RUNTIME)
	@Target({METHOD})
	public @interface ToggleRunner {
		Class<? extends Runner> value();
	}
```
	
应用举例：

McDonalds.java

``` java
	public class McDonalds {
		...
		public String makeHamburg() {
			...
			Material material = new Material();    // 创建普通对象
			desc.append(material.sauce(country));
			...
		}
	}
```

Material.java

``` java
	class Material {
		...
		@ToggleRunner(LocationDependingRunner.class) // 为需要注入条件的方法标记Runner类
		@Location(Others)
		public String sauce(Country country) {  // 使用方法参数将需要参与条件判断的变量提供给Runner类
			return "TomatoSauce|";
		}
	}
```


**通过上述分析我们可以看出，作为特性开关的实现，以上两种方案都是很好的选择。那么它们之间又有何不同之处呢？下面我们会从多个角度进行比较。**

1. 是否需要使用特殊的方法创建对象：

	* “代理模式方案”在创建对象时，需要使用类似反射的方式
	
		<pre>area(country).create(Material.class)</pre>

	* “AspectJ动态编译方案”则没有特殊要求
	
2. 是否需要添加特殊标记：

	* “代理模式方案”不需要在方法上添加额外标记
	
	* “AspectJ动态编译方案”需要为Runner添加特殊标记
	
		<pre>
		@ToggleRunner(LocationDependingRunner.class)
		@Location(Others)
	    public String sauce(Country country) { … }</pre>
    	
3. 是否会产生对额外参数的依赖：

	* “代理模式方案”不需要依赖额外参数
	
	* “AspectJ动态编译方案”由于需要通过参数获取参与条件判断的变量，所以会出现不必要的参数
	
		<pre>
		public String sauce(Country country) {
        	return "TomatoSauce|";
	    }</pre>
	
通过上面的比较，我们可以看出“AspectJ动态编译方案”由于条件变量的传入，会在一定程度上破坏业务的清晰表达，对代码整洁也会产生一定影响。所以在我们的项目中，最终选择了以“代理模式”创建特性开关。


### 应用

下面与大家分享一下，在我们当前项目中是如何一步步引入特性开关的。

首先，我们来看看需要加入特性开关的类。

OwnerDetailAction.java

``` java
	public class OwnerDetailAction {
		private PolicyIdentifier policyIdentifier;
		private OwnerDetailFetchingService service;
		private OwnerDetail ownerDetail;
		...
		
		public View onNext() {
			if (policyIdentifier.getCurrentBrand() == Brand.AMMI && 
					policyIdentifier.getCurrentChannel() == Channel.Internet) {
				Scale scale = service.getScale(ownerDetail.getId());
				ownerDetail.setScale(scale);
			}
			
			setCommonInfo(ownerDetails);
			return View.Continue;
		}
		...
	}
```
	
可以看到，代码中关于品牌（Brand）和渠道(Channel)的判断与正常的业务逻辑混淆在一起，严重影响了业务的清晰表达。下面，将使用特性开关重新组织上述业务结构。

*首先，将ownerDetail.setScale()抽取到一个新类中。*

OwnerDetailAction.java

``` java
	public class OwnerDetailAction {
		private PolicyIdentifier policyIdentifier;
		private OwnerDetailFetchingService service;
		private OwnerDetail ownerDetail;
		...
		
		public View onNext() {
			Brand brand = policyIdentifier.getCurrentBrand();
			Channel channel = policyIdentifier.getCurrentChannel();
		
			new ScaleSetting(brand, channel).set(ownerDetail);
			setCommonInfo(ownerDetails);
			return View.Continue;
		}
		...
	}
```
	
ScaleSetting.java

``` java
	public class ScaleSetting {
		private Brand brand;
		private Channel channel;
	
		public ScaleSetting(Brand brand, Channel channel) {
			this.brand = brand;
			this.channel = channel;
		}
	
		public set(OwnerDetail ownerDetail) {
			if (brand == Brand.AMMI && channel == Channel.Internet) {
				Scale scale = service.getScale(ownerDetail.getId());
				ownerDetail.setScale(scale);
			}
		}
	}
```

可以看出代码在业务表达上已经清爽了很多，不多判断仍然存在，只是隐藏到了ScaleSetting类中。

*接下来，我们使用由“动态代理”实现的特性开关继续改进判定逻辑的表达。*

OwnerDetailAction.java

``` java
	public class OwnerDetailAction {
		private PolicyIdentifier policyIdentifier;
		private OwnerDetailFetchingService service;
		private OwnerDetail ownerDetail;
		...
		
		public View onNext() {
			Brand brand = policyIdentifier.getCurrentBrand();
			Channel channel = policyIdentifier.getCurrentChannel();
		
			brand(brand).channel(channel).create(ScaleSetting.class);
			setCommonInfo(ownerDetails);
			return View.Continue;
		}
		...
	}
```
	
ScaleSetting.java

``` java
	public class ScaleSetting {
	
		@BrandAndChannels(AMMI_INTERNET)
		public set(OwnerDetail ownerDetail) {
			Scale scale = service.getScale(ownerDetail.getId());
			ownerDetail.setScale(scale);
		}
	}
```
	
Brand和Channel的判断逻辑已经被完全从代码中剥离了出来，业务表达变得更加清晰。并且，如果我们发现需要对AMMI上的Extranet渠道提供支持，我们只需要做如下改动：

``` java
	@BrandAndChannels({AMMI_INTERNET, AMMI_EXTRANET})
	public set(OwnerDetail ownerDetail) { … }
```
	
同理，如果未来ScaleSetting将对所有品牌开放，只需简单地将@Brands标注去掉即可。

最后我们来揭开神秘的brand().channel().create()的面纱。

BrandToggle.java

``` java
	public final class BrandToggle {
		private Brand currentBrand;
		private Channel currentChannel;
		
		private BrandToggle(Brand brand) {
			this.currentBrand = brand;
		}
		
		public static BrandToggle brand(Brand brand) {
			return new BrandToggle(brand);
		}
		
		public BrandToggle channel(Channel channel) {
			this.currentChannel = channel;
			return this;
		}
		
		public <T> T create(Class<T> targetClass) {
			return create(targetClass, new Object[0]);
		}
		
		public <T> T create(Class<T> targetClass, Object[] args) {
			return new ProxyGenerator<T>(targetClass, args).generate(new MarkedByBrands(), 
					new BrandDependingHandler());
		}
		
		private static class MarkedByBrands implements MethodFilter {
			@Override
			public boolean isHandled(Method method) {
				return method.getAnnotation(Brands.class) != null;
			}
		}
		
		private class BrandDependingHandler implements MethodHandler {
			@Override
			public Object invoke(Object targe, Method method, Method methodDelegation, Object[] args) throws Throwable {
				BrandAndChannels annotation = method.getAnnotation(BrandAndChannels.class);
				if (brands == null || containsCurrentBrand(annotation.value())) {
					return methodDelegation.invoke(target, args);
				}
				
				return new DefaultValue(method.getReturnType()).value();
			}
			
			private boolean containsCurrentBrand(BrandAndChannel[] brandAndChannels) {
				if (BrandAndChannel brandAndChannel : brandAndChannels) {
					if (brandAndChannel.is(currentBrand, currentChannel)) {
						return true;
					}
				}
				
				return false;
			}
		}
	}
```

BrandToggle类通过brand()，channel()方法很好地表达了特性开关中的“开关”概念，create()方法则将ProxyGenerator类的实现细节完全隐藏了起来。

至此，我们的系统已经能够通过特性开关实现功能的有效屏蔽，并且这个过程对于使用者来说几乎是透明的。

### 小结

“特性开关”在许多场景中都比“特性分支”具有更好的适用性和更高的效率。但是，就像所有的解决方案一样，特性开关同样也不是银弹，也存在使用的界限。只有我们很好地掌握其原理，合理地应用技术，不断改进，才能使“特性开关”这一利器在我们的项目中发挥更大的作用。

最后，衷心感谢ThoughtWorks公司高级咨询师张逸先生在本文写作过程中提供的无私帮助与建议。


---
