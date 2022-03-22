---
title: Google Provider Inject Service
tags: 编程基础
categories: 编程基础
abbrlink: deea827d
date: 2022-02-16 17:49:29
---

### 定义接口

```
import com.google.inject.ProvidedBy;
public interface Service {
    public void execute();
}
```

### 定义实现类
```
public class OneService implements Service {
    @Override
    public void execute() {
        System.out.println("Hello!  I'M Service 1!");

    }
}
```

### Provider实现类
```
import com.google.inject.Provider;

public class OneServiceProvider implements Provider<Service> {
    @Override
    public Service get() {
        return new OneService();
    }
}
```

### 测试类

使用Module关联：

```
import com.google.inject.Binder;
import com.google.inject.Guice;
import com.google.inject.Inject;
import com.google.inject.Module;

public class ProviderServiceDemo {
    @Inject
    private Service service;

    public static void main(String[] args) {
        ProviderServiceDemo instance = Guice.createInjector(new Module() {

            @Override
            public void configure(Binder binder) {
                binder.bind(Service.class).toProvider(OneServiceProvider.class);
            }
        }).getInstance(ProviderServiceDemo.class);
        instance.service.execute();// Hello! I'M Service 1!

    }
}
```

也可以使用@ProviderBy注解：

```
import com.google.inject.ProvidedBy;

@ProvidedBy(OneServiceProvider.class)
public interface Service {
    public void execute();
}


import com.google.inject.Guice;

public class ProviderServiceDemo2 {
    public static void main(String[] args) {
        ProviderServiceDemo2 instance = Guice.createInjector().getInstance(ProviderServiceDemo2.class);
        instance.service.execute(); 
    }
}
```

```
import com.google.inject.Binder;
import com.google.inject.Guice;
import com.google.inject.Inject;
import com.google.inject.Module;
import com.google.inject.Provider;

public class ProviderServiceDemo3 {
    @Inject
    private Provider<Service> provider;

    public static void main(String[] args) {

        ProviderServiceDemo3 instance = Guice.createInjector(new Module() {

            @Override
            public void configure(Binder binder) {
                binder.bind(Service.class).toProvider(OneServiceProvider.class);
            }
        }).getInstance(ProviderServiceDemo3.class);

        instance.provider.get().execute();// Hello! I'M Service 1!
    }
}
```
