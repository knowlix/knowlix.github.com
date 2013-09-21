---
layout: post
title: "Netty: Запуск socket сервера"
date: 2012-08-04 12:39
comments: true
categories: Netty
---
Как было отмечено ранее, стабильная ветка [Netty](http://netty.io/) обладает отличным [JavaDoc](http://static.netty.io/3.5/api/), а более новая четвертая ветка практически не задокументирована. Хочется еще отметить, что в *JavaDoc'е* *Netty* используется специальный *JavaDoc* доклет [ApiViz](http://code.google.com/p/apiviz/), который позволяет отображать связи компонент прямо в документации ввиде графиков. В связи с этим обзор архитектуры будет вестись по 3ей версии. Именно по ней будет изучена терминология проекта, а уже с какими-то знаниями о компонентах системы мы посмотрим на отличия в реализации той или иной фичи в новой четвертой версии *Netty*.

<!--more-->

В качестве отправной точки, будем использовать учебный проект Echo сервера, идущий вместе с исходниками *Netty*.

{% codeblock %}
ServerBootstrap bootstrap = new ServerBootstrap(
	new NioServerSocketChannelFactory(
			Executors.newCachedThreadPool(),
			Executors.newCachedThreadPool()));

bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
	public ChannelPipeline getPipeline() throws Exception {
		return Channels.pipeline(new EchoServerHandler());
	}
});

bootstrap.bind(new InetSocketAddress(port));
{% endcodeblock %}

Запуск сервера начинается с создания объекта helper класса `ServerBootstrap`. Этот объект позволяет сконфигурировать сервер, наполнить его ключевыми компонентами  и наконец, запустить. В первую очередь необходимо указать фабрику каналов. Каналы представляют собой сетевое соединение и *Netty* поддерживает несколько их реализаций. Основные из них это соединения использующие:

1. Old I/O Java package
2. New I/O Java package

Если первый тип реализации - это скорее пережиток прошлого и нужен лишь для поддержки старых программ, то второй тип - это то, что нам нужно. `ChannelFactory` можно не задавать в конструкторе `ServerBootstrap`, а использовать `ServerBootstrap#setChannelFactory` метод. Setter-метод можно использовать только один раз. Это лишь логическое ограничение, которое проверяется естественно только в runtime'e.

`NioServerSocketChannelFactory` оперирует двумя пулами потоков для создание асинхронных соединений. Пулы можно задать в конструкторе `ChannelFactory`. В противном случае создаются стандартные кеширующие java пулы:

{% codeblock %}
Executors.newCachedThreadPool()
{% endcodeblock %}

Зачем фабрике каналов два пула потоков? Первый из них - boss pool. Создает новые потоки для входящих соединений. Второй же worker pool, обеспечивает потоки для неблокирующих операций записи/чтения. 

При создании соединения для канала создается свой pipeline - цепочка обработчиков асинхронных событий каналов. Поэтому `ServerBootstrap` должен обладать фабрикой pileline'ов. Создавая фабрику pipeline'ов определяются обработчики. В данном случае это один обработчик `EchoServerHandler`. Удивительно, что без фабрик каналов и pipeline'ов сервер работать не будет, но в архитектуре возможность создания сервера без фабрик есть, т.к. задаются они setter методами.

`ServerBootstrap#bind` запускает сервер, а точнее вешает сервер на определенный адрес. Рассмотрим на данном примере, как запускается *NIO Socet Server*.

{% codeblock %}
public Channel bind(final SocketAddress localAddress) {
	if (localAddress == null) {
		throw new NullPointerException("localAddress");
	}

	final BlockingQueue<ChannelFuture> futureQueue = new LinkedBlockingQueue<ChannelFuture>();

	ChannelHandler binder = new Binder(localAddress, futureQueue);
	ChannelHandler parentHandler = getParentHandler();

	ChannelPipeline bossPipeline = pipeline();
	bossPipeline.addLast("binder", binder);
	if (parentHandler != null) {
		bossPipeline.addLast("userHandler", parentHandler);
	}

	Channel channel = getFactory().newChannel(bossPipeline);

	// Wait until the future is available.
	ChannelFuture future = null;
	boolean interrupted = false;
	do {
		try {
			future = futureQueue.poll(Integer.MAX_VALUE, TimeUnit.SECONDS);
		} catch (InterruptedException e) {
			interrupted = true;
		}
	} while (future == null);

	if (interrupted) {
		Thread.currentThread().interrupt();
	}

	future.awaitUninterruptibly();
	if (!future.isSuccess()) {
		future.getChannel().close().awaitUninterruptibly();
		throw new ChannelException("Failed to bind to: " + localAddress, future.getCause());
	}

	return channel;
}
{% endcodeblock %}

Запуск сервера начинается с открытия нового канала, который создается с помощью фабрики каналов, которую мы засетили. В нашем случае будет создан `NioServerSocketChannel`. В конструкторе этого канала будет создан Java `SocetChannel`:

{% codeblock %}
socket = ServerSocketChannel.open()
{% endcodeblock %}

И если socket открыт удачно, то канал сигнализирует событие об открытии нового канала.

{% codeblock %}
public static void fireChannelOpen(Channel channel) {
	if (channel.getParent() != null) {
		fireChildChannelStateChanged(channel.getParent(), channel);
	}

	channel.getPipeline().sendUpstream(
			new UpstreamChannelStateEvent(
					channel, ChannelState.OPEN, Boolean.TRUE));
}
{% endcodeblock %}

Стоит рассказать, что в событийной модели *Netty* присутствуют два вида сообщений:

1. *Upstream* - входящие
2. *Downstream* - исходящие

Все сообщения циркулируют по pipeline. Можно рассматривать pipeline, как цепочку обработчиков событий. Любой обработчик может остановить на себе обработку сообщения. Посыл сообщений происходит соответственно через pipeline.

Канал может быть создан другим каналом. Тот будет для нового канала родительским. Для каналов клиентской стороны родительским каналом будет канал серверной стороны. Входящее сообщение об открытии канала посылается и родительскому каналу, если он есть (метод `fireChildChannelStateChanged`), и обработчикам данного канала.

Boss-канал, открывающий соединение, на данном этапе кроме открытия сокета и порождения события об этом, ничего не делает и не может делать, так как не обладает нужной для этого информацией. Как же происходит слежение за сообщениями данного канала? Очевидно, что у данного канала должны быть свои обработчики, которые будут ожидать события открытия соединения и в этот момент выполнять всю необходимую для нас работу.

При создании в `ServerBootstrap` нового сокет канала инициализирует boss pipeline, куда добавляется обязательный обработчик `Binder` и пользовательский обработчик, который можно задать через метод `ServerBootstrap#setParentHandler`. Класс обязательного обработчика является внтуренним для класса `ServerBootstrap`, т.к. является неотъемлемой его частью. В момент получения события об открытии нового socket-канала, `Binder` получает ссылку на Netty канал из объекта события и задает ему пользовательский `PipelineFactory`, пользовательские настройки и привязывает открытое соединение к конкретному адресу. По аналогии с Java сокетами, Netty сокеты также обладают методом bind, который привязывает его к конкретному адресу.

{% codeblock %}
public static ChannelFuture bind(Channel channel, SocketAddress localAddress) {
	if (localAddress == null) {
		throw new NullPointerException("localAddress");
	}
	ChannelFuture future = future(channel);
	channel.getPipeline().sendDownstream(new DownstreamChannelStateEvent(
			channel, future, ChannelState.BOUND, localAddress));
	return future;
}
{% endcodeblock %}

Асинхронная модель работы *Netty* основана не тольно на событиях, но также и на Future объектах. Если события обеспечивают работу через callback'и и в данном случае поток, вызвавший операцию, никак не связан с асинхронным процессом, то Future объект предоставляет loop алгоритм, связывая асинхронный вызванный процесс с его родителем. Запуск сервера - как раз такой случай, т.к. этот метод должен вернуть открытый и сконфигурированный канал. Но если до данного момента работа шла в синхронном режиме, то привязка адреса к открытому соединению полноценно открывает сервер для клиентских соединений, запуская соответствующие потоки. Поэтому данный метод возвращает `ChannelFuture` объект с сылкой на текущий канал и вызывает исходящее событие привязки сервера к адресу.

В методах pipeline'а отправки сообщения происходит некоторое проксирование сообщений, т.к. разные реализации каналов могут работать по разному с разными сообщениями. Очевидно, что проксирование сообщений специфично для разных видов реализаций каналов, поэтому фабрика каналов, помимо типа самих каналов определяет конкретную реализацию проксирующего класса. Проксирующий класс определяет какие сообщения поддерживает канал и возможно даже обрабатывает какие-то из них, которые служат только для внутреннего использования. Такие объекты в *Netty* именуются *Sink* объектами. В данном случае обработка события привязки сервера к адресу обрабатывается внутри `NioServerSocketPipelineSink`:

{% codeblock %}
private void bind(
		NioServerSocketChannel channel, ChannelFuture future,
		SocketAddress localAddress) {

	boolean bound = false;
	boolean bossStarted = false;
	try {
		channel.socket.socket().bind(localAddress, channel.getConfig().getBacklog());
		bound = true;

		future.setSuccess();
		fireChannelBound(channel, channel.getLocalAddress());

		Executor bossExecutor =
			((NioServerSocketChannelFactory) channel.getFactory()).bossExecutor;
		DeadLockProofWorker.start(bossExecutor,
				new ThreadRenamingRunnable(new Boss(channel),
						"New I/O server boss #" + id + " (" + channel + ')'));
		bossStarted = true;
	} catch (Throwable t) {
		future.setFailure(t);
		fireExceptionCaught(channel, t);
	} finally {
		if (!bossStarted && bound) {
			close(channel, future);
		}
	}
}
{% endcodeblock %}

В первую очередь вызывается привязка адреса к *Java* сокету. Далее `ChannelFuture` переводится в состояние успешного выполнения операции и посылается уже входящее сообщение pipeline'у канала о привязке сервера к адресу. Далее получается пул потоков, который мы передали фабрике каналов и с помощью него запускается boss-поток, который будет следить за соединениями по данному каналу. Теперь наш сервер может принимать соединения и обрабатывать их нашим pipeline'ом.

Т.е. на данном этапе *Binder* обработчик связал сокет с адресом и осталось оповестить `ServerBootstrap` о статусе конфигурирования канала и вернуть наружу ссылку на сам канал. Но тут есть две проблемы:

1. Событийная модель *Netty* работает в асинхронном режиме, что не дает вернуть напрямую результат работы обработчика
2. Так как режим асинхронный, то `ServerBootstrap` работает в своем пространстве и, если не предпринято каких-то шаманских действий, то вполне возможно уже отработал.

А шаманские действия, как вы уже возможно успели заметить, предприняты. В `ServerBootstrap` до инициализации *Binder* обработчика создается `BlockingQueue`, которая передается в *Binder*. Далее, проинициализировав boss обработчик и создав канал, `ServerBootstrap` пытается прочитать из очереди `ChannelFuture`, но так как очередь блокирующая и в ней нет на данный момент сообщений, то поток `ServerBootstrap` подвешивается пока очередь не будет готова вернуть сообщение.

{% codeblock %}
future = futureQueue.poll(Integer.MAX_VALUE, TimeUnit.SECONDS);
{% endcodeblock %}

Как только *Binder* сконфигурировал открытый канал, он помещает `ChannelFuture` объект, который мы ранее рассмотрели в методе `bind` самого канала, в эту очередь:

{% codeblock %}
boolean finished = futureQueue.offer(evt.getChannel().bind(localAddress));
{% endcodeblock %}

В этот момент поток выполнения `ServerBootstrap` оживает и получает это сообщение из очереди и если статус `ChannelFuture` говорит, что все прошло хорошо, то наружу возвращает ссылка на открытый канал.

Вот так просто и изящно создаются socket соединения в *Netty 3.5*.
