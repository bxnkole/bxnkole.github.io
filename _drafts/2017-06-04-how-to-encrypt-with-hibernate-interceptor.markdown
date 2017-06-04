---
title:  "Encrypting database contents with Hibernate Interceptor"
description: How to encrypt database contents with Hibernate Interceptor
category: java
---

There comes a time in the process of engineering a piece of software where you just want more. More, in the context of this
post, means adding an extra functionality for every interaction with the database, just like a hook. The most common use case for database
interception is for auditing purposes. However, I have personally encountered cases where the requirement demands for
encryption of actual database contents.

To demonstrate this functionality, I'll start with a **Person** model. To keep it simple, every person object is going to contain four
properties; **age, name, email, activeStatus** of types **Integer, String, String, Boolean** respectively.

*I'm declaring the member variables as public, because I can :) (Don't try this at home)*

{% highlight java %}
public class Person {
    public int age;
    public String name;
    public String email;
    public Boolean activeStatus;
}
{% endhighlight %}

Luckily for us, Hibernate already provides an EmptyInterceptor class which, as the name implies, does practically nothing. We are going to extend the
functionality of this class by creating an EncryptionInterceptor class (because its encryption we're interested in. Call it whatever you want, really).
This is what the class will look like:

{% highlight java %}

{% endhighlight %}

Next, we add an interceptor to the session factory configuration phase:

{% highlight java %}
Configuration cfg = new Configuration();
cfg.setProperty("hibernate.connection.url", connectionUrl);
cfg.setProperty("hibernate.connection.username", username);
cfg.setProperty("hibernate.connection.password", password);

//WE ADD THE INTERCEPTOR HERE
cfg.setInterceptor(new EncryptionInterceptor());

StandardServiceRegistryBuilder ssrb = new StandardServiceRegistryBuilder();
ssrb.applySettings(cfg.configure().getProperties());
SessionFactory sessionFactory = cfg.buildSessionFactory(ssrb.build());
{% endhighlight %}