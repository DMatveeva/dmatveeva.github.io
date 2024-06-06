---
layout: post
title: Readiness probe и EventListener
categories: Рабочее
---

Возникла проблема при старте сервиса - долго не отвечают heathchecks сервиса в openshift.

Для данного сервиса healthchecks в Openshift проверяли liveness и readiness эндпойнты актуатора в Spring boot, и проблема оказалась в readiness probe.
Выяснилось, что при старте сервиса запускается продолжительный по времени метод  формирования отчета, который сохраняется в БД в ожидании, когда его считает другой сервис.

Разобралась с тем, что происходит при старте сервиса.

Метод формирования отчета запускался с помощью аннотации:
```
@EventListener(AppricationReadyEvent.class)
public void generate() {
//логика создания отчета
}
```
В документации к классу `AppricationReadyEvent` сказано:

_Event published as late as conceivably possible to indicate that the application is ready to service requests._

В документации к классу `ReadinessState`:

_"Readiness" state of the application.
An application is considered ready when it's live and willing to accept traffic. "Readiness" failure means that the application is not able to accept traffic and that the infrastructure should stop routing requests to it._

Выглядит логично, что `AppricationReadyEvent` должен быть опубликован после того, как readiness probe станет UP.
Посмотрим, когда выставляется статус UP.

В методе `SpingApplication.run(...)`, после создания, инициализации и подготовки контекста идет вызов `listeners.ready(...)`
```
public ConfigurableApplicationContext run(String... args) {
    ...
    try {
        ...
        context = createApplicationContext();
        ...
    }
    try {
        if (context.isRunning()) {
            listeners.ready(context, startup.ready());
        }
    }
    ...
}
```
Здесь вызывается `EventPublishingRunListener#ready`, который и выставляет статус readiness probe в UP:

`org.springframework.boot.context.event.EventPublishingRunListener#ready`
```
@Override
public void ready(ConfigurableApplicationContext context, Duration timeTaken) {
    context.publishEvent(new ApplicationReadyEvent(this.application, this.args, context, timeTaken));
    AvailabilityChangeEvent.publish(context, ReadinessState.ACCEPTING_TRAFFIC);
}
```
А `context.publishEvent` в свою очередь вызывает
`org.springframework.context.event.SimpleApplicationEventMulticaster#multicastEvent`
```
@Override
public void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType) {
    ...
    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        if (executor != null && listener.supportsAsyncExecution()) {
            try {
                executor.execute(() -> invokeListener(listener, event));
            }
            ...
     }
```
`invokeListeners(...)` - как раз приводит к вызову нашего метода `generate`, помеченного аннотацией 
`@EventListener(AppricationReadyEvent.class)`.
Поэтому, пока он не завершится, readiness probe не будет переведена в статус UP.
Переопределить метод `EventPublishingRunListener#ready` нет возможности, поэтому при старте запускаем 
формирование отчета в новом потоке. Т.к. продолжительный метод генерации выполняется в новом потоке, в основном потоке  метод `generateAtStartup` быстро завершается, не блокируясь, и readiness probe переводится в UP.
```
@Autowired
ScheduledExecutorService executorService;

@EventListener(ApplicationReadyEvent.class)
public void generateAtStartup() {
    executorService.schedule(this::generateReport, 20, TimeUnit.SECONDS);
}

private void generate() { //логика создания отчета }
```