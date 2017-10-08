---
title:  "Encrypting database contents with Hibernate Interceptor"
description: How to encrypt database contents with Hibernate Interceptor
category: java, hibernate
---

There comes a time in the process of engineering a piece of software where you want more control of database interactions, just like a hook. 
The most common use case for database interception is for auditing purposes. However, I have personally encountered cases where the requirement demands for
encryption of actual database contents. This comes in handy for desktop application development (Been doing a lot of that lately).

To demonstrate this functionality, I'll start with a **Person** model. To keep it simple, every person object is going to contain four
properties; **age, name, email, activeStatus** of types **Integer, String, String, Boolean** respectively.

*All sources are available on my [Github](https://github.com/bxnkole/hibernate-encryption-interceptor) page*

{% highlight java %}
@Entity
@Getter
@Setter
@NoArgsConstructor
@ToString
public class Person {
    @Id 
    @GeneratedValue 
    private Long id;
    
    @Column private int age;
    @Column private String name;
    @Column private String email;
    @Column private Boolean activeStatus;
}
{% endhighlight %}

Luckily for us, Hibernate already provides an EmptyInterceptor class which, as the name implies, does practically nothing. We are going to extend the
functionality of this class by overriding three methods from parent class. They are: 

1.**onSave**

   This is invoked whenever a save or update operation is performed using Hibernate's session. 
   For this example, I am only interested in the String fields, hence the instanceof check. 
   As seen below, I am encrypting the plain text just before it is persisted to the database. 
   The boolean value that is returned from this method is just a pointer for Hibernate to know whether the entity was modified or not.
{% highlight java %}
public boolean onSave(Object entity, Serializable id, Object[] state, String[] propertyNames, Type[] types) {
   try {
       for (int i = 0; i < state.length; i++) {
           if (state[i] instanceof String) {
               String plainText = (String) state[i];
               if (plainText != null && !plainText.trim().isEmpty()) {
                   state[i] = SimpleAES.encrypt(plainText);
               }
           }
       }
       return true;
   } catch (Exception e) {
       //log error
   }
   return false;
}
{% endhighlight %}
   
2.**onFlushDirty**

   This is invoked when a persistent object (entity) is detected to be dirty i.e. has been modified. 
   It exposes the current state and previous states of the object as method parameters.
{% highlight java %}
@Override
public boolean onFlushDirty(Object entity, Serializable id, Object[] currentState, Object[] previousState, String[] propertyNames, Type[] types) {
  try {
      for (int i = 0; i < currentState.length; i++) {
          if (currentState[i] instanceof String) {
              String plainText = (String) currentState[i];
              if (plainText != null && !plainText.trim().isEmpty()) {
                  currentState[i] = SimpleAES.encrypt(plainText);
              }
          }
      }
      return true;
  } catch (Exception e) {
      //log error
  }
  return false;
}
{% endhighlight %}
   
3.**postFlush**

   This method is invoked after any transaction is committed. After a typical transaction, the database entities are propagated back to the objects used to save them.
   The implication here is that when we save, our objects from code will have encrypted properties. We don't want that. This method will be used to decrypt them. 
   Invoked after read/load operations. 
   
{% highlight java %}
@Override
public void postFlush(Iterator entities) {
   entities.forEachRemaining(entity -> {
       Field[] declaredFields = entity.getClass().getDeclaredFields();
       for (Field declaredField : declaredFields) {
           Class<?> type = declaredField.getType();
           if (!type.equals(String.class)) {
               continue;
           }

           boolean isColumn = declaredField.isAnnotationPresent(Column.class);
           if (!isColumn) {
               continue;
           }

           try {
               PropertyDescriptor pd = new PropertyDescriptor(declaredField.getName(), entity.getClass());
               String value = (String) pd.getReadMethod().invoke(entity);
               if (value != null && !value.trim().isEmpty()) {
                   pd.getWriteMethod().invoke(entity, SimpleAES.decrypt(value));
               }
           } catch (Exception e) {
               //log error here
           }
       }
   });
}
{% endhighlight %}


Next, we add this interceptor to the session factory configuration phase:

{% highlight java %}
Configuration cfg = new Configuration();

//WE ADD THE INTERCEPTOR HERE
cfg.setInterceptor(new EncryptionInterceptor());

StandardServiceRegistryBuilder ssrb = new StandardServiceRegistryBuilder();
ssrb.applySettings(cfg.configure().getProperties());
SessionFactory sessionFactory = cfg.buildSessionFactory(ssrb.build());
{% endhighlight %}

To demonstrate this, I am going to create a **Person** object and write it to the database.

{% highlight java %}
Person person = new Person();
person.setName("Bankole");
person.setEmail("janedoe@gmail.com");
person.setAge(10);
person.setActiveStatus(true);

new HibernateUtil().create(person);
System.out.println(person); //we can do this, thanks to toString()
{% endhighlight %}

A new row is added to the database:

![New Person](/assets/images/hib-enc-post/1.png)

But clear text is printed out to the console:

{% highlight java %}
Person(id=1, age=10, name=Bankole, email=janedoe@gmail.com, activeStatus=false)
{% endhighlight %}

Pretty cool yeah? Let me know what you think in the comments below.

You can find the complete source code on my [Github Page](https://github.com/bxnkole/hibernate-encryption-interceptor)

**Useful Links**

[https://docs.jboss.org/hibernate/core/3.6/javadocs/org/hibernate/Interceptor.html](https://docs.jboss.org/hibernate/core/3.6/javadocs/org/hibernate/Interceptor.html)