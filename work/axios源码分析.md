#		axios源码分析

* axios和Axiso的关系

  * 1.axios不是Axios的实例化对象,但是axios具有Axios实例化对象的功能<span style="color:red">(axios具有Axios原型对象上的所有方法,有Axios对象上的所有属性)</span>

  * ```js
    在lib/axios文件下
    var axios = createInstance(defaults)
    function createInstance(defaultConfig){
    　　    var context = new Axios(defaultConfig);
    　　    var instance = bind(Axios.prototype.request, context);
        	//这里的extend方法会把Axios的方法和属性复制给instance(axios)并且调用bind改变this的指向
       	    utils.extend(instance, Axios.prototype, context);
    　　    utils.extend(instance, context)
    　　    return instance;
    　　}
    ```

  * 2.axios的来源:axios是`Axios.prototype.request`函数bind()返回的函数<span style="color:red">(bind改变课this的指向)</span>

  * ```js
    在/lib/helps/bind.js文件下
    //这是bind函数
    function bind(fn, thisArg){
    　	return function wrap(){
    　	var args = new Array(arguments.length);
    　	for(var i = 0; i < args.length; i++){
    　　  	args[i] = arguments[i]
    　	 }
    　	return fn.apply(thisArg, args)//此处改变了this的指向
    　	}
    　}
    
    ```

* instance与axios的区别

  * 1.相同点

    * 都是可以发送任意请求的函数   `request(config)`

    * 都有发特定请求的各种方法   `get/post/put/delete`

    * 都有默认配置和拦截器的属性  `defaults/interceptors`

    * ```js
      在lib/axios.js文件下
      axios.create = function create(instanceConfig) {
          //此处返回了一个新的instance对象,并且这个对象的属性是复制的axios.defaults的
        return createInstance(mergeConfig(axios.defaults, instanceConfig));
      };
      ```

  * 2.不同点

    * 默认配置很可能不一样
    * `return createInstance(mergeConfig(axios.defaults, instanceConfig));`由此代码可以推断两者的默认配置可能不一样
    * instance没有axios后添加的一些方法:`create/CancelToken/all`

* 运行流程图

  * ![图片1](C:\Users\大帅逼\Desktop\图片1.png)

  * 1.整体流程

  * ```js
    request(config) ==> dispatchReqquest(config)  ==> 	xhrAdapter(config)
    *1.request(config):
    	将 请求拦截器/dispatchRequest()/响应拦截器  通过promise串链起来,返回promise
    *2.dispatchRequest(config)
    	转换请求数据 ===> 调用 xhrAdapter()发请求 ===> 请求返回后转换响应数据,返回promise
    *3.xhrAdapter(config)
    	创建xhr对象,依据config进行相关设置,发送	特定请求,并接受响应数据,返回promise
    ```

  * 请求/响应拦截器

    * 1.请求拦截器

      * 在真正发请求前执行的回调函数
      * 可以对请求进行检查或配置进行特定处理
      * 成功的回调函数,传递的是config
      * 失败的传递的默认是error

    * 2.响应拦截器

      * 请求得到响应后执行的回调函数
      * 可以对响应数据进行特定处理
      * 成功的回调,传的默认是response
      * 失败传的默认error

    * 3.如果同时有两个或者两个以上的拦截器,请求拦截器的执行顺序是从后往前,响应拦截器执行的顺序是顺序执行,造成这种现象的原因是

    * ```js
      在lib/core/Axios.js下
      Axios.prototype.request = function request(config) {
      		var chain = [dispatchRequest, undefined];
      		this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
                  //1.此处用了unshift方法,是从前面添加元素到数组,会造成后边的拦截器在数组的最前面
          	chain.unshift(interceptor.fulfilled, interceptor.rejected);
        });
        this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
            	//2.此处用了push方法,是从后面添加元素到数组
          	chain.push(interceptor.fulfilled, interceptor.rejected);
        });
        		while (chain.length) {
      		//3.而执行的时候是用循环依次取出数组中元素,      
              	promise = promise.then(chain.shift(), chain.shift());
        	}
      }
      ```

    * 

  * axios的请求/响应数据转换器

    * 1.请求转换器:对请求头和请求体数据进行特定处理的函数(在发请求之前进行处理)

    * 2.响应转换器:将响应体JSON字符串解析为js对象或者数组的函数(在得到响应之后进行处理)

    * ```js
      在lib/defaults.js文件下
      //请求转换器
      transformRequest: [function transformRequest(data, headers) {
      	if (utils.isObject(data)) {
            	setContentTypeIfUnset(headers, 'application/json;charset=utf-8');
            return JSON.stringify(data);
          }
      }
      //响应转换器
       transformResponse: [function transformResponse(data) {
       	if (typeof data === 'string') {
           	 try {
           	   data = JSON.parse(data);
          	  } catch (e) { /* Ignore */ }
        	  }
        	  return data;
       }
      ```

      

