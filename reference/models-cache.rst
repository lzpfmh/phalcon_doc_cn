缓存对象关系映射Caching in the ORM
=====================================
现实中的每个应用都不同，一些应用的模型数据经常改变而另一些模型的数据几乎不同。访问数据库在很多时候对我们应用的来说 是个瓶颈。这是由于我们每次访问应用时都会和数据库数据通信，和数据库进行通信的代价是很大的。因此在必要时我们可以通过增加缓存层来获取更高的性能。 本章内容的重点即是探讨实施缓存来提高性能的可行性。Phalcon框架给我们提供了灵活的缓存技术来实现我们的应用缓存。

Every application is different, we could have models whose data change frequently and others that rarely change.
Accessing database systems is often one of the most common bottlenecks in terms of performance. This is due to
the complex connection/communication processes that PHP must do in each request to obtain data from the database.
Therefore, if we want to achieve good performance we need to add some layers of caching where the
application requires it.


This chapter explains the possible points where it is possible to implement caching to improve performance.
The framework gives you the tools to implement the cache where you demand of it according to the architecture
of your application.

缓存结果集Caching Resultsets
---------------------------------
一个非常可行的方案是我们可以为那些不经常改变且经常访问的数据库数据进行缓存，比如把他们放入内存，这样可以加快程序的执行速度。当 :doc:`Phalcon\\Mvc\\Model <../api/Phalcon_Mvc_Model>` 需要使用缓存数据的服务时Model可以直接从DI中取得此缓存服务modelsCache(惯例名).Phalcon提供了一个组件（服务）可以用来 :doc:`cache <cache>` 任何种类的数据，下面我们会解释如何在model使用它。第一步我们要在启动文件注册这个服务:

A well established technique to avoid the continuous access to the database is to cache resultsets that don't change
frequently using a system with faster access (usually memory).

When :doc:`Phalcon\\Mvc\\Model <../api/Phalcon_Mvc_Model>` requires a service to cache resultsets, it will
request it to the Dependency Injector Container with the convention name "modelsCache".

As Phalcon provides a component to :doc:`cache <cache>` any kind of data, we'll explain how to integrate it with Models.
First, you must register it as a service in the services container:

.. code-block:: php

    <?php

    use Phalcon\Cache\Frontend\Data as FrontendData;
    use Phalcon\Cache\Backend\Memcache as BackendMemcache;

    //Set the models cache service
    $di->set('modelsCache', function() {

        //Cache data for one day by default
        $frontCache = new FrontendData(array(
            "lifetime" => 86400
        ));

        //Memcached connection settings
        $cache = new BackendMemcache($frontCache, array(
            "host" => "localhost",
            "port" => "11211"
        ));

        return $cache;
    });

在注册缓存服务时我们可以按照我们的所需进行配置。一旦完成正确的缓存设置之后，我们可以按如下的方式缓存查询的结果了:	
	
You have complete control in creating and customizing the cache before being used by registering the service
as an anonymous function. Once the cache setup is properly defined you could cache resultsets as follows:

.. code-block:: php

    <?php

    // Get products without caching
    $products = Products::find();

    // Just cache the resultset. The cache will expire in 1 hour (3600 seconds)
    $products = Products::find(array(
        "cache" => array("key" => "my-cache")
    ));

    // Cache the resultset for only for 5 minutes
    $products = Products::find(array(
        "cache" => array("key" => "my-cache", "lifetime" => 300)
    ));

    // Using a custom cache
    $products = Products::find(array("cache" => $myCache));

有关系查询得到的结果集同样可以加入缓存：	
	
Caching could be also applied to resultsets generated using relationships:

.. code-block:: php

    <?php

    // Query some post
    $post     = Post::findFirst();

    // Get comments related to a post, also cache it
    $comments = $post->getComments(array(
        "cache" => array("key" => "my-key")
    ));

    // Get comments related to a post, setting lifetime
    $comments = $post->getComments(array(
        "cache" => array("key" => "my-key", "lifetime" => 3600)
    ));

如果想删除已经缓存的结果，则只需要使用前面指定的缓存的键值进行删除即可。	
	
When a cached resultset needs to be invalidated, you can simply delete it from the cache using the previously specified key.

注意并不是所有的结果都必须缓存下来。那些经常改变的数据就不应该被缓存，这样做只会影响应用的性能。另外对于那些特别大的不易变的数据集，开发者应用根据实际情况进行选择是否进行缓存。

Note that not all resultsets must be cached. Results that change very frequently should not be cached since they
are invalidated very quickly and caching in that case impacts performance. Additionally, large datasets that
do not change frequently could be cached, but that is a decision that the developer has to make based on the
available caching mechanism and whether the performance impact to simply retrieve that data in the
first place is acceptable.

重写 find 与 findFirst 方法 Overriding find/findFirst
--------------------------------------------------------
从上面的我们可以看到这两个方法是从:doc:`Phalcon\\Mvc\\Model <../api/Phalcon_Mvc_Model>`继承而来 <../api/Phalcon_Mvc_Model>:

As seen above, these methods are available in models that inherit :doc:`Phalcon\\Mvc\\Model <../api/Phalcon_Mvc_Model>`:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Model;

    class Robots extends Model
    {

        public static function find($parameters=null)
        {
            return parent::find($parameters);
        }

        public static function findFirst($parameters=null)
        {
            return parent::findFirst($parameters);
        }

    }

这样做会影响到所有此类的对象对这两个函数的调用，我们可以在其中添加一个缓存层，如果未有其它缓存的话（比如modelsCache）。例如，一个基本的缓存实现是我们在此类中添加一个静态的变量以避免在同一请求中多次查询数据库：	
	
By doing this, you're intercepting all the calls to these methods, this way, you can add a cache
layer or run the query if there is no cache. For example, a very basic cache implementation, uses
a static property to avoid that a record would be queried several times in a same request:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Model;

    class Robots extends Model
    {

        protected static $_cache = array();

        /**
         * Implement a method that returns a string key based
         * on the query parameters
         */
        protected static function _createKey($parameters)
        {
            $uniqueKey = array();
            foreach ($parameters as $key => $value) {
                if (is_scalar($value)) {
                    $uniqueKey[] = $key . ':' . $value;
                } else {
                    if (is_array($value)) {
                        $uniqueKey[] = $key . ':[' . self::_createKey($value) .']';
                    }
                }
            }
            return join(',', $uniqueKey);
        }

        public static function find($parameters=null)
        {

            //Create an unique key based on the parameters
            $key = self::_createKey($parameters);

            if (!isset(self::$_cache[$key])) {
                //Store the result in the memory cache
                self::$_cache[$key] = parent::find($parameters);
            }

            //Return the result in the cache
            return self::$_cache[$key];
        }

        public static function findFirst($parameters=null)
        {
            // ...
        }

    }

访问数据要远比计算key值慢的多，我们在这里定义自己需要的key生成方式。注意好的键可以避免冲突，这样就可以依据不同的key值取得不同的缓存结果。

Access the database is several times slower than calculate a cache key, you're free in implement the
key generation strategy you find better for your needs. Note that a good key avoids collisions as much as possible,
this means that different keys returns unrelated records to the find parameters.

上面的例子中我们把缓存放在了内存中，这做为第一级的缓存。当然我们也可以在第一层缓存的基本上实现第二层的缓存比如使用 APC/XCache或是使用NoSQL数据库（如MongoDB等）：	

In the above example, we used a cache in memory, it is useful as a first level cache. Once we have the memory cache,
we can implement a second level cache layer like APC/XCache or a NoSQL database:

.. code-block:: php

    <?php

    public static function find($parameters=null)
    {

        //Create an unique key based on the parameters
        $key = self::_createKey($parameters);

        if (!isset(self::$_cache[$key])) {

            //We're using APC as second cache
            if (apc_exists($key)) {

                $data = apc_fetch($key);

                //Store the result in the memory cache
                self::$_cache[$key] = $data;

                return $data;
            }

            //There are no memory or apc cache
            $data = parent::find($parameters);

            //Store the result in the memory cache
            self::$_cache[$key] = $data;

            //Store the result in APC
            apc_store($key, $data);

            return $data;
        }

        //Return the result in the cache
        return self::$_cache[$key];
    }

这样我们可以对可模型的缓存进行完全的控制，如果多个模型需要进行如此缓存可以建立一个基础类：	
	
This gives you full control on how the the caches must be implemented for each model, if this strategy is common to several models
you can create a base class for all of them:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Model;

    class CacheableModel extends Model
    {

        protected static function _createKey($parameters)
        {
            // .. create a cache key based on the parameters
        }

        public static function find($parameters=null)
        {
            //.. custom caching strategy
        }

        public static function findFirst($parameters=null)
        {
            //.. custom caching strategy
        }
    }

然后把这个类作为其它缓存类的基类：	
	
Then use this class as base class for each 'Cacheable' model:

.. code-block:: php

    <?php

    class Robots extends CacheableModel
    {

    }

强制缓存Forcing Cache
-----------------------
前面的例子中我们在Phalcon\\Mvc\\Model中使用框架内建的缓存组件。为实现强制缓存我们传递了cache作为参数：

Earlier we saw how Phalcon\\Mvc\\Model has a built-in integration with the caching component provided by the framework. To make a record/resultset
cacheable we pass the key 'cache' in the array of parameters:

.. code-block:: php

    <?php

    // Cache the resultset for only for 5 minutes
    $products = Products::find(array(
        "cache" => array("key" => "my-cache", "lifetime" => 300)
    ));

为了自由的对特定的查询结果进行缓存我们，比如我们想对模型中的所有查询结果进行缓存我们可以重写find/findFirst方法：	
	
This gives us the freedom to cache specific queries, however if we want to cache globally every query performed over the model,
we can override the find/findFirst method to force every query to be cached:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Model;

    class Robots extends Model
    {

        protected static function _createKey($parameters)
        {
            // .. create a cache key based on the parameters
        }

        public static function find($parameters=null)
        {

            //Convert the parameters to an array
            if (!is_array($parameters)) {
                $parameters = array($parameters);
            }

            //Check if a cache key wasn't passed
            //and create the cache parameters
            if (!isset($parameters['cache'])) {
                $parameters['cache'] = array(
                    "key"      => self::_createKey($parameters),
                    "lifetime" => 300
                );
            }

            return parent::find($parameters);
        }

        public static function findFirst($parameters=null)
        {
            //...
        }

    }

缓存 PHQL 查询 Caching PHQL Queries
--------------------------------------
ORM中的所有查询，不管多么高级的查询方法内部使用使用PHQL进行实现的。这个语言可以让我们非常自由的创建各种查询，当然这些查询也可以被缓存：

All queries in the ORM, no matter how high level syntax we used to create them are handled internally using PHQL.
This language gives you much more freedom to create all kinds of queries. Of course these queries can be cached:

.. code-block:: php

    <?php

    $phql = "SELECT * FROM Cars WHERE name = :name:";

    $query = $this->modelsManager->createQuery($phql);

    $query->cache(array(
        "key"      => "cars-by-name",
        "lifetime" => 300
    ));

    $cars = $query->execute(array(
        'name' => 'Audi'
    ));

如果不想使用隐式的缓存尽管使用你想用的缓存方式：	
	
If you don't want to use the implicit cache just save the resulset into your favorite cache backend:

.. code-block:: php

    <?php

    $phql = "SELECT * FROM Cars WHERE name = :name:";

    $cars = $this->modelsManager->executeQuery($phql, array(
        'name' => 'Audi'
    ));

    apc_store('my-cars', $cars);

重用的相关记录Reusable Related Records
---------------------------------------------
一些模型有关联的数据表我们直接使用在内存中的实例关联数据：

Some models may have relationships to other models. This allows us to easily check the records that relate to instances in memory:

.. code-block:: php

    <?php

    //Get some invoice
    $invoice  = Invoices::findFirst();

    //Get the customer related to the invoice
    $customer = $invoice->customer;

    //Print his/her name
    echo $customer->name, "\n";

这个例子非常简单，依据查询到的订单信息取得用户信息之后再取得用户名。下面的情景也是如何：我们查询了一些订单的信息，然后取得这些订单相关联 用户的信息，之后取得用户名：	
	
This example is very simple, a customer is queried and can be used as required, for example, to show its name.
This also applies if we retrieve a set of invoices to show customers that correspond to these invoices:

.. code-block:: php

    <?php

    //Get a set of invoices
    // SELECT * FROM invoices;
    foreach (Invoices::find() as $invoice) {

        //Get the customer related to the invoice
        // SELECT * FROM customers WHERE id = ?;
        $customer = $invoice->customer;

        //Print his/her name
        echo $customer->name, "\n";
    }

每个客户可能会有一个或多个帐单，这就意味着客户对象没必须取多次。为了避免一次次的重复取客户信息，我们这里设置关系为reusable为true, 这样ORM即知可以重复使用客户信息：	
	
A customer may have one or more bills, this means that the customer may be unnecessarily more than once.
To avoid this, we could mark the relationship as reusable, this way, we tell the ORM to automatically reuse
the records instead of re-querying them again and again:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Model;

    class Invoices extends Model
    {

        public function initialize()
        {
            $this->belongsTo("customers_id", "Customer", "id", array(
                'reusable' => true
            ));
        }

    }

此Cache存在于内存中，这意味着当请示结束时缓存数据即被释放。我们也可以通过重写模型管理器的方式实现更加复杂的缓存：	
	
This cache works in memory only, this means that cached data are released when the request is terminated. You can
add a more sophisticated cache for this scenario overriding the models manager:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Model\Manager as ModelManager;

    class CustomModelsManager extends ModelManager
    {

        /**
         * Returns a reusable object from the cache
         *
         * @param string $modelName
         * @param string $key
         * @return object
         */
        public function getReusableRecords($modelName, $key){

            //If the model is Products use the APC cache
            if ($modelName == 'Products'){
                return apc_fetch($key);
            }

            //For the rest, use the memory cache
            return parent::getReusableRecords($modelName, $key);
        }

        /**
         * Stores a reusable record in the cache
         *
         * @param string $modelName
         * @param string $key
         * @param mixed $records
         */
        public function setReusableRecords($modelName, $key, $records){

            //If the model is Products use the APC cache
            if ($modelName == 'Products'){
                apc_store($key, $records);
                return;
            }

            //For the rest, use the memory cache
            parent::setReusableRecords($modelName, $key, $records);
        }
    }

别忘记注册模型管理器到DI中：	
	
Do not forget to register the custom models manager in the DI:

.. code-block:: php

    <?php

    $di->setShared('modelsManager', function() {
        return new CustomModelsManager();
    });

缓存相关记录 Caching Related Records
---------------------------------------
当使用find或findFirst查询关联数据时，ORM内部会自动的依据以下规则创建查询条件于：

When a related record is queried, the ORM internally builds the appropriate condition and gets the required records using find/findFirst
in the target model according to the following table:

+---------------------+---------------------------------------------------------------------------------------------------------------+
| Type                | Description                                                                          | Implicit Method        |
+=====================+===============================================================================================================+
| Belongs-To          | Returns a model instance of the related record directly                              | findFirst              |
+---------------------+---------------------------------------------------------------------------------------------------------------+
| Has-One             | Returns a model instance of the related record directly                              | findFirst              |
+---------------------+---------------------------------------------------------------------------------------------------------------+
| Has-Many            | Returns a collection of model instances of the referenced model                      | find                   |
+---------------------+---------------------------------------------------------------------------------------------------------------+

这意味着当我们取得关联记录时，我们需要解析如何如何取得数据的方法：

This means that when you get a related record you could intercept how these data are obtained by implementing the corresponding method:

.. code-block:: php

    <?php

    //Get some invoice
    $invoice  = Invoices::findFirst();

    //Get the customer related to the invoice
    $customer = $invoice->customer; // Invoices::findFirst('...');

    //Same as above
    $customer = $invoice->getCustomer(); // Invoices::findFirst('...');

因此，我们可以替换掉Invoices模型中的findFirst方法然后实现我们使用适合的方法	
	
Accordingly, we could replace the findFirst method in the model Invoices and implement the cache we consider most appropriate:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Model;

    class Invoices extends Model
    {

        public static function findFirst($parameters=null)
        {
            //.. custom caching strategy
        }
    }

递归缓存相关记录Caching Related Records Recursively
---------------------------------------------------------
在这种场景下我们假定我们每次取主记录时都会取模型的关联记录，如果我们此时保存这些记录把相关记录也保存下来可能会为我们的系统带来一些性能上的提升：

In this scenario, we assume that everytime we query a result we also retrieve their associated records.
If we store the records found together with their related entities perhaps we could reduce a bit the overhead required
to obtain all entities:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Model;

    class Invoices extends Model
    {

        protected static function _createKey($parameters)
        {
            // .. create a cache key based on the parameters
        }

        protected static function _getCache($key)
        {
            // returns data from a cache
        }

        protected static function _setCache($key)
        {
            // stores data in the cache
        }

        public static function find($parameters=null)
        {
            //Create a unique key
            $key     = self::_createKey($parameters);

            //Check if there are data in the cache
            $results = self::_getCache($key);

            // Valid data is an object
            if (is_object($results)) {
                return $results;
            }

            $results = array();

            $invoices = parent::find($parameters);
            foreach ($invoices as $invoice) {

                //Query the related customer
                $customer = $invoice->customer;

                //Assign it to the record
                $invoice->customer = $customer;

                $results[] = $invoice;
            }

            //Store the invoices in the cache + their customers
            self::_setCache($key, $results);

            return $results;
        }

        public function initialize()
        {
            // add relations and initialize other stuff
        }
    }

从已经缓存的订单中取得用户信息，可以减少系统的负载。注意我们也可以使用PHQL来实现这个，下面使用了PHQL来实现：	
	
Getting the invoices from the cache already obtains the customer data in just one hit, reducing the overall overhead of the operation.
Note that this process can also be performed with PHQL following an alternative solution:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Model;

    class Invoices extends Model
    {

        public function initialize()
        {
            // add relations and initialize other stuff
        }

        protected static function _createKey($conditions, $params)
        {
            // .. create a cache key based on the parameters
        }

        public function getInvoicesCustomers($conditions, $params=null)
        {
            $phql  = "SELECT Invoices.*, Customers.*
            FROM Invoices JOIN Customers WHERE " . $conditions;

            $query = $this->getModelsManager()->executeQuery($phql);

            $query->cache(array(
                "key"      => self::_createKey($conditions, $params),
                "lifetime" => 300
            ));

            return $query->execute($params);
        }

    }

基于条件的缓存Caching based on Conditions
------------------------------------------
此例中，我依据当的条件实施缓存：根据主键值不同的范围分配不同的缓存后端：

In this scenario, the cache is implemented conditionally according to current conditions received.
According to the range where the primary key is located we choose a different cache backend:

+---------------------+--------------------+
| Type                | Cache Backend      |
+=====================+====================+
| 1 - 10000           | mongo1             |
+---------------------+--------------------+
| 10000 - 20000       | mongo2             |
+---------------------+--------------------+
| > 20000             | mongo3             |
+---------------------+--------------------+

最简单的方式即是为模型类添加一个静态的方法，此方法中我们指定要使用的缓存：

The easiest way is adding an static method to the model that chooses the right cache to be used:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Model;

    class Robots extends Model
    {

        public static function queryCache($initial, $final)
        {
            if ($initial >= 1 && $final < 10000) {
                return self::find(array(
                    'id >= ' . $initial . ' AND id <= '.$final,
                    'cache' => array('service' => 'mongo1')
                ));
            }
            if ($initial >= 10000 && $final <= 20000) {
                return self::find(array(
                    'id >= ' . $initial . ' AND id <= '.$final,
                    'cache' => array('service' => 'mongo2')
                ));
            }
            if ($initial > 20000) {
                return self::find(array(
                    'id >= ' . $initial,
                    'cache' => array('service' => 'mongo3')
                ));
            }
        }

    }

这个方法是可以解决问题，不过如果我们需要添加其它的参数比如排序或条件等我们还要创建更复杂的方法。另外当我们使用find/findFirst来查询关联数据时此方法亦会失效：	
	
This approach solves the problem, however, if we want to add other parameters such orders or conditions we would have to create
a more complicated method. Additionally, this method does not work if the data is obtained using related records or a find/findFirst:

.. code-block:: php

    <?php

    $robots = Robots::find('id < 1000');
    $robots = Robots::find('id > 100 AND type = "A"');
    $robots = Robots::find('(id > 100 AND type = "A") AND id < 2000');

    $robots = Robots::find(array(
        '(id > ?0 AND type = "A") AND id < ?1',
        'bind'  => array(100, 2000),
        'order' => 'type'
    ));

为了实现这个我们需要拦截中间语言解析，然后书写相关的代码以定制缓存： 首先我们需要创建自定义的创建器，然后我们可以使用它来创建守全自己定义的查询：	
	
To achieve this we need to intercept the intermediate representation (IR) generated by the PHQL parser and
thus customize the cache everything possible:

The first is create a custom builder, so we can generate a totally customized query:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Model\Query\Builder as QueryBuilder;

    class CustomQueryBuilder extends QueryBuilder
    {

        public function getQuery()
        {
            $query = new CustomQuery($this->getPhql());
            $query->setDI($this->getDI());
            return $query;
        }

    }

这里我们返回的是CustomQuery而不是不直接的返回Phalcon\\Mvc\\Model\\Query， 类定义如下所示：	
	
Instead of directly returning a Phalcon\\Mvc\\Model\\Query, our custom builder returns a CustomQuery instance,
this class looks like:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Model\Query as ModelQuery;

    class CustomQuery extends ModelQuery
    {

        /**
         * The execute method is overridden
         */
        public function execute($params=null, $types=null)
        {
            //Parse the intermediate representation for the SELECT
            $ir = $this->parse();

            //Check if the query has conditions
            if (isset($ir['where'])) {

                //The fields in the conditions can have any order
                //We need to recursively check the conditions tree
                //to find the info we're looking for
                $visitor = new CustomNodeVisitor();

                //Recursively visits the nodes
                $visitor->visit($ir['where']);

                $initial = $visitor->getInitial();
                $final   = $visitor->getFinal();

                //Select the cache according to the range
                //...

                //Check if the cache has data
                //...
            }

            //Execute the query
            $result = $this->_executeSelect($ir, $params, $types);

            //cache the result
            //...

            return $result;
        }

    }

这里我们实现了一个帮助类用以递归的的检查条件以查询字段用以识我们知了需要使用缓存的范围（即检查条件以确认实施查询缓存的范围）：	
	
Implementing a helper (CustomNodeVisitor) that recursively checks the conditions looking for fields that
tell us the possible range to be used in the cache:

.. code-block:: php

    <?php

    class CustomNodeVisitor
    {

        protected $_initial = 0;

        protected $_final = 25000;

        public function visit($node)
        {
            switch ($node['type']) {

                case 'binary-op':

                    $left  = $this->visit($node['left']);
                    $right = $this->visit($node['right']);
                    if (!$left || !$right) {
                        return false;
                    }

                    if ($left=='id') {
                        if ($node['op'] == '>') {
                            $this->_initial = $right;
                        }
                        if ($node['op'] == '=') {
                            $this->_initial = $right;
                        }
                        if ($node['op'] == '>=')    {
                            $this->_initial = $right;
                        }
                        if ($node['op'] == '<') {
                            $this->_final = $right;
                        }
                        if ($node['op'] == '<=')    {
                            $this->_final = $right;
                        }
                    }
                    break;

                case 'qualified':
                    if ($node['name'] == 'id') {
                        return 'id';
                    }
                    break;

                case 'literal':
                    return $node['value'];

                default:
                    return false;
            }
        }

        public function getInitial()
        {
            return $this->_initial;
        }

        public function getFinal()
        {
            return $this->_final;
        }
    }

最后，我们替换Robots模型中的查询方法以使用我们创建的自定义类：	
	
Finally, we can replace the find method in the Robots model to use the custom classes we've created:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Model;

    class Robots extends Model
    {
        public static function find($parameters=null)
        {

            if (!is_array($parameters)) {
                $parameters = array($parameters);
            }

            $builder = new CustomQueryBuilder($parameters);
            $builder->from(get_called_class());

            if (isset($parameters['bind'])) {
                return $builder->getQuery()->execute($parameters['bind']);
            } else {
                return $builder->getQuery()->execute();
            }

        }
    }

缓存PHQL查询计划 Caching of PHQL planning
---------------------------------------------
像大多数现代的操作系统一样PHQL内部会缓存执行计划，如果同样的语句多次执行，PHQL会使用之前生成的查询计划以提升系统的性能， 对开发者来说只采用绑定参数的形式传递参数即可实现：

As well as most moderns database systems PHQL internally caches the execution plan,
if the same statement is executed several times PHQL reuses the previously generated plan
improving performance, for a developer to take better advantage of this is highly recommended
build all your SQL statements passing variable parameters as bound parameters:

.. code-block:: php

    <?php

    for ($i = 1; $i <= 10; $i++) {

        $phql   = "SELECT * FROM Store\Robots WHERE id = " . $i;
        $robots = $this->modelsManager->executeQuery($phql);

        //...
    }

上面的例子中，Phalcon产生了10个查询计划，这导致了应用的内存使用量增加。重写以上代码，我们使用绑定参数的这个优点可以减少系统和数据库的过多操作：	
	
In the above example, ten plans were generated increasing the memory usage and processing in the application.
Rewriting the code to take advantage of bound parameters reduces the processing by both ORM and database system:

.. code-block:: php

    <?php

    $phql = "SELECT * FROM Store\Robots WHERE id = ?0";

    for ($i = 1; $i <= 10; $i++) {

        $robots = $this->modelsManager->executeQuery($phql, array($i));

        //...
    }

重用PHQL查询也可以提高性能：	
	
Performance can be also improved reusing the PHQL query:

.. code-block:: php

    <?php

    $phql  = "SELECT * FROM Store\Robots WHERE id = ?0";
    $query = $this->modelsManager->createQuery($phql);

    for ($i = 1; $i <= 10; $i++) {

        $robots = $query->execute($phql, array($i));

        //...
    }

`prepared statements`_ 的查询计划亦可以被大多数的数据库所缓存，这样可以减少执行的时间，也可以使用我们的系统免受`SQL Injections`_的影响。	
	
Execution plans for queries involving `prepared statements`_ are also cached by most database systems
reducing the overall execution time, also protecting your application against `SQL Injections`_.

.. _`prepared statements` : http://en.wikipedia.org/wiki/Prepared_statement
.. _`SQL Injections` : http://en.wikipedia.org/wiki/SQL_injection
