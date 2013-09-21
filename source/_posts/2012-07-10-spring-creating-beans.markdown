---
layout: post
title: "Прицельный обзор: Как создаются бины в Spring Framework"
date: 2012-07-10 01:09
comments: true
published: true
categories: [Spring Framework, Sighting review]
---
<p>Основной интерфейс - <code>BeanFactory</code>. Основной метод - <code>getBean</code>. Его расширяют такие интерфейсы как:</p>

<ul class="enum">
    <li><code>ListableBeanFactory</code> для доступа к спискам экземпляров созданных bean'ов</li>
    <li><code>HierarchicalBeanFactory</code> для организации иерархичных фабрик бинов</li>
    <li><code>ConfigurableBeanFactory</code> определяет методы, конфигурирующие бины</li>
</ul>

<!--more-->

<p>Этих интерфейсов бесчисленное множество. По отдельности они практически не используются, а служат для создания больших многофункциональных объектов фабрик. Для инициации загрузки необходим конфигурационный файл. За обработку источников конфигурации бинов отвечают BeanDefinition объекты:</p>

<ul class="enum">
    <li><code>PropertiesBeanDefinitionReader</code> для конфигурирования бинов в property файлах</li>
    <li><code>XmlBeanDefinitionReader</code> для конфигурирования бинов с помощью xml</li>
</ul>

<p>Естественно, наиболее популярен XML BeanDefinition, но очевидно, что скоро и от XML откажутся, как от пережитка прошлого, в пользу <em>Java</em> синтаксиса. Но это будущее, а сейчас вершина <code>FactoryBean</code> иерархии - <code>XmlBeanFactory</code>. Этот объект определяет все специфичные для создания бинов с помощью xml конфигураций реализации, такие как <code>XmlBeanDefinitionReader</code>. Настройки бина, обявленного в конфиге, хранятся в BeanDefinition объекте. Все эти объекты хранятся в реестре бинов - <code>BeanDefinitionRegistry</code>.</p>

<p>Как же строятся бины?</p>

{% codeblock %}
public <T> T getBean(Class<T> requiredType) throws BeansException {
    Assert.notNull(requiredType, "Required type must not be null");
    String[] beanNames = getBeanNamesForType(requiredType);
    if (beanNames.length > 1) {
        ArrayList<String> autowireCandidates = new ArrayList<String>();
        for (String beanName : beanNames) {
            if (getBeanDefinition(beanName).isAutowireCandidate()) {
                autowireCandidates.add(beanName);
            }
        }
        if (autowireCandidates.size() > 0) {
            beanNames = autowireCandidates.toArray(new String[autowireCandidates.size()]);
        }
    }
    if (beanNames.length == 1) {
        return getBean(beanNames[0], requiredType);
    }
    else if (beanNames.length == 0 && getParentBeanFactory() != null) {
        return getParentBeanFactory().getBean(requiredType);
    }
    else {
        throw new NoSuchBeanDefinitionException(requiredType, "expected single bean but found " +
                beanNames.length + ": " + StringUtils.arrayToCommaDelimitedString(beanNames));
    }
}
{% endcodeblock %}

<p>Сразу же бросается в глаза некий утилитарный класс <code>Assert</code>, который лишь визуально напоминает конструкцию языка java <code>assert</code>. На этом сходство заканчивается, т.к. внутри все те же стандартные java исключения типа <code>IllegalArgumentException</code>. С помощью метода <code>getBeanNamesForType</code> получается массив всех имен бинов из реестра и отбираются только те, которые соответствуют по типу и другим логическим параметрам, что, к примеру, не позволяет создавать бины из абстрактных классов. Далее, если не нашлось ни одного имени, то пытаемся получить бин из родительской фабрики бинов. <code>parentBeanFactory</code> задается в конструкторе фабрики при её создании. Если имен бинов найдено несколько, то значит конфиг некорректен. И тем не менее пытаемся в данном случае сузить круг поиска только бинами, которые помечены как внедряемые в другие классы.</p>

<p>Теперь, когда у нас есть имя бина и его тип, мы можем попытаться получить экземпляр бина. Метод создания бина достаточно большой (135 строк без сторонних используемых методов), поэтому покажу только кусочки. В первую очередь пытаемся получить объект, если это singleton и он был загружен ранее. Т.к. объект бина может быть фабрикой, то идет проверка на соответствие интерфейсу <code>FactoryBean</code>. Если это действительно фабрика, то вызывается ее фабричный метод.</p>

<p>Порадовала проверка на параллельный запуск создания одного и того же бина в одном потоке. </p>

{% codeblock %}
if (isPrototypeCurrentlyInCreation(beanName)) {
    throw new BeanCurrentlyInCreationException(beanName);
}
{% endcodeblock %}

<p>Таким образом предотвращаются циклические ссылки. Статус создания того или иного бина хранится в NamedThreadLocal переменной. </p>

<blockquote>
Далее идут иерархические хитросплетения, которые сводятся к одному, к библиотке <a href="http://cglib.sourceforge.net/" rel="nofollow">CGLib</a>. С помощью неё создаются бины в spring'e.
</blockquote>
