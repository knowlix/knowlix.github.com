---
layout: post
title: "Mockito: Как создаются mock'и"
date: 2012-06-30 21:17
comments: true
published: true
categories: Mockito
---
<p>Работа с <a href="https://code.google.com/p/mockito/" rel="nofollow">Mockito</a> начинается, как ни странно, с класса <code>org.mockito.Mockito</code>. Этот класс содержит в себе несколько статичных методов, которые обычно и импортируются в тест-класс. Первое же, что бросается в глаза, это огромные JavaDoc комментарии. Разработчики решили вести настоящий Tutorial прямо в коде. Обнаружить код среди такого обилия комментариев без использования инструментов IDE весьма непросто. Класс <em>Mockito</em> делегирует все вызовы классу <code>org.mockito.internal.MockitoCore</code>. Если класс <em>Mockito</em> - точка входа внешнего API, то <code>MockitoCore</code> - точка входа во внутреннюю реализацию проекта. </p>

<!--more-->

<p>Создание mock объекта начинается с вызова метода <code>mock(Class&lt;T&gt; typeToMock, MockSettings settings)</code>, где</p>

<ul class="enum">
    <li>typeToMock - класс типа будущего mock'а</li>
    <li>settings - настройки mock'a</li>
</ul>

<p>Типичный вызов создания мока выглядит так:</p>

{% codeblock %}
Mockito.mock(SomeClass.class)
{% endcodeblock %}

<p>При этом настройки mock'а инициализируются по умолчанию с помощью метода <code>Mockito.withSettings()</code>, который возвращает реализацию интерфейсов настроек. А интерфейсов класса с настройками mock объектов два - <code>org.mockito.MockSettings</code> и <code>org.mockito.mock.MockCreationSettings</code>. Первый определяет методы установки параметров mock'ов, второй определяет интерфейс для доступа к этим параметрам без возможности их изменения. Соответственно, запись значений в объект производится через первый интерфейс, а все остальные классы работают с параметрами через второй интерфейс, чтобы ничего в нем случайно не изменить. Настройки можно задавать вручную при создании mock объекта. В частности можно указать произвольное имя mock'а, чтобы логи выполнения тестов стали понятнее.</p>

<p>При создании mock'а, все его методы stubb'ируются. Результат вызова метода определяется Answer объектом. Для важных методов тестирования мы задаем <code>Answer</code> объект вручную через конструкцию <code>Mockito.when(Matcher).thenAnswer(Answer)</code>, для других же объектов вызывается <code>Answer</code> по умолчанию. Конкретная реализация ответа также определяется в настройках mock'а и её можно переопределить. В настройках по умолчанию используется <code>org.mockito.internal.stubbing.defaultanswers.GloballyConfiguredAnswer</code>. Этот ответ в свою очередь делегирует запрос реализации ответа, которая определена в глобальном объекте конфигурации, который был рассмотрен в статье <a href="http://queuepy.com/blog/2012/06/26/mockito-init-mocks/">Mockito: Обработка аннотаций</a>. В конфигурации же определена реализация <code>org.mockito.internal.stubbing.defaultanswers.ReturnsEmptyValues</code>, которая возвращает пустые значения для разных типов объектов.</p>

<p>Помимо <code>GloballyConfiguredAnswer</code>, в <em>Mockito</em>, реализовано ещё несколько реализаций ответов, ссылки на эти объекты хранятся в Enum'е <code>org.mockito.Answers</code>. Также в <em>Mockito</em> ещё имеются и внутренние реализации ответов, такие как <code>ReturnsEmptyValues</code> и <code>ReturnsMoreEmptyValues</code>. Последний делегирует запрос <code>ReturnsEmptyValues</code> и дополняет его логику определением еще нескольких типов данных. Дело в том, что <code>ReturnsEmptyValues</code> возвращает null для всех объектов, кроме определенных реализаций коллекций, примитивных типов и их оберток. В какой-то момент разработчики решили поддержать еще строки и массивы, но так как кто-то, скорее всего, успел написать тесты с учетом этой специфики, то просто обновить существующую реализацию уже недостаточно. Пришлось создавать новый тип ответа <code>ReturnsMoreEmptyValues</code>. Новый ответ сейчас используется в реализации ответа <code>ReturnsSmartNulls</code>, который вместо null пытается создать и вернуть mock'реализацию данного типа.</p>

<p>Есть и другие внутренние реализации ответов, но в них нет ничего интересного. Так же имеется возможность задать в настройках объекты - слушатели, которые будут оповещены в момент вызова метода. Эти объекты реализуют интерфейс <code>org.mockito.MockSettings.InvocationListener</code>.</p>

<p>Помимо прочего, базовый класс настроек имеет метод самопроверки, который в аргументе принимает класс объекта, mock которого планируется создать, и проверяет возможность создания mock'а с данными настройками. В случае успешной проверки создается новый экземпляр объекта настроек, куда устанавливается объект имени mock'а и обработанные настройки, определенные раннее. Объект имени мока с интерфейсом <code>org.mockito.internal.util.MockName</code> служит для хранения имени мока и для выполнения некоторых операций по обработке имени мока.</p>

<p>Основной объект системы, выполняющий создание объекта, реализует интерфейс <code>org.mockito.plugins.MockMaker</code>, но напрямую с этим объектом работает только утилитарный класс <code>org.mockito.internal.util.MockUtil</code>. Именно метод <code>createMock</code> класса <code>MockUtil</code> вызывается в <code>MockitoCore#mock</code>. В первую очередь создается объект с интерфейсом <code>org.mockito.invocation.MockHandler</code>, который перехватывает вызов стаббированных методов в mock объекте. Объекты перехватчиков создаются фабрикой <code>org.mockito.internal.handler.MockHandlerFactory</code>. Создается базовая реализация обработчика <code>org.mockito.internal.handler.MockHandlerImpl</code> и оборачивается разными wrapper'ами (паттерн <em>Wrapper</em>) для навешивания дополнительного функционала. На данный момент используются следующие обертки:</p>

<ul class="enum">
    <li><code>org.mockito.internal.handler.NullResultGuardian</code> для предотвращения возвращения null'овых значений из методов, которые возвращают примитивные типы или их обертки.</li>

    <li><code>org.mockito.internal.handler.MockHandlerFactory</code> оповещает слушателей, объявленных в настройках о вызове метода.</li>
</ul>
    
<p>Далее вызывается метод <code>MockMaker#createMock</code>, куда передаются настройки и <code>MockHandler</code>. Объект <code>MockMaker'а</code> создается в статичном поле класса <code>MockUtil</code> с помощью <code>ClassPathLoader</code>, который уже использовался для загрузки пользовательской реализации класса конфигурации. Метод <code>СlassPathLoader#getMockMaker</code> возвращает заранее загруженную реализацию <code>MockMaker'a</code>. Так же как и с объектом конфигурации, имеется возможность использовать свою реализацию <code>MockMaker'а</code>, но принцип загрузки отличается:</p>

{% codeblock %}
static <T> List<T> loadImplementations(Class<T> service) {
    ClassLoader loader = Thread.currentThread().getContextClassLoader();
    if (loader == null) {
        loader = ClassLoader.getSystemClassLoader();
    }

    Enumeration<URL> resources;
    try {
        resources = loader.getResources("mockito-extensions/" + service.getName());
    } catch (IOException e) {
        throw new MockitoException("Failed to load " + service, e);
    }

    List<T> result = new ArrayList<T>();
    for (URL resource : Collections.list(resources)) {
        InputStream in = null;
        try {
            in = resource.openStream();
            for (String line : readerToLines(new InputStreamReader(in, "UTF-8"))) {
                String name = stripCommentAndWhitespace(line);
                if (name.length() != 0) {
                    result.add(service.cast(loader.loadClass(name).newInstance()));
                }
            }
        } catch (Exception e) {
            throw new MockitoConfigurationException(
                    "Failed to load " + service + " using " + resource, e);
        } finally {
            closeQuietly(in);
        }
    }
    return result;
}
{% endcodeblock %}

<p>С помощью ClassLoader'a загружается файл с именем класса, переданного в аргументе <code>service</code>. На данный момент в качестве service используется класс <code>MockMaker</code>. Файл ищется в папке с именем "mockito-extensions". В файле указываются полные имена классов - один класс на строчку. Далее эти классы загружаются и создаются их объекты. Несмотря на возможность загрузить несколько <code>MockMaker'ов</code>, будет применен только первый найденный. Если пользовательских классов нет, то загружается <code>MockMaker</code> по умолчанию <code>org.mockito.internal.creation.CglibMockMaker</code>.</p>

<blockquote>
Создание прокси классов, а mock объекты фактически ими и являются, по умолчанию обеспечивается библиотекой <a href="http://cglib.sourceforge.net/">CGLib</a>. Исходники этой библиотеки включены в проект, чтобы не заморачиваться с версией проекта и перепаковкой исходников. Конечно, <em>Maven</em> облегчил бы задачу и структура проекта стала бы яснее, но что сделано, то сделано. Разработчики не модифицируют эти исходники и в целом это запрещается. <em>CGLib</em> - библиотека, позволяющая создавать, расширять классы и интерфейсы в runtime'e.
</blockquote>

<p>Внутри <code>MockMaker</code>'a обработчик вызова стаббированных методов - <code>MockHandler</code> оборачивается внутрь класса <code>MethodInterceptorFilter</code>, реализующего интерфейс <code>MethodInterceptor</code>, который является частью <em>CGLib</em> библиотеки.</p>

<p>Как ни странно, логика создания объектов с использованием <em>CGLib</em> размыта и описана в неком утилитарном классе <code>org.mockito.internal.creation.jmock.ClassImposterizer</code>, который уже использовался при проверке настроек mock'объекта. Этот класс позаимствован из библиотеки <a href="http://www.jmock.org/">jMock</a>.</p>

<blockquote>
#Thanks to jMock guys for this handy class that wraps all the cglib magic.
</blockquote>

<p>Кстати, это не единственный класс в <em>Mockito</em> с похожим комментарием.</p>

<p>Этот класс вносит разнородность в <em>Mockito</em>, что очень некрасиво. Например, для создания экземпляров классов используется библиотека <a href="http://code.google.com/p/objenesis/">objenesis</a>. В <em>Mockito</em> же эта процедура выполняется напрямую, с помощью рефлексии.</p>

<p>В первую очередь устанавливается видимость конструкторов, класс для которого создается Mock объект. Это мы уже проходили. Решается парой методов рефлексии. Далее создается класс proxy объекта. Эта работа выполняется с помощью <em>CGLib</em> класса <code>Enhancer</code>. Этот класс конструирует класс для будущего proxy по параметрам, которые мы в него передали.</p>

{% codeblock %}
private Class<?> createProxyClass(Class<?> mockedType, Class<?>...interfaces) {
    if (mockedType == Object.class) {
        mockedType = ClassWithSuperclassToWorkAroundCglibBug.class;
    }
    
    Enhancer enhancer = new Enhancer() {
        @Override
        @SuppressWarnings("unchecked")
        protected void filterConstructors(Class sc, List constructors) {
            // Don't filter
        }
    };
    enhancer.setClassLoader(SearchingClassLoader.combineLoadersOf(mockedType));
    enhancer.setUseFactory(true);
    if (mockedType.isInterface()) {
        enhancer.setSuperclass(Object.class);
        enhancer.setInterfaces(prepend(mockedType, interfaces));
    } else {
        enhancer.setSuperclass(mockedType);
        enhancer.setInterfaces(interfaces);
    }
    enhancer.setCallbackTypes(new Class[]{MethodInterceptor.class, NoOp.class});
    try {
        return enhancer.createClass(); 
    } catch (CodeGenerationException e) {
    }
}
{% endcodeblock %}  
  
<p>Я опустил некоторые моменты по управлению секьюрностью классов и обработки исключения. Они незначительны.</p>

<p>Для начала решаем проблему с багом <em>CGLib</em>, который отказывается обрабатывать класс <code>Object</code>, так как где-то, видимо, завязывается на родителе класса. Т.е. если тип mock объекта - <code>Object</code>, то мы создаем пустой объект <code>ClassWithSuperclassToWorkAroundCglibBug</code>, с которым дальше работаем. Затем создаем объект <code>Enhancer'a</code>, и ему передаются classloader'ы классов будущих mock'ов, обернутых во внутренний класс <em>CGLib</em> для работы со всеми найденными classloader'ами как с одним. Класс, помимо прочего, будет реализовывать интерфейс <code>Factory</code>, который определяет методы установки обработчиков вызовов методов и инстанцирования этих объектов. Это внутренняя особенность <em>CGLib</em>. Далее указываем <code>Enhancer'у</code> типы классов и интерфейсов, которые он должен будет унаследовать, и типы обработчиков-перехватчиков методов классов. В данном случае это стандартные классы <code>MethodInterceptor</code>, который позволяет определять обработчики, и <code>NoOp</code>, который передает управление методам объекта напрямую - нужно для <code>Spy</code> моков.</p>

<p>Следующим шагом создается объект прокси:</p>


{% codeblock %}
private Object createProxy(Class<?> proxyClass, final MethodInterceptor interceptor) {
    Factory proxy = (Factory) objenesis.newInstance(proxyClass);
    proxy.setCallbacks(new Callback[] {interceptor, SerializableNoOp.SERIALIZABLE_INSTANCE });
    return proxy;
}
{% endcodeblock %}

<p>Как видно, objensis намного упрощает создание объектов. У разработчиков, видимо, не доходят руки, чтобы переписать старый код с использованием этой библиотеки. В качестве обработчиков событий указываем созданный ранее InternalMockHandler и пустую реализацию интерфейса NoOp.</p>

<p>Теперь остается только оповестить все компоненты системы о событии создания мока:</p>

{% codeblock %}
mockingProgress.mockingStarted(mock, typeToMock);
{% endcodeblock %}

<p>и вернуть его в класс-тест.</p>

<p>Хочу заметить, что в коде немало хардкода. Немало мест, где метод принимает параметры по общему интерфейсу, а внутри метода проверяется объект на соответствие определенному типу, который реализует этот интерфейс:</p>

{% codeblock %}
private InternalMockHandler cast(MockHandler handler) {
    if (!(handler instanceof InternalMockHandler)) {
        throw new MockitoException("At the moment you cannot provide own implementations of MockHandler." +
                "\nPlease see the javadocs for the MockMaker interface.");
    }
    return (InternalMockHandler) handler;
}
{% endcodeblock %}

<p>Но стоит отдать разработчикам должное за заботу о пользователях - расширяя проект, они стараются не менять старую часть API. Обидно было бы переписывать кучу тестов при переходе на новую версию <em>Mockito</em>.</p>