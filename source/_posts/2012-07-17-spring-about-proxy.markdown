
---
layout: post
title: "Прицельный обзор: Зачем Spring'у proxy объекты"
date: 2012-07-17 00:01
published: true
comments: true
categories: [Spring Framework, Sighting review]
---
Как уже стало известно, в основе процесса создания [Spring](http://www.springsource.org/spring-framework) бинов лежит *CGLib* - очень распространенная библиотека для создания классов в режиме реального времени. Ее используют такие библиотеки как *Hibernate*, *iBATIS* и [многие другие](http://cglib.sourceforge.net).

<!--more-->

Иногда в *spring'e* появляется необходимость переопределить некоторые методы бина. Типичный случай - разные жизненные циклы связанных бинов. Например, singleton bean должен работать при каждом вызове своих методов с новым экземпляром связанного с ним не singleton бина. В данном случае *Spring* не сможет обеспечить такую логику, т.к. связанный бин заинжектируется в момент создания singleton бина.

>В Spring'е есть возможность указать родной для первого бина метод, при вызове которого будет создаваться и возвращаться новый экземпляр связанного бина. Помимо этого есть возможность переопределять методы с помощью `MethodReplacer`'a. Подробнее [здесь](http://static.springsource.org/spring/docs/1.2.x/reference/beans.html#beans-factory-method-injection).

Эти фичи требуют создания proxy объектов бинов, а реализуется все это безобразие с помощью всем известной библиотеки *CGLib*. После того, как beanFactory определил все метаданные для создания бина и не нашел его в кеше, то он пытается его создать следующим образом:

{% codeblock %}
public Object instantiate(RootBeanDefinition beanDefinition, String beanName, BeanFactory owner) {
	if (beanDefinition.getMethodOverrides().isEmpty()) {
		Constructor<?> constructorToUse;
		synchronized (beanDefinition.constructorArgumentLock) {
			constructorToUse = (Constructor<?>) beanDefinition.resolvedConstructorOrFactoryMethod;
			if (constructorToUse == null) {
				final Class clazz = beanDefinition.getBeanClass();
				if (clazz.isInterface()) {
					throw new BeanInstantiationException(clazz, "Specified class is an interface");
				}
				try {
					if (System.getSecurityManager() != null) {
						constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor>() {
							public Constructor run() throws Exception {
								return clazz.getDeclaredConstructor((Class[]) null);
							}
						});
					}
					else {
						constructorToUse =	clazz.getDeclaredConstructor((Class[]) null);
					}
					beanDefinition.resolvedConstructorOrFactoryMethod = constructorToUse;
				}
				catch (Exception ex) {
					throw new BeanInstantiationException(clazz, "No default constructor found", ex);
				}
			}
		}
		return BeanUtils.instantiateClass(constructorToUse);
	}
	else {
		return instantiateWithMethodInjection(beanDefinition, beanName, owner);
	}
}	
{% endcodeblock %}

В методе проверяется есть ли в `BeanDefinition` объекте переопределенные методы. Если нет, то объект просто инстанцируется с помощью *reflection*'а, иначе вызывается метод `instantiateWithMethodInjection`, в котором и реализована работа создания proxy объектов бинов через `CglibSubclassCreator`. `SubclassCreator` имеет основной метод `instantiate`:

{% codeblock %}
public Object instantiate(Constructor ctor, Object[] args) {
       Enhancer enhancer = new Enhancer();
       enhancer.setSuperclass(this.beanDefinition.getBeanClass());
       enhancer.setCallbackFilter(new CallbackFilterImpl());
       enhancer.setCallbacks(new Callback[] {
          NoOp.INSTANCE,
          new LookupOverrideMethodInterceptor(),
	  new ReplaceOverrideMethodInterceptor()
       });

       return (ctor == null) ? 
          enhancer.create() : 
	  enhancer.create(ctor.getParameterTypes(), args);
}
{% endcodeblock %}

В этом методе создается Enhancer - объект библиотеки CGLib, который конструирует proxy классы. Указывается класс, от которого стоит унаследоваться. В данном случае это класс самого бина. Далее указываются обработчики переопределяемых методов. На данный момент это три вида `MethodInterceptor`'a:

1. `NoOp.INSTANCE` - константа библиотеки CGLib, передающая управление реальным методам
2. `LookupOverrideMethodInterceptor` оборабатывает методы, возвращающие новые экземпляры бинов.
3. `ReplaceOverrideMethodInterceptor` обрабатывает методы, переопределенные с помощью `MethodReplacer`'a.

Оба эти класса объявляены внутри `SubclassCreator`'а. Метод `enhancer.create()` конструирует класс и создает экземпляр объекта этого класса. 

Стоит заметить, что `beanDefintion` и `beanFactory` хранятся в полях `SubclassCreator`'а, и объявленные в нем классы имеют доступ к ним. Эти поля помечены модификатором final, что гарантирует доступ только на чтение к значениям этих полей. 

Объект бина создан и его можно вернуть для инжектирования в клиентскую программу. При вызове методов бинов, созданных с помощью CGLib, передается управление обработчикам методов, которые указаны у proxy объекта. При создании бина были указаны 3 обработчика и `CallbackFilter`. При добавлении обработчика он получает индекс, как в массиве. `CallbackFilter` определяет какой из обработчиков выполнится, указывая его индекс. В spring'e создается свой фильтр обработчиков `CallbackFilterImpl`:

{% codeblock %}
private class CallbackFilterImpl extends CglibIdentitySupport implements CallbackFilter {
    
    public int accept(Method method) {
        MethodOverride methodOverride = beanDefinition.getMethodOverrides().getOverride(method);
        if (logger.isTraceEnabled()) {
            logger.trace("Override for '" + method.getName() + "' is [" + methodOverride + "]");
        }
        if (methodOverride == null) {
            return PASSTHROUGH;
        }
        else if (methodOverride instanceof LookupOverride) {
            return LOOKUP_OVERRIDE;
        }
        else if (methodOverride instanceof ReplaceOverride) {
            return METHOD_REPLACER;
        }
        throw new UnsupportedOperationException(
                "Unexpected MethodOverride subclass: " + methodOverride.getClass().getName());
    }
}
{% endcodeblock %}

Как можно заметить, переопределяемые методы представляют собой `MethodOverride` объект, который определяет его имя, имя объекта, которому он принадлежит и другую мета информацию о методе. Но сам по себе этот класс абстрактный, а наследуют его два класса:

1. `LookupOverride` для объявления порождающих методов
2. `ReplaceOverride` для объявления методов, переопределяемых произвольной логикой

Определяя тип переопределяемого метода, определяется индекс обработчика. Индекс обработчика задан в виде константы.

При вызове обработчика `LookupOverrideMethodInterceptor` вызывается создание бина с помощью beanFactory текущего бина по имени, которое задано в LookupOverride. Это имя попадает туда из соответствующей директивы конфигурации бинов:

{% codeblock %}
<lookup-method name="createSingleShotHelper" bean="singleShotHelper"/>
{% endcodeblock %}

При вызове `ReplaceOverrideMethodInterceptor` получается указанный в `ReplaceOverride` `MethodReplacer` и вызывается его основной метод с логикой, куда передается информация о вызванном методе, аргументах и самом объекте. Объект `MethodReplacer` также определяется в конфигурации директивой:

{% codeblock %}
<replaced-method name="computeValue" replacer="replacementComputeValue">
{% endcodeblock %}

Как видно из строчки конфигурации, задается имя бина `MethodReplacer`'a. Поэтому в `ReplaceOverride` объекте хранится просто имя, а при обработке вызова метода данным обработчиком по имени бина получается объект с помощью все той же `BeanFactory`.

Как видно из данного обзора, подход оперирования именами бинов во всех частях конфигурации в некоторых ситуациях может вызывать достаточно большой overhead при работе с proxy объектами.
