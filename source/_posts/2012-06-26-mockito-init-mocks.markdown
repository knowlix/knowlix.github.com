---
layout: post
title: "Mockito: Обработка аннотаций"
date: 2012-06-26 21:33
comments: true
published: true
categories: Mockito
---
<p>Как говорилось ранее, метод <code>MockitoAnnotations#initMocks</code> инициализирует объекты, аннотированные одной из ключевых аннотаций (<code>@Spy</code>, <code>@Mock</code>, <code>@Captor</code>, <code>@InjectMocks</code>). Сегодня мы увидим как обрабатываются аннотации. Архитектура должна быть гибкой, чтобы позволить удобно добавлять новые аннотации.</p>

<!--more-->

<p>В метод передается единственный аргумент - ссылка на объект теста. Именно по полям этого объекта будет произведен поиск аннотаций. Т.к. объект <code>MockitoAnnotations</code> является частью внешнего API <a href="https://code.google.com/p/mockito/" rel="nofollow">Mockito</a>, то любые аргументы должны быть проверены на <code>null</code>, что и сделано. В случае обнаружения <code>null</code> выкидывается <code>MockitoException</code>.</p>

<blockquote>
    В <em>Mockito</em> все исключения собраны в утилитарном объекте org.mockito.exceptions.Reporter, каждый метод которого выкидывает определенное исключение. Вообще сомнительное решение, но в данном случае (в методе initMocks этот подход проигнорирован и исключение выбрасывается напрямую).
</blockquote>

<p>Далее в методе инициализируется объект, реализующий интерфейс <code>org.mockito.configuration.AnnotationEngine</code>. Этот объект непосредственно ищет аннотации и вызывает создание соответствующего объекта. Различные реализации AnnotationEngine отвечают за разные группы аннотаций и, соответственно, подход к обработке этих групп отличается. </p>

<p>Реализация <code>AnnotationEngine</code> определена в <code>org.mockito.internal.configuration.GlobalConfiguration</code>. Это объект, хранящий базовую конфигурацию системы. Переопределить настройки можно только создав свой собственный объект, реализующий интерфейс <code>org.mockito.internal.configuration.IMockitoConfiguration</code>. <em>Mockito</em> подцепит его автоматически, если назвать и расположить его так, чтобы его наименование соответствовало наименованию, которое хранится в публичном поле <code>org.mockito.internal.configuration.ClassPathLoader#MOCKITO_CONFIGURATION_CLASS_NAME</code>. На данный момент это "org.mockito.configuration.MockitoConfiguration". За загрузку custom'ной конфигурации отвечает класс <code>ClassPathLoader</code>, который загружает её по захардкоденному наименованию. Вот так выглядит метод <code>GlobalConfiguration#createConfig</code>:</p>

{% codeblock %}
private IMockitoConfiguration createConfig() {
    IMockitoConfiguration defaultConfiguration = new DefaultMockitoConfiguration();
    IMockitoConfiguration config = new ClassPathLoader().loadConfiguration();
    if (config != null) {
        return config;
    } else {
        return defaultConfiguration;
    }
}
{% endcodeblock %}

<p>Таким образом, в случае отсутствия внешней конфигурации загружается конфигурация по умолчанию. Стоит отметить, что <code>GlobalConfiguration</code> является singletone'ом для объектов одного потока, т.к. реализация конфигурации хранится в <code>ThreadLocal</code> переменной класса <code>GlobalConfiguration</code>:</p>

{% codeblock %}
private static ThreadLocal<IMockitoConfiguration> globalConfiguration = new ThreadLocal<IMockitoConfiguration>();

public GlobalConfiguration() {
    if (globalConfiguration.get() == null) {
        globalConfiguration.set(createConfig());
    }
}
{% endcodeblock %}

<blockquote>
    В <em>Mockito</em> способ использования единого экземпляра объекта с помощью ThreadLocal переменных очень распространен и используется повсеместно. Так как тесты, в подавляющем своем большинстве, запускаются в одном потоке, то этот способ работает.
</blockquote>

<p>В <code>DefaultMockitoConfiguration#getAnnotationEngine</code> возвращается <code>org.mockito.internal.configuration.InjectingAnnotationEngine</code> - реализация <code>AnnotationEngine</code>, обрабатывающая стаббирующие аннотации (<code>@Spy</code>, <code>@Mock</code>, <code>@Captor</code>) и внедряющая их объекты в <code>@InjectMocks</code> объект. </p>

<blockquote>
InjectingAnnotationEngine - делегат. Дело в том, что аннотации добавлялись постепенно. С появлением аннотации InjectMocks появилась необходимость работать с проинициализированными объектами. Разработчики решили использовать в такой ситуации создающие AnnotationEngine внутри инжектируемого. Таким образом, инжектирующий AnnotationEngine делегирует создающим AnnotationEngine обязанность по созданию объектов, а сам затем занимается уже внедрением этих объектов.
</blockquote>

<p>Таким образом, в <code>InjectingAnnotationEngine</code> присутствуют два <code>AnnotationEngine</code>, которым делегируется обработка старых аннотаций, а именно:</p>

<ol class="enum">
    <li><code>DefaultAnnotationEngine</code> обрабатывает аннотации <code>@Mock</code> и <code>@Captor</code></li>
    <li><code>SpyAnnotationEngine</code> обрабатывает <code>@Spy</code> аннотации</li>
</ol>

<p>Сам же класс <code>InjectingAnnotationEngine</code> обрабатывает аннотации <code>@InjectMocks</code>.</p>

<p>Основной метод <code>InjectingAnnotationEngine#process</code> сначала вызывает инициализацию стаббируемых объектов, делегируя эти функции другим реализациям, затем вторым шагом внедряет эти объекты в поле, аннотированное как <code>@InjectMocks</code>:</p>

{% codeblock %}
private void processIndependentAnnotations(final Class<?> clazz, final Object testInstance) {
    Class<?> classContext = clazz;
    while (classContext != Object.class) {
        //this will create @Mocks, @Captors, etc:
        delegate.process(classContext, testInstance);
        //this will create @Spies:
        spyAnnotationEngine.process(classContext, testInstance);

        classContext = classContext.getSuperclass();
    }
}
{% endcodeblock %}

<p>Цикл while обходит все родительские классы класса-теста, чтобы проинициализировать и их аннотации. Затем вызываются последовательно методы <code>DefaultAnnotationEngine#process</code> и <code>SpyAnnotationEngine#process</code>. В них передается ссылка на класс объекта теста и на объект теста. Реализации этих методов достаточно тривиальны и сводятся к обходу полей класса, полученных с помощью метода <code>clazz.getDeclaredFields()</code> и затем для каждого поля обходится список аннотаций, полученных методом <code>field.getAnnotations()</code>. <code>DefaultAnnotationEngine#process</code> позволяет добавлять обработчики аннотаций. </p>

<p>Обработчик аннотаций реализует интерфейс <code>FieldAnnotationProcessor&lt;A&gt;</code>, где под типом параметра A подставляется тип аннотации, например <code>Mock</code>. Список поддерживаемых процессоров аннотаций хранится в Map'е по типу аннотации и заполняется в конструкторе <code>DefaultAnnotationEngine</code>. <code>FieldAnnotationProcessor#process</code> вызывает создание соответствующего объекта, как это делается пользователями библиотеки без использования аннотаций, т.е. с помощью вызова статических методов класса <code>Mockito</code> или <code>ArgumentCaptor</code>. <code>SpyAnnotationEngine#process</code> реализует логику процессорa аннотаций внутри себя, т.к. создан для обработки только одной аннотации - <code>@Spy</code>. Непонятно почему разработчики вынесли обработку этой аннотации в отдельный AnnotationEngine, а не добавили ещё один <code>FieldAnnotationProcessor</code>.</p>

<p>Процесс внедрения mock-объектов в аннотированное <code>@InjectMocks</code> поле немного сложнее. Его мы рассмотрим отдельно.</p>