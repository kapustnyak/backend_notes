EntityManager — это API, с помощью которого Вы управляете жизненным циклом сущностей и выполняете запросы. Каждый EntityManager связан с **контекстом постоянства** (persistence context) — «рабочим пространством» управляемых объектов. Внутри одного контекста для каждой @Id существует **ровно один** объект — это и есть **first-level cache (L1)**. Он гарантирует **тождественность** объектов, **dirty checking** и отложенную запись при flush/commit.


Ключевые методы, влияющие на L1:
- find / getReference (ленивая ссылка) — загрузка/привязка к контексту, повторный find той же сущности вернёт **тот же** объект из L1. 
- persist, merge, remove — управление состояниями. 
- flush — синхронизация изменений с БД; clear / detach — отцепление объектов, очистка L1. 

В Spring (типично) один EntityManager привязан к транзакции — значит, **область жизни L1 = границы транзакции**. Это следует из модели JPA: «transaction-scoped persistence context».

Одно чтение из БД, второе — из L1:
```java
@Transactional
public void demo(EntityManager em) {
    Order o1 = em.find(Order.class, 42L); // SQL SELECT
    Order o2 = em.find(Order.class, 42L); // без SQL — тот же объект из L1
    assert o1 == o2;                      // true: тождественность внутри контекста
    o1.setStatus("PAID");                 // dirty checking
    // при commit/flush — UPDATE
}
```

Cбрасываем L1, следующая загрузка снова пойдёт в БД:
```java
@Transactional
public void clearDemo(EntityManager em) {
    Order o1 = em.find(Order.class, 42L);
    em.clear();                           // все управляемые -> DETACHED
    Order o2 = em.find(Order.class, 42L); // снова SQL SELECT
}
```


#### Что такое second-level cache (shared cache)

Спецификация JPA определяет **второй уровень (L2)** как _общий кэш_ на уровень EntityManagerFactory/пула контекстов. Он **не обязателен**: реализация JPA может его не использовать. Работать с ним позволяет интерфейс javax.persistence.Cache: contains, evict(...), evictAll(). Если L2 не включён, методы, кроме contains, «без эффекта». 

Поведение провайдера относительно L2 настраивается через shared-cache-mode в persistence.xml (значения ALL, NONE, ENABLE_SELECTIVE, DISABLE_SELECTIVE, UNSPECIFIED). Это именно «как провайдер ДОЛЖЕН использовать общий кэш».

Пример безопасного обращения к L2 по JPA-API:
```java
public void evictCustomer(EntityManagerFactory emf, Long id) {
    Cache cache = emf.getCache();
    if (cache != null && cache.contains(Customer.class, id)) {
        cache.evict(Customer.class, id); // убрать конкретную сущность из L2
    }
}
```

