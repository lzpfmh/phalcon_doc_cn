资源文件管理Assets Management
==============================
Phalcon\\Assets 是一个让开发者管理静态资源的组件，如管理css，javascript等。

Phalcon\\Assets is a component that allows the developer to manage static resources
such as css stylesheets or javascript libraries in a web application.

 :doc:`Phalcon\\Assets\\Manager <../api/Phalcon_Assets_Manager>` 存在于DI容器中，所以我们可以在服务容器存在的任何地方使用它来添加/管理资源。

:doc:`Phalcon\\Assets\\Manager <../api/Phalcon_Assets_Manager>` is available in the services
container, so you can add resources from any part of the application where the container
is available.

添加资源Adding Resources
---------------------------
Assets支持两个内置的资源管理器：css和javascripts.我们可以根据需要创建其它的资源。资源管理器内部保存了两类资源集合一为 javascript另一为css.

Assets supports two built-in resources: css and javascripts. You can create other
resources if you need. The assets manager internally stores two default collections
of resources one for javascript and another for css.

我们可以非常简单的向这两个集合里添加资源，如下：

You can easily add resources to these collections like follows:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Controller;

    class IndexController extends Controller
    {
        public function index()
        {

            //Add some local CSS resources
            $this->assets
                ->addCss('css/style.css')
                ->addCss('css/index.css');

            //and some local javascript resources
            $this->assets
                ->addJs('js/jquery.js')
                ->addJs('js/bootstrap.min.js');

        }
    }

然后我们可以在视图中输出资源：	
	
Then in the views added resources can be printed:

.. code-block:: html+php

    <html>
        <head>
            <title>Some amazing website</title>
            <?php $this->assets->outputCss() ?>
        </head>
        <body>

            <!-- ... -->

            <?php $this->assets->outputJs() ?>
        </body>
    <html>

Volt语法：	
	
Volt syntax:

.. code-block:: html+jinja

    <html>
        <head>
            <title>Some amazing website</title>
              {{ assets.outputCss() }}
        </head>
        <body>

            <!-- ... -->

              {{ assets.outputJs() }}
        </body>
    <html>

本地与远程资源Local/Remote resources
-------------------------------------------
本地资源是同一应用中的资源，这些资源存在于应用的根目录中。 :doc:`Phalcon\\Mvc\\Url <../api/Phalcon_Mvc_Url>` 用来生成 本地的url. 

Local resources are those who're provided by the same application and they're located in the document root
of the application. URLs in local resources are generated by the 'url' service, usually
:doc:`Phalcon\\Mvc\\Url <../api/Phalcon_Mvc_Url>`.

远程资源即是一种存在于CDN或其它远程服务器上的资源，比如常用的jquery, bootstrap等资源。

Remote resources are those such as common library like jquery, bootstrap, etc. that are provided by a CDN.

.. code-block:: php

    <?php

    public function indexAction()
    {

        //Add some local CSS resources
        $this->assets
            ->addCss('//netdna.bootstrapcdn.com/twitter-bootstrap/2.3.1/css/bootstrap-combined.min.css', false)
            ->addCss('css/style.css', true);
    }

集合Collections
----------------------
集合即是把一同类的资源放在一些，资源管理器隐含的创建了两个集合：css和js. 当然我们可以创建其它的集合以归类其它的资源， 这样我们可以很容易的 在视图里显示：

Collections groups resources of the same type, the assets manager implicitly creates two collections: css and js.
You can create additional collections to group specific resources for ease of placing those resources in the views:

.. code-block:: php

    <?php

    //Javascripts in the header
    $this->assets
        ->collection('header')
        ->addJs('js/jquery.js')
        ->addJs('js/bootstrap.min.js');

    //Javascripts in the footer
    $this->assets
        ->collection('footer')
        ->addJs('js/jquery.js')
        ->addJs('js/bootstrap.min.js');

然后在视图中如下使用：		
		
Then in the views:

.. code-block:: html+php

    <html>
        <head>
            <title>Some amazing website</title>
            <?php $this->assets->outputJs('header') ?>
        </head>
        <body>

            <!-- ... -->

            <?php $this->assets->outputJs('footer') ?>
        </body>
    <html>

Volt语法：	
	
Volt syntax:

.. code-block:: html+jinja

    <html>
        <head>
            <title>Some amazing website</title>
              {{ assets.outputCss('header') }}
        </head>
        <body>

            <!-- ... -->

              {{ assets.outputJs('footer') }}
        </body>
    <html>

前缀Prefixes
----------------
集合可以添加前缀，这可以实现非常简单的更换服务器：

Collections can be URL-prefixed, this allows to easily change from a server to other at any moment:

.. code-block:: php

    <?php

    $scripts = $this->assets->collection('footer');

    if ($config->environment == 'development') {
        $scripts->setPrefix('/');
    } else {
        $scripts->setPrefix('http:://cdn.example.com/');
    }

    $scripts->addJs('js/jquery.js')
            ->addJs('js/bootstrap.min.js');

我们也可以使用链式语法，如下：			
			
A chainable syntax is available too:

.. code-block:: php

    <?php

    $scripts = $assets
        ->collection('header')
        ->setPrefix('http://cdn.example.com/')
        ->setLocal(false)
        ->addJs('js/jquery.js')
        ->addJs('js/bootstrap.min.js');

压缩与过滤Minification/Filtering
-----------------------------------
Phalcon\\Assets提供了内置的js及css压缩工具。 开发者可以设定资源管理器以确定对哪些资源进行压缩啊些资源不进行压缩。除了上面这些之外 我们还可以使用Douglas Crockford书写的Jsmin压缩工具，及Ryan Day提供的CSSMin来对js及css文件进行压缩. 

Phalcon\\Assets provides built-in minification of Javascript and CSS resources. The developer can create a collection of
resources instructing the Assets Manager which ones must be filtered and which ones must be left as they are.
In addition to the above, Jsmin by Douglas Crockford is part of the core extension offering minification of javascript files
for maximum performance. In the CSS land, CSSMin by Ryan Day is also available to minify CSS files:

下面的例子中展示了如何使用集合对资源文件进行压缩：

The following example shows how to minify a collection of resources:

.. code-block:: php

    <?php

    $manager

        //These Javascripts are located in the page's bottom
        ->collection('jsFooter')

        //The name of the final output
        ->setTargetPath('final.js')

        //The script tag is generated with this URI
        ->setTargetUri('production/final.js')

        //This is a remote resource that does not need filtering
        ->addJs('code.jquery.com/jquery-1.10.0.min.js', false, false)

        //These are local resources that must be filtered
        ->addJs('common-functions.js')
        ->addJs('page-functions.js')

        //Join all the resources in a single file
        ->join(true)

        //Use the built-in Jsmin filter
        ->addFilter(new Phalcon\Assets\Filters\Jsmin())

        //Use a custom filter
        ->addFilter(new MyApp\Assets\Filters\LicenseStamper());

开始部分我们通过资源管理器取得了一个命名的集合，集合中可以包含javascript或css资源但不能同时包含两个。一些资源可能位于远程的服务器上 这上结资源我们可以通过http取得。为了提高性能建议把远程的资源取到本地来，以减少加载远程资源的开销。		
		
It starts getting a collection of resources from the assets manager, a collection can contain javascript or css
resources but not both. Some resources may be remote, that is, they're obtained by HTTP from a remote source
for further filtering. It is recommended to convert the external resources to local eliminating the overhead
of obtaining them.

.. code-block:: php

    <?php

    //These Javascripts are located in the page's bottom
    $js = $manager->collection('jsFooter');

如上面，addJs方法用来添加资源到集合中，第二个参数指示了资源是否为外部的，第三个参数指示是否需要压缩资源：	
	
As seen above, method addJs is used to add resources to the collection, the second parameter indicates
whether the resource is external or not and the third parameter indicates whether the resource should
be filtered or left as is:

.. code-block:: php

    <?php

    // This a remote resource that does not need filtering
    $js->addJs('code.jquery.com/jquery-1.10.0.min.js', false, false);

    // These are local resources that must be filtered
    $js->addJs('common-functions.js');
    $js->addJs('page-functions.js');

过滤器被注册到集合内，我们可以注册我个过滤器，资源内容被过滤的顺序和过滤器注册的顺序是一样的。	
	
Filters are registered in the collection, multiple filters are allowed, content in resources are filtered
in the same order as filters were registered:

.. code-block:: php

    <?php

    //Use the built-in Jsmin filter
    $js->addFilter(new Phalcon\Assets\Filters\Jsmin());

    //Use a custom filter
    $js->addFilter(new MyApp\Assets\Filters\LicenseStamper());

注意：不管是内置的还是自定义的过滤器对集合来说他们都是透明的。最后一步用来确定所有写到同一个文件中还是分开保存。如果要让集合中所有的文件合成 一个文件只需要使用join函数：	
	
Note that both built-in and custom filters can be transparently applied to collections.
Last step is decide if all the resources in the collection must be joined in a single file or serve each of them
individually. To tell the collection that all resources must be joined you can use the method 'join':

.. code-block:: php

    <?php

    // This a remote resource that does not need filtering
    $js->join(true);

    //The name of the final file path
    $js->setTargetPath('public/production/final.js');

    //The script html tag is generated with this URI
    $js->setTargetUri('production/final.js');

如果资源写入同一文件，则我们需要定义使用哪一个文件来保存要写入的资源数据，及使用一个ur来展示资源。这两个设置可以使用setTargetPath() 和setTargetUri()两个函数来配置。	
	
If resources are going to be joined, we need also to define which file will be used to store the resources
and which uri will be used to show it. These settings are set up with setTargetPath() and setTargetUri().

内置过滤器Built-In Filters
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Phalcon内置了两个过滤器以分别实现对js及css的压缩，由于二者是使用c实现的故极大的减少了性能上的开销：

Phalcon provides 2 built-in filters to minify both javascript and css respectively, their C-backend provide
the minimum overhead to perform this task:

+-----------------------------------+-----------------------------------------------------------------------------------------------------------+
| Filter                            | Description                                                                                               |
+===================================+===========================================================================================================+
| Phalcon\\Assets\\Filters\\Jsmin   | Minifies Javascript removing unnecessary characters that are ignored by Javascript interpreters/compilers |
+-----------------------------------+-----------------------------------------------------------------------------------------------------------+
| Phalcon\\Assets\\Filters\\Cssmin  | Minifies CSS removing unnecessary characters that are already ignored by browsers                         |
+-----------------------------------+-----------------------------------------------------------------------------------------------------------+

自定义过滤器Custom Filters
^^^^^^^^^^^^^^^^^^^^^^^^^^^
除了使用Phalcon内置的过滤器外，开发者还可以创建自己的过滤器。这样我们就可以使用YUI_, Sass_, Closure_,等。

In addition to built-in filters, a developer can create his own filters. These can take advantage of existing
and more advanced tools like YUI_, Sass_, Closure_, etc.:

.. code-block:: php

    <?php

    use Phalcon\Assets\FilterInterface;

    /**
     * Filters CSS content using YUI
     *
     * @param string $contents
     * @return string
     */
    class CssYUICompressor implements FilterInterface
    {

        protected $_options;

        /**
         * CssYUICompressor constructor
         *
         * @param array $options
         */
        public function __construct($options)
        {
            $this->_options = $options;
        }

        /**
         * Do the filtering
         *
         * @param string $contents
         * @return string
         */
        public function filter($contents)
        {

            //Write the string contents into a temporal file
            file_put_contents('temp/my-temp-1.css', $contents);

            system(
                $this->_options['java-bin'] .
                ' -jar ' .
                $this->_options['yui'] .
                ' --type css '.
                'temp/my-temp-file-1.css ' .
                $this->_options['extra-options'] .
                ' -o temp/my-temp-file-2.css'
            );

            //Return the contents of file
            return file_get_contents("temp/my-temp-file-2.css");
        }
    }

用法:	
	
Usage:

.. code-block:: php

    <?php

    //Get some CSS collection
    $css = $this->assets->get('head');

    //Add/Enable the YUI compressor filter in the collection
    $css->addFilter(new CssYUICompressor(array(
         'java-bin'      => '/usr/local/bin/java',
         'yui'           => '/some/path/yuicompressor-x.y.z.jar',
         'extra-options' => '--charset utf8'
    )));

自定义输出Custom Output
---------------------------
OutputJs及outputCss方法可以依据不同的资源类来创建需要的html代码。我们可以重写这个方法或是手动的输出这些资源方法如下：

Methods outputJs and outputCss are available to generate the necessary HTML code according to each type of resources.
You can override this method or print the resources manually in the following way:

.. code-block:: php

    <?php

    use Phalcon\Tag;

    foreach ($this->assets->collection('js') as $resource) {
        echo Tag::javascriptInclude($resource->getPath());
    }

.. _YUI : http://yui.github.io/yuicompressor/
.. _Closure : https://developers.google.com/closure/compiler/?hl=fr
.. _Sass : http://sass-lang.com/
