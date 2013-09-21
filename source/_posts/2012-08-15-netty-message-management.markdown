---
layout: post
title: "Netty: Управление сообщениями"
date: 2012-08-15 22:37
comments: true
categories: Netty
---
После открытия сокета вызывается так называемая регистрация открытого *Netty* канала, чтобы сообщения этого соединения могли быть обработаны в рамках *Netty*. Сегодня мы посмотрим, как происходит управление сообщениями.

<!--more-->

Все начинается с этого метода, который вызывается после принятия открытого соединения:

{% codeblock %}
private void registerAcceptedChannel(SocketChannel acceptedSocket, Thread currentThread) {
	try {
		ChannelPipeline pipeline =
			channel.getConfig().getPipelineFactory().getPipeline();
		NioWorker worker = nextWorker();
		worker.register(new NioAcceptedSocketChannel(
				channel.getFactory(), pipeline, channel,
				NioServerSocketPipelineSink.this, acceptedSocket,
				worker, currentThread), null);
	} catch (Exception e) {
		try {
			acceptedSocket.close();
		} catch (IOException e2) {
		}
	}
}
{% endcodeblock %}

Обработка сообщений в *Netty* производится цепочкой обработчиков, которая в терминологии *Netty* называется `pipeline`. Поэтому в первую очередь создается новый pipeline с помощью фабрики, которую мы проинициализировали в момент создания сервера. Для обработки сообщений в конкретном соединении канала создается отдельный Worker поток. За обработку операции ввода/вывода отвечают `Worker'ы`. В частности nio операции управляются `NioWorker'ом`. `Worker'ы` создаются внутри пула воркеров. Передавая в `NioServerSocketChannelFactory` пул воркер-потоков, этот пул оборачивается в `NioWorkerPool`, в конструкторе которого создаются экземпляры `NioWorker'ов`. Количество воркеров ограничено количеством процессоров, доступных JVM из расчета - на каждый процессор два воркера.

{% codeblock %}
static final int DEFAULT_IO_THREADS = Runtime.getRuntime().availableProcessors() * 2;
{% endcodeblock %}

Объекты воркеров хранятся в обычном массиве внутри пула. При регистрации канала, из пула воркеров последовательно достается воркер с помощью метода `nextWorker`:

{% codeblock %}
public E nextWorker() {
	return (E) workers[Math.abs(workerIndex.getAndIncrement() % workers.length)];
}
{% endcodeblock %}

Индекс текущего воркера хранится в `AtomicInteger` переменной. Целочисленное деление индекса на количество воркеров в пуле гарантирует получение воркера в пределах массива воркеров.

Далее создается новый канал `NioAcceptedSocketChannel` и регистрируется внутри воркера. За принятие соединений отвечает один канал, за получение и отправку сообщений отвечает только что созданный канал. Регистрация канала начинается с создания объекта задачи регистрации `RegisterTask`. Этот объект специфичен для конкретного воркера и объявляется внутри него. Далее запускается поток воркера:

{% codeblock %}
public void run() {
	thread = Thread.currentThread();

	boolean shutdown = false;
	Selector selector = this.selector;
	for (;;) {
		try {
			SelectorUtil.select(selector);

			cancelledKeys = 0;
			processRegisterTaskQueue();
			processEventQueue();
			processWriteTaskQueue();
			processSelectedKeys(selector.selectedKeys());

			if (selector.keys().isEmpty()) {
				if (shutdown ||
					executor instanceof ExecutorService && ((ExecutorService) executor).isShutdown()) {

					synchronized (startStopLock) {
						if (registerTaskQueue.isEmpty() && selector.keys().isEmpty()) {
							started = false;
							try {
								selector.close();
							} catch (IOException e) {
							} finally {
								this.selector = null;
							}
							break;
						} else {
							shutdown = false;
						}
					}
				} else {
					if (allowShutdownOnIdle) {
						shutdown = true;
					}
				}
			} else {
				shutdown = false;
			}
		} catch (Throwable t) {
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				// Ignore.
			}
		}
	}
}
{% endcodeblock %}

Стоит отметить, что запуск потока вызывается в блоке синхронизации с объектом блокировки, который принадлежит конкретному объекту worker'а. Эта блокировка гарантирует последовательную обработку сообщений с помощью одного и того же воркера, взятого из пула.

В основе работы потока все тот же бесконечный цикл и `Selector` для обеспечения эффективных пауз и обработки сообщений. Внутри `process*` методов выполняется обработка задач, которые висят в очереди, причем алгоритм работы этих методов идентичен. Различаются только очереди, из которых достаются задачи:

{% codeblock %}
for (;;) {
	final Runnable task = someQueue.poll();
	if (task == null) {
		break;
	}
	task.run();
	cleanUpCancelledKeys();
}
{% endcodeblock %}

> Зачем разведен Copy/Paste в методах обработки очередей, остается загадкой.

Задачи реализуют интерфейс `Runnable`. Метод регистрации регистрирует канал в `Selector'е` на события чтения/записи наподобие того, как регистрировался канал
 принятия соединений на события принятия соединений.
 
Интересен метод обработки ключей, обнаруженных селектором (принцип работы селектора описывался в предыдущей статье):

{% codeblock %}
private void processSelectedKeys(Set<SelectionKey> selectedKeys) throws IOException {
	for (Iterator<SelectionKey> i = selectedKeys.iterator(); i.hasNext();) {
		SelectionKey k = i.next();
		i.remove();
		try {
			int readyOps = k.readyOps();
			if ((readyOps & SelectionKey.OP_READ) != 0 || readyOps == 0) {
				if (!read(k)) {
					continue;
				}
			}
			if ((readyOps & SelectionKey.OP_WRITE) != 0) {
				writeFromSelectorLoop(k);
			}
		} catch (CancelledKeyException e) {
			close(k);
		}

	}
}
{% endcodeblock %}

Сразу бросается в глаза интересный способ обхода итератора по ключам селектора. В ключах Nio используется работа с константами на основе битов. Так метод `k.readyOps()` возвращает бит готовой операции. Сравнение констант производится с помощью побитовых операций. Сообщений в канале по большому счету может быть два - чтение и запись. О них мы поговорим в следующих статьях. 

После обработки всех задач выполняется проверка на наличие ключей в селекторе на всякий случай, и если ключи все обработались, то происходит закрытие селектора и прерывание цикла. В случае исключения, как и в Boss потоке, вызывается секундный `sleep` для предотвращения чрезмерной загрузки процессора в случае сбоев.

