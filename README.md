# spring_master
spring源码剖析，大部分都是基于Springframework5.0  

交代： spring最核心的两个类： `DefaultListableBeanFactory` && `XmlBeanDefinitionReader ` 

### 容器的基础XMLBeanFactory  

  `XmlBeanFactory`继承自`DefaultListableBeanFactory`，而DefaultListableBeanFactory是整个bean加载的核心部分，是spring注册及加载bean的默认实现，而XmlBeanFactory与DefaultListableBeanFactory不同的地方其实是使用了自定义的XML读取器`XmlBeanDefinitionReader`,实现了个性化的`BeanDefinitionReader`读取 
  
  **DefaultListableBeanFactory层次结构图**
![DefaultListableBeanFactory层次结构图](https://github.com/momokanni/spring_master/blob/master/UML_img/DefaultListableBeanFactory.png) 

  XmlBeanFactory对DefaultListableBeanFactory进行了扩展，主要用于从XML文件中读取BeanDefinition，注册及获取Bean都是使用从父类继承的方法去实现。唯独不同的个性化实现就是增加了XmlBeanDefinitionReader类型的reader属性。在XmlBeanFactory中主要使用reader属性对资源文件进行读取和注册。  
  
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



**XmlBeanDefinitionReader**  
![XmlBeanDefinitionReader层次结构图](https://github.com/momokanni/spring_master/blob/master/UML_img/XmlBeanDefinitionReader.png)  

> 1. 通过继承AbstractBeanDefinitionReader中的方法，来使用ResourceLoader将资源文件路径转换为对应的Resource文件  
> 2. 通过DocumentLoader对Resource文件进行转换，将Resource文件转换为Document文件。  
> 3. 通过实现接口：BeanDefinitionDocumentReader的DefaultBeanDefinitionDocumentReaDer类对Document进行解析，并使用BeanDefinitionParserDelegate对Element进行解析。  


