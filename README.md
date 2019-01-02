# spring_master
spring源码剖析，大部分都是基于Springframework5.0  

### 容器的基础XMLBeanFactory  

  XmlBeanFactory继承自DefaultListableBeanFactory，而DefaultListableBeanFactory是整个bean加载的核心部分，是spring注册及加载bean的默认实现，而XmlBeanFactory与DefaultListableBeanFactory不同的地方其实是使用了自定义的XML读取器XmlBeanDefinitionReader,实现了个性化的BeanDefinitionReader读取 
  
  **DefaultListableBeanFactory层次结构图**
![DefaultListableBeanFactory层次结构图](https://github.com/momokanni/spring_master/blob/master/UML_img/DefaultListableBeanFactory.png) 

  XmlBeanFactory对DefaultListableBeanFactory进行了扩展，主要用于从XML文件中读取BeanDefinition，注册及获取Bean都是使用从父类继承的方法去实现。唯独不同的个性化实现就是增加了XmlBeanDefinitionReader类型的reader属性。在XmlBeanFactory中主要使用reader属性对资源文件进行读取和注册。  
  
`BeanFactory bf = new XMLBeanFactory(new ClassPathResource("beanFactoryTest.xml"))`  

![XmlBeanFactory初始化时序图](https://github.com/momokanni/spring_master/blob/master/UML_img/XmlBeanFactory_%E5%88%9D%E5%A7%8B%E5%8C%96%E6%97%B6%E5%BA%8F%E5%9B%BE.jpg)  
**注：**  
> 1. 调用ClassPathResource的构造函数来构造Resource资源文件的实例对象。  
> 2. 配置文件封装：  
```  
      public interface InputStreamSource {  
          InputStream getInputStream() throws IOException;  
      }  
      
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
          Resource createRelative(String relativePath) throws IOException; // Create a resource relative to this resource.  
          
          @Nullable
	        String getFilename(); //Determine a filename for this resource  
          
          String getDescription(); // return a description for this resource  
          
          // Java8以后接口都提供默认方法 return 基础资源的字节通道
          default ReadableByteChannel readableChannel() throws IOException {
            return Channels.newChannel(getInputStream());
          }
      }  
 ```  
      
![XmlBeanFactory结构图](https://github.com/momokanni/spring_master/blob/master/UML_img/XmlBeanFactory.png)  

**XmlBeanDefinitionReader**  
![XmlBeanDefinitionReader层次结构图](https://github.com/momokanni/spring_master/blob/master/UML_img/XmlBeanDefinitionReader.png)  

> 1. 通过继承AbstractBeanDefinitionReader中的方法，来使用ResourceLoader将资源文件路径转换为对应的Resource文件  
> 2. 通过DocumentLoader对Resource文件进行转换，将Resource文件转换为Document文件。  
> 3. 通过实现接口：BeanDefinitionDocumentReader的DefaultBeanDefinitionDocumentReaDer类对Document进行解析，并使用BeanDefinitionParserDelegate对Element进行解析。  


