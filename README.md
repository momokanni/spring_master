# 容器的基本实现
spring源码剖析，大部分都是基于Springframework5.0  

交代： spring最核心的两个类： `DefaultListableBeanFactory` && `XmlBeanDefinitionReader ` 

### 容器的基础XMLBeanFactory  

  `XmlBeanFactory`继承自`DefaultListableBeanFactory`，而DefaultListableBeanFactory是整个bean加载的核心部分，是spring注册及加载bean的默认实现，而XmlBeanFactory与DefaultListableBeanFactory不同的地方其实是使用了自定义的XML读取器`XmlBeanDefinitionReader`,实现了个性化的`BeanDefinitionReader`读取 
  
  **DefaultListableBeanFactory层次结构图**
![DefaultListableBeanFactory层次结构图](https://github.com/momokanni/spring_master/blob/master/UML_img/DefaultListableBeanFactory.png) 

  XmlBeanFactory对DefaultListableBeanFactory进行了扩展，主要用于从XML文件中读取BeanDefinition，注册及获取Bean都是使用从父类继承的方法去实现。唯独不同的个性化实现就是增加了XmlBeanDefinitionReader类型的reader属性。在XmlBeanFactory中主要使用reader属性对资源文件进行读取和注册。  
  
**XmlBeanDefinitionReader**  
XML读取是spring重要功能，也就可以从XmlBeanDefinitionReader中梳理：资源文件读取、解析、注册的大致脉络。  

![XmlBeanDefinitionReader层次结构图](https://github.com/momokanni/spring_master/blob/master/UML_img/XmlBeanDefinitionReader.png)  

> 1. 通过继承AbstractBeanDefinitionReader中的方法，来使用ResourceLoader将资源文件路径转换为对应的Resource文件  
> 2. 通过DocumentLoader对Resource文件进行转换，将Resource文件转换为Document文件。  
> 3. 通过实现接口：BeanDefinitionDocumentReader的DefaultBeanDefinitionDocumentReaDer类对Document进行解析，并使用				      BeanDefinitionParserDelegate对Element进行解析。 

`BeanFactory bf = new XMLBeanFactory(new ClassPathResource("beanFactoryTest.xml"))`  

![XmlBeanFactory初始化时序图](https://github.com/momokanni/spring_master/blob/master/UML_img/XmlBeanFactory_%E5%88%9D%E5%A7%8B%E5%8C%96%E6%97%B6%E5%BA%8F%E5%9B%BE.jpg)  
**注：**  
> 1. 调用ClassPathResource的构造函数来构造Resource资源文件的实例对象。  
> 2. 配置文件封装：通过Resource相关类完成对配置文件进行封装后配置文件的读取工作就转交给XmlBeanDefinitionReader来处理了  
```  
      // 封装任何能返回InputStream的类，返回一个新的InputStream对象  
      public interface InputStreamSource {  
          InputStream getInputStream() throws IOException;  
      }  
      
      /**  
       * 有了Resource接口便可以对所有资源文件进行统一处理  
       * 例： Resource resource = new ClassPathResource("beanFactoryTest.xml");  
       *     InputStream ins = resource.getInputStream();  
       * 得到InputStream后，我们就可以照旧进行开发，并且可以利用Resource及其子类提供的诸多特性  
       */
      public interface Resource extends InputStreamSource {  
          boolean exists();  //是否存在  
          default boolean isReadable() {return exists();} // 是否可读  
          default boolean isOpen() {return false;}  // 是否打开  
          default boolean isFile() {return false;} // 是否是一个文件  
          URL getURL() throws IOException;  // Return a URL handle for this resource.  
          URI getURI() throws IOException; // Return a URI handle for this resource.  
          File getFile() throws IOException; // Return a File handle for this resource.  
          long contentLength() throws IOException; // Determine the content length for this resource.  
          long lastModified() throws IOException;  // Determine the last-modified timestamp for this resource.  
	  
	  // 基于当前资源创建一个相对资源  
	  // Create a resource relative to this resource.  
          Resource createRelative(String relativePath) throws IOException;   
          
          @Nullable
	  String getFilename(); //Determine a filename for this resource  
          
	  // 用于在错误处理中打印出错资源文件的详细信息
          String getDescription(); // return a description for this resource  
          
          // Java8 接口都提供默认方法 return 基础资源的字节通道
          default ReadableByteChannel readableChannel() throws IOException {
            return Channels.newChannel(getInputStream());
          }
      }  
 ```  
> 3. 查看XmlBeanFactory.java，分析以Resource实例作为构造函数参数的办法  
```  
	@Deprecated
	@SuppressWarnings({"serial", "all"})
	public class XmlBeanFactory extends DefaultListableBeanFactory {

		private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);

		public XmlBeanFactory(Resource resource) throws BeansException {  
			this(resource, null); // 调用XmlBeanFactory(Resource,BeanFactory)构造方法
		}
		
		public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
			super(parentBeanFactory); // 跟踪代码到父类AbstractAutowireCapableBeanFactory构造函数，下边进行分析
			this.reader.loadBeanDefinitions(resource); // 资源加载的真正实现，重点之一
		}

	}  
	
	
```  

> 3.1  由 `super(parentBeanFactory);` 跟踪到父类AbstractAutowireCapableBeanFactory的构造方法
```  
	/**
	 * ignoreDependencyInterface方法主要作用是忽略给定接口的自动装配功能。
	 * spring官方介绍： 自动装配时忽略给定的依赖接口，典型应用是通过其他方式解析Application上下文注册依赖，类似于 BeanFactory 通过
	 * 		   BeanFactoryAware 进行注入或者 ApplicationContext通过ApplicationContextAware进行注入。
	 * 例：A中由属性B，当spring获取A的bean时，其属性B还未初始化，spring默认会初始化B，这是spring重要特性。
	 *     但某些情况下，B不会被初始化，一种情况就是B implements BeanNameAware接口。
	 */
	public AbstractAutowireCapableBeanFactory() {
		super();
		ignoreDependencyInterface(BeanNameAware.class);
		ignoreDependencyInterface(BeanFactoryAware.class);
		ignoreDependencyInterface(BeanClassLoaderAware.class);
	}  
```  

> 3.2 加载bean: `this.reader.loadBeanDefinitions(resource); `  

![loadBeanDefinitions函数执行时序图](https://github.com/momokanni/spring_master/blob/master/UML_img/loadBeanDefinitions%E5%87%BD%E6%95%B0%E6%89%A7%E8%A1%8C%E6%97%B6%E5%BA%8F%E5%9B%BE.jpg)  

> 3.2.1 封装资源文件  
> 3.2.2 获取输入流,从Resource中获取对应的InputStream并构造InputSource  
> 3.2.3 继续调用doLoadBeanDefinitions(inputSource, encodedResource.getResource());  

**根据上图的顺序看源码即可。**  

#### 源码中涉及的注意点  
> 1. EncodeResource ： 对资源文件的编码进行处理，主要逻辑体现在getReader()方法中，当设置了编码属性，spring会使用相应的编码作为输入流的编码  
> 2. 数据准备阶段的逻辑 `this.reader.loadBeanDefinitions(resource);`：  
```  
	1. 对传入的resource参数进行封装
	2. 通过SAX读取XML文件的方式来准备InputSource对象 
	3. doLoadBeanDefinitions(inputSource,encodeResource.getResource()) // 核心处理  
	  3.1：获取对XML文件的验证模式
	  3.2：加载XML文件，并得到对应的Document
	  3.3：根据返回的Document注册Bean信息
```  
> 3. XML常用验证模式有两种：DTD && XSD
     DTD(Document Type Definition)文档类型定义，是一种保证XML文档格式正确有效的验证方法，可通过比较XML文档和DTD文件来看文档是否符合规范.
```
<!DOCTYPE beans PUBILC "-//Spring//DTD BEAN2.0//EN""http://www.Springframework.org/dtd/Spring-beans-2.0.dtd">
<beans>
...
</beans>
```  

    XSD(XML Schemas Definition) = XML Schema语言：描述了XML文档的结构。  (名称空间，名称空间对应的XML Schema文档的存储位置)  
    
> 4. 获取Document,XmlBeanFactoryReader将读取任务委托(delegate)给了DocumentLoader  
     `private DocumentLoader documentLoader = new DefaultDocumentLoader();`  
     
> 5. EntityResolver:如果SAX应用程序需要实现自定义处理外部实体，则必须实现此接口并使用setEntityResolver方法向SAX驱动器注册一个实例  
     作用： 项目本身就可以提供一个如何寻找DTD生命的方法。  
 
> 6. 源码中大量使用了面向对象中单一职责的原则，将逻辑处理委托给单一的类进行处理。

 


