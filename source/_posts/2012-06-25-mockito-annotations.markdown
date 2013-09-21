---
layout: post
title: "Mockito: Использование аннотаций"
date: 2012-06-25 15:03
comments: true
categories: Mockito
published: true
---
<p><a href="https://code.google.com/p/mockito/" rel="nofollow">Mockito</a> поддерживает аннотации с 0.9'ой версии. В той версии была введена аннотация <code>@Mock</code>, которая позволяет создавать mock-объекты. Далее были добавлены и другие аннотации:</p>

<ul class="enum">
    <li><code>@Spy</code> создает spy-объекты</li>
    <li><code>@Captor</code> создает экземпляр ArgumentCapture</li>
    <li><code>@InjectMocks</code> внедряет mock- и spy- объекты в аннотированный объект</li>
    <li><code>@Incubating</code> определяет недавно добавленный класс, с возможностью его изменения.</li>
</ul>

<!--more-->

Обработка аннотаций начинается с класса <code>org.mockito.MockitoAnnotaions</code>. В период существования только одной анотации <code>@Mock</code>, она была объявлена внутри этого класса, о чем свидетельствует deprecated аннотация. Позднее аннотации были вынесены в отдельные файлы.

Запуск обработки аннотаций осуществляется вызовом метода <code>MockitoAnnotaions#initMocks</code>. Разработчики предоставили нам два пути по работе с этим классом:

<ol class="enum">
    <li>Вручную, передав в качестве параметра ссылку на объект теста: <code>MockitoAnnotations.initMocks(this)</code></li>
    <li>С помощью junit runner'a: <code>@RunWith(MockitoJUnitRunner.class)</code></li>
</ol>

Работа Runner'a организована следующим образом. Наружу из библиотеки выглядывает объект, реализующий интерфейс <code>org.junit.runner.Runner</code> для использования непосредственно в <em>JUnit</em>. В его конструкторе с помощью <code>org.mockito.internal.runners.RunnerFactory</code> создается экземпляр внутренней реализации Runner'a.

<blockquote>
    RunnerFactory инкапсулирует логику создания новых экземпляров Runner'ов в отдельный объект org.mockito.internal.runners.util.RunnerProvider. Такое решение основано на  странном способе создания Runner'ов с помощью рефлексии. Об этом ниже.
</blockquote>

 Внутренняя реализация Runner'a (назовем его Mockito Runner) определяется версией <em>JUnit</em>, т.к. начиная с версии 4.5 основной реализацией JUnit Runner'a является класс <code>org.junit.runners.BlockJUnit4ClassRunner</code>. Соответственно, версия JUnit определяется наличием класса в classpath'е проекта:

{% codeblock %}
try {
    Class.forName("org.junit.runners.BlockJUnit4ClassRunner");
    hasJUnit45OrHigher = true;
} catch (Throwable t) {
    hasJUnit45OrHigher = false;
}
{% endcodeblock %}

А создаются эти реализации с помощью рефлексии по имени класса.
    
{% codeblock %}
if (runnerProvider.isJUnit45OrHigherAvailable()) {
    return runnerProvider.newInstance("org.mockito.internal.runners.JUnit45AndHigherRunnerImpl", klass);
} else {
    return runnerProvider.newInstance("org.mockito.internal.runners.JUnit44RunnerImpl", klass);
}
{% endcodeblock %}
    
Необходимость такого решения на данный момент выясняется у разработчиков <em>Mockito</em>.

Реализации Mockito Runner в пакете <code>org.mockito.internal.runners</code>:

<ol class="enum">
    <li><code>JUnit44RunnerImpl</code> использует <code>org.junit.internal.runners.JUnit4ClassRunner</code></li>
    <li><code>JUnit45AndHigherRunnerImpl</code> использует <code>org.junit.runners.BlockJUnit4ClassRunner</code></li>
</ol>

Mockito Runner в конструкторе создает анонимный класс, унаследованный от соответствующего JUnit Runner'a, где вызывается <code>MockitoAnnotations#initMocks</code>.

<blockquote>
    При использовании runner'ов для инициализации аннотаций Mockito автоматически добавляет свой org.junit.runner.notification.RunListener - FrameworkUsageValidator. Этот валидатор подписывается на событие testFinished и вызывает org.mockito.Mockito#validateMockitoUsage для самодиагностики.
</blockquote>

В комплекте с MockitoJUnitRunner идет еще одна реализация Runner'a - ConsoleSpammingMockitoJUnitRunner. От первого она отличается тем, что добавляет свой RunListener, который по событию testFailure логирует все сообщения. На данный момент логируются события создания mock-объектов. За коллекционирование сообщений системы отвечает класс <code>org.mockito.internal.debugging.WarningsCollector</code>, который по глобальному событию создания mock'ов логирует что и как создалось. 

Система событий в Mockito реализована своеобразно. Принцип работы отчасти позаимствован у JUnit, т.е. есть класс с методами событий. Переопределяя некоторые из них мы описываем обработчик соответствующего события. Но разработчики mockito пошли немного другим путем. Для каждого события создается отдельный интерфейс listener'a, который расширяет общий для всех listener'ов интерфейс <code>MockingProgressListener</code>. Самое интересное это то, что <code>MockingProgressListener</code> - это пустой интерфейс-маркер, а в интерфейс конкретного обработчика добавляются методы совершенно произвольной сигнатуры. На практике это выглядит следующим образом. Есть интерфейс обработчика события создания mock-объектов:

{% codeblock %}
public interface MockingStartedListener extends MockingProgressListener {
    void mockingStarted(Object mock, Class classToMock);
}
{% endcodeblock %}

И есть непосредственно обработчик <code>org.mockito.internal.listeners.CollectCreatedMocks</code>, реализующий этот интерфейс.

В Mockito есть центральный класс, который реализует интерфейс MockingProgress. Этот класс хранит состояние процесса тестирования с помощью <em>Mockito</em>. В момент установки состояния старта создания mock-объекта он оповещает обработчик соответствующего события следующим образом:

{% codeblock %}
if (listener != null && listener instanceof MockingStartedListener) {
    ((MockingStartedListener) listener).mockingStarted(mock, classToMock);
}
{% endcodeblock %}

А если ещё принять во внимание, что объект <code>MockingProgress</code> в большинстве случаев будет синглтоном, он принимает только один обработчик и метод setListener вынесен в его интерфейс, то становится страшно (надеюсь разработчики Mockito прокомментируют этот момент).