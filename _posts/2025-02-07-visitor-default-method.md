---
layout: post
title: Паттерн Visitor  
categories: Теория
---

В книге Head First "Паттерны Проектирования" не рассказывается про один из известных паттернов - Visitor (Посетитель).

Он используется, когда мы хотим добавить новое поведение классам из иерархии, но не хотим сильно править сами классы.

Допустим у нас в проекте есть иерархия спортсменов различного уровня.
У спортсмена есть методы, где реализована логика занятия спортом.
```
public abstract class Athlete {
    public abstract void doSport();
}

public class Novice extends Athlete {
    public void doSport() {
        //do sport like Novice athlete
    }
}

public class Pro extends Athlete {
    public void doSport() {
        //do sport like Professional athlete
    }
}
```
Поступило требование, чтобы каждый спортсмен сдавал норматив по отжиманиям и подтягиваниям.

Нам может быть желательно не вносить новый код в иерархию. Например, наши классы удачно спроектированы, покрыты тестами и мы не хотим ничего сломать. 
К тому же, при внесении новых методов может нарушаться Single Responsibility Principle. Мы хотим сконцентрировать в классах код, который описывает занятие конкретным видом спорта, а нормативы - это просто побочная рутина. К тому же вполне возможно что нормативы будут меняться, и каждый раз 
придется править классы иерархии.

Решение -- используя паттерн Посетитель, вынести методы, которые описывают сдачу нормативов в отдельный класс FitTestVisitor, и вызывать их в классах иерархии (остальные методы опущены для краткости):
```
class FitTestVisitor {
    public void doNoviceFitTest(Novice athlete) {
        System.out.println("Сделать 10*3 подтягиваний.");  
        System.out.println("Сделать 10*3 отжиманий.");  
    }
    public void doProFitTest(Pro athlete) {
        System.out.println("Сделать 30*3 подтягиваний.");
        System.out.println("Сделать 30*3 отжиманий.");  
    }
}

public abstract class Athlete {
    public abstract void operate(FitTestVisitor visitor);
}

public class Novice extends Athlete {
    public void operate(FitTestVisitor visitor) {
        visitor.doNoviceFitTest(this);
    }
}

public class Pro extends Athlete {
    public void operate(FitTestVisitor visitor) {
        visitor.doProFitTest(this);
    }
}
```
По сути, с помощью Посетителя мы отделяем новое поведение от данных. 

В Java есть механизм, который позволяет сделать то же самое - дефолтные методов интерфейсов. Дефолтные методы - это реализация примесей.

Примесь (mix in) - это элемент языка программирования, реализующий четко выделенное поведение. Позволяет дополнить классы новым поведением, без необходимости наследовать класс от данного элемента.  

Для реализации нашей задачи можем создать интерфейсы PullupSkill, PushupSkill.
И дефолтные методы для спортсмена каждого уровня.
Объявляем, что Athlete реализовывает интерфейсы PullupSkill, PushupSkill. И добавляем абстрактный метод fitTest().
Для каждого потомка в методе fitTest вызываем соответствующие дефолтные методы интерфейсов.
```

public interface PullupSkill {
    default void novicePullup() { System.out.println("Сделать 10*3 подтягиваний."); }
    default void proPullup() { System.out.println("Сделать 5*3 подтягиваний на одной руке."); }
}

public interface PushupSkill {
    default void novicePushup() { System.out.println("Сделать 10*3 отжиманий."); }
    default void proPushup() { System.out.println("Сделать 5*3 отжиманий на одной руке."); }
}

public abstract class Athlete implements PullupSkill, PushupSkill {
    public abstract void fitTest();
}

public class Novice extends Athlete {
    public void fitTest() {
        novicePullup();
        novicePushup();
    }
}

public class Pro extends Athlete {
    public void fitTest() {
        proPullup();
        proPushup();
    }
}
```

По сути, мы достигли того же эффекта - отделили поведение от данных.

Плюсы второй реализации:
- Проще понять человеку, незнакомому с паттерном Visitor.
- Проще и нагляднее добавить поведение новому классу, например, тренеру не из иерархии Athlete.
- Более говорящие имена методов в отличии от operate(Visitor visitor).