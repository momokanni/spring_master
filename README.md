# spring_master
spring源码剖析，大部分都是基于Springframework5.0  

### 容器的基础XMLBeanFactory  

  XmlBeanFactory继承自DefaultListableBeanFactory，而DefaultListableBeanFactory是整个bean加载的核心部分，是spring注册及加载bean的默认实现，而XmlBeanFactory与DefaultListableBeanFactory不同的地方其实是使用了自定义的XML读取器XmlBeanDefinitionReader,实现了个性化的BeanDefinitionReader读取 
  
  XmlBeanFactory对DefaultListableBeanFactory进行了扩展，主要用于从XML文件中读取BeanDefinition，注册及获取Bean都是使用从父类继承的方法去实现。唯独不同的个性化实现就是增加了XmlBeanDefinitionReader类型的reader属性。在XmlBeanFactory中主要使用reader属性对资源文件进行读取和注册。  
  
![XmlBeanFactory结构图](https://github.com/momokanni/spring_master/blob/master/UML_img/XmlBeanFactory.png)  

  
**DefaultListableBeanFactory层次结构图**
![DefaultListableBeanFactory层次结构图](https://github.com/momokanni/spring_master/blob/master/UML_img/DefaultListableBeanFactory.png)  

**XmlBeanDefinitionReader**  
![XmlBeanDefinitionReader层次结构图](https://github.com/momokanni/spring_master/blob/master/UML_img/XmlBeanDefinitionReader.png)  

> 1. 通过继承AbstractBeanDefinitionReader中的方法，来使用ResourceLoader将资源文件路径转换为对应的Resource文件  

`BeanFactory bf = new XMLBeanFactory(new ClassPathResource("beanFactoryTest.xml"))`  

