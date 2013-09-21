---
layout: post
title: "Mockito: @InjectMocks"
date: 2012-06-29 00:33
comments: true
published: true
categories: Mockito
---
<p>Процесс внедрения mock'ов разделен на несколько этапов:</p>

<ol class="enum">
    <li>Поиск полей, отмеченных аннотацией <code>@InjectMocks</code></li>
    <li>Поиск мок объектов в тесте</li>
    <li>Внедрение найденных моков в поля, помеченные аннотацией <code>@InjectMocks</code></li>
</ol>

<!--more-->

{% codeblock %}
Set<Field> mockDependentFields = new HashSet<Field>();
Set<Object> mocks = newMockSafeHashSet();
        
while (clazz != Object.class) {
    new InjectMocksScanner(clazz).addTo(mockDependentFields);
    new MockScanner(testClassInstance, clazz).addPreparedMocks(mocks);
    clazz = clazz.getSuperclass();
}
{% endcodeblock %}

<p>В <a href="https://code.google.com/p/mockito/" rel="nofollow">Mockito</a> используется собственный утилитарный класс <code>Sets</code>, с помощью которого создаются внутренние реализации множеств. В данном случае заимпортирован один из его статичных методов newMockSafeHashSet. Метод создает реализацию под названием <code>org.mockito.internal.util.collections.HashCodeAndEqualsSafeSet</code>. Создана отдельная реализация <code>Set</code> интерфейса лишь для того, чтобы скрыть особенность реализации моков. А именно, есть некоторая проблема в стаббинге методов <code>Object.equals</code> и <code>Object.hashCode</code>, и они могут выкидывать <code>NullPointerException</code>. Истинную причину выясним при разборе процесса subbing'а. При работе в коллекциях эти методы интенсивно используются при поиске и добавлении элементов, поэтому мы оборачиваем мок объект в этот wrapper, который использует свои методы <code>equals</code> и <code>hashCode</code>. Оборачивание моков происходит в момент добавления элементов в множество.</p>

<p>Вернемся к примеру. В множество mocks у нас будут добавлены все найденные mock-объекты. Стоит заметить, что в Mockito много переменных модифицируются, будучи переданными через аргументы. Так и в данном случае коллекции наполяются именно так. Immutability здесь явно не хватает. <code>org.mockito.internal.configuration.injection.scanner.InjectMocksScanner</code> реализует алгоритм поиска полей, отмеченных аннотацией <code>@InjectMocks</code>. Основные два метода класса:</p>

{% codeblock %}
private Set<Field> scan() {
    Set<Field> mockDependentFields = new HashSet<Field>();
    Field[] fields = clazz.getDeclaredFields();
    for (Field field : fields) {
        if (null != field.getAnnotation(InjectMocks.class)) {
            assertNoAnnotations(field, Mock.class, MockitoAnnotations.Mock.class, Captor.class);
            mockDependentFields.add(field);
        }
    }

    return mockDependentFields;
}

void assertNoAnnotations(final Field field, final Class... annotations) {
    for (Class annotation : annotations) {
        if (field.isAnnotationPresent(annotation)) {
            new Reporter().unsupportedCombinationOfAnnotations(annotation.getSimpleName(), InjectMocks.class.getSimpleName());
        }
    }
}
{% endcodeblock %}
    
<p>Именно результат метода scan и добавляет в наше множество. В цикле пробегаем по полям класса-теста, проверяем есть ли у поля аннотация <code>@InjectMocks</code>. Стоит отметить, что используются системные java классы полей и классов. Пока, по-моему, ничего сверхъестественного. Далее, если метод аннотирован, то проверяем не аннотирован ли он еще и стаббирующими аннотациями (<code>@Mock</code>, <code>@Captor</code>), т.к. это противоречит логике и не может быть допущено. В случае нарушения правил выбрасывается <code>RuntimeException</code>. Если все в порядке, то мы можем добавить поле в наше множество.</p>

<p><code>org.mockito.internal.configuration.injection.scanner.MockScanner</code> реализуется второй этап алгоритма внедрения моков - поиск мок объектов в тесте. Основные методы класса:</p>

{% codeblock %}
private Set<Object> scan() {
    Set<Object> mocks = newMockSafeHashSet();
    for (Field field : clazz.getDeclaredFields()) {
        // mock or spies only
        FieldReader fieldReader = new FieldReader(instance, field);

        Object mockInstance = preparedMock(fieldReader.read(), field);
        if (mockInstance != null) {
            mocks.add(mockInstance);
        }
    }
    return mocks;
}

private Object preparedMock(Object instance, Field field) {
    if (isAnnotatedByMockOrSpy(field)) {
        return instance;
    } else if (isMockOrSpy(instance)) {
        mockUtil.maybeRedefineMockName(instance, field.getName());
        return instance;
    }
    return null;
}

private boolean isAnnotatedByMockOrSpy(Field field) {
    return null != field.getAnnotation(Spy.class)
            || null != field.getAnnotation(Mock.class)
            || null != field.getAnnotation(MockitoAnnotations.Mock.class);
}

private boolean isMockOrSpy(Object instance) {
    return mockUtil.isMock(instance)
            || mockUtil.isSpy(instance);
}
{% endcodeblock %}

<p>Класс выполнен в стиле <code>InjectMocksScanner</code>, хотя общего интерфейса у классов нет. Но это не страшно, т.к. выполняют они сугубо утилитарные функции. Результат метода scan помещается в множество найденных мок объектов для внедрения. За чтение значений полей класса из конкретных объектов отвечает FieldReader. Основная строчка его метода read:</p>

{% codeblock %}
return field.get(target)
{% endcodeblock %}

<p>В методе <code>MockScanner#preparedMock</code> определяется, является ли полученный из поля объект mock'ом. Определяется он по наличию в поле аннотаций <code>@Mock</code> или <code>@Spy</code>. Если аннотаций нет, то есть вероятность, что объекты были стаббированы вручную. Это проверяется в методе <code>MockScanner#isMockOrSpy</code>. В случае, если объект все-таки является <em>Mock</em> объектом, то возможно требуется его переименование. Необходимость такого решения возможно выяснится позднее, когда мы разберем процесс создания <em>Mock</em> объектов. На данный момент это выглядит каким то костылем.

<p>Осталось дело за малым - внедрить мок-объекты. Отправная точка <code>org.mockito.internal.configuration#DefaultInjectionEngine</code> - объект, который инициализирует обработчиков-внедренцев. И запускает процесс инжектирования. Внедрением занимаются <code>org.mockito.internal.configuration.injection.MockInjectionStrategy</code> объекты. На данный момент моки инжектируются через конструктор (<code>ConstructorInjection</code>), поля класса и сеттеры (<code>PropertyAndSetterInjection</code>).</p>

<blockquote>
    Поле, помеченное аннотацией @InjectMocks, может быть помечено ещё и аннотацией @Spy, что заставляет, в случае неуспешного внедрения моков в объект, создавать из него Spy-объект. Хотя в целом JavaDoc полный, этот факт скрыт от глаз пользователей.
</blockquote>

<p>Для обработки <code>@Spy</code> аннотации, если не получилось создать объект, в который внедрялись моки, используется <code>SpyOnInjectedFieldsHandler</code>. Основной метод <code>DefaultInjectionEngine</code> - <code>#injectMocksOnFields</code>:

{% codeblock %}
public void injectMocksOnFields(Set<Field> needingInjection, Set<Object> mocks, Object testClassInstance) {
    MockInjection.onFields(needingInjection, testClassInstance)
            .withMocks(mocks)
            .tryConstructorInjection()
            .tryPropertyOrFieldInjection()
            .handleSpyAnnotation()
            .apply();
}
{% endcodeblock %}

<p>Как видно, объект-конфигуратор процесса внедрения - <code>org.mockito.internal.configuration.injection.MockInjection</code>. Он определяет внутри себя реализации той или иной стратегии. Поэтому его методы <code>tryConstructorInjection</code>, <code>tryPropertyOrFieldInjection</code> и <code>handleSpyAnnotation</code> возвращают именно те классы стратегий, которые мы рассмотрели выше. Метод <code>onFields</code> возвращает объект <code>OngoingMockInjection</code> - внутренний класс класса <code>MockInjection</code>. Его тип нас не должен интересовать, т.к. проектировался этот объект для использования здесь и сейчас в виде Builder'а, что обеспечивается возвращением из каждого его метода ссылки на самого себя. Этот объект-builder сохраняет в конструкторе поля с аннотацией <code>@InjectMocks</code>. Далее метод withMocks сохраняет список mock объектов. Странный подход к инициализации базовых данных. Практичнее было бы внести сохранение списка mock объектов в конструктор, т.к. инжектирование без них смысла не имеет. А так как порядок выполнения Builder методов не регулируется, то можно пропустить этот метод. Далее 3'мя следующими методами объявляем стратегии инжектирования, которые хотим использовать. Стратегии инжектирования образуют цепочку стратегий по примеру паттерна Composite. Вызывая основной метод первого из них, он обрабатывает основную логику, затем вызывается аналогичный метод связанного с ним объекта-внедренца. И так до последнего. На практике это реализуется следующим образом:</p>
 
{% codeblock %}
private MockInjectionStrategy injectionStrategies = MockInjectionStrategy.nop();
{% endcodeblock %}
 
<p>В <code>OngoingMockInjection</code> создается поле с пустой стратегией обработчиком, где nop() - фабричный метод возвращающий пустую реализацию:</p>
 
{% codeblock %}
 public static final MockInjectionStrategy nop() {
    return new MockInjectionStrategy() {
        protected boolean processInjection(Field field, Object fieldOwner, Set<Object> mockCandidates) {
            return false;
        }
    };
}
{% endcodeblock %}
    
<p><code>processInjection</code> - основной абстрактный метод MockInjectionStrategy, в котором определяется логика внедрения моков. Методы добавления следующего в цепочке обработчика выглядят следующим образом:</p>

{% codeblock %}
public OngoingMockInjection tryConstructorInjection() {
    injectionStrategies.thenTry(new ConstructorInjection());
    return this;
}

public OngoingMockInjection tryPropertyOrFieldInjection() {
    injectionStrategies.thenTry(new PropertyAndSetterInjection());
    return this;
}
{% endcodeblock %}

<p>Где метод <code>thenTry</code> передает добавляемый объект следующему в цепочке объекту, пока не будет достигнут последний объект в цепочке. Он то и примет к себе новое звено:</p>
    
{% codeblock %}
public MockInjectionStrategy thenTry(MockInjectionStrategy strategy) {
    if(nextStrategy != null) {
        nextStrategy.thenTry(strategy);
    } else {
        nextStrategy = strategy;
    }
    return strategy;
}
{% endcodeblock %}

<p>Метод <code>OngoingMockInjection#apply</code>, который завершает цепочку методов Builder'а, выполняет метод injectionStrategies.process, первого из цепочки стратегий обработки объекта для каждого поля, аннотированного <code>@InjectMocks</code>:</p>

{% codeblock %}
public boolean process(Field onField, Object fieldOwnedBy, Set<Object> mockCandidates) {
    if(processInjection(onField, fieldOwnedBy, mockCandidates)) {
        return true;
    }
    return relayProcessToNextStrategy(onField, fieldOwnedBy, mockCandidates);
}
{% endcodeblock %}
    
<p>Где осуществляется вызов метода с логикой обработки, а затем в методе relayProcessToNextStrategy передается управление следующему объекту в цепи.</p>

<p>Метод <code>OngoingMockInjection#handleSpyAnnotation</code> прикрепляет финальный обработчик <code>@Spy</code> аннотаций в другой <code>MockInjectionStrategy#nop()</code>, который также вызывается для каждого поля в методе apply после вызова цепочки стратегий внедрения.</p>

<p>Рассмотрим алгоритм внедрения через конструктор:</p>

<p>ConstructorInjection определяет внутри себя реализацию ConstructorArgumentResolver'а, где определяется алгоритм получения mock'ов из найденного заранее множества, которые могут быть приравнены к типам аргументов конструктора.</p>

{% codeblock %}
public Object[] resolveTypeInstances(Class<?>... argTypes) {
    List<Object> argumentInstances = new ArrayList<Object>(argTypes.length);
    for (Class<?> argType : argTypes) {
        argumentInstances.add(objectThatIsAssignableFrom(argType));
    }
    return argumentInstances.toArray();
}
{% endcodeblock %}
        
<p>Где метод objectThatIsAssignableFrom перебирает множество mock объектов и ищет первое из них, которое удовлетворяет условию:</p>

{% codeblock %}
argType.isAssignableFrom(mock.getClass())
{% endcodeblock %}

<p>Выбор конструктора, определение его параметров и создание нового экземпляра класса для поля, аннотированного <code>@InjectMocks</code>, осуществляется в классе <code>org.mockito.internal.util.reflection.FieldInitializer</code>. Внутри определены две реализации <code>ConstructorInstantiator</code> для обработки - конструктор с параметрами и без параметров. В ConstructorInjection используется <code>ParameterizedConstructorInstantiator</code>, так как внедрение mock объектов в конструктор без параметров смысла не имеет. Соответственно, первое что делается - это проверка - не создан ли объект в целевом поле.</p>

{% codeblock %}
private FieldInitializationReport acquireFieldInstance() throws IllegalAccessException {
    Object fieldInstance = field.get(fieldOwner);
    if(fieldInstance != null) {
        return new FieldInitializationReport(fieldInstance, false, false);
    }

    return instantiator.instantiate();
}
{% endcodeblock %}

<p>Значение объекта получается уже знакомым способом через метод get. Если объект уже в поле создан, то нет смысла идти дальше - возвращаем результат о неуспешном создании объекта. <code>FieldInitializationReport</code> - класс с данными результатов создания класса. <code>instantiator</code> - это и есть один из экземпляров <code>ConstructorInstantiator</code>. По методу <code>instantiate</code> выполняется основная логика поиска соответствующего конструктора и создания самого объекта. Основное содержание метода:</p>

{% codeblock %}
constructor = biggestConstructor(field.getType());
changer.enableAccess(constructor);

final Object[] args = argResolver.resolveTypeInstances(constructor.getParameterTypes());
Object newFieldInstance = constructor.newInstance(args);
new FieldSetter(testClass, field).set(newFieldInstance);

return new FieldInitializationReport(field.get(testClass), false, true);
{% endcodeblock %}
    
<p>С помощью метода <code>biggestConstructor</code> (кстати, неудачное название, т.к. не отражает действия) определяется конструктор с наибольшим количеством параметров. Реализация тривиальна.</p>

<p>С помощью метода changer.enableAccess изменяем доступность конструктора:</p>

{% codeblock %}
constructor.setAccessible(true);
{% endcodeblock %}

<p>При этом сохраняем старое значение accesible, чтобы вернуть его в конце метода в блоке finally:</p>

{% codeblock %}
finally {
    if(constructor != null) {
        changer.safelyDisableAccess(constructor);
    }
}
{% endcodeblock %}

<p><code>argResolver</code> - реализация <code>ConstructorArgumentResolver</code>, которая определяется внутри <code>ConstructorInjection</code> (о нем мы говорили выше). С помощью него мы получаем массив mock объектов в нужном нам порядке и создаем с помощью них новый экземпляр класса целевого поля. <code>FieldSetter</code> - противоположность <code>FieldReader'а</code>. Вставляет значение в поле, при этом также меняя видимость поля. В конце возвращается <code>FieldInitializationReport</code> с отчетом об успешном создании объекта. В дальнейшем из этого объекта получится флаг <code>fieldWasInitializedUsingContructorArgs</code>,и если он равен <code>true</code>, то вызов следующих обработчиков в цепочке прекращается.</p>

<p>Остальные обработчики работают по такому же принципу. Так что нет особого смысла их детально рассматривать.</p>

<p>На этом рассмотрение аннотаций заканчивается. В комментариях отвечу на любые вопросы.</p>