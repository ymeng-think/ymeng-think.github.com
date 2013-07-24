---
layout: post
title: "在项目中透明地引入特性开关"
date: 2013-07-14 18:22
comments: true
author: Meng Yu
published: false
categories: ContinuousDelivery
Tags: [ContinuousDelivery, FeatureToggle]

---

{% img right /images/tech/import_feature_toggle_transparently/toggle.jpg 370 245 'Toggle' 'toggle' %}

在前几期的InfoQ专栏中刊登了一篇名为“[使用功能开关更好地实现持续部署](http://www.infoq.com/cn/articles/function-switch-realize-better-continuous-implementations)”的文章，文中讲解了特性开关与Spring的集成应用。但如果项目没有依赖Spring，又如何很好地使用特性开关呢？又如何透明地引入之呢？

接下来就向大家介绍，在我们的项目中是如何使用特性开关透明地实现功能屏蔽的。


### 问题

我们的团队正在开发一款在线保险产品，该产品下包括若干品牌，每个品牌有不同的目标用户群，但所提供的服务基本相同。当第一个品牌正式上线后，我们就面临一个很大挑战——既要修正上线后发现的Bug，又要继续为其它品牌继续添加新特性，且这些特性暂时不能反映到已上线的品牌中。

最终我们决定选择[特性开关](http://martinfowler.com/bliki/FeatureToggle.html)来解决这个问题。

要解决这个问题，我们可以选择 if… else… 这样简单的特性开关模型。但是如果这样做又会引入其它问题：

* *条件式特型开关会对现有的业务结构产生影响*

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

   可以看到，简单的条件分支虽然实现了特性开关——部分代码只有在满足条件时才会执行，但却破坏了原有清晰的业务结构⋯⋯
   
* *当前程序中如果已存在了某些类似特性开关的判断，条件式特性开关会造成逻辑混淆*

   添加特性开关前：
   
``` java
	inputVehicleDetails();
	createQuote();
	if (currentBrand == brandB) {	// 原有依赖于品牌的条件判断
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
   		 
   可以看到，新添加入的条件式特性开关 brandA.isActive() 与原有业务判定逻辑 currentBrand == brashA 十分相似，很难区分，在使用过程中更是特别容易混淆。更糟的是，随着项目的不断深入，越来越多的条件式特性开关会被放置在代码中，代码中会充满“坏味道“，这样只会使混淆的情况进一步恶化！
   
* *条件式特性开关并不具有可扩展性*
	
   条件式特性开关通常只是简单的条件判断，并不具有可扩展性。添加第一个条件判断与添加第十个需要写同样多的代码，并且由于判断逻辑越来越多，会令添加代码所用时间和维护成本持续增加。
   
   例如：
   
``` java
	if (brandA.isActive()) {	    // 特性开关
		inputVehicleDetails();
	}
	if (brandB.isActive()) {	    // 特性开关
		createQuote();
	}
	if (brandC.isActive()) {	    // 特性开关
		inputPersonalDetails();
	}
	if (brandD.isActive()) {       // 特性开关
		buyInsurance();
	}
```

* *当需要移除特性开关时，我们必须删除代码*

  例如：
``` java
	inputVehicleDetails();
	if (brandA.isActive()) {	// 移除特性开关时，需要删除这行
		createQuote();
	}							// 移除特性开关时，需要删除这行
	inputPersonalDetails();
	buyInsurance();
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

通过上面的问题描述和期待特点分析，我们可以看出，特性开关作为一种基础结构不应与业务代码相混淆，它们之间不应存在强耦合的关系。我们既需要保持原有业务逻辑，又要在合适的位置将判断逻辑注入其中，这会使我们想到设计模式中的“代理(Proxy)模式”。

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

为了能更加清楚地说明如何使用特性开关，我们举一个生活中的小例子：<br />
在日本，由于绝大多数人不喜欢吃番茄酱，所以麦当劳中销售的汉堡默认是没有加番茄酱的；但是在世界的其它地方，番茄酱却是汉堡的必备佐料。

应用举例：

McDonalds.java —— 代表麦当劳，它会将汉堡卖给世界上所有喜爱它的人 ^o^

``` java
	public class McDonalds {
		private Country country;
		
		public McDonalds(Country country) {
			this.country = country;
		}

		public String makeHamburg() {
			StringBuilder desc = new StringBuilder();

			Material material = area(country).create(Material.class);
			desc.append(material.bread());
			desc.append(material.sauce());
			desc.append(material.lettuce());
			desc.append(material.cutlet());
			desc.append(material.bread());

			return desc.toString();
		}
	}
```
    		
Material.java —— 代表汉堡中的材料，包括：面包、生菜、肉饼和重要的蕃茄酱
	
``` java
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

		@Location(Others)
		public String sauce() {
			return "TomatoSauce|";
		}
	}
```

细心的读者可能已经注意到，McDonalds类中material局部变量的创建是通过 area(country).create(Material.class) 来完成的。通过area()方法我们将国家信息添加到了选材的过程中。那么，<br />
当country是日本时，汉堡的组成就会是：Bread|Lettuce|Meat|Bread|<br />
当country是其它国家时，汉堡中就会被加入番茄酱：Bread|**TomatoSauce**|Lettuce|Meat|Bread|

如果你对用代理模式生成的特性开关还心存疑问，别着急，你会从下面的“应用”环节中找到答案。

除了以上介绍的这种方法，我们还可以通过控制编译器，在编译阶段将判定条件注入到生成的代码中，以实现特性开关。

#### 2. 使用ASpectJ动态编译创建特性开关

[AspectJ](http://eclipse.org/aspectj/)是一个面向切面的框架，它扩展了Java语言。AspectJ定义了AOP语法,所以它有一个专门的编译器用来生成遵守Java字节编码规范的Class文件。

通过AspectJ，我们可以将判定条件在编译时注入到代码中。

{% img middle /images/tech/import_feature_toggle_transparently/aspectj.png 432 259 'Compile Object with AspectJ' 'aspectj' %}

如下代码所示，FeatureToggleAspect类通过AspectJ在编译时将切入点(Runner接口的实现类)置入被@ToggleRunner所标记的方法的前部。当方法被调用时，将首先执行Runner接口的实现类，对是否满足条件作出判断。如果满足，则原逻辑才会被执行。

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

我们仍用上面麦当劳的例子，来看看由AspectJ创建的特性开关是如何工作的。

McDonalds.java

``` java
	public class McDonalds {
		private Country country;

		public McDonalds(Country country) {
			this.country = country;
		}

		public String makeHamburg() {
			StringBuilder desc = new StringBuilder();

			Material material = new Material();
			desc.append(material.bread());
			desc.append(material.sauce(country));
			desc.append(material.lettuce());
			desc.append(material.cutlet());
			desc.append(material.bread());

			return desc.toString();
		}
	}
```

可以看到，Material的创建使用了标准的new运算符。但是country却被作为参数传到Material.sauce()方法中。

Material.java

``` java
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

		@ToggleRunner(LocationDependingRunner.class)
		@Location(Others)
		public String sauce(Country country) {      // 参数country的加入纯粹是为了满足特性开关的需要，并非属于逻辑代码
			return "TomatoSauce|";
		}
	}
```


**通过上述分析我们可以看出，作为特性开关的实现，以上两种方案都是很好的选择。那么它们之间又有何不同之处呢？下面我们会从多个角度进行比较。**

1. 是否需要使用特殊的方法创建对象：

	* “代理方式”在创建对象时，需要使用类似反射的方式
	
		<pre>area(country).create(Material.class)</pre>

	* “AspectJ编译方式”则没有特殊要求
	
2. 是否需要添加特殊标记：

	* “代理方式”不需要在方法上添加额外标记
	
	* “AspectJ编译方式”需要为Runner添加特殊标记
	
		<pre>
		@ToggleRunner(LocationDependingRunner.class)
		@Location(Others)
	    public String sauce(Country country) { … }</pre>
    	
3. 是否会产生对额外参数的依赖：

	* “代理方式”不会依赖额外参数
	
	* “AspectJ编译方式”由于需要通过参数获取参与条件判断的变量，所以会出现不必要的参数
	
		<pre>
		public String sauce(Country country) {  // country为“不必要”参数
        	return "TomatoSauce|";
	    }</pre>
	
通过上面的比较，我们可以看出由“AspectJ编译方式”创建的特性开关由于条件变量的传入，会在一定程度上破坏业务的清晰表达，对代码整洁也会产生一定影响。所以在我们的项目中，最终选择了以“代理模式”创建特性开关。


### 应用

下面与大家分享一下，在我们的项目中是如何一步步引入特性开关的。

首先，让我们来看看需要加入特性开关的类。

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
				InsuranceCoverage converage = service.getInsuranceCoverage(ownerDetail.getId());
				ownerDetail.setInsuranceCoverage(converage);
			}
			
			updateFamilyInfo(ownerDetails);
			return View.Continue;
		}
		...
	}
```

通过观察上面的代码可以看出，service.getInsuranceCoverage() 与 ownerDetail.setInsuranceCoverage() 是属于业务范畴的操作，而 getCurrentBrand() == Brand.AMMI && getCurrentChannel() == Channel.Internet 则是针对品牌与渠道的判断，属于特性判断的范畴，与业务并没有直接的联系。当这样的逻辑判断与正常的业务逻辑混杂在一起时，严重影响了业务的清晰表达。<br />
所以第一步我们现将这团混乱的代码抽取到一个新类中，从而保证主流程的清晰表达。

*首先，将ownerDetail.setInsuranceCoverage()抽取到一个新类中。*

InsuranceCoverageUpdater.java

``` java
	public class InsuranceCoverageUpdater {
		private Brand brand;
		private Channel channel;
	
		public InsuranceCoverageUpdater(Brand brand, Channel channel) {
			this.brand = brand;
			this.channel = channel;
		}
	
		public void update(OwnerDetail ownerDetail) {
			if (brand == Brand.AMMI && channel == Channel.Internet) {
				InsuranceCoverage converage = service.getInsuranceCoverage(ownerDetail.getId());
				ownerDetail.setInsuranceConverage(converage);
			}
		}
	}
```

之后，原来OwnerDetailAction类将被修改为：

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
		
			InsuranceCoverageUpdater insuranceConverage = new InsuranceCoverageUpdater(brand, channel)
			insuranceConverage.update(ownerDetail);
			updateFamilyInfo(ownerDetails);
			return View.Continue;
		}
		...
	}
```

虽然只是简单地做了类的抽取，但是对比之前，现在的代码在业务表达上已经清爽了很多。不过判断依然存在，只是被隐藏到了InsuranceCoverageUpdater类中。

*接下来，我们使用特性开关进一步改进逻辑表达。*

OwnerDetailAction.java

``` java
	import static com.corp.domain.BrandToggle.brand;

	public class OwnerDetailAction {
		private PolicyIdentifier policyIdentifier;
		private OwnerDetailFetchingService service;
		private OwnerDetail ownerDetail;
		...
		
		public View onNext() {
			Brand brand = policyIdentifier.getCurrentBrand();
			Channel channel = policyIdentifier.getCurrentChannel();
		
			InsuranceCoverageUpdater insuranceConverage = brand(brand).channel(channel).create(InsuranceCoverageUpdater.class);
			insuranceConverage.update(ownerDetail);
			updateFamilyInfo(ownerDetails);
			return View.Continue;
		}
		...
	}
```

可以看出，原来的 new InsuranceCoverageUpdater(brand, channel) 方法被 brand(brand).channel(channel).create(InsuranceCoverageUpdater.class) 方法所取代。<br />
此处的brand()方法是静态导入的Brand.brand()方法。通过静态导入，使对brand和channel的设定表现为链式结构，进一步增强了代码的可读性。尔后，再通过create()方法创建InsuranceCoverageUpdater类的实例。

InsuranceCoverageUpdater类中的代码也得到进一步精简。
	
InsuranceCoverageUpdater.java

``` java
	public class InsuranceCoverageUpdater {
	
		@BrandAndChannels(AMMI_INTERNET)  // 标明只有当brand为AMMI，channel为Internet时update功能才会被执行。
		public void update(OwnerDetail ownerDetail) {
			Scale scale = service.getScale(ownerDetail.getId());
			ownerDetail.setScale(scale);
		}
	}
```

虽然原有的new表达式被create方法调用所取代，但是，InsuranceCoverageUpdater类中恼人if判断逻辑却被完全移除，没有留下任何特性开关使用的痕迹。复杂的条件判断已被简单的Annotation所取代，整个代码都变得非常清爽。
	
另外，如果有进一步的开关要求需要——如对AMMI上的Extranet渠道提供支持，只需要简单地在annotation中添加AMMI_EXTRANET即可：

``` java
	@BrandAndChannels({AMMI_INTERNET, AMMI_EXTRANET})
	public void update(OwnerDetail ownerDetail) { ... }
```
	
同理，如果未来InsuranceCoverageUpdater.update()功能将对所有品牌开放，只需简单地将@BrandAndChannels标记移除即可。

最后我们来揭开 brand().channel().create() 的神秘面纱。

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

至此，我们的系统已经能够通过特性开关实现功能的有效屏蔽。通过这种方式，我们使由于特性开关的添加而造成的对原有业务的影响降到了最低，不再有恼人的if...else...表达式，只有清爽的业务结构。更重要的是，这种方式易于操作与实现，对于特性开关的使用者来说整个过程几乎是透明的。

Note: 如果您想了解特性开关的更多实现细节，可以在我的[Github](https://github.com/ymeng-think/feature-toggle)中找到相应的源代码。

### 小结

“特性开关”在许多场景中都比“特性分支”具有更好的适用性和更高的效率。但是，就像所有的解决方案一样，特性开关同样也不是银弹，也存在使用的界限。只有我们很好地掌握其原理，合理地应用技术，不断改进，才能使“特性开关”这一利器在我们的项目中发挥更大的作用。

最后，衷心感谢ThoughtWorks公司高级咨询师张逸在本文写作过程中提供的无私帮助与建议。

---

### 成文数日后的一次思考（Jul 24, 2013）

在本文写成的几日后，曾向一位同事推荐本文中的做法，因为恰好他所在的项目组需要使用特性开关来暂时隐藏一些未完成的功能，并且也希望能通过配置特性开关实现业务分支。另外，他还问道：“我们为什么不把特性开关做为一种产品或解决方案来发布呢？”

初听起来，这是一个不错的建议。但是我并不完全赞同，理由如下：

1. 虽然特性开关提供了分支选择的可能，但我们应该明确：特性开关只是用来解决项目中那5％的不同。换句话说，项目中通用的部分应该大于95％，即主要业务流程是完全相同的，只是在个别步骤上存在些许差异。如果一个项目中通用的部分只有50％，而其余的50％完全不同，我建议应该考虑其它业务分支解决方法，而不要使用特性开关，因为这不是特性开关所擅长解决的问题。
2. 当我们将特性开关作为一个产品或者一揽子的解决方案兜售给他人时，必然面对各种各样的需求，必须满足无数的特殊情况。这会使原本单纯的特性开关变得不再简单，很可能会变成一种重型框架。这不是一个好的方向，至少不是我所希望的。所以我的建议是，保持特性开关的单纯，保持功能实现的最小集，并且使他人可以根据自己的需要轻松扩展。在我看来，本文中讲到的特性开关更适合作为一种方法，一种解决特殊问题的推荐方式。

如果大家能够通过遵循正确的步骤使用特性开关，解决了困扰自己多时的问题，那么特性开关就已经产生了它最大的价值。


